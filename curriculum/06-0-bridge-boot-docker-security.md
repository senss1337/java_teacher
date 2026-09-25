# Мостик М6. Docker, Spring Boot, Spring Security, Spring Data

**Зачем мостик.** Проект «Облачное хранилище» — первый на Spring Boot, с Docker Compose, Spring Security, Redis и S3.
Каждая из этих технологий сама по себе — отдельная тема для изучения. Здесь по очереди разбираем их на игрушках:
«hello world» контейнеров, минимальное Boot-приложение, Security на двух эндпоинтах, репозиторий Spring Data.

**Как проходить.** Песочница `projects/06-0-bridge/`: одно Boot-приложение, которое ты постепенно наращиваешь,
плюс `docker-compose.yml`.

**Сколько времени:** 7–10 дней.

---

## Часть 1. Docker с нуля

### Простыми словами

До сих пор ты ставил Postgres, Tomcat и Java прямо в систему. Проблемы: разные версии на разных машинах,
«у меня работает, а на сервере нет», мусор в системе. **Docker** упаковывает программу **вместе со всем окружением**
(библиотеки, настройки, даже урезанная ОС) в **образ**. Образ запускается одинаково везде, где есть Docker.

- **Образ (image)** — неизменяемый «слепок»: `postgres:16`, `redis:7`, `minio/minio`. Скачивается с Docker Hub.
  Аналогия: класс.
- **Контейнер** — запущенный экземпляр образа, изолированный процесс. Аналогия: объект класса. Из одного образа можно
  запустить много контейнеров.
- **Volume** — хранилище данных **вне** контейнера. Контейнер удалил → данные в volume остались. Без volume база
  потеряет всё при пересоздании контейнера.
- **Порт** — контейнер изолирован, чтобы достучаться до Postgres с хоста, «пробрасываешь» порт: `-p 5432:5432`
  (хост:контейнер).

```bash
docker run -d --name pg \
  -e POSTGRES_PASSWORD=secret \        # переменная окружения внутри контейнера
  -p 5432:5432 \                       # порт хоста → порт контейнера
  -v pgdata:/var/lib/postgresql/data \ # именованный volume для данных
  postgres:16
docker ps                              # запущенные контейнеры
docker logs -f pg                      # логи
docker exec -it pg psql -U postgres    # зайти внутрь и выполнить команду
docker stop pg && docker rm pg         # остановить и удалить (volume pgdata остался!)
docker volume rm pgdata                # удалить данные
```

### Docker на Mac

Docker работает только на Linux, поэтому на Mac он запускается внутри лёгкой виртуальной машины. Удобнее всего
**Docker Desktop** ([docker.com](https://www.docker.com/products/docker-desktop/), выбери версию для Apple Silicon или Intel).
Лёгкие альтернативы — **OrbStack** или **Colima** (`brew install colima docker`): команды `docker` и `docker compose` те же.

Что важно знать на Mac:
- **Память.** Виртуальной машине выдаётся ограниченная память (*Docker Desktop → Settings → Resources*). Для Postgres +
  Redis + MinIO (проект 6) или Kafka (проект 7) поставь не меньше 4–6 ГБ.
- **Apple Silicon (arm64).** Официальные образы (`postgres`, `redis`, `minio/minio`, `apache/kafka`, `eclipse-temurin`)
  собраны и под arm64, всё работает нативно. Если какой-то образ есть только под amd64, Docker скажет
  `no matching manifest for linux/arm64`. Тогда добавь в сервис compose `platform: linux/amd64`: заработает через эмуляцию,
  но медленнее.
- **Testcontainers** находят Docker Desktop автоматически. С Colima иногда нужно указать `DOCKER_HOST`
  (см. [документацию Testcontainers](https://java.testcontainers.org/supported_docker_environment/)).

### Docker Compose

Когда контейнеров несколько (Postgres + Redis + MinIO), команды `docker run` с кучей флагов неудобны.
**Compose** описывает весь набор в одном YAML-файле и запускает его одной командой:

```yaml
# docker-compose.yml (нейтральный пример: блог)
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: blog
      POSTGRES_PASSWORD: ${DB_PASSWORD}      # из файла .env рядом (его — в .gitignore)
    ports: ["5432:5432"]
    volumes: ["blog-db:/var/lib/postgresql/data"]
    healthcheck:                             # как понять, что БД готова принимать соединения
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
  cache:
    image: redis:7
    ports: ["6379:6379"]
volumes:
  blog-db:
```

```bash
docker compose up -d        # поднять всё в фоне
docker compose ps
docker compose logs -f db
docker compose down         # остановить и удалить контейнеры (volumes остаются)
docker compose down -v      # ...и удалить volumes (данные!)
```

Внутри compose контейнеры видят друг друга **по имени сервиса**: приложение в контейнере подключается к `db:5432`,
а не к `localhost`. С хоста (из IDEA) — через проброшенный порт `localhost:5432`.

---

## Часть 2. Spring Boot: что он делает за тебя

### Простыми словами

В проектах 4–5 ты руками: регистрировал `DispatcherServlet`, настраивал Jackson, `DataSource`, `SessionFactory`,
менеджер транзакций, собирал WAR и клал в Tomcat. **Spring Boot** делает это сам:

1. **Стартеры** — зависимости-наборы. `spring-boot-starter-webmvc` = Spring MVC + Jackson + встроенный Tomcat + логирование
   (в Boot 3 и старых статьях он назывался `spring-boot-starter-web`; в Boot 4 это имя помечено устаревшим).
   `spring-boot-starter-data-jpa` = Hibernate + Spring Data + HikariCP.
2. **Автоконфигурация** — Boot смотрит, что лежит в classpath, и создаёт нужные бины. Видит драйвер Postgres
   и настройки `spring.datasource.*` → создаёт `DataSource`. Видит Hibernate → создаёт `EntityManagerFactory` и транзакции.
   **Главное правило:** если ты объявил свой бин того же типа, Boot свой не создаёт (`@ConditionalOnMissingBean`).
3. **Встроенный сервер** — приложение собирается в **исполняемый JAR** с Tomcat внутри: `java -jar app.jar`. WAR не нужен.
4. **Конфигурация** — `application.yml`/`.properties`, профили, переменные окружения переопределяют файл
   (`SPRING_DATASOURCE_PASSWORD` → `spring.datasource.password`).

> ⚠️ Мы используем **Spring Boot 4**. Большинство туториалов написано под Boot 3: концепции те же, но отличаются
> имена стартеров, пакеты Jackson 3, моки в тестах (`@MockitoBean`). Список отличий — в
> [«Версии стека»](00-setup.md#что-в-старых-туториалах-выглядит-иначе).

```java
@SpringBootApplication                     // = @Configuration + @ComponentScan + @EnableAutoConfiguration
public class BlogApplication {
    public static void main(String[] args) { SpringApplication.run(BlogApplication.class, args); }
}
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/blog
    username: postgres
    password: ${DB_PASSWORD}               # из окружения
  jpa:
    hibernate.ddl-auto: validate
app:
  posts:
    page-size: 20                          # своя настройка
```

**Свои настройки** — через `@ConfigurationProperties` (типобезопасно, с валидацией), а не россыпь `@Value`:

```java
@Validated                                         // без неё @Min молча НЕ проверяется!
@ConfigurationProperties(prefix = "app.posts")
record PostsProperties(@Min(1) int pageSize) {}
// + @EnableConfigurationProperties(PostsProperties.class) или @ConfigurationPropertiesScan
// + зависимость spring-boot-starter-validation (реализация Bean Validation)
```

При `page-size: 0` приложение не стартует и пишет понятную ошибку. Лучше упасть при запуске, чем работать с неверным конфигом.

**«Откуда взялся этот бин?»** Запусти с `--debug`: Boot напечатает **condition evaluation report** — какие автоконфигурации
сработали и почему, какие нет.

> 🎯 **Спросят на собесе:** Как работает автоконфигурация Spring Boot?
> **Ответ:** Boot импортирует классы из `META-INF/spring/...AutoConfiguration.imports`. Каждый из них — `@Configuration`
> с условиями (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`): бин создаётся, только если
> нужный класс есть в classpath и пользователь не определил свой. Отчёт — `--debug`.

---

## Часть 3. Spring Data JPA

### Простыми словами

В проекте 4 ты писал DAO с HQL руками. **Spring Data** генерирует реализацию репозитория **по интерфейсу**:

```java
interface PostRepository extends JpaRepository<Post, Long> {     // CRUD уже есть: save, findById, findAll, delete...
    Optional<Post> findBySlug(String slug);                       // запрос выводится из имени метода
    boolean existsBySlug(String slug);                            // лучше, чем findBySlug(...).isPresent()
    Page<Post> findByAuthorId(Long authorId, Pageable pageable);  // пагинация из коробки

    @Query("select p from Post p join fetch p.author where p.published = true")   // сложное — руками
    List<Post> findPublishedWithAuthors();
}
```

Всё, что ты знаешь про Hibernate (LAZY, N+1, состояния), остаётся в силе: Spring Data работает поверх него.
`@Transactional(readOnly = true)` на методах чтения — подсказка Hibernate не делать dirty checking.

---

## Часть 4. Spring Security

### Простыми словами

В проекте 5 ты руками делал: проверку пароля, сессию, куку, interceptor для защиты страниц. **Spring Security** делает то же,
но это большой фреймворк со своей архитектурой. Главное — понять его устройство, иначе он кажется магией.

**Это цепочка фильтров.** Security встраивается в обработку запроса набором сервлетных фильтров (ты знаешь их по проекту 3):

```
HTTP-запрос
  → DelegatingFilterProxy (обычный фильтр контейнера, передаёт в Spring)
  → FilterChainProxy → выбирает SecurityFilterChain по пути
      → SecurityContextHolderFilter      достаёт аутентификацию из сессии (если была)
      → ... фильтры аутентификации ...   (форма логина, basic, JWT — что настроено)
      → ExceptionTranslationFilter       превращает ошибки доступа в 401/403
      → AuthorizationFilter              проверяет: можно ли этому пользователю на этот путь
  → DispatcherServlet → твой контроллер
```

**Ключевые понятия:**
- **Аутентификация** — «кто ты» (проверка логина/пароля). **Авторизация** — «что тебе можно».
- `Authentication` — объект «кто залогинен и с какими правами».
- `SecurityContextHolder` — где лежит `Authentication` текущего запроса (в `ThreadLocal`: у каждого потока своя копия).
- `SecurityContextRepository` — где контекст хранится **между** запросами (по умолчанию в `HttpSession`).
- `UserDetailsService` — «найди пользователя по имени» (ты реализуешь поверх своего репозитория).
- `PasswordEncoder` — BCrypt.
- `AuthenticationManager` — «проверь эти логин и пароль».
- `AuthenticationEntryPoint` — что ответить неаутентифицированному (по умолчанию редирект на форму логина,
  для REST нужен 401 JSON).

```java
@Configuration
@EnableWebSecurity
class SecurityConfig {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated())
            .exceptionHandling(e -> e.authenticationEntryPoint(
                (req, resp, ex) -> resp.sendError(401)))          // для API: 401 вместо редиректа
            .build();
    }
    @Bean PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
}
```

**Логин через свой REST-эндпоинт** (как требует ТЗ проекта 6) — самое неочевидное место. Ты вызываешь
`AuthenticationManager.authenticate(...)`, кладёшь результат в `SecurityContext` **и явно сохраняешь** контекст
через `SecurityContextRepository`: начиная с Security 6 (и в нашей 7) это не происходит автоматически. Прочитай
[Persisting Authentication](https://docs.spring.io/spring-security/reference/servlet/authentication/persistence.html) — там это описано.

**CSRF.** По умолчанию Security требует CSRF-токен для POST/PUT/DELETE. Для SPA на сессиях есть варианты
(токен в куке `XSRF-TOKEN`, который фронт читает и шлёт в заголовке) или выключение с пониманием рисков. Решение надо обосновать.

> 🎯 **Спросят на собесе:** Как устроена цепочка фильтров Spring Security?
> **Ответ:** `DelegatingFilterProxy` в контейнере → `FilterChainProxy` → выбор `SecurityFilterChain` по запросу →
> упорядоченные фильтры (`SecurityContextHolderFilter`, аутентификация, `ExceptionTranslationFilter`, `AuthorizationFilter`...).
> `SecurityContext` хранится в `ThreadLocal` на время запроса и сохраняется в сессию через `SecurityContextRepository`.

> 🎯 **Спросят на собесе:** Где Spring Security хранит данные аутентифицированного пользователя во время запроса и почему это работает в многопоточном сервере?
> **Ответ:** В `SecurityContextHolder`, по умолчанию стратегия `ThreadLocal`: каждый поток обработки запроса видит свой контекст.
> Между запросами контекст хранится в `HttpSession` (или Redis через Spring Session), при новом запросе восстанавливается фильтром.
> **Типичная ошибка:** забыть, что при `@Async`/своих пулах потоков `ThreadLocal`-контекст не переносится автоматически.

---

## Часть 5. Redis и Spring Session

**Простыми словами.** По умолчанию `HttpSession` живёт в памяти приложения: перезапуск — все разлогинены;
два инстанса — сессия есть только на одном. Решение: хранить сессии во внешнем хранилище.
**Redis** — быстрое key-value хранилище в памяти со встроенным **TTL** (время жизни записи): истёкшие сессии удаляются сами.
**Spring Session** подменяет хранилище `HttpSession` на Redis — твой код не меняется, меняется зависимость и пара настроек.

---

## Часть 6. S3 и MinIO

**Простыми словами.** **S3** — объектное хранилище (протокол Amazon, стал стандартом). В нём нет папок и файлов
в привычном смысле:
- **бакет (bucket)** — «диск»;
- **объект** — данные + **ключ** (строка, например `photos/2024/cat.jpg`).

«Папки» — иллюзия: `photos/2024/` — просто **общее начало ключей** (**prefix**). Из этого следствия:
- **пустой папки не бывает** — её изображают объектом-маркером нулевого размера с ключом `photos/2024/`;
- **переименовать папку** нельзя — нужно скопировать **каждый** объект под новым ключом и удалить старые;
- «содержимое папки» — `listObjects(prefix="photos/2024/", recursive=false)`.

**MinIO** — S3-совместимое хранилище, которое запускается локально в Docker. Работа из Java — MinIO Java SDK.

---

## Часть 7. Testcontainers

**Простыми словами.** В проекте 5 тесты шли на тестовой схеме реальной БД, которую ты поднимал сам. **Testcontainers**
поднимает **настоящие** Postgres/Redis/MinIO в Docker **автоматически на время тестов** и удаляет после. Тесты становятся
самодостаточными: `./mvnw verify` на любой машине с Docker.

```java
// зависимости (test): spring-boot-testcontainers, org.testcontainers:testcontainers-junit-jupiter,
// org.testcontainers:testcontainers-postgresql   ← имена модулей Testcontainers 2.x
import org.testcontainers.postgresql.PostgreSQLContainer;   // новый пакет в 2.x, класс без <?>

@SpringBootTest
@Testcontainers
class PostRepositoryTest {
    @Container
    @ServiceConnection                                   // Boot сам подставит url/user/password
    static PostgreSQLContainer pg = new PostgreSQLContainer("postgres:17");

    @Autowired PostRepository repo;

    @Test void savesPost() { ... }
}
```

Для MinIO в Testcontainers есть готовый модуль (`testcontainers-minio`), но ТЗ проекта 6 предлагает написать
свой `GenericContainer` (образ, порт, переменные окружения, стратегия ожидания готовности). Сделай сам, это полезное упражнение.

---

📚 **Читать:**
- Найджел Поултон, *Docker Deep Dive*: главы про образы, контейнеры, volumes, Compose. Или [Docker docs: Get started](https://docs.docker.com/get-started/) (части 1–6) + [Compose file reference](https://docs.docker.com/reference/compose-file/services/).
- SIA: гл. 1 (Boot, стартеры), гл. 3 «Working with data» (Spring Data JPA), гл. 5 «Securing Spring», гл. 6 «Working with configuration properties».
- [Spring Boot Reference: Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html), [Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html).
- [Spring Data JPA Reference: Query Methods](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html).
- SSIA: главы про архитектуру Spring Security (фильтры, `AuthenticationManager`, `UserDetailsService`, `PasswordEncoder`), CSRF и CORS.
- [Spring Security Reference: Architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html) и [Persisting Authentication](https://docs.spring.io/spring-security/reference/servlet/authentication/persistence.html) — **обязательно**.
- [Spring Session — Redis](https://docs.spring.io/spring-session/reference/getting-started/using-redis.html).
- [AWS S3 User Guide: Organizing objects using prefixes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-prefixes.html).
- [Testcontainers: Getting started](https://java.testcontainers.org/quickstart/junit_5_quickstart/), [Spring Boot: Testcontainers support](https://docs.spring.io/spring-boot/reference/testing/testcontainers.html).

---

## Упражнения и вехи

### Веха B6.1 — Docker и Boot

**Что нужно сделать**
1. `docker run` Postgres с volume; создай таблицу через `docker exec ... psql`; пересоздай контейнер — данные на месте;
   удали volume — данных нет.
2. `docker-compose.yml`: Postgres + Redis + MinIO (консоль MinIO на порту 9001 — зайди в неё браузером, создай бакет,
   загрузи файл). `.env` в `.gitignore`, `.env.example` в git.
3. Spring Boot-приложение «заметки» (Web + Data JPA + Postgres из compose + Flyway или Liquibase): `GET/POST /notes`,
   репозиторий Spring Data, свои настройки через `@ConfigurationProperties`.
4. Запусти с `--debug`, найди в отчёте, почему создался `DataSource`. Объяви свой бин `ObjectMapper` и убедись, что
   автоконфигурированный отключился.

**С чего начать.** С п. 1, это 15 минут.

**Как проверить себя**
- [ ] `docker compose up -d` поднимает всё, `docker compose down` + `up` — данные на месте.
- [ ] `./mvnw package && java -jar target/*.jar` — приложение работает без Tomcat.
- [ ] Пароль БД не в git.
- [ ] Можешь объяснить, как Boot решил создать `DataSource` и почему не создал второй `ObjectMapper`.

### Веха B6.2 — Security, Redis, Testcontainers

**Что нужно сделать** (в том же приложении «заметки»):
1. Пользователи в БД, BCrypt, `UserDetailsService` поверх своего репозитория.
2. Логин **JSON-эндпоинтом** `POST /api/login` (не формой!) с сохранением контекста в сессию; `GET /api/me`;
   `/notes` — только аутентифицированным; неаутентифицированный → 401 JSON (не редирект).
3. Заметки видны только владельцу.
4. Spring Session + Redis: залогинься, перезапусти приложение — сессия жива.
5. Интеграционный тест на Testcontainers (Postgres): регистрация → логин → кука → `/api/me`.

**С чего начать.** С п. 2 без БД: пользователь в памяти (`InMemoryUserDetailsManager`), просто чтобы логин через JSON
заработал. Потом переключи на БД.

**Как проверить себя**
- [ ] `curl` без куки к `/notes` → 401 с JSON, не 302.
- [ ] После рестарта приложения кука всё ещё работает (Redis).
- [ ] Можешь нарисовать путь запроса через фильтры Security до контроллера.
- [ ] Можешь объяснить своё решение по CSRF.

---

## Контрольные вопросы

1. Образ vs контейнер vs volume. Почему БД без volume теряет данные?
2. Как контейнеры находят друг друга в compose? Чем `localhost` из контейнера отличается от `localhost` хоста?
3. Что делает Spring Boot по сравнению с «голым» Spring? Как работает автоконфигурация и как её переопределить?
4. `@ConfigurationProperties` vs `@Value`.
5. Как Spring Data строит запрос из имени метода? Когда нужен `@Query`?
6. Архитектура Spring Security: фильтры, `SecurityContextHolder`, `SecurityContextRepository`.
7. Аутентификация vs авторизация. Как вернуть 401 вместо редиректа?
8. Зачем выносить сессии в Redis?
9. S3: бакет, ключ, префикс. Почему «переименовать папку» дорого?
10. Testcontainers vs H2 vs тестовая схема: плюсы и минусы.
