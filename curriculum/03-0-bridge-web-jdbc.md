# Мостик М3. Веб на Java и JDBC с нуля

**Зачем мостик.** В проекте «Обмен валют» разом появляются: сервер Tomcat, сервлеты, фильтры, JSON, SQL-база через JDBC,
слоистая архитектура и деплой на Linux. По отдельности каждая вещь несложная, но все вместе — стена.
Здесь разбираем их по одной на маленьких упражнениях.

**Как проходить.** Проект-песочница `projects/03-0-bridge/` (можно несколько маленьких Maven-модулей или проектов).
Упражнения **намеренно не похожи** на валютный API: цель — понять механику, а не заготовить решение.

**Сколько времени:** 5–7 дней.

---

## Часть 1. Как Java обрабатывает HTTP: сервлеты и Tomcat

### Простыми словами

В Python-вебе две роли: **приложение** (Django/Flask/FastAPI-код) и **сервер**, который принимает HTTP-соединения
и вызывает приложение (gunicorn, uvicorn). Между ними стандартный интерфейс — WSGI/ASGI.

В Java то же самое:
- **Сервлет-контейнер** (Tomcat, Jetty) — сервер: слушает порт, разбирает HTTP, управляет пулом потоков.
- **Сервлет** — твой класс, который обрабатывает запросы. Стандартный интерфейс между ними —
  спецификация **Jakarta Servlet** (пакет `jakarta.servlet`; в старых версиях — `javax.servlet`).

Ты пишешь класс-наследник `HttpServlet` и переопределяешь методы `doGet`, `doPost` и т. д.:

```java
@WebServlet("/hello")                                   // по какому пути вызывать (как @app.route)
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        String name = req.getParameter("name");         // ?name=... (как request.args.get)
        resp.setContentType("text/plain; charset=UTF-8");
        resp.getWriter().write("Hello, " + (name == null ? "stranger" : name));
    }
}
```

**Как это попадает в Tomcat.** Проект собирается в **WAR** (`<packaging>war</packaging>`). WAR кладётся в папку
`webapps/` Tomcat-а, и Tomcat разворачивает его по пути с именем файла: `myapp.war` → `http://localhost:8080/myapp/hello`.
Это «контекстный путь» (context path).

### Жизненный цикл сервлета — важно!

1. Tomcat создаёт **один** экземпляр твоего сервлета (не по одному на запрос!).
2. Вызывает `init()`.
3. На **каждый** запрос вызывает `service()` → `doGet()`/`doPost()`... **в отдельном потоке из пула**.
   Одновременно 50 запросов = 50 потоков в **одном и том же** объекте сервлета.
4. При остановке — `destroy()`.

**Следствие:** поле в сервлете видят все запросы одновременно. Хранить в нём данные конкретного запроса нельзя:
будет гонка (мостик М2, часть 5). Поля сервлета — только общие, неизменяемые или потокобезопасные зависимости
(например, ссылка на сервис).

```java
public class BadServlet extends HttpServlet {
    private String currentUser;                          // ОШИБКА: общий для всех потоков
    protected void doGet(...) {
        currentUser = req.getParameter("user");          // запрос А записал "alice"
        // ... в этот момент запрос Б записал "bob"
        resp.getWriter().write(currentUser);             // запрос А отдаст "bob"!
    }
}
```

> 🎯 **Спросят на собесе:** Жизненный цикл сервлета?
> **Ответ:** Контейнер загружает класс, создаёт **один** экземпляр, вызывает `init()`; на каждый запрос — `service()`
> (диспетчеризует в `doGet/doPost/...`) в потоке из пула контейнера; при остановке — `destroy()`.
> **Типичная ошибка:** «на каждый запрос создаётся новый сервлет». Отсюда баги с полями-состоянием.

### Фильтры — middleware

**Фильтр** — код, который выполняется **до и после** сервлета для набора путей. Аналог middleware в Django/FastAPI.
Нужен для общих вещей: кодировка, заголовки, логирование, CORS, обработка ошибок.

```java
@WebFilter("/*")                                          // для всех путей
public class TimingFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        long start = System.nanoTime();
        chain.doFilter(req, resp);                        // передать дальше по цепочке (к следующему фильтру/сервлету)
        long ms = (System.nanoTime() - start) / 1_000_000;
        System.out.println(((HttpServletRequest) req).getRequestURI() + " took " + ms + " ms");
    }
}
```

Цепочка фильтров → сервлет — это паттерн **Chain of Responsibility**: каждый обработчик решает, передать дальше или ответить сам.

**Front Controller** — идея «один входной сервлет на все пути, внутри него маршрутизация по обработчикам».
Так устроен `DispatcherServlet` в Spring MVC.

### Старт и остановка приложения: `ServletContextListener`

Где создать объекты, которые нужны всем сервлетам (пул соединений к БД, сервисы)? При старте приложения:

```java
@WebListener
public class AppStartup implements ServletContextListener {
    public void contextInitialized(ServletContextEvent e) {
        var clock = Clock.systemUTC();
        e.getServletContext().setAttribute("clock", clock);    // положить в общий контекст
    }
    public void contextDestroyed(ServletContextEvent e) { /* закрыть ресурсы */ }
}
// в сервлете: в init() достать getServletContext().getAttribute("clock")
```

Это твой **composition root** в веб-приложении (как `main` в консольном).

### Частые ошибки
- Изменяемые поля в сервлете.
- Смешать `javax.servlet` и `jakarta.servlet`. Tomcat 10+ — только `jakarta`.
- `jakarta.servlet-api` без `<scope>provided</scope>`: сервер уже содержит этот API, вторая копия в WAR конфликтует.
- Копипаста `setContentType`/`setCharacterEncoding` во всех сервлетах вместо фильтра.
- Нет `doPatch`: в `HttpServlet` его просто нет. Для PATCH придётся переопределить `service()` и направить запрос сам.

---

## Часть 2. JSON: Jackson

**Простыми словами.** Jackson — стандартная библиотека JSON ↔ Java-объекты (как `json` + pydantic).
Главный класс — `ObjectMapper`.

```java
record Book(String title, int year) {}

ObjectMapper mapper = new ObjectMapper();                         // создать ОДИН на приложение
String json = mapper.writeValueAsString(new Book("Dune", 1965));  // {"title":"Dune","year":1965}
Book b = mapper.readValue(json, Book.class);
mapper.writeValue(resp.getWriter(), List.of(b));                  // сразу в HTTP-ответ
```

`ObjectMapper` дорогой в создании и **потокобезопасный** после настройки: создай один и переиспользуй.
Для `BigDecimal` и дат есть настройки (`WRITE_BIGDECIMAL_AS_PLAIN`, модуль `jackson-datatype-jsr310`).

---

## Часть 3. JDBC с нуля

### Простыми словами

**JDBC** — стандартный низкоуровневый API Java для SQL-баз. Аналог Python DB-API (`sqlite3`, `psycopg`).
Для каждой БД нужен **драйвер** (зависимость Maven: `org.xerial:sqlite-jdbc`, `org.postgresql:postgresql`).

Три главных объекта:
- `Connection` — соединение с БД (как `conn = sqlite3.connect(...)`);
- `PreparedStatement` — SQL-запрос с параметрами `?` (как `cursor.execute(sql, params)`);
- `ResultSet` — результат, по которому идёшь курсором (как `for row in cursor`).

Все три — ресурсы, их **обязательно закрывать**, иначе соединения кончатся и приложение встанет.
Закрываем через try-with-resources:

```java
String sql = "SELECT id, title, year FROM books WHERE year > ?";
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setInt(1, 2000);                                        // параметры нумеруются с 1!
    try (ResultSet rs = ps.executeQuery()) {
        List<Book> result = new ArrayList<>();
        while (rs.next()) {                                    // сдвинуть курсор; false — строки кончились
            result.add(new Book(rs.getLong("id"), rs.getString("title"), rs.getInt("year")));
        }
        return result;
    }
} catch (SQLException e) {                                     // checked — придётся обработать
    throw new DataAccessException("Failed to load books", e);  // трансляция в своё unchecked, с причиной
}
```

Для `INSERT` с автоинкрементом получить новый id: `conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)`,
потом `ps.getGeneratedKeys()`.

### SQL-инъекции: почему только `PreparedStatement`

```java
// ОПАСНО: пользователь передаст name = "x' OR '1'='1" — и получит всех
String sql = "SELECT * FROM users WHERE name = '" + name + "'";

// БЕЗОПАСНО: параметр передаётся отдельно от текста запроса и не может стать частью SQL
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE name = ?");
ps.setString(1, name);
```

> 🎯 **Спросят на собесе:** `Statement` vs `PreparedStatement`?
> **Ответ:** `PreparedStatement` — параметризованный запрос: защищает от SQL-инъекций, т. к. параметры передаются отдельно
> от текста SQL, и позволяет БД переиспользовать план. `Statement` с конкатенацией строк — уязвимость.

### Транзакции

**Простыми словами.** Транзакция — группа операций «всё или ничего». Перевод денег: списать с одного счёта
и зачислить на другой. Если второе упало, первое должно откатиться.

По умолчанию JDBC в режиме **autocommit**: каждый запрос — отдельная транзакция. Для группы:

```java
try (Connection conn = dataSource.getConnection()) {
    conn.setAutoCommit(false);                   // начать транзакцию
    try {
        withdraw(conn, from, amount);            // оба метода используют ОДНО соединение
        deposit(conn, to, amount);
        conn.commit();                           // зафиксировать
    } catch (Exception e) {
        conn.rollback();                         // откатить всё
        throw e;
    }
}
```

**ACID** — свойства транзакции: Atomicity (всё или ничего), Consistency (из корректного состояния в корректное),
Isolation (параллельные транзакции не мешают друг другу — в какой степени, задаёт **уровень изоляции**),
Durability (зафиксированное не потеряется).

> 🎯 **Спросят на собесе:** Какие уровни изоляции транзакций бывают и какие аномалии они предотвращают?
> **Ответ:** READ UNCOMMITTED (грязное чтение возможно), READ COMMITTED (нет грязного чтения, возможно неповторяемое),
> REPEATABLE READ (нет неповторяемого, возможны фантомы по стандарту), SERIALIZABLE (как последовательное выполнение).
> Postgres по умолчанию — READ COMMITTED, MySQL InnoDB — REPEATABLE READ.

### Пул соединений

**Простыми словами.** Открыть соединение с БД дорого: сеть, аутентификация, десятки миллисекунд. Открывать новое на каждый
запрос — медленно, а при нагрузке соединения кончатся. **Пул** держит N открытых соединений и выдаёт их «в аренду».
`conn.close()` у соединения из пула **не закрывает** его, а возвращает в пул. Стандарт в Java — **HikariCP**.

```java
var config = new HikariConfig();
config.setJdbcUrl("jdbc:sqlite:library.db");
config.setMaximumPoolSize(10);
DataSource dataSource = new HikariDataSource(config);     // создать ОДИН раз при старте
```

> 🎯 **Спросят на собесе:** Зачем пул соединений?
> **Ответ:** Открытие соединения с БД дорогое (TCP, аутентификация). Пул держит готовые соединения и выдаёт их в аренду;
> `close()` на соединении из пула возвращает его в пул, а не закрывает.

### Ограничения БД — последняя линия обороны

`UNIQUE`, `FOREIGN KEY`, `NOT NULL`, `CHECK` — это правила, которые БД гарантирует **всегда**, даже при параллельных запросах.
Проверка в Java «сначала `SELECT`, есть ли такой email, потом `INSERT`» **не защищает** от дублей: два запроса могут
одновременно выполнить `SELECT`, оба ничего не найти, оба сделать `INSERT`. Правильно: `UNIQUE`-ограничение в БД + ловить
ошибку нарушения ограничения при `INSERT` и превращать её в понятный ответ.

⚠️ В **SQLite** внешние ключи по умолчанию **выключены**: нужно `PRAGMA foreign_keys = ON` для каждого соединения
(в URL: `jdbc:sqlite:file.db?foreign_keys=on`).

### Индексы

Индекс — отсортированная структура (обычно B-дерево) по колонке, позволяющая находить строки за O(log n) вместо полного
перебора таблицы. `UNIQUE` автоматически создаёт индекс. Индексы ускоряют чтение и замедляют запись. Отлично объяснено
на [Use The Index, Luke](https://use-the-index-luke.com/ru/sql/anatomy).

### Проблема N+1

Нужно показать 100 книг с авторами. Наивно: 1 запрос за книгами + по запросу за автором для **каждой** книги = 101 запрос.
Правильно: один запрос с `JOIN`. Такое встретится в каждом проекте, и в ORM (проект 4) особенно коварно.

> 🎯 **Спросят на собесе:** Что такое проблема N+1?
> **Ответ:** Сначала загружаем N записей одним запросом, затем для каждой делаем ещё по запросу за связанными данными:
> N+1 запросов вместо 1–2. Решается `JOIN`/батч-загрузкой (`WHERE id IN (...)`). В ORM — `JOIN FETCH`, `@EntityGraph`, batch size.

---

## Часть 4. Слоистая архитектура: Controller → Service → DAO

**Простыми словами.** Когда в одном сервлете и разбор HTTP, и бизнес-правила, и SQL, получается спагетти.
Классическое разделение (**MVC(S)**):

```
HTTP ──> Controller (сервлет)  разобрать запрос, провалидировать формат, вызвать сервис,
                               превратить результат/исключение в HTTP-ответ (код + JSON)
             │
             v
         Service               бизнес-правила: что можно, что нельзя, как считать.
                               Ничего не знает про HTTP.
             │
             v
         DAO                   только SQL: достать/сохранить. Ничего не знает про правила и HTTP.
             │
             v
            БД
```

- **DAO (Data Access Object)** — класс со всем SQL для одной сущности: `findById`, `findAll`, `save`.
- **DTO (Data Transfer Object)** — простой объект того, что уходит клиенту (JSON). Он может отличаться от строки таблицы:
  скрыть поля, добавить вложенные объекты, переименовать. Как pydantic-схема ответа в FastAPI.
- **Mapper** — код, перекладывающий данные «модель ↔ DTO».

**Ошибки идут снизу вверх исключениями.** DAO бросает `NotFound`/`Conflict`, сервис пропускает или бросает свои,
а контроллер (или один общий фильтр) превращает их в HTTP-коды. Одно место трансляции ошибок, а не `try/catch` в каждом сервлете.

**Singleton через `static` — антипаттерн.** Хочется сделать `CurrencyDao.getInstance()`. Не надо: создай объекты один раз
в `ServletContextListener` (composition root) и передай куда нужно.

> 🎯 **Спросят на собесе:** Почему Singleton называют антипаттерном?
> **Ответ:** Глобальное состояние, скрытые зависимости (не видно из конструктора), сложно подменить в тестах, проблемы с
> многопоточной ленивой инициализацией. «Один экземпляр на приложение» лучше обеспечивать контейнером/composition root.

---

## Часть 5. Сервер на Linux: минимум для деплоя

- **VPS** — арендованная виртуальная машина (Timeweb, Selectel, Hetzner, DigitalOcean...). Подключение: `ssh user@ip`.
- Установка: `sudo apt install openjdk-21-jre-headless` (или Temurin), Tomcat — скачать архив и распаковать в `/opt/tomcat`.
- **systemd** — менеджер служб Linux: описываешь сервис в `/etc/systemd/system/tomcat.service`, дальше
  `systemctl start|stop|status tomcat`, `journalctl -u tomcat` — логи. Служба сама поднимется после перезагрузки.
- Не запускать под `root`: создать пользователя `tomcat`.
- Файлы на сервер: `scp app.war user@ip:/opt/tomcat/webapps/`.
- Открытые порты: `ufw allow 8080`.

---

📚 **Читать:**
- [Jakarta Servlet 6.0 Specification](https://jakarta.ee/specifications/servlet/6.0/jakarta-servlet-spec-6.0): гл. 2 «The Servlet Interface» (жизненный цикл, многопоточность), гл. 6 «Filtering» — по диагонали, это первоисточник.
- [Baeldung: Introduction to Java Servlets](https://www.baeldung.com/intro-to-servlets), [Baeldung: Servlet Filters](https://www.baeldung.com/intercepting-filter-pattern-in-java).
- [Baeldung: Jackson ObjectMapper](https://www.baeldung.com/jackson-object-mapper-tutorial).
- CJ2, гл. 5 «Database Programming» (JDBC): соединения, `PreparedStatement`, транзакции — **целиком по диагонали**.
- [Oracle Tutorial: JDBC Basics](https://docs.oracle.com/javase/tutorial/jdbc/basics/) — разделы Using Prepared Statements, Using Transactions.
- HPJP, часть I «JDBC and Database Essentials»: главы про управление соединениями (connection pooling) и транзакции.
- DDIA, гл. 7 «Transactions» — первые разделы (ACID, уровни изоляции). Это одна из лучших глав книги.
- [Use The Index, Luke — «Анатомия индекса»](https://use-the-index-luke.com/ru/sql/anatomy).
- PoEAA: гл. 9 (Service Layer), гл. 10 (Table Data Gateway), гл. 14 (Front Controller), гл. 15 (Data Transfer Object).
- EJ 3 и EJ 5 (синглтоны vs DI), EJ 73 (трансляция исключений).
- [DigitalOcean: How To Install Apache Tomcat 10 on Ubuntu](https://www.digitalocean.com/community/tutorials) (найди по названию актуальную версию) — пример systemd-юнита.

---

## Упражнения и вехи

### Веха B3.1 — Сервлеты и Tomcat

**Что нужно сделать**
1. Скачать Tomcat 10.1, запустить локально (`bin/startup.sh`), открыть `http://localhost:8080`.
2. Maven-проект с `packaging=war` и `jakarta.servlet-api` (`provided`).
3. `HelloServlet` из части 1 + сервлет `/time`, отдающий текущее время **в JSON** через Jackson
   (`{"utc": "...", "zone": "..."}`, зона — из параметра `?zone=Europe/Moscow`, по умолчанию UTC; неверная зона → 400 с JSON-ошибкой).
4. Фильтр кодировки/`Content-Type` и фильтр-замер времени из части 1.
5. `ServletContextListener`, который создаёт `ObjectMapper` и `Clock` и кладёт в контекст; сервлеты берут их оттуда.
6. Эксперимент: сервлет с полем-счётчиком запросов (`int`) + нагрузка (`ab -n 10000 -c 50` или скрипт на Python с потоками).
   Покажи, что счётчик врёт, исправь.

**С чего начать.** С п. 1–3: добейся, чтобы `curl localhost:8080/<контекст>/hello?name=Ann` вернул текст.

**Как проверить себя**
- [ ] Деплой WAR в Tomcat работает (копированием в `webapps` или через IDEA Ultimate).
- [ ] Ни один сервлет не выставляет кодировку сам, это делает фильтр.
- [ ] `ObjectMapper` создан один раз.
- [ ] Можешь объяснить, почему счётчик в п. 6 врал и что именно исправило ситуацию.

### Веха B3.2 — JDBC

**Что нужно сделать** (обычное консольное приложение, не веб):
1. SQLite-база «библиотека»: таблицы `authors`, `books` (FK на автора, `UNIQUE` на ISBN), скрипт создания в ресурсах.
2. Класс доступа к данным: добавить автора (вернуть сгенерированный id), найти книги по году, добавить книгу.
   Все ресурсы в try-with-resources, `SQLException` → своё unchecked-исключение с причиной.
3. **Инъекция своими глазами.** Метод поиска по названию с конкатенацией; покажи тестом, что строка `' OR '1'='1`
   возвращает все книги. Исправь на `PreparedStatement`, тест теперь должен падать на уязвимой версии и проходить на исправленной.
4. **Транзакция.** Таблица `accounts`; перевод между счетами в транзакции; тест, что при ошибке во второй операции
   первая откатилась.
5. **N+1 своими глазами.** Вывести 50 книг с именами авторов двумя способами: N+1 и одним `JOIN`. Посчитай запросы
   (просто счётчиком в своём коде) и сравни.
6. **Уникальность.** Добавление книги с существующим ISBN → поймать нарушение ограничения и бросить своё
   `DuplicateIsbnException`. Никакого предварительного `SELECT`.
7. HikariCP вместо `DriverManager`.

**С чего начать.** С п. 1–2: создать таблицы и вставить одного автора. Проверь в DBeaver, что строка появилась.

**Как проверить себя**
- [ ] Нет ни одной конкатенации SQL с параметрами (кроме намеренно уязвимой в п. 3, помеченной комментарием).
- [ ] Все `Connection`/`Statement`/`ResultSet` закрываются.
- [ ] Внешние ключи в SQLite включены (проверь: вставка книги с несуществующим автором падает).
- [ ] Можешь объяснить, почему предварительный `SELECT` не защищает от дублей.

---

## Контрольные вопросы

1. Роли сервлет-контейнера и сервлета. Аналогия с Python и где она ломается.
2. Жизненный цикл сервлета; почему в сервлете нельзя хранить данные запроса.
3. Фильтр vs сервлет; `FilterChain`; какой это паттерн.
4. Что такое WAR и контекстный путь? Почему servlet-api — `provided`?
5. `Connection`, `PreparedStatement`, `ResultSet`: что это, как закрывать.
6. SQL-инъекция: механизм и защита.
7. Транзакция, autocommit, ACID, уровни изоляции.
8. Пул соединений: зачем, что делает `close()`.
9. Почему ограничения БД важнее проверок в Java? Race condition «SELECT перед INSERT».
10. Controller / Service / DAO / DTO: кто за что отвечает.
