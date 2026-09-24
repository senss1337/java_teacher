# Проект 3. Обмен валют

**ТЗ:** [zhukovsd — Обмен валют](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/)
REST API для валют и курсов: `GET/POST /currencies`, `GET /currency/{code}`, `GET/POST /exchangeRates`,
`GET/PATCH /exchangeRate/{pair}`, `GET /exchange?from=&to=&amount=` (прямой курс, обратный, кросс через USD).
**Фреймворки не используем**: голые Jakarta Servlets + JDBC + SQLite, деплой WAR в Tomcat на VPS.

Зачем без фреймворков, если есть опыт с FastAPI/Django: чтобы понимать, **что Spring делает за тебя**. На собесе спросят
«как работает `DispatcherServlet`» — и ответ будет «это сервлет, который я писал сам в проекте 3».

## Цели обучения

- Жизненный цикл сервлета, контейнер (Tomcat), `web.xml`/аннотации, фильтры, WAR.
- JDBC: `Connection`, `PreparedStatement`, `ResultSet`, транзакции, пул соединений.
- Слоистая архитектура MVC(S): Controller (сервлет) → Service → DAO, DTO и маппинг.
- Обработка ошибок через иерархию исключений и **одну** точку трансляции в HTTP-ответы.
- `BigDecimal`: масштаб, округление, сравнение.
- Первый деплой на Linux-сервер.

## Что изучить до старта

| Тема | Обрати внимание |
|------|-----------------|
| Jakarta Servlet 6: `HttpServlet`, `doGet/doPost`, `@WebServlet`, `ServletContextListener`, `Filter` | **`doPatch` нет** — разберись, как обработать PATCH (метод `service`). Сервлет — **синглтон** на весь контейнер, обрабатывает запросы из многих потоков: какие поля в нём допустимы? |
| Tomcat 10.1: установка, `webapps`, manager app, контекстный путь | `javax.servlet` (Tomcat 9) vs `jakarta.servlet` (Tomcat 10+) — не смешивать |
| JDBC: `DriverManager`, `DataSource`, `PreparedStatement`, `ResultSet`, `getGeneratedKeys`, транзакции (`setAutoCommit(false)`) | try-with-resources для **всех** трёх ресурсов |
| HikariCP | зачем пул, что такое `maximumPoolSize` |
| SQLite: типы, `UNIQUE`, `FOREIGN KEY` (**включить `PRAGMA foreign_keys`!**), где лежит файл БД в WAR | путь к БД в ресурсах vs файловая система сервера |
| Jackson: `ObjectMapper` (один на приложение!), сериализация `BigDecimal`, records | `ObjectMapper` потокобезопасен и дорог в создании |
| `BigDecimal`: `scale`, `setScale`, `RoundingMode`, `divide` с `MathContext`, `compareTo` vs `equals`, `stripTrailingZeros` | `new BigDecimal("1.0").equals(new BigDecimal("1.00")) == false` |
| REST: коды ответов, идемпотентность методов, `x-www-form-urlencoded` | PATCH с form-urlencoded: `getParameter` для PATCH не работает в Tomcat — почему и что делать? |
| Фаулер, PoEAA: Table Data Gateway / DAO, Data Transfer Object, Service Layer | — |

📚 **Читать:**
- [Jakarta Servlet 6.0 Specification](https://jakarta.ee/specifications/servlet/6.0/jakarta-servlet-spec-6.0): гл. 2 «The Servlet Interface» (жизненный цикл, многопоточность), гл. 6 «Filtering» — по диагонали, это первоисточник.
- [Baeldung: Introduction to Java Servlets](https://www.baeldung.com/intro-to-servlets), [Baeldung: Servlet Filters](https://www.baeldung.com/intercepting-filter-pattern-in-java).
- CJ2, гл. 5 «Database Programming» (JDBC): соединения, `PreparedStatement`, транзакции, метаданные — **целиком по диагонали**.
- [Oracle Tutorial: JDBC Basics](https://docs.oracle.com/javase/tutorial/jdbc/basics/) — разделы Using Prepared Statements, Using Transactions.
- HPJP, часть I «JDBC and Database Essentials»: главы про управление соединениями (connection pooling) и транзакции.
- PoEAA: гл. 9 «Domain Logic Patterns» (Transaction Script, Service Layer), гл. 10 «Data Source Architectural Patterns» (Table Data Gateway), гл. 14 «Web Presentation Patterns» (Front Controller, Page Controller), гл. 15 «Distribution Patterns» (Data Transfer Object), гл. 18 «Base Patterns» (Gateway, Money).
- EJ 60 (не используйте `double` для точных вычислений), EJ 73 (трансляция исключений), EJ 3 и EJ 5 (синглтоны vs DI).
- DDIA, гл. 7 «Transactions» — первые разделы (ACID, уровни изоляции). Это одна из лучших глав книги.
- [Use The Index, Luke — «Анатомия индекса»](https://use-the-index-luke.com/ru/sql/anatomy) — как работают индексы.
- [Javadoc `BigDecimal`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/math/BigDecimal.html) — вводная часть про scale, precision и rounding.

> 🎯 **Спросят на собесе:** Жизненный цикл сервлета?
> **Ответ:** Контейнер загружает класс, создаёт **один** экземпляр, вызывает `init()`; на каждый запрос — `service()`
> (диспетчеризует в `doGet/doPost/...`) в потоке из пула контейнера; при остановке — `destroy()`.
> **Типичная ошибка:** «на каждый запрос создаётся новый сервлет». Отсюда баги с полями-состоянием.

> 🎯 **Спросят на собесе:** `Statement` vs `PreparedStatement`?
> **Ответ:** `PreparedStatement` — параметризованный запрос: защищает от SQL-инъекций, т. к. параметры передаются отдельно
> от текста SQL, и позволяет БД переиспользовать план. `Statement` с конкатенацией строк — уязвимость.

> 🎯 **Спросят на собесе:** Зачем пул соединений?
> **Ответ:** Открытие соединения с БД дорогое (TCP, аутентификация). Пул держит готовые соединения и выдаёт их в аренду;
> `close()` на соединении из пула возвращает его в пул, а не закрывает.

## Паттерны и принципы в фокусе

- **Layered architecture / MVC(S).** Сервлет: разобрать запрос, провалидировать, вызвать сервис, сериализовать.
  Сервис: бизнес-логика (как искать курс для обмена — прямой/обратный/кросс). DAO: только SQL.
- **DAO.** Интерфейс + JDBC-реализация. Нужен ли интерфейс, если реализация одна? Сформулируй аргументы за и против.
- **DTO + Mapper.** Что уходит в JSON — не то же самое, что строка таблицы. Где живёт маппинг?
- **Front Controller / Filter (Chain of Responsibility).** Общие вещи (кодировка, `Content-Type`, CORS, обработка
  исключений) — в фильтрах, а не копипастой в каждом сервлете.
- **Иерархия исключений.** DAO/сервис кидают доменные исключения (не найдено, конфликт), **одно место** превращает их в
  HTTP-коды + `{"message": ...}`.
- **Ручной DI / Composition Root.** Где создаются DAO, сервисы, `DataSource`, `ObjectMapper`? Как сервлет их получает?
  Подсказка: `ServletContextListener` + атрибуты контекста. Синглтоны через `static getInstance()` — антипаттерн
  (почему? как тестировать?).

> 🎯 **Спросят на собесе:** Почему Singleton называют антипаттерном?
> **Ответ:** Глобальное состояние, скрытые зависимости (не видно из конструктора), сложно подменить в тестах, проблемы с
> многопоточной ленивой инициализацией. «Один экземпляр на приложение» лучше обеспечивать контейнером/composition root.

## Вехи

### Веха 3.1 — Каркас: Maven WAR + Tomcat + первый сервлет
Критерии приёмки:
- `packaging=war`, `jakarta.servlet-api` в scope `provided` (объясни, почему).
- Приложение деплоится в локальный Tomcat, тестовый эндпоинт отвечает JSON.
- Фильтр, выставляющий кодировку и `Content-Type` для всех ответов.
- Composition root в `ServletContextListener`.

### Веха 3.2 — Схема БД и DAO
Вопросы до кода:
- Какие ограничения на уровне БД? (`UNIQUE` на `Code`, `UNIQUE(BaseCurrencyId, TargetCurrencyId)`, `FOREIGN KEY`,
  `CHECK`?) Почему их нельзя заменить проверками в Java?
- Как хранить `Rate` в SQLite, чтобы не потерять точность? (подсказка: в SQLite нет настоящего `DECIMAL` — изучи type affinity)
- Как получить список курсов **вместе с** валютами одним запросом, без N+1?

Критерии приёмки:
- DDL-скрипт + начальные данные (5+ валют, 5+ курсов) в репозитории; БД создаётся/проверяется при старте.
- DAO на `PreparedStatement`, все ресурсы в try-with-resources, `SQLException` транслируется в свои исключения.
- Пул соединений (HikariCP).
- Список курсов — **один** SQL-запрос с `JOIN`.

> 🎯 **Спросят на собесе:** Что такое проблема N+1?
> **Ответ:** Сначала загружаем N записей одним запросом, затем для каждой делаем ещё по запросу за связанными данными:
> N+1 запросов вместо 1–2. Решается `JOIN`/батч-загрузкой (`WHERE id IN (...)`). В ORM — `JOIN FETCH`, `@EntityGraph`, batch size.

### Веха 3.3 — Валюты: `GET /currencies`, `GET /currency/{code}`, `POST /currencies`
Вопросы до кода:
- Как проверить уникальность кода валюты? `SELECT` перед `INSERT` — чем опасно? (race condition!)
- Кто валидирует, что `code` — это 3 латинские буквы? Какой ответ при нарушении?
- Как разобрать путь `/currency/EUR` в сервлете?

Критерии приёмки:
- Коды ответов строго по ТЗ (200, 201, 400, 404, 409, 500) и тело ошибки `{"message": ...}`.
- Конфликт уникальности ловится через ограничение БД, а не через предварительный `SELECT`.
- Сервлет не содержит SQL и не собирает JSON руками.

### Веха 3.4 — Курсы: `GET/POST /exchangeRates`, `GET/PATCH /exchangeRate/{pair}`
Критерии приёмки:
- PATCH работает с `x-www-form-urlencoded` телом.
- Пара разбирается корректно, валидация кодов, 404 если валюты или пары нет.
- `BigDecimal` на всём пути: из запроса, в БД, в JSON. Нигде нет `double`.

### Веха 3.5 — Обмен: `GET /exchange`
Вопросы до кода:
- Три сценария (AB, BA, USD-A + USD-B). Где эта логика? Как сделать так, чтобы четвёртый сценарий добавлялся без
  переписывания трёх? (подсказка: Chain of Responsibility / список стратегий)
- Какую точность держать в промежуточных вычислениях, на каком шаге округлять и каким `RoundingMode`?
- Что возвращать в `rate` для обратного курса — сколько знаков?

Критерии приёмки:
- Все три сценария + ошибка, если курс не вычисляется.
- `convertedAmount` округлён до 2 знаков, режим округления осознанный (объясни выбор).
- Юнит-тесты сервиса обмена с подменой DAO (фейк или Mockito) на все сценарии, включая деление с бесконечной дробью.

> ⚠️ **Типичная ловушка:** `BigDecimal.ONE.divide(rate)` без `MathContext`/scale кидает `ArithmeticException` на
> непериодическом результате (например, 1/3).

### Веха 3.6 — Обработка ошибок и полировка
Критерии приёмки:
- Одно место трансляции исключений → HTTP (фильтр или базовый сервлет). Неожиданные ошибки → 500 с общим сообщением
  (stack trace **не** уходит клиенту, но уходит в лог).
- Логирование через SLF4J + Logback, не `System.out`.
- Проверено [тестовым фронтендом](https://github.com/zhukovsd/currency-exchange-frontend) из ТЗ. Если раздаёшь его отдельно через Docker/nginx, понадобится CORS — сделай это фильтром.
- Интеграционные тесты DAO на временной БД (SQLite in-memory или временный файл).

### Веха 3.7 — Деплой
Критерии приёмки:
- VPS (любой провайдер), JRE 21 + Tomcat 10.1, WAR задеплоен, API доступно по `http://ip:8080/<context>/currencies`.
- Tomcat запущен как systemd-сервис, не от root. Файл SQLite лежит там, где у процесса есть права на запись.
- `README.md`: как собрать и задеплоить; ссылка на рабочий инстанс.
- Пройден [чеклист из ТЗ](https://zhukovsd.github.io/java-backend-learning-course/projects/currency-exchange/#чеклист-для-самопроверки).

## Челленджи

- ★ **MapStruct** для маппинга entity ↔ DTO. Что он генерирует? Сравни с ручным маппером.
- ★ **Валидация** через Jakarta Bean Validation (Hibernate Validator) без Spring: собери `Validator` сам.
- ★★ **Свой мини-роутер.** Один `FrontControllerServlet` на `/*`, маршрутизация по методу+шаблону пути
  на обработчики. Ты только что написал упрощённый `DispatcherServlet`. Объясни в README, чем он отличается от Spring-овского.
- ★★ **Транзакции.** Эндпоинт массового импорта курсов (много пар за раз): или всё, или ничего.
- ★★★ **Кросс-курс через любую валюту** (не только USD): поиск пути в графе валют (BFS из проекта 2!),
  без циклов, кратчайшая цепочка.
- ★★★ **Postgres вместо SQLite** за тем же DAO-интерфейсом, выбор реализации конфигом. Сколько кода изменилось?

## За что будет 🔴

- `double`/`float` для денег и курсов; `new BigDecimal(double)`.
- Конкатенация SQL; незакрытые `Connection`/`Statement`/`ResultSet`.
- SQL в сервлетах; бизнес-логика обмена в сервлете; сервлет с изменяемыми полями-состоянием.
- `SELECT` перед `INSERT` для проверки уникальности без ограничения в БД.
- Нет `UNIQUE`/`FOREIGN KEY`, выключенные внешние ключи в SQLite.
- Копипаста `setContentType`/`setCharacterEncoding` во всех сервлетах.
- Stack trace в ответе клиенту; `catch (Exception e) { e.printStackTrace(); }`.
- `static getInstance()`-синглтоны для сервисов и DAO; новый `ObjectMapper` на каждый запрос.
- Расхождение кодов/формата ответов с ТЗ.

## Контрольные вопросы (защита)

1. Жизненный цикл сервлета. Почему сервлет не должен иметь изменяемого состояния?
2. Путь HTTP-запроса от Tomcat до твоего DAO и обратно: какие объекты, какие потоки?
3. Фильтр vs сервлет. Как устроена `FilterChain`? Какой паттерн?
4. `PreparedStatement` и SQL-инъекции. Как работает пул соединений, что делает `close()`?
5. Что такое транзакция, ACID, `autoCommit`? Какой уровень изоляции по умолчанию в SQLite и Postgres?
6. Почему `SELECT` перед `INSERT` — race condition? Как правильно?
7. `BigDecimal`: `scale` vs `precision`, `equals` vs `compareTo`, `HALF_UP` vs `HALF_EVEN` (банковское округление).
8. Зачем DTO, если можно сериализовать модель напрямую?
9. PUT vs PATCH vs POST: идемпотентность.
10. Что такое WAR, чем отличается от JAR? Где в WAR лежат зависимости?

> 🎯 **Спросят на собесе:** Какие уровни изоляции транзакций бывают и какие аномалии они предотвращают?
> **Ответ:** READ UNCOMMITTED (грязное чтение возможно), READ COMMITTED (нет грязного чтения, возможно неповторяемое),
> REPEATABLE READ (нет неповторяемого, возможны фантомы по стандарту), SERIALIZABLE (как последовательное выполнение).
> Postgres по умолчанию — READ COMMITTED, MySQL InnoDB — REPEATABLE READ.
