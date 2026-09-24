# Проект 6. Облачное хранилище файлов

**ТЗ:** [zhukovsd — Облачное хранилище файлов](https://zhukovsd.github.io/java-backend-learning-course/projects/cloud-file-storage/)
Многопользовательский «Google Drive»: регистрация/вход/выход, загрузка файлов и папок, создание папок, удаление,
переименование/перемещение, скачивание (папка — zip), поиск. REST API под `/api` (RPC для auth, REST для ресурсов),
пользователи в Postgres, **файлы в MinIO (S3)**, сессии в **Redis**, готовый [React-фронтенд](https://github.com/zhukovsd/cloud-storage-frontend),
Swagger, Docker Compose, Testcontainers, деплой JAR.

Первый проект на **Spring Boot**. Главная ловушка — позволить Boot думать за тебя. Ты должен понимать каждый автоконфиг,
который используешь.

## Цели обучения

- Spring Boot: автоконфигурация, стартеры, `application.yml`, профили, `@ConfigurationProperties`.
- Spring Security: `SecurityFilterChain`, аутентификация по сессии для REST, `UserDetailsService`, `PasswordEncoder`,
  обработка 401 без редиректов на форму.
- Spring Data JPA: репозитории, derived queries, миграции (в этот раз **Liquibase**, если в 5 был Flyway).
- S3-модель хранения (бакеты, ключи, «папок нет») и MinIO Java SDK.
- Стриминг больших файлов (без загрузки в память), zip на лету.
- Docker Compose для инфраструктуры, Testcontainers для тестов.
- Проектирование REST API и ошибок, OpenAPI/Swagger.

## Что изучить до старта

| Тема | Обрати внимание |
|------|-----------------|
| Spring Boot: как работает автоконфигурация (`@Conditional*`, `AutoConfiguration.imports`), стартеры, `--debug` отчёт | «откуда взялся этот бин?» — отвечай через condition report |
| `@ConfigurationProperties` + record, валидация конфигов | вместо россыпи `@Value` |
| Spring Security 6: `SecurityFilterChain`, `AuthenticationManager`, `SecurityContextHolder`, `SecurityContextRepository`, `AuthenticationEntryPoint` | для REST: логин через свой JSON-эндпоинт, сохранить контекст в сессию **явно** (Security 6 перестал делать это автоматически) |
| Spring Session Data Redis, TTL | сериализация сессии |
| Spring Data JPA: `JpaRepository`, derived queries, `@Query`, `@Transactional(readOnly = true)` | что генерирует `save()` для новой сущности и почему `existsBy...` лучше `findBy... != null` |
| Liquibase changelogs | в проекте 5 был Flyway — сравни |
| S3: bucket, object key, prefix, delimiter, «папка» = префикс, пустая папка = объект-маркер, `copyObject` + `removeObjects` вместо rename | **переименование папки — это копирование всех объектов** |
| MinIO Java SDK: `putObject` со стримом и размером, `getObject`, `listObjects(recursive)`, `statObject`, `removeObjects` (ленивый результат — ошибки видны только при итерации!) | — |
| Multipart в Spring: `MultipartFile`, лимиты `spring.servlet.multipart.*`, имена файлов с путями (загрузка папки) | не вычитывать файл в `byte[]` |
| `StreamingResponseBody`, `ZipOutputStream` | скачивание папки без временного файла и без загрузки в память |
| Docker/Compose: образы, volumes, сети, `depends_on` + healthcheck | Postgres, Redis, MinIO |
| Testcontainers: `@Testcontainers`, `@Container`, `@ServiceConnection` (Boot 3.1+), `GenericContainer` для MinIO | переиспользование контейнеров между тестами |
| springdoc-openapi | Swagger UI, аннотации `@Operation`, `@ApiResponse` |
| Path traversal, валидация путей и имён | `../`, `//`, пустые сегменты, запрещённые символы |

📚 **Читать:**
- SIA: гл. 1 (Boot, стартеры), гл. 3 «Working with data» (Spring Data JPA), гл. 5 «Securing Spring», гл. 6 «Working with configuration properties», гл. 7 «Creating REST services».
- SSIA: главы про архитектуру Spring Security (фильтры, `AuthenticationManager`, `UserDetailsService`, `PasswordEncoder`), CSRF и CORS — **основной учебник для вехи 6.2**.
- [Spring Boot Reference: Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html), [Creating Your Own Auto-configuration](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html) (чтобы понять условия), [Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html).
- [Spring Security Reference: Architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html) и [Persisting Authentication](https://docs.spring.io/spring-security/reference/servlet/authentication/persistence.html) — **обязательно**.
- [Spring Session — Redis](https://docs.spring.io/spring-session/reference/getting-started/using-redis.html).
- [Spring Data JPA Reference: Query Methods](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html).
- [AWS S3 User Guide: Organizing objects using prefixes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-prefixes.html) — почему «папок нет». [MinIO Java SDK API Reference](https://min.io/docs/minio/linux/developers/java/API.html).
- [Testcontainers: Getting started](https://java.testcontainers.org/quickstart/junit_5_quickstart/), [Spring Boot: Testcontainers support](https://docs.spring.io/spring-boot/reference/testing/testcontainers.html).
- Найджел Поултон, *Docker Deep Dive*: главы про образы, контейнеры, volumes, Compose. Или [Docker docs: Get started](https://docs.docker.com/get-started/) + [Compose file reference](https://docs.docker.com/reference/compose-file/).
- [OWASP: Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal).
- PoEAA, гл. 18: Gateway, Separated Interface — теория фасада над хранилищем.

> 🎯 **Спросят на собесе:** Как работает автоконфигурация Spring Boot?
> **Ответ:** Boot импортирует классы из `META-INF/spring/...AutoConfiguration.imports`. Каждый из них — `@Configuration`
> с условиями (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`): бин создаётся, только если
> есть нужный класс в classpath и пользователь не определил свой. Отчёт — `--debug`.

> 🎯 **Спросят на собесе:** Как устроена цепочка фильтров Spring Security?
> **Ответ:** `DelegatingFilterProxy` в контейнере → `FilterChainProxy` → выбор `SecurityFilterChain` по запросу →
> упорядоченные фильтры (`SecurityContextHolderFilter`, аутентификация, `ExceptionTranslationFilter`, `AuthorizationFilter`...).
> `SecurityContext` хранится в `ThreadLocal` на время запроса и сохраняется в сессию через `SecurityContextRepository`.

## Паттерны и принципы в фокусе

- **Facade / Port & Adapter** над MinIO. Весь S3-специфичный код (префиксы `user-{id}-files/`, маркеры пустых папок,
  копирование при rename) — за интерфейсом хранилища. Контроллер и доменный сервис работают с «ресурсами» и «путями»
  пользователя. Утечка `user-1-files/` в ответ API — прямо указанная в ТЗ ошибка.
- **Value Object для пути.** Путь к ресурсу — это не `String`. Нормализация, валидация, «родитель», «имя», «папка ли» —
  в одном типе. Сколько багов этим предотвращается?
- **Стриминг** — данные идут потоком клиент → приложение → S3 и обратно, не оседая в памяти.
- **Единый формат ошибок** через `@RestControllerAdvice` + `ProblemDetail` или свой `{"message"}` по ТЗ.
- **Authorization by design.** Пользователь физически не может обратиться к чужому префиксу: корень определяется из
  аутентификации, а не из параметра запроса.

## Вехи

### Веха 6.1 — Инфраструктура и каркас
Критерии приёмки:
- `docker-compose.yml`: Postgres (сначала), volumes, healthchecks. Переменные — через `.env` (в `.gitignore`) + `.env.example` в git.
- Spring Boot 3.x, Java 21, `@ConfigurationProperties` для своих настроек, профили.
- Liquibase-миграция таблицы пользователей (уникальный индекс на username, длины, not null).

### Веха 6.2 — Регистрация, вход, выход, текущий пользователь
Вопросы до кода:
- Как Security понимает, что пользователь залогинен при следующем запросе? Где лежит контекст?
- Что вернёт защищённый эндпоинт неавторизованному: редирект на `/login` (дефолт) или 401? Как настроить?
- Регистрация сразу логинит (по ТЗ). Как это сделать без повторного запроса логина?
- CSRF для SPA на сессиях: выключить или настроить? Обоснуй.

Критерии приёмки:
- Эндпоинты `/api/auth/sign-up|sign-in|sign-out`, `/api/user/me` строго по ТЗ (коды, тела, `201` при регистрации).
- Валидация (Bean Validation) с ответом 400 `{"message"}`; 409 при занятом имени (через ограничение БД).
- 401 JSON для неавторизованных, без HTML-страниц и редиректов.

### Веха 6.3 — Интеграционные тесты пользователей (Testcontainers)
Критерии приёмки:
- Postgres в Testcontainers, контейнер переиспользуется между тестовыми классами.
- Кейсы из ТЗ: создание пользователя пишет запись, дубликат → ожидаемое исключение. Плюс HTTP-уровень
  (`MockMvc` или `TestRestTemplate`/`RestClient`): регистрация → кука → `/user/me`.

### Веха 6.4 — MinIO и сервис хранилища
Вопросы до кода:
- Как выглядит интерфейс хранилища **с точки зрения приложения**? Какие операции? Какие типы параметров и результатов?
- Как отличить файл от папки? Как представить пустую папку? Где это знание живёт?
- Путь от пользователя: `folder1/../../user-2-files/secret.txt`. Что произойдёт?

Критерии приёмки:
- MinIO в compose, бакет создаётся при старте, если его нет.
- Сервис-фасад над MinIO: префикс пользователя добавляется/убирается **только** внутри него.
- Тип «путь ресурса» с валидацией и нормализацией, покрыт юнит-тестами (обход каталогов, двойные слэши, пустые имена,
  запрещённые символы, папка vs файл).

### Веха 6.5 — Загрузка файлов и папок
Критерии приёмки:
- `POST /api/resource?path=` с multipart, вложенные папки из имён файлов, `201` + список ресурсов.
- Файлы не буферизуются целиком в память (объясни, куда Spring пишет multipart и как это настроить).
- Разумные лимиты размера, 409 при существующем файле.

### Веха 6.6 — Операции с ресурсами: инфо, список, создание папки, удаление, перемещение, скачивание
Вопросы до кода:
- Перемещение папки с 1000 файлов: сколько запросов к S3? Что если на 500-м упадёт? (атомарности нет — как минимизировать ущерб?)
- Переименование `a/2` в `a/1`, когда `a/1` уже есть: что должно произойти по ТЗ?
- Скачивание папки: как отдать zip, не создавая временный файл?

Критерии приёмки:
- Все эндпоинты `/resource`, `/resource/move`, `/resource/download`, `/directory` по ТЗ, со всеми кодами ошибок.
- Ни одна операция не затирает существующий ресурс.
- Zip и файлы скачиваются неповреждёнными (проверь: скачай, распакуй, сравни чексуммы).
- 404 для несуществующей папки (нельзя «войти» в несуществующую).

### Веха 6.7 — Поиск и фронтенд
Критерии приёмки:
- `GET /api/resource/search?query=` находит **только** свои ресурсы.
- React-фронтенд из ТЗ раздаётся Spring Boot с корня; SPA-роутинг работает при обновлении страницы на вложенном URL
  (подумай, почему по умолчанию будет 404).
- Swagger UI документирует все эндпоинты, включая коды ошибок.

### Веха 6.8 — Redis-сессии, тесты хранилища, деплой
Критерии приёмки:
- Сессии в Redis (Spring Session), переживают рестарт приложения — проверь.
- ★ из ТЗ: интеграционные тесты сервиса файлов на MinIO в Testcontainers: загрузка, rename, delete, **доступ к чужим файлам
  невозможен**, поиск не находит чужие.
- JAR на VPS, инфраструктура через compose, секреты через env.
- Сверка с [чеклистом из ТЗ](https://zhukovsd.github.io/java-backend-learning-course/projects/cloud-file-storage/#чеклист-для-самопроверки).

## Челленджи

- ★ **Квоты** на объём хранилища пользователя. Где считать занятый объём и как не пересчитывать его каждый раз?
- ★ **ArchUnit**: пакет `web` не зависит от `io.minio`, домен не зависит от Spring Web.
- ★★ **Presigned URLs** для скачивания больших файлов напрямую из MinIO. Плюсы, минусы, безопасность (TTL).
- ★★ **Докеризация приложения** (multi-stage Dockerfile, non-root user, layered jar) и запуск всего стека одной командой.
- ★★★ **Multipart upload** в S3 для файлов > 100 МБ с возобновлением. Как поведёт себя твой текущий код с файлом 5 ГБ?
- ★★★ **Наблюдаемость**: Spring Boot Actuator, Micrometer, метрики загрузок/скачиваний, Prometheus + Grafana в compose.

## За что будет 🔴

- `user-{id}-files` в ответах API, в контроллерах, во фронтенде. MinIO-классы за пределами адаптера.
- Путь как голая `String` с ручными `split("/")` по всему коду; возможность path traversal.
- Доступ к чужим файлам через поиск/скачивание/move с ручным URL.
- Затирание ресурсов при move/rename; битые zip; `byte[]` для целых файлов.
- Форматы ответов/коды не по ТЗ, HTML-редиректы вместо 401.
- Отключённый CSRF без понимания последствий и обоснования.
- Игнорирование результата `removeObjects` (ошибки молча теряются).
- Секреты в `application.yml` в git.
- Логика в контроллерах; `@Transactional` на контроллере.

## Контрольные вопросы (защита)

1. Как работает автоконфигурация Boot? Как переопределить автоконфигурированный бин?
2. Путь запроса через Spring Security до контроллера. Где хранится аутентификация между запросами?
3. Сессии в Redis: что именно сериализуется, как работает TTL, что будет при двух инстансах приложения?
4. S3 vs файловая система: почему «переименование папки» дорогое? Что такое prefix и delimiter?
5. Как ты гарантируешь изоляцию пользователей? Приведи атаку и покажи, почему она не пройдёт.
6. Как загружаемый файл проходит от браузера до MinIO? Где он лежит в процессе? Сколько памяти занимает?
7. Testcontainers vs H2: плюсы и минусы. Как ускорить тесты с контейнерами?
8. Spring Data JPA: как из имени метода получается запрос? Когда `@Query`? Что делает `readOnly = true`?
9. Docker: образ vs контейнер, volume vs bind mount, как контейнеры находят друг друга в compose-сети?
10. REST vs RPC: почему в ТЗ auth — RPC, а ресурсы — REST?

> 🎯 **Спросят на собесе:** Где Spring Security хранит данные аутентифицированного пользователя во время запроса и почему это работает в многопоточном сервере?
> **Ответ:** В `SecurityContextHolder`, по умолчанию стратегия `ThreadLocal`: каждый поток обработки запроса видит свой контекст.
> Между запросами контекст хранится в `HttpSession` (или Redis через Spring Session), при новом запросе восстанавливается фильтром.
> **Типичная ошибка:** забыть, что при `@Async`/своих пулах потоков `ThreadLocal`-контекст не переносится автоматически.
