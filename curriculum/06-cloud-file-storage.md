# Проект 6. Облачное хранилище файлов

**ТЗ:** [zhukovsd — Облачное хранилище файлов](https://zhukovsd.github.io/java-backend-learning-course/projects/cloud-file-storage/)

Многопользовательский «Google Drive»:
- регистрация, вход, выход (`/api/auth/sign-up|sign-in|sign-out`, `/api/user/me`);
- загрузка файлов и целых папок, создание пустой папки, удаление, переименование и перемещение, скачивание
  (папка отдаётся zip-архивом), поиск;
- REST API под `/api`: RPC-стиль для авторизации, REST для ресурсов, ошибки `{"message": ...}`.

Пользователи в Postgres, **файлы в MinIO (S3)**, сессии в **Redis**. Готовый
[React-фронтенд](https://github.com/zhukovsd/cloud-storage-frontend), Swagger, Docker Compose, Testcontainers, деплой JAR.

Первый проект на **Spring Boot**. Перед ним **обязательно** пройди [мостик М6](06-0-bridge-boot-docker-security.md):
Docker, Boot, Security, Redis, S3 и Testcontainers разобраны там. Главная ловушка проекта — позволить Boot думать
за тебя. Ты должен понимать каждую автоконфигурацию, которой пользуешься.

**Сколько времени:** 4–6 недель.

---

## Цели обучения

- Собрать реальное Boot-приложение: Web, Security, Data JPA, Session, миграции (в этот раз **Liquibase**, если в проекте 5 был Flyway).
- Спроектировать фасад над S3, чтобы детали хранения не протекали в API.
- Передавать большие файлы потоком, не загружая их в память.
- Гарантировать изоляцию пользователей: чужие файлы недоступны никак.
- Поднимать инфраструктуру через Docker Compose и тестировать на Testcontainers.

---

## Что изучить до старта

Boot, Security, Redis, S3, Docker, Testcontainers — в [мостике М6](06-0-bridge-boot-docker-security.md). Здесь — специфика проекта.

### 1. Путь к ресурсу — это тип, а не строка

**Простыми словами.** Весь проект крутится вокруг путей: `folder1/folder2/file.txt`, `folder1/folder2/` (папка оканчивается на `/`).
Если путь везде — голая `String`, то в десятке мест появятся `split("/")`, `endsWith("/")`, `substring(...)`, и в каждом
месте свои баги: двойные слэши, пустые сегменты, `../`, забытый завершающий `/`.

Решение — **value object** (как координата в проекте 2): один класс, который при создании **нормализует и валидирует** путь
и умеет отвечать на вопросы: это папка? какое имя? какой родитель? присоединить дочерний элемент.
Создал объект — значит, путь корректен; невалидный путь просто нельзя создать.

```java
// Нейтральный пример: email как value object
public record Email(String value) {
    private static final Pattern FORMAT = Pattern.compile("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$");
    public Email {
        value = value.strip().toLowerCase(Locale.ROOT);                    // нормализация
        if (!FORMAT.matcher(value).matches()) throw new InvalidEmailException(value);   // валидация
    }
    public String domain() { return value.substring(value.indexOf('@') + 1); }   // поведение
}
// дальше по коду ходит Email, а не String — невалидный email в систему попасть не может
```

**Path traversal.** Пользователь присылает `path=../user-2-files/secret.txt`. Если просто склеить с корнем пользователя —
получит чужой файл. Нормализация пути (запрет `..`, `.`, пустых сегментов) — часть value object.

### 2. Фасад над хранилищем

**Простыми словами.** В MinIO файлы пользователя 1 лежат под ключами `user-1-files/docs/a.txt`, пустая папка —
объект-маркер, переименование папки — копирование всех объектов. Всё это **детали хранения**. Контроллеры, DTO и фронтенд
должны видеть только `docs/a.txt` и «ресурс: файл/папка, имя, размер».

Поэтому весь S3-код прячется за **фасадом** — классом (или парой классов) с интерфейсом в терминах приложения:
«загрузить в папку», «список содержимого папки», «переместить ресурс», «удалить», «найти по имени», «открыть поток на чтение».
Внутри фасада — префикс пользователя, маркеры папок, копирование при rename. Снаружи этого нет. Чеклист ТЗ прямо называет
протекание `user-${id}-files` в API ошибкой.

**Откуда берётся «чей это корень».** Не из параметра запроса, а из **аутентификации** (текущий пользователь из Security).
Тогда обратиться к чужому префиксу невозможно по построению, а не «потому что мы проверили».

### 3. Потоковая передача файлов

**Простыми словами.** Файл 2 ГБ нельзя прочитать в `byte[]`: память кончится при паре одновременных загрузок.
Данные должны идти **потоком**: кусками от клиента через приложение в S3 и обратно, не оседая целиком в памяти.

- **Загрузка:** Spring разбирает multipart и отдаёт `MultipartFile`. Маленькие файлы он держит в памяти, большие пишет
  во временный файл на диске (порог и лимиты — `spring.servlet.multipart.*`). У `MultipartFile` есть `getInputStream()`
  и `getSize()` — ровно то, что нужно SDK MinIO для `putObject`.
- **Скачивание файла:** отдавай поток из S3 прямо в ответ: `InputStreamResource`/`StreamingResponseBody`.
- **Скачивание папки zip-ом:** `StreamingResponseBody` + `ZipOutputStream`, который пишет **прямо в ответ**,
  по одному файлу из S3 за раз. Ни временного файла, ни всего архива в памяти.

```java
// Нейтральный пример: выгрузка отчёта CSV потоком — строки пишутся в ответ по мере чтения из БД
@GetMapping("/reports/orders.csv")
ResponseEntity<StreamingResponseBody> export() {
    StreamingResponseBody body = out -> {
        try (var writer = new BufferedWriter(new OutputStreamWriter(out, StandardCharsets.UTF_8))) {
            orderRepository.streamAll().forEach(o -> write(writer, o));   // никакого List всех заказов
        }
    };
    return ResponseEntity.ok().contentType(MediaType.parseMediaType("text/csv")).body(body);
}
```

> 🎯 **Спросят на собесе:** Как отдать клиенту большой файл, не загружая его в память?
> **Ответ:** Потоково: копировать `InputStream` источника в `OutputStream` ответа буфером фиксированного размера
> (`StreamingResponseBody`, `InputStreamResource`, `transferTo`). Для генерируемых данных (zip, CSV) — писать в выходной поток по мере генерации.
> **Типичная ошибка:** `readAllBytes()`/`byte[]` для всего файла.

### 4. Особенности MinIO SDK, которые больно бьют

- `putObject` требует размер потока (или размер части для multipart-загрузки).
- `removeObjects(...)` возвращает **ленивый** результат: удаление реально происходит, только когда ты итерируешь результат,
  и ошибки видны тоже только там. Не проитерировал — ничего не удалилось, и ты об этом не узнал.
- `listObjects(recursive=false)` возвращает и объекты, и «папки» (общие префиксы) — различай их.
- Исключения SDK — checked и разнообразные: транслируй их в свои внутри фасада.

### 5. Ошибки API и документация

- Единый формат `{"message": ...}` — через `@RestControllerAdvice` (можно на основе `ProblemDetail`).
- Коды ошибок по ТЗ для каждого эндпоинта: 400, 401, 404, 409, 500.
- **Swagger/OpenAPI** — `springdoc-openapi`: подключаешь зависимость, получаешь `/swagger-ui.html`; аннотации `@Operation`,
  `@ApiResponse` описывают эндпоинты и коды ошибок.

### 6. SPA-фронтенд внутри Boot

Статические файлы из `src/main/resources/static` Boot раздаёт сам. Но у SPA свой роутинг: при обновлении страницы
`/files/folder1/` браузер пойдёт на сервер, а такого файла нет → 404. Нужно, чтобы неизвестные не-API пути отдавали `index.html`.
Разберись, как это сделать (контроллер-«catch-all» или настройка обработчика ресурсов).

### Шпаргалка

| Тема | Обрати внимание |
|------|-----------------|
| Путь — value object | нормализация, `..`, папка vs файл |
| Фасад над MinIO | префикс пользователя только внутри |
| Корень пользователя из аутентификации | не из параметра |
| `MultipartFile` → `putObject` потоком | лимиты multipart |
| `StreamingResponseBody` + `ZipOutputStream` | без временных файлов |
| `removeObjects` — ленивый | итерировать результат |
| `@RestControllerAdvice`, springdoc | коды по ТЗ |
| SPA fallback на `index.html` | обновление вложенной страницы |

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
- [springdoc-openapi: Getting started](https://springdoc.org/#getting-started).
- [Baeldung: Spring StreamingResponseBody](https://www.baeldung.com/spring-mvc-sse-streams) (раздел про `StreamingResponseBody`).

---

## Паттерны и принципы в фокусе

- **Facade / Port & Adapter** над MinIO: весь S3-специфичный код за интерфейсом.
- **Value Object для пути.**
- **Стриминг** вместо буферизации.
- **Единый формат ошибок** через `@RestControllerAdvice`.
- **Authorization by design:** корень пользователя — из аутентификации.

---

## Вехи

### Веха 6.1 — Инфраструктура и каркас

**Что нужно сделать**
1. `docker-compose.yml` с Postgres (healthcheck, volume), `.env` + `.env.example`.
2. Spring Boot 3.x, Java 21, `@ConfigurationProperties`, профили.
3. Liquibase-миграция таблицы пользователей.

**С чего начать.** Скопируй compose и каркас из песочницы мостика М6.

**Вопросы для размышления**
1. Какие колонки нужны таблице пользователей, если ориентироваться на Spring Security? *Направление: что возвращает `UserDetails`?*
2. Какие ограничения? *Направление: уникальный username (индекс!), длины, not null.*

**Как проверить себя**
- [ ] `docker compose up -d` + `./mvnw spring-boot:run` — приложение стартует, миграция применена.
- [ ] Секретов в git нет.

### Веха 6.2 — Регистрация, вход, выход, текущий пользователь

**Что нужно сделать.** `/api/auth/sign-up|sign-in|sign-out`, `/api/user/me` строго по ТЗ.

**С чего начать.** С `sign-in` и `me` — это ровно веха B6.2 мостика. Потом `sign-up` (с автоматическим входом) и `sign-out`.

**Вопросы для размышления**
1. Как Security понимает, что пользователь залогинен при следующем запросе? *Направление: `SecurityContextRepository`.*
2. Регистрация сразу логинит. Как это сделать без повторного запроса логина?
3. CSRF для SPA на сессиях: выключить или настроить? Обоснуй.

**Как проверить себя**
- [ ] Коды и тела по ТЗ, `201` при регистрации.
- [ ] Валидация (Bean Validation) → 400 `{"message"}`; занятое имя → 409 (через ограничение БД).
- [ ] 401 JSON для неавторизованных, без HTML и редиректов.

### Веха 6.3 — Интеграционные тесты пользователей

**Что нужно сделать.** Тесты на Testcontainers (Postgres).

**Как проверить себя**
- [ ] Контейнер переиспользуется между тестовыми классами (разберись, как — статический контейнер/singleton container pattern).
- [ ] Кейсы из ТЗ: создание пользователя пишет запись; дубликат → ожидаемое исключение.
- [ ] HTTP-уровень (`MockMvc` или `RestClient`/`TestRestTemplate`): регистрация → кука → `/api/user/me`.

### Веха 6.4 — MinIO и фасад хранилища

**Что нужно сделать**
1. MinIO в compose; бакет создаётся при старте, если его нет.
2. Value object пути с тестами.
3. Фасад над MinIO с операциями приложения; префикс пользователя — только внутри.

**С чего начать.** С value object пути и его тестов: это чистая Java без MinIO. Список тест-кейсов: `a/b/c.txt`, `a/b/`,
`a//b`, `../x`, `/a`, пустая строка, имя с запрещёнными символами, очень длинное имя.

**Вопросы для размышления**
1. Как выглядит интерфейс хранилища **с точки зрения приложения**: операции, типы параметров и результатов?
2. Как отличить файл от папки? Как представить пустую папку? Где живёт это знание? *Направление: раздел 2 — только в фасаде.*
3. `folder1/../../user-2-files/secret.txt` — что произойдёт? *Направление: раздел 1.*

**Как проверить себя**
- [ ] Классы MinIO SDK (`io.minio.*`) импортируются только в пакете фасада.
- [ ] Тип пути покрыт тестами на все кейсы выше.

### Веха 6.5 — Загрузка файлов и папок

**Что нужно сделать.** `POST /api/resource?path=` с multipart, вложенные папки из имён файлов, `201` + список ресурсов.

**С чего начать.** Один файл в корень. Потом файл с подпапкой в имени (`upload_folder/test.txt`). Потом несколько файлов.

**Вопросы для размышления**
1. Куда Spring пишет большой multipart-файл, пока ты его обрабатываешь? *Направление: `spring.servlet.multipart.file-size-threshold`, `location`.*
2. Файл уже существует — 409. Как это проверить и есть ли здесь гонка?

**Как проверить себя**
- [ ] Файл 500 МБ загружается, и память приложения не растёт на 500 МБ (посмотри в VisualVM/`jcmd`).
- [ ] Разумные лимиты размера в конфиге.
- [ ] 409 при существующем файле.

### Веха 6.6 — Операции с ресурсами

**Что нужно сделать.** Все эндпоинты `/resource`, `/resource/move`, `/resource/download`, `/directory` по ТЗ, со всеми кодами ошибок.

**С чего начать.** С `GET /directory` (список содержимого): на нём проверишь, как фасад отдаёт папки и файлы.

**Вопросы для размышления**
1. Перемещение папки с 1000 файлов — сколько запросов к S3? Что если на 500-м упадёт?
   *Направление: атомарности нет. Как минимизировать ущерб: порядок «скопировать всё → удалить старое»?*
2. Переименование `a/2` в `a/1`, когда `a/1` уже есть: что должно произойти по ТЗ? *Направление: 409, а не затирание.*
3. Как отдать zip папки без временного файла? *Направление: раздел 3.*

**Как проверить себя**
- [ ] Ни одна операция не затирает существующий ресурс.
- [ ] Скачанные файлы и zip не повреждены: скачай, распакуй, сравни чексуммы (`sha256sum`).
- [ ] Несуществующая папка → 404.
- [ ] Результат `removeObjects` итерируется, ошибки не теряются.

### Веха 6.7 — Поиск, фронтенд, Swagger

**Что нужно сделать**
1. `GET /api/resource/search?query=` — только свои ресурсы.
2. React-фронтенд из ТЗ раздаётся Boot с корня.
3. Swagger UI документирует все эндпоинты с кодами ошибок.

**Вопросы для размышления**
1. Как искать по имени в S3, где нет индекса по именам? Во что это обойдётся при 100 000 файлов? (ответь словами)
2. Почему при обновлении страницы на вложенном URL фронтенда будет 404? *Направление: раздел 6.*

**Как проверить себя**
- [ ] Два пользователя с файлами одинаковых имён: каждый находит только свои.
- [ ] Весь сценарий из ТЗ проходится через фронтенд.

### Веха 6.8 — Redis-сессии, тесты хранилища, деплой

**Что нужно сделать**
1. Сессии в Redis (Spring Session), переживают рестарт — проверь.
2. ★ из ТЗ: интеграционные тесты фасада на MinIO в Testcontainers (`GenericContainer`): загрузка, rename, delete,
   **чужие файлы недоступны**, поиск не находит чужие.
3. JAR на VPS, инфраструктура через compose, секреты через env.
4. Сверка с [чеклистом из ТЗ](https://zhukovsd.github.io/java-backend-learning-course/projects/cloud-file-storage/#чеклист-для-самопроверки).

---

## Челленджи

- ★ **Квоты** на объём хранилища пользователя. Где считать занятый объём и как не пересчитывать его каждый раз?
- ★ **ArchUnit:** пакет `web` не зависит от `io.minio`, домен не зависит от Spring Web.
- ★★ **Presigned URLs** для скачивания больших файлов напрямую из MinIO. Плюсы, минусы, безопасность (TTL).
- ★★ **Докеризация приложения** (multi-stage Dockerfile, non-root, layered jar) и запуск всего стека одной командой.
- ★★★ **Multipart upload** в S3 для файлов > 100 МБ с возобновлением. Как поведёт себя текущий код с файлом 5 ГБ?
- ★★★ **Наблюдаемость:** Actuator, Micrometer, метрики загрузок/скачиваний, Prometheus + Grafana в compose.

## За что будет 🔴

- `user-{id}-files` в ответах API, в контроллерах, во фронтенде. Классы MinIO за пределами фасада.
- Путь — голая `String` с ручными `split("/")` по всему коду; возможен path traversal.
- Доступ к чужим файлам через поиск/скачивание/move с ручным URL.
- Затирание ресурсов при move/rename; битые zip; `byte[]` для целых файлов.
- Форматы/коды ответов не по ТЗ; HTML-редиректы вместо 401.
- CSRF выключен без обоснования.
- Результат `removeObjects` не проверяется.
- Секреты в `application.yml` в git.
- Логика в контроллерах; `@Transactional` на контроллере.

## Контрольные вопросы (защита)

1. Как работает автоконфигурация Boot? Как переопределить автоконфигурированный бин?
2. Путь запроса через Spring Security до контроллера. Где хранится аутентификация между запросами?
3. Сессии в Redis: что сериализуется, как работает TTL, что будет при двух инстансах приложения?
4. S3 vs файловая система: почему «переименование папки» дорогое? Что такое prefix и delimiter?
5. Как ты гарантируешь изоляцию пользователей? Приведи атаку и покажи, почему она не пройдёт.
6. Как загружаемый файл проходит от браузера до MinIO? Где он лежит в процессе? Сколько памяти занимает?
7. Testcontainers vs H2: плюсы и минусы. Как ускорить тесты с контейнерами?
8. Spring Data JPA: как из имени метода получается запрос? Когда `@Query`? Что даёт `readOnly = true`?
9. Docker: образ vs контейнер, volume vs bind mount, как контейнеры находят друг друга в compose?
10. REST vs RPC: почему в ТЗ auth — RPC, а ресурсы — REST?
