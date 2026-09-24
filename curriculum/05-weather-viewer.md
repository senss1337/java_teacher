# Проект 5. Погода

**ТЗ:** проекта больше нет на сайте курса. Полный текст из истории репозитория лежит в
[`specs/05-weather-viewer-tz.md`](specs/05-weather-viewer-tz.md).
Коротко: серверный рендеринг (Thymeleaf), регистрация/авторизация на **самописных сессиях и куках**
(без `JSESSIONID`, без Spring Security), поиск локаций и погоды через [OpenWeatherMap API](https://openweathermap.org/api),
список избранных локаций пользователя с текущей погодой. Таблицы `Users`, `Locations`, `Sessions`,
миграции Flyway/Liquibase, интеграционные тесты сервисов и HTTP-клиента с моками, деплой WAR в Tomcat.
Готовая вёрстка: [weather-viewer-html-layouts](https://github.com/zhukovsd/weather-viewer-html-layouts).

## Цели обучения

- Понять **изнутри**, как работают сессии и куки, чтобы в проекте 6 Spring Security перестал быть магией.
- Безопасно хранить пароли (BCrypt), понимать атаки на сессии.
- Интегрироваться с внешним HTTP API: клиент, таймауты, десериализация, ошибки, изоляция за интерфейсом.
- Spring глубже: профили, конфигурация через properties, `@ControllerAdvice`, интерсепторы/фильтры, Spring Test.
- Миграции схемы БД.
- Интеграционные тесты: отдельная БД, моки внешнего API.

## Что изучить до старта

| Тема | Обрати внимание |
|------|-----------------|
| Cookies: атрибуты `HttpOnly`, `Secure`, `SameSite`, `Max-Age`, `Path` | какие из них защищают от XSS, а какие от CSRF? |
| Сессии на сервере: session ID, хранение, истечение, инвалидация при logout | ID сессии — `UUID.randomUUID()` достаточно случаен? (разберись, какой там генератор) |
| Хеширование паролей: BCrypt (jBCrypt или `spring-security-crypto` — отдельная библиотека **без** Security), соль, cost factor | почему не SHA-256? |
| Session fixation, CSRF, XSS (экранирование в Thymeleaf `th:text` vs `th:utext`) | OWASP Top 10 — пробеги глазами |
| Thymeleaf: выражения, `th:each`, `th:if`, формы (`th:object`, `th:field`), фрагменты (`th:replace`) для общего заголовка | Template View (PoEAA) |
| Spring: профили (`@Profile`, `spring.profiles.active`), `@PropertySource`, `Environment`, `@Value` | секрет API-ключа — через env, не в git |
| Spring `HandlerInterceptor` или `Filter` для проверки сессии; `HandlerMethodArgumentResolver` для инъекции текущего пользователя в контроллер | не проверять сессию руками в каждом методе контроллера |
| HTTP-клиент: `RestClient` (Spring 6.1+) или `java.net.http.HttpClient`; таймауты, `URI` builder, URL-encoding параметров | город с пробелом или кириллицей в названии |
| Jackson: `@JsonProperty`, `@JsonIgnoreProperties(ignoreUnknown = true)`, records | ответ OpenWeather большой — маппить только нужное |
| Flyway: версионированные миграции `V1__...sql`, почему нельзя править применённую миграцию | `validate` vs `ddl-auto` Hibernate |
| Spring Test: `@SpringJUnitConfig`, `@ActiveProfiles`, транзакционные тесты с откатом; Mockito; MockRestServiceServer или WireMock | test pyramid |
| `@Scheduled` (опционально) | чистка истёкших сессий |

📚 **Читать:**
- [MDN: HTTP cookies](https://developer.mozilla.org/ru/docs/Web/HTTP/Cookies) и [Set-Cookie](https://developer.mozilla.org/ru/docs/Web/HTTP/Headers/Set-Cookie) — все атрибуты.
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) и [Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — **обязательно**, это чеклист для вехи 5.2.
- [OWASP Top 10](https://owasp.org/Top10/) — пробеги заголовки и описания.
- [Thymeleaf + Spring tutorial](https://www.thymeleaf.org/doc/tutorials/3.1/thymeleafspring.html) — разделы про формы и фрагменты.
- [Spring Framework Reference: Environment Abstraction / Profiles](https://docs.spring.io/spring-framework/reference/core/beans/environment.html), [Interception (HandlerInterceptor)](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/interceptors.html), [REST Clients — RestClient](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html).
- [Spring Framework Reference: Testing](https://docs.spring.io/spring-framework/reference/testing.html) — разделы Spring TestContext Framework и MockRestServiceServer.
- [Flyway: Concepts — Migrations](https://documentation.red-gate.com/fd/migrations-271585107.html).
- PoEAA, гл. 14: Template View; гл. 18: Gateway, Service Stub.
- UTPP, гл. 5 «Mocks and test fragility», гл. 7–8 про интеграционные тесты (что мокать, а что нет) — **главное чтение проекта**.
- SIA, гл. 2 (Thymeleaf, формы, валидация).

> 🎯 **Спросят на собесе:** Почему пароли хешируют BCrypt, а не SHA-256?
> **Ответ:** SHA-256 быстрый — это плохо для паролей: перебор миллиардов вариантов в секунду на GPU. BCrypt/scrypt/Argon2
> специально медленные (настраиваемый cost), содержат соль (защита от rainbow tables) и хранят её в самом хеше.

> 🎯 **Спросят на собесе:** Что такое session fixation и как защититься?
> **Ответ:** Атакующий навязывает жертве известный ему ID сессии до логина; после логина жертвы сессия становится
> аутентифицированной. Защита — выдавать **новый** ID сессии при аутентификации.

> 🎯 **Спросят на собесе:** Что делают `HttpOnly` и `SameSite`?
> **Ответ:** `HttpOnly` — кука недоступна из JavaScript (снижает ущерб от XSS). `SameSite=Lax/Strict` — кука не
> отправляется в кросс-сайтовых запросах (защита от CSRF).

## Паттерны и принципы в фокусе

- **Adapter / Gateway** для внешнего API. Приложение зависит от **своего** интерфейса «сервис погоды», возвращающего
  **свои** модели. DTO OpenWeather не выходят за пределы адаптера. Смена провайдера погоды = новый адаптер.
- **Anti-Corruption Layer** (DDD) — то же с другой стороны: чужая модель не проникает в твой домен.
- **Interceptor / Filter** для аутентификации — сквозная функциональность отдельно от бизнес-логики.
- **Template View** (Thymeleaf) + **PRG** (Post/Redirect/Get) после форм.
- **Configuration by profile** — dev/test/prod отличаются конфигом, а не кодом.

## Вехи

### Веха 5.1 — Каркас: Spring MVC + Thymeleaf + Flyway + Postgres
Критерии приёмки:
- Spring MVC (без Boot, как в ТЗ), WAR, Thymeleaf-вьюхи из вёрстки, общий заголовок через фрагменты.
- Миграции Flyway создают `users`, `locations`, `sessions` с индексами и ограничениями
  (уникальный логин, FK, индекс на `sessions.expires_at` — зачем?). Hibernate схему **не** генерирует (`validate`).
- Профили `dev`/`test`, конфиги в `.properties`, секреты через переменные окружения.

### Веха 5.2 — Регистрация, вход, выход, сессии
Вопросы до кода:
- Что хранится в куке, а что в БД? Что если кука подделана/устарела?
- Где проверяется «пользователь авторизован» для защищённых страниц — один раз для всех?
- Как контроллер получает текущего пользователя, не доставая куку руками?
- Регистрация: гонка «логин занят» — снова через ограничение БД, как в проекте 3?
- Что делать с истёкшими сессиями в БД?

Критерии приёмки:
- BCrypt, кука с `HttpOnly` + `SameSite`, срок жизни сессии из конфига, logout удаляет сессию.
- Своя кука (не `JSESSIONID`), контейнерную `HttpSession` не используем.
- Проверка сессии — в интерсепторе/фильтре; текущий пользователь — через аргумент-резолвер или атрибут запроса.
- Ошибки форм (логин занят, пароли не совпадают, неверные данные) отображаются на странице, PRG после успеха.

### Веха 5.3 — Интеграционные тесты пользователей и сессий
Критерии приёмки:
- Отдельная БД или схема для тестов, чистится перед каждым тестом (подумай: `@Transactional` на тесте с откатом —
  когда это **не** работает?).
- Кейсы из ТЗ: регистрация создаёт запись, дубликат логина → ожидаемое исключение, истёкшая сессия не авторизует.
- Время в тестах управляется через `Clock`, а не `Thread.sleep`.

### Веха 5.4 — Клиент OpenWeather
Вопросы до кода:
- Какой интерфейс у твоего сервиса погоды? Какие модели он возвращает? (Не DTO OpenWeather!)
- Как обрабатываешь 401 (неверный ключ), 404, 429 (лимит), 5xx, таймаут? Какие исключения наружу?
- Температура в Кельвинах или в Цельсиях — где конвертируешь?

Критерии приёмки:
- Поиск локаций по названию (Geocoding API), погода по координатам. Таймауты настроены.
- API-ключ только из окружения.
- Тесты с моком HTTP (MockRestServiceServer или WireMock): успешный разбор, 4xx/5xx → ожидаемое исключение.
  Ни одного реального запроса в тестах.

### Веха 5.5 — Локации: поиск, добавление, удаление, главная
Вопросы до кода:
- У пользователя 10 локаций — 10 запросов к API на каждое открытие главной. Последовательно или параллельно?
  Что с лимитом 60 запросов в минуту? (подсказка: кеш)
- Удаление: как не дать удалить **чужую** локацию подменой ID в форме?
- Координаты — `BigDecimal` или `double`? Сравнение локаций при добавлении дубликата?

Критерии приёмки:
- Функционал по ТЗ: поиск → добавить → главная с карточками → удалить.
- Удаление/просмотр только своих локаций (проверка владельца на уровне запроса к БД, не только UI).
- Нет N+1 при загрузке локаций пользователя.

### Веха 5.6 — Деплой и финал
Критерии приёмки:
- WAR в Tomcat на VPS, Postgres, ключ API через переменные окружения systemd-сервиса.
- Сверка с [чеклистом из ТЗ](specs/05-weather-viewer-tz.md) (раздел «Чеклист для самопроверки», после работающей версии).

## Челленджи

- ★ **Кеш погоды** (Caffeine) с TTL 5–10 минут. Как инвалидировать? Как тестировать TTL без `sleep`?
- ★ **Чистка сессий** по расписанию `@Scheduled`. Что если два инстанса приложения? (ответь словами)
- ★★ **Параллельные запросы** погоды для главной страницы: `CompletableFuture` + свой `Executor` с ограничением.
  Почему нельзя использовать `ForkJoinPool.commonPool()` для I/O? А что изменят **virtual threads** (Java 21)?
- ★★ **Resilience**: retry с backoff на 5xx, circuit breaker (Resilience4j) — и деградация UI при недоступном API.
- ★★★ **Remember-me** со скользящим продлением сессии и ротацией ID. Опиши модель угроз в README.

## За что будет 🔴

- Пароли в открытом виде или простым хешем; креды или API-ключ в репозитории.
- Проверка сессии копипастой в каждом контроллере; бизнес-логика в контроллерах.
- Кука без `HttpOnly`; сессия не удаляется при logout; ID сессии предсказуем.
- DTO OpenWeather в контроллерах/шаблонах; HTTP-клиент без таймаутов; `catch` ошибок API с возвратом `null`.
- Возможность удалить/увидеть чужую локацию.
- `th:utext` с пользовательскими данными (XSS).
- Hibernate `ddl-auto=update` вместо миграций; правка уже применённой миграции.
- Реальные HTTP-запросы в тестах; тесты, зависящие от порядка или от общей грязной БД.

## Контрольные вопросы (защита)

1. Как работает твоя аутентификация: от POST формы логина до рендера главной при следующем запросе?
2. Куки: атрибуты и от чего они защищают. Сессии vs JWT: плюсы и минусы.
3. BCrypt: что внутри хеша, что такое cost factor и соль?
4. XSS, CSRF, session fixation — что это и как защищён твой проект?
5. Filter vs HandlerInterceptor vs AOP: в чём разница, что где выполняется?
6. Как изолирован OpenWeather от остального кода? Что изменится при смене провайдера?
7. Юнит-тест vs интеграционный: где граница в твоём проекте? Mock vs Stub vs Fake.
8. Flyway: как работает, где хранит историю, почему нельзя менять применённые миграции?
9. Spring-профили: как устроена конфигурация dev/test/prod?
10. PRG: зачем редирект после POST?

> 🎯 **Спросят на собесе:** Mock vs Stub vs Fake?
> **Ответ:** Stub возвращает заготовленные ответы. Mock — ещё и проверяет взаимодействия (какие методы вызваны и с чем).
> Fake — рабочая упрощённая реализация (in-memory репозиторий). Мокай то, что тебе не принадлежит, только на границах.
