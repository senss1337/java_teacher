# Мостик М4. Spring и Hibernate с нуля

**Зачем мостик.** В проекте 3 ты всё делал руками: создавал объекты в listener-е, писал SQL, перекладывал `ResultSet`
в объекты. В проекте 4 появляются два фреймворка, которые делают это за тебя: **Spring** (создаёт и связывает объекты,
маршрутизирует HTTP, управляет транзакциями) и **Hibernate** (превращает объекты в строки таблиц и обратно).
Оба работают через «магию» (прокси, рефлексию, отложенную загрузку), и без понимания механики ты будешь ловить
загадочные ошибки. Здесь разбираем механику на игрушечных примерах.

**Как проходить.** Песочница `projects/04-0-bridge/`: сначала маленькое Spring-приложение без БД, потом Hibernate
на игрушечной схеме «библиотека» (авторы и книги).

**Сколько времени:** 7–10 дней.

---

## Часть 1. IoC и DI: от ручной сборки к Spring

### Простыми словами

В проекте 3 у тебя был **composition root** — listener, где ты руками писал:

```java
var dataSource = new HikariDataSource(config);
var currencyDao = new JdbcCurrencyDao(dataSource);
var rateDao = new JdbcRateDao(dataSource);
var exchangeService = new ExchangeService(rateDao, currencyDao);
```

Когда классов 50, этот код становится огромным. **Spring** делает это за тебя: ты помечаешь классы аннотациями,
а Spring сам создаёт объекты и передаёт их в конструкторы друг другу. Это называется **IoC (Inversion of Control)** —
«инверсия управления»: не ты создаёшь зависимости, а контейнер создаёт и **внедряет** их в тебя (**DI**).

- **ApplicationContext (IoC-контейнер)** — объект Spring, который хранит все созданные объекты.
- **Бин (bean)** — объект, которым управляет Spring.

```java
@Component                                   // «Spring, создай бин этого класса»
class JdbcBookDao implements BookDao {
    private final DataSource dataSource;
    JdbcBookDao(DataSource dataSource) {     // Spring найдёт бин DataSource и передаст сюда
        this.dataSource = dataSource;
    }
}

@Service                                     // то же, что @Component, но по смыслу — сервис
class LibraryService {
    private final BookDao books;             // зависимость от ИНТЕРФЕЙСА
    LibraryService(BookDao books) { this.books = books; }  // Spring найдёт единственную реализацию BookDao
}

@Configuration                               // класс-конфигурация: здесь вручную объявляют бины
@ComponentScan("dev.me.library")             // «найди все @Component в этом пакете»
class AppConfig {
    @Bean                                    // бин из чужого класса, на который аннотацию не повесить
    DataSource dataSource() { return new HikariDataSource(...); }
}

var ctx = new AnnotationConfigApplicationContext(AppConfig.class);   // запуск контейнера
LibraryService service = ctx.getBean(LibraryService.class);
```

**Способы внедрения:** через конструктор (правильно: зависимости видны, `final`, объект сразу валиден),
через сеттер, через поле `@Autowired private X x;` (не делай так: скрытая зависимость, нельзя `final`, неудобно в тестах).

### Скоупы бинов

По умолчанию бин — **singleton**: **один** экземпляр на весь контейнер, общий для всех потоков.
Как сервлет в проекте 3! Поэтому правило то же: **бины-сервисы не хранят данных конкретного запроса в полях** (stateless).
Другие скоупы: `prototype` (новый при каждом запросе бина), `request`, `session` (в вебе).

> ⚠️ Не путай: «singleton-бин» Spring — это «один на контейнер, создан контейнером». Паттерн Singleton со `static getInstance()`
> — другое, и он по-прежнему антипаттерн.

### Прокси: как Spring добавляет поведение

**Простыми словами.** Часть возможностей Spring (транзакции, безопасность, кеширование) работает так: вместо твоего объекта
Spring подсовывает другим бинам **обёртку** — прокси. Прокси перехватывает вызов, делает что-то до (открыть транзакцию),
вызывает твой настоящий метод, делает что-то после (commit/rollback). Это **AOP** — аспектно-ориентированное
программирование. Ближайшая аналогия в Python — декоратор, только навешивается снаружи и в рантайме.

```
  OrderController ──вызывает──> [Прокси OrderService] ──> [настоящий OrderService]
                                 1. begin transaction
                                 2. вызвать метод ───────> placeOrder()
                                 3. commit / rollback
```

**Главное следствие — ловушка self-invocation.** Если метод класса вызывает **другой свой** метод (`this.other()`),
вызов идёт мимо прокси, и `@Transactional` у `other()` не сработает. То же для `private`-методов: прокси не может их перехватить.

```java
@Service
class OrderService {
    public void placeAll(List<Order> orders) {
        orders.forEach(this::place);         // this — НЕ прокси: @Transactional у place() проигнорирован!
    }
    @Transactional
    public void place(Order o) { ... }
}
```

> 🎯 **Спросят на собесе:** Почему `@Transactional` не срабатывает при вызове метода из того же класса?
> **Ответ:** Spring оборачивает бин в прокси; транзакционная логика — в прокси. Вызов `this.method()` идёт мимо прокси.
> То же для `private`/`final` методов (CGLIB не может их переопределить).

---

## Часть 2. Spring MVC

### Простыми словами

Spring MVC — это то, что ты писал руками в проекте 3 (Front Controller + роутинг + JSON + обработка ошибок), только готовое.

- **`DispatcherServlet`** — один сервлет на все пути (Front Controller). Получает запрос, находит нужный метод
  контроллера (**HandlerMapping**), разбирает параметры, вызывает метод, превращает результат в JSON (**HttpMessageConverter** —
  внутри Jackson), ловит исключения.
- **`@RestController`** — класс с методами-обработчиками, результат которых идёт в тело ответа как JSON.

```java
@RestController
@RequestMapping("/books")
class BookController {
    private final LibraryService service;
    BookController(LibraryService service) { this.service = service; }

    @GetMapping("/{isbn}")                                       // /books/978-...
    BookDto get(@PathVariable String isbn) {                     // {isbn} из пути
        return service.findByIsbn(isbn);                         // вернул объект → Jackson → JSON
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    BookDto create(@RequestBody CreateBookRequest request) {     // JSON тела → объект
        return service.create(request);
    }
}

@RestControllerAdvice                                            // глобальный обработчик ошибок
class ErrorHandler {
    @ExceptionHandler(BookNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    ErrorDto notFound(BookNotFoundException e) { return new ErrorDto(e.getMessage()); }
}
```

Сравни с FastAPI: `@app.get("/books/{isbn}")` ≈ `@GetMapping("/{isbn}")`; `exception_handler` ≈ `@ExceptionHandler`.

**Без Spring Boot** (как в проекте 4) `DispatcherServlet` регистрируют сами: класс, реализующий
`WebApplicationInitializer` (или наследник `AbstractAnnotationConfigDispatcherServletInitializer`), Tomcat найдёт его при старте.
Плюс `@EnableWebMvc` на конфигурации.

---

## Часть 3. ORM и Hibernate

### Простыми словами

В проекте 3 ты писал SQL и перекладывал `ResultSet` в объекты руками. **ORM** (Object-Relational Mapping) делает это
автоматически: описываешь класс, а ORM сама генерирует `INSERT/SELECT/UPDATE`. Ты знаешь это по SQLAlchemy ORM
или Django ORM.

- **JPA** (Jakarta Persistence) — **стандарт** (набор аннотаций и интерфейсов).
- **Hibernate** — самая популярная **реализация** JPA.

```java
@Entity                                              // класс ↔ таблица
@Table(name = "books", indexes = @Index(columnList = "title"))
public class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 200)
    private String title;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)   // много книг → один автор
    @JoinColumn(name = "author_id", nullable = false)
    private Author author;

    protected Book() {}                              // JPA требует конструктор без аргументов (может быть protected)
    public Book(String title, Author author) { ... } // а твой код пользуется нормальным
}
```

### Сессия и состояния объекта — главное, что надо понять

**Persistence context** (в Hibernate — `Session`, в JPA — `EntityManager`) — «рабочая область», в которой Hibernate
**следит** за загруженными объектами. Аналог `Session` в SQLAlchemy.

Состояния сущности:
- **Transient** — просто `new Book(...)`, Hibernate о нём не знает.
- **Managed (persistent)** — объект в сессии: загружен через неё или сохранён `persist()`. Hibernate отслеживает изменения.
- **Detached** — сессия закрыта, объект остался, но изменения больше не отслеживаются.
- **Removed** — помечен на удаление.

**Dirty checking.** Изменил поле managed-объекта — **ничего не вызывай**, при коммите Hibernate сам сравнит с исходным
состоянием и выполнит `UPDATE`:

```java
try (Session s = sessionFactory.openSession()) {
    Transaction tx = s.beginTransaction();
    Book b = s.find(Book.class, 1L);       // SELECT; b — managed (find — стандартный метод JPA)
    b.rename("New title");                 // просто меняем объект
    tx.commit();                           // Hibernate сам сделает UPDATE books SET title=... WHERE id=1
}
```

**`SessionFactory`** — тяжёлый объект: настраивается один раз при старте, живёт всё приложение, закрывается при остановке.
**`Session`** — лёгкая, на одну единицу работы (обычно один запрос/одну транзакцию).

> 🎯 **Спросят на собесе:** Состояния сущности в Hibernate/JPA?
> **Ответ:** Transient (новый объект, не связан с сессией), Managed/Persistent (в контексте персистентности, изменения
> отслеживаются dirty checking и сбрасываются при flush), Detached (сессия закрыта, изменения не отслеживаются),
> Removed (помечен на удаление).

### Ленивая загрузка и `LazyInitializationException`

**Простыми словами.** Загрузил книгу — нужен ли сразу её автор? Если связь **LAZY**, Hibernate вместо автора кладёт
**прокси-заглушку** и сделает `SELECT` за автором только при первом обращении (`book.getAuthor().getName()`).
Если к этому моменту сессия уже закрыта — `LazyInitializationException`.

**EAGER** — грузить сразу. Кажется удобным, но грузит лишнее всегда и провоцирует N+1.
Коварство: у `@ManyToOne` и `@OneToOne` **по умолчанию EAGER**. Рекомендация: **всё LAZY**, а нужное грузить явно.

> 🎯 **Спросят на собесе:** `FetchType.LAZY` vs `EAGER`, какие значения по умолчанию?
> **Ответ:** `@ManyToOne`/`@OneToOne` — EAGER, `@OneToMany`/`@ManyToMany` — LAZY. Рекомендация — всё LAZY,
> а нужные связи подгружать явно (`JOIN FETCH`, `@EntityGraph`) в конкретных запросах.

### N+1 в ORM

```java
List<Book> books = session.createQuery("from Book", Book.class).getResultList();  // 1 запрос
for (Book b : books) {
    System.out.println(b.getAuthor().getName());      // + по запросу на КАЖДУЮ книгу (LAZY) = N+1
}
```

Лечение — сказать в запросе, что автор нужен сразу:

```java
session.createQuery("select b from Book b join fetch b.author", Book.class)   // 1 запрос с JOIN
```

Чтобы **видеть** N+1, включи логирование SQL (`hibernate.show_sql=true` или логгер `org.hibernate.SQL`).
Всегда смотри, сколько запросов реально уходит.

### Пагинация

`query.setFirstResult(offset).setMaxResults(size)` → `LIMIT/OFFSET` в SQL. Плюс отдельный `select count(...)`
для общего числа страниц. Всегда с **сортировкой** (`order by`), иначе порядок страниц не определён.

> ⚠️ `JOIN FETCH` **коллекции** (`@OneToMany`) + пагинация = Hibernate грузит всё в память и режет там (предупреждение
> HHH90003004). Для `@ManyToOne` проблемы нет.

### Транзакции в Spring + Hibernate

С Spring не нужно открывать сессии и транзакции руками: `@Transactional` на методе сервиса, и прокси откроет транзакцию,
привяжет сессию к потоку и закоммитит/откатит. DAO получает текущую сессию через `sessionFactory.getCurrentSession()`
(или `EntityManager`). Для этого нужен бин `HibernateTransactionManager` (или `JpaTransactionManager`).
В Spring 7 классы интеграции с Hibernate лежат в пакете `org.springframework.orm.jpa.hibernate`
(в старых статьях — `org.springframework.orm.hibernate5`, его больше нет)
и `@EnableTransactionManagement`.

Антипаттерн **session-per-operation** — новая сессия на каждый вызов DAO: теряются кеш первого уровня, dirty checking и атомарность.

---

📚 **Читать:**
- SIA, гл. 1 «Getting started with Spring» (IoC, DI, конфигурация) и гл. 2 «Developing web applications» (Spring MVC). Всё, что про Boot, мысленно переводи в ручную конфигурацию.
- [Spring Framework Reference: Core — The IoC Container](https://docs.spring.io/spring-framework/reference/core/beans.html): разделы Introduction, Bean Overview, Dependencies, Bean Scopes, Java-based Container Configuration.
- [Spring: AOP Proxies — Understanding AOP Proxies](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html) — **обязательно**, там картинка про self-invocation.
- [Spring Web MVC — DispatcherServlet](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html), [Annotated Controllers](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller.html).
- [Spring: Declarative Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html).
- [Hibernate ORM User Guide](https://hibernate.org/orm/documentation/) (на странице документации выбери версию 7.x → User Guide): разделы Domain Model (Entity types, Associations), Bootstrap, Persistence Context, Fetching.
- HPJP, часть II «JPA and Hibernate»: главы про маппинг связей (Relationships), Flushing, **Fetching**.
- Блог Михалчи: [N+1 query problem](https://vladmihalcea.com/n-plus-1-query-problem/), [The best way to map @ManyToOne](https://vladmihalcea.com/manytoone-jpa-hibernate/), [LazyInitializationException](https://vladmihalcea.com/the-best-way-to-handle-the-lazyinitializationexception/).
- PoEAA, гл. 11 (Unit of Work, Identity Map, Lazy Load) — это ровно то, что делает Hibernate.

---

## Упражнения и вехи

### Веха B4.1 — Spring без веба и Spring MVC

**Что нужно сделать**
1. Консольное приложение: 3–4 класса (например, «сервис уведомлений» + два канала отправки + репозиторий в памяти),
   связанные через Spring-контейнер (`@Configuration` + `@ComponentScan`). Все зависимости — через конструктор.
2. Два бина одного интерфейса (два канала). Разберись, как Spring выбирает, какой внедрить (`@Primary`, `@Qualifier`,
   внедрение `List<Channel>` — всех сразу).
3. **Прокси своими глазами:** аннотация `@Transactional` пока не нужна, сделай свой аспект через Spring AOP
   (`@Aspect`, `@Around`), логирующий время выполнения методов сервиса. Потом вызови метод сервиса из другого метода
   **того же** класса и убедись, что аспект не сработал. Объясни почему.
4. Маленькое Spring MVC-приложение (без Boot, WAR в **Tomcat 11** — Spring 7 требует Servlet 6.1): `GET /greetings/{name}`, `POST /greetings` (JSON),
   `@RestControllerAdvice` для ошибки валидации → 400 `{"message": ...}`.

**С чего начать.** С п. 1: заставь `ctx.getBean(...)` вернуть полностью собранный сервис.

**Как проверить себя**
- [ ] Нет `@Autowired` на полях.
- [ ] Можешь нарисовать схему «кто кого вызывает» с прокси в п. 3 и показать, где вызов обошёл прокси.
- [ ] `curl` к Spring MVC приложению даёт правильные коды и JSON.

### Веха B4.2 — Hibernate на игрушечной схеме

**Что нужно сделать** (консольное приложение, Postgres в Docker или H2):
1. Сущности `Author` и `Book` (`@ManyToOne` LAZY, ограничения `nullable`, `length`, уникальный ISBN).
2. `SessionFactory` создаётся один раз и закрывается при выходе.
3. **Dirty checking:** загрузи книгу, поменяй название без вызова `update`/`merge`, закоммить; убедись по логу SQL, что был `UPDATE`.
4. **LazyInitializationException:** загрузи книгу, закрой сессию, обратись к автору. Поймай исключение. Исправь двумя способами
   (`join fetch` в запросе; обращение к автору внутри сессии) и объясни, почему «держать сессию открытой подольше» — не решение.
5. **N+1:** 50 книг разных авторов, выведи «книга — автор». Посчитай запросы в логе. Исправь `join fetch`.
6. **Пагинация:** страница 3 по 10 книг, отсортированных по названию + общее число страниц. Посмотри сгенерированный SQL.
7. **Состояния:** тест или консольная демонстрация transient → managed → detached → merge.

**С чего начать.** С п. 1–2 и включённого `show_sql`: сохрани одного автора и посмотри, какой SQL ушёл.

**Как проверить себя**
- [ ] Все связи LAZY, EAGER нигде явно не указан.
- [ ] Для каждого пункта 3–6 в README: сколько SQL-запросов ушло и почему.
- [ ] Можешь без подсказок рассказать про 4 состояния сущности и dirty checking.

---

## Контрольные вопросы

1. IoC и DI: что это, чем конструкторная инъекция лучше полевой?
2. Бин, контекст, скоупы. Почему singleton-бин должен быть stateless?
3. Как Spring выбирает бин при нескольких реализациях интерфейса?
4. Прокси и AOP: как работает `@Transactional`, что такое self-invocation?
5. Путь запроса через `DispatcherServlet` до твоего контроллера.
6. JPA vs Hibernate. `SessionFactory` vs `Session`.
7. Состояния сущности, dirty checking, first-level cache.
8. LAZY vs EAGER, дефолты, `LazyInitializationException`.
9. N+1: как увидеть и как лечить.
10. Session-per-operation: почему антипаттерн.
