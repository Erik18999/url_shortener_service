# URL Shortener Service

## Описание

`url_shortener_service` — это микросервис, который является частью приложения "CorporationX". Данный микросервис позволяет пользователям отправлять свои длинные ссылки (URL) в наше приложение и сокращать их до очень маленьких. Это полезно для тех, кто хочет поделиться ссылкой в социальных сетях, где длинные ссылки могут выглядеть непривлекательно и быть неудобными для использования.

[Сервис](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/service/UrlServiceImpl.java) реализует два [REST-эндпоинта](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/controller/UrlController.java) — создание короткой ссылки и редирект по ней на оригинальный длинный URL — и при этом обладает функциональностью многопоточного [кэширования](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/cache/HashCacheImpl.java), работой с [PostgreSQL](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/repository/UrlJdbcRepository.java) и [Redis](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/repository/UrlCacheRepositoryImpl.java), а также [планировщиком](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/scheduler/CleanerScheduler.java) переиспользования ресурсов. Взаимодействие с основным приложением строится через REST API, то есть это полноценное микросервисное взаимодействие с решением характерных для такой архитектуры проблем: race condition при параллельных запросах, нагрузка на БД, повторное использование ресурсов.

**Ключевая инженерная задача** этого сервиса — обеспечить [генерацию](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/generator/HashGeneratorImpl.java) коротких ссылок без задержек и без дублей даже при высокой параллельной нагрузке, не перегружая при этом базу данных на каждый запрос.

## Архитектурная схема

![Архитектура URL Shortener](docs/UrlShortener%28light%29.png)

## Как это работает

### 1. Создание короткой ссылки

Пользователь отправляет запрос: `POST /api/v1/url-shortener/url` с телом `{ "url": "https://very-long-link.com/..." }`.

1. [`UrlController`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/controller/UrlController.java) принимает запрос, [`UrlDto`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/dto/UrlDto.java) валидируется аннотациями `@NotBlank` и `@Pattern` (регулярное выражение проверяет, что это действительно корректный `http(s)://` URL, а не пустая строка или случайный текст).
2. Запрос передаётся в [`UrlService.createShortUrl(UrlDto urlDto)`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/service/UrlServiceImpl.java).
3. `UrlService` обращается в [`HashCache.getHash()`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/cache/HashCacheImpl.java) — получает **уже готовый** хэш из внутреннего кэша (не генерирует новый на лету, это была бы медленная операция под нагрузкой).
4. Полученная пара `hash → url` сохраняется сперва в PostgreSQL ([таблица](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/entity/Url.java) [`url`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/resources/db/changelog/changeset/V001_url_shortener-service_url_hash.sql), через [`UrlRepository`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/repository/UrlJdbcRepository.java)), а затем сохраняется в Redis (через [`UrlCacheRepository`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/repository/UrlCacheRepositoryImpl.java), так как свежесозданная ссылка с высокой вероятностью будет запрошена сразу же).
5. Когда все данные успешно сохранены, `UrlService` просто возвращает хэш в `UrlController`, который формирует из него [ответ](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/dto/ShortUrlResponse.java) (полный короткий URL: статический адрес нашего сервиса + хэш. Например, `http://localhost:8080/api/v1/url-shortener/xz`) и возвращает его пользователю с кодом `201 Created`.

### 2. Переход по короткой ссылке на длинный (оригинальный) URL

Пользователь отправляет запрос: `GET /api/v1/url-shortener/{hash}`.

1. `UrlController` вынимает `hash` из пути и передаёт его в [`UrlService.getUrl(String hash)`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/service/UrlServiceImpl.java).
2. `UrlService` сначала проверяет Redis ([`UrlCacheRepository.getUrlByHash(String hash)`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/repository/UrlCacheRepositoryImpl.java)) — если хэш там есть, оригинальный URL возвращается моментально.
3. Если хэша в Redis не нашлось — идёт запрос в PostgreSQL ([`UrlRepository.findUrlByHash(String hash)`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/repository/UrlJdbcRepository.java)). Если URL найден в БД, он дополнительно кладётся в Redis (прогрев кэша), чтобы повторные переходы по этой же ссылке обслуживались мгновенно.
4. Если хэша нет ни в Redis, ни в БД — выбрасывается [`DataNotFoundException`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/exception/DataNotFoundException.java), которую перехватывает [`UrlExceptionHandler`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/handler/UrlExceptionHandler.java) и превращает в ответ `404 Not Found`.
5. Если оригинальный URL найден, `UrlController` оборачивает его в `RedirectView`, и пользователь получает `302 Found` с редиректом на оригинальный адрес.

### 3. Как устроен [`HashCache`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/cache/HashCacheImpl.java) — сердце фичи

Самая интересная инженерная часть сервиса: генерация хэшей на лету на каждый запрос была бы слишком медленной и создавала бы огромную нагрузку на БД при большом количестве пользователей. Поэтому хэши **готовятся заранее** и хранятся во внутренней памяти приложения (Оперативная память), а конкретно в специально созданной структуре данных - `ConcurrentLinkedQueue<String>`.

- **Структура данных**: `ConcurrentLinkedQueue<String>` — потокобезопасная lock-free очередь. Каждый запрос забирает хэш через `poll()`, что тоже потокобезопасно.
- **Проверка порога**: при каждом обращении `HashCache` проверяет (с помощью метода `shouldRefillQueue()`), не опустилась ли заполненность Очереди (`ConcurrentLinkedQueue`) ниже порога (по умолчанию 20% от максимального размера, значение настраивается через [конфиг](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/resources/application.yaml)). Если хэшей ещё достаточно — просто отдаётся следующий хэш из Очереди без каких-либо задержек.
- **Асинхронное пополнение**: если заполненность ниже порога, запускается (с помощью метода `refillCache()`) асинхронное пополнение кэша из БД через отдельный [Executor](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/properties/HashCacheExecutorProperties.java), не блокируя текущий и последующие запросы — у сервиса ещё остаётся 20% хэшей "про запас", пока пополнение не завершится.
- **Защита от повторного пополнения**: чтобы несколько потоков не запустили пополнение кэша одновременно (что было бы избыточной нагрузкой на БД), используется `AtomicBoolean` - `isRefilling`, с атомарной операцией `compareAndSet(false, true)` — гарантированно только один поток запускает процесс пополнения, остальные его просто пропускают, пока флаг не сброшен.
- **Прогрев при старте**: кэш заполняется (с помощью метода `init()`) сразу при старте приложения через `@PostConstruct`, а не ждёт первого запроса пользователя.

### 4. Как генерируются сами хэши — [`HashGenerator`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/generator/HashGeneratorImpl.java) (алгоритм [Base62](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/encoder/Base62Encoder.java))

Когда [`HashCache`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/cache/HashCacheImpl.java) берёт батч (порция) хэшей из БД (с помощью [`HashRepository.getHashBatch()`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/repository/HashJdbcRepository.java)) и пополняет свой внутренний кэш (`ConcurrentLinkedQueue<String>`), он идёт в БД не за случайными значениями, а за гарантированно уникальными целыми числами:

1. `HashGenerator` запрашивает у `HashRepository` пачку уникальных целых чисел прямо из PostgreSQL sequence (последовательность) ([`unique_number_seq`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/resources/db/changelog/changeset/V001_url_shortener-service_url_hash.sql)) — специальной сущности БД, которая атомарно увеличивает счётчик и никогда не выдаёт повторов, даже при параллельных запросах. Это делается одним SQL-запросом через `generate_series` + `nextval`, без цикла на уровне приложения.
2. Полученные числа передаются в [`Base62Encoder`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/encoder/Base62Encoder.java) (с помощью `BaseEncoder.encode(List<Long>)`), который кодирует каждое число в хэш длиной до 6 символов с помощью алгоритма Base62. Данный алгоритм реализован в методе `private String encodeToString(Long number)`.

   Алгоритм Base62 — это простой алгоритм взятия остатка от деления числа на 62, где этот остаток является индексом в массиве из 62 символов, которые один за другим и формируют хэш. Т.о. число делят взятие остатка на 62 до тех пор, пока она не обратится в 0, и каждый раз добавляют новый символ в хэш по индексу остатка.
3. Поскольку исходные целые числа из БД уникальны, а Base62 — это взаимно однозначное соответствие, то результирующие хэши тоже гарантированно уникальны — никаких проверок на дубли и повторных попыток генерации не требуется.
4. Сгенерированные хэши сохраняются в БД батчем, а конкретно в таблицу [`hash`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/resources/db/changelog/changeset/V001_url_shortener-service_url_hash.sql) — отдельный "резервный пул" свободных хэшей, из которого потом и берёт `HashCache` (с помощью метода [`HashRepository.getHashBatch()`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/repository/HashJdbcRepository.java)). 
Важно, что `HashGenerator` генерирует хэшей **с запасом** (значительно больше, чем `HashCache` берёт за раз), чтобы пул в БД никогда не иссякал. Это указано в конфигурационном классе [`HashBatchProperties`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/properties/HashBatchProperties.java) и в файле [`application.yaml`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/resources/application.yaml).
5. Вся операция генерации хэшей — асинхронная ([`@Async`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/generator/HashGeneratorImpl.java) с именованным [thread pool](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/properties/ThreadPoolProperties.java) - [`hashGeneratorThreadPool`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/config/context/async/AsyncConfig.java)), поэтому пользователи не замечают этого процесса даже под нагрузкой.

### 5. Переиспользование хэшей — очистка старых ссылок

Раз в сутки (по [cron-расписанию](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/resources/application.yaml) из конфига) запускается [`CleanerScheduler`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/scheduler/CleanerScheduler.java) - это класс, который содержит джобу запускающаяся по расписанию.

Принцип работы данной джобы:

1. Находит в БД (таблица [`url`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/resources/db/changelog/changeset/V001_url_shortener-service_url_hash.sql)) все ассоциации (URL-Hash) старше 1 года.
2. Удаляет (с помощью `UrlRepository.deleteOldUrlsAndReturnHashes(LocalDateTime)`) их одним SQL-запросом с `DELETE ... RETURNING hash`, который атомарно и удаляет записи, и возвращает освободившиеся хэши.
3. Эти освободившиеся хэши не выбрасываются, а переносятся обратно в БД (таблица `hash`) — то есть возвращаются в пул свободных хэшей для повторного использования. Т.о. мы экономим место, а наш сервис "UrlShortener" работает ещё оптимальнее.
4. Вся операция выполняется в рамках одной транзакции ([`@Transactional`](https://github.com/Erik18999/url_shortener_service/blob/werewolf-stream8-erkin/src/main/java/faang/school/urlshortenerservice/service/UrlServiceImpl.java)) — либо всё удаление и перенос хэшей проходит успешно, либо откатывается целиком.

Так как исходная sequence в БД - это монотонно возрастающая последовательность и никогда не выдаёт повторов (дубликатов), конфликты между "новыми" и "переиспользованными" хэшами не возможны.

## Структура классов

| Класс | Роль |
|---|---|
| [`UrlController`](src/main/java/faang/school/urlshortenerservice/controller/UrlController.java) | REST-контроллер: создание короткой ссылки и редирект по хэшу |
| [`UrlService`](src/main/java/faang/school/urlshortenerservice/service/UrlService.java) / [`UrlServiceImpl`](src/main/java/faang/school/urlshortenerservice/service/UrlServiceImpl.java) | Основная бизнес-логика: генерация, получение и очистка ссылок |
| [`HashCache`](src/main/java/faang/school/urlshortenerservice/cache/HashCache.java) / [`HashCacheImpl`](src/main/java/faang/school/urlshortenerservice/cache/HashCacheImpl.java) | Потокобезопасный in-memory кэш свободных хэшей с асинхронным пополнением |
| [`HashGenerator`](src/main/java/faang/school/urlshortenerservice/generator/HashGenerator.java) / [`HashGeneratorImpl`](src/main/java/faang/school/urlshortenerservice/generator/HashGeneratorImpl.java) | Асинхронная генерация новых хэшей и сохранение их в БД |
| [`Base62Encoder`](src/main/java/faang/school/urlshortenerservice/encoder/Base62Encoder.java) | Кодирование уникальных чисел в хэши по алгоритму Base62 |
| [`CleanerScheduler`](src/main/java/faang/school/urlshortenerservice/scheduler/CleanerScheduler.java) | Ежедневная джоба очистки устаревших ссылок и переиспользования хэшей |
| [`UrlRepository`](src/main/java/faang/school/urlshortenerservice/repository/UrlRepository.java) / [`UrlJdbcRepository`](src/main/java/faang/school/urlshortenerservice/repository/UrlJdbcRepository.java) | Хранение и получение ассоциаций хэш ↔ URL в PostgreSQL |
| [`HashRepository`](src/main/java/faang/school/urlshortenerservice/repository/HashRepository.java) / [`HashJdbcRepository`](src/main/java/faang/school/urlshortenerservice/repository/HashJdbcRepository.java) | Получение уникальных чисел из sequence, батчевое сохранение/извлечение свободных хэшей |
| [`UrlCacheRepository`](src/main/java/faang/school/urlshortenerservice/repository/UrlCacheRepository.java) / [`UrlCacheRepositoryImpl`](src/main/java/faang/school/urlshortenerservice/repository/UrlCacheRepositoryImpl.java) | Кэширование популярных/недавних URL в Redis |
| [`AsyncConfig`](src/main/java/faang/school/urlshortenerservice/config/context/async/AsyncConfig.java) | Именованный thread pool для асинхронной генерации хэшей |
| [`UrlExceptionHandler`](src/main/java/faang/school/urlshortenerservice/handler/UrlExceptionHandler.java) | Глобальный обработчик исключений |
| [`Url`](src/main/java/faang/school/urlshortenerservice/entity/Url.java) | JPA-сущность ассоциации хэш ↔ URL |

## Тестирование

Ключевая бизнес-логика покрыта unit-тестами (JUnit 5 + Mockito), включая проверку правильного порядка вызовов зависимостей и корректность обработки edge-кейсов (пустой кэш, ненайденный хэш, отсутствие старых ссылок для очистки).

- [`UrlServiceImplTest`](src/test/java/faang/school/urlshortenerservice/service/UrlServiceImplTest.java)

## Стек

- Java 17
- Spring Boot 3
- Spring Data JPA / JDBC (`JdbcTemplate`)
- PostgreSQL (sequence, batch-операции, `RETURNING`)
- Redis (Jedis)
- Spring Async (`@Async`, `ThreadPoolTaskExecutor`)
- Spring Scheduling (`@Scheduled`, cron)
- Liquibase
- MapStruct
- SpringDoc OpenAPI (Swagger)
- Testcontainers (PostgreSQL, Redis)
- JUnit 5, Mockito, AssertJ

## Конфигурация

Все параметры сервиса вынесены в `application.yaml` и не захардкожены в коде:

- `hash.batch-size`, `hash.hash-generation-size` — размер батча свободных хэшей и размер батча генерации новых
- `hash.thread-pool.*` — параметры thread pool для генерации хэшей
- `cache.cache-size`, `cache.refill-threshold-percent` — размер `HashCache` и порог пополнения (по умолчанию 500 хэшей / 20%)
- `hash-cache-executor.*` — параметры thread pool для пополнения `HashCache`
- `cleaner.older-than-years`, `cleaner.cron` — возраст устаревших ссылок и расписание очистки

Файл: [`application.yaml`](src/main/resources/application.yaml)

## Запуск

### Предварительные требования
- Docker и Docker Compose
- JDK 17

### Шаги

1. Поднять инфраструктуру (Postgres, Redis, MinIO, Kafka):
```bash
git clone https://github.com/Erik18999/infra.git
cd infra
./run.sh
```
2. Склонировать и запустить сам сервис:
```bash
git clone https://github.com/Erik18999/url_shortener_service.git
cd url_shortener_service
```
Открыть проект в IntelliJ IDEA и запустить [`UrlShortenerApplication`](src/main/java/faang/school/urlshortenerservice/UrlShortenerApplication.java).

## Swagger UI

http://localhost:8080/swagger-ui.html

## Архитектурная схема (тёмная тема)

![Архитектура URL Shortener (dark)](docs/UrlShortener%28dark%29.png)
