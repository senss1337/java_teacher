# Проект 7. Планировщик задач

**ТЗ:** [zhukovsd — Планировщик задач](https://zhukovsd.github.io/java-backend-learning-course/projects/task-tracker/)
Четыре сервиса + инфраструктура:
- **backend**: REST API, пользователи, задачи, JWT, Spring Security;
- **scheduler**: раз в сутки собирает по каждому пользователю сделанные и оставшиеся задачи и оркестрирует отчёт;
- **summarization**: LLM по HTTP (без SDK), генерирует текст отчёта, общается через Kafka в стиле RPC;
- **email-sender**: читает `EMAIL_SENDING_TASKS` из Kafka и шлёт письма через SMTP.

Плюс Postgres, Kafka, Docker Compose, Dockerfile на каждый сервис, CI/CD на GitHub Actions (сборка и push образов), деплой.
Часть API проектируешь сам.

## Цели обучения

- Микросервисная декомпозиция: границы сервисов, владение данными, контракты сообщений.
- Kafka: топики, партиции, consumer groups, offset, семантики доставки, сериализация.
- Надёжность в распределённой системе: идемпотентность, повторы, dead letter, transactional outbox.
- Stateless-аутентификация на JWT внутри Spring Security.
- Интеграция с LLM по голому HTTP: промпт, формат ответа, таймауты, ошибки, стоимость.
- Dockerfile (multi-stage), CI/CD, деплой стека.

## Что изучить до старта

| Тема | Обрати внимание |
|------|-----------------|
| Микросервисы: зачем и когда **не** нужны, database-per-service, синхронная vs асинхронная интеграция | проект намеренно маленький для микросервисов, так что их цену ты почувствуешь |
| Kafka: broker, topic, partition, offset, consumer group, ключ сообщения и порядок, retention; KRaft (без ZooKeeper) | порядок гарантирован только внутри партиции |
| Семантики доставки: at-most-once, at-least-once, exactly-once; commit offset до/после обработки | at-least-once ⇒ консьюмер обязан быть **идемпотентным** |
| Spring Kafka: `KafkaTemplate`, `@KafkaListener`, `JsonSerializer`/`JsonDeserializer` и trusted packages, `DefaultErrorHandler`, DLT; `ReplyingKafkaTemplate` для request-reply | — |
| [Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html) | «записал в БД и отправил в Kafka» — две системы, нет общей транзакции |
| JWT: структура (header.payload.signature), подпись HS256 vs RS256, `exp`, почему payload не секретен, отзыв токенов | хранение на клиенте, XSS |
| Spring Security + JWT: свой фильтр (`OncePerRequestFilter`) или Resource Server (`oauth2-resource-server` с `JwtDecoder`) | stateless: `SessionCreationPolicy.STATELESS` |
| Spring `@Scheduled`, cron, ShedLock (почему при 2 инстансах задача выполнится дважды) | часовые пояса: «полночь» — чья? |
| Spring Mail: `JavaMailSender`, SMTP, шаблоны писем | секреты SMTP |
| LLM HTTP API (OpenAI-совместимый chat completions): system/user сообщения, `temperature`, лимиты токенов, structured output, таймауты, 429 | промпт-инъекции из заголовков задач пользователя |
| Dockerfile: multi-stage, слои и кеш, `eclipse-temurin:21-jre`, non-root, layered jar | размер образа |
| GitHub Actions: workflow, jobs, matrix, secrets, `docker/build-push-action` | секреты Docker Hub |
| Maven multi-module (если монорепо): parent POM, `<modules>`, общий модуль контрактов | или отдельные репозитории — выбери и обоснуй |

📚 **Читать:**
- MSP: гл. 1 «Escaping monolithic hell», гл. 2 «Decomposition strategies», гл. 3 «Interprocess communication in a microservice architecture» (**transactional outbox, идемпотентность, request/reply — здесь**), гл. 11 про безопасность (JWT между сервисами). Или онлайн: [microservices.io/patterns](https://microservices.io/patterns/).
- KDG: гл. 1 «Meet Kafka», гл. 3 «Kafka Producers», гл. 4 «Kafka Consumers» (consumer groups, commit offset'ов), гл. 7 «Reliable Data Delivery», гл. 8 «Exactly-Once Semantics» — по диагонали.
- DDIA, гл. 11 «Stream Processing» (первая половина: брокеры, логи, семантики доставки) и гл. 12 «The Future of Data Systems» (раздел про end-to-end идемпотентность).
- [Spring for Apache Kafka Reference](https://docs.spring.io/spring-kafka/reference/): разделы Receiving Messages, Error Handling / DLT, Request/Reply (`ReplyingKafkaTemplate`), Serialization.
- SIA, гл. 9 «Sending messages asynchronously» (раздел про Kafka).
- SSIA: главы про OAuth2 Resource Server и JWT. [RFC 7519 (JWT)](https://datatracker.ietf.org/doc/html/rfc7519) — разделы 1–4, [jwt.io/introduction](https://jwt.io/introduction). [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html).
- [Spring Framework Reference: Task Scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html), [ShedLock README](https://github.com/lukas-krecan/ShedLock).
- [OpenAI API: Chat Completions](https://platform.openai.com/docs/api-reference/chat) (или аналог выбранного провайдера), [OWASP LLM Top 10: Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/).
- [Docker docs: Multi-stage builds](https://docs.docker.com/build/building/multi-stage/), [Spring Boot: Efficient container images](https://docs.spring.io/spring-boot/reference/packaging/container-images/efficient-images.html).
- [GitHub Actions: Quickstart](https://docs.github.com/en/actions/writing-workflows/quickstart), [Publishing Docker images](https://docs.github.com/en/actions/use-cases-and-examples/publishing-packages/publishing-docker-images).
- [The Twelve-Factor App](https://12factor.net/ru/) — прочитать целиком, это 20 минут.

> 🎯 **Спросят на собесе:** Как Kafka гарантирует порядок сообщений?
> **Ответ:** Только внутри партиции. Сообщения с одинаковым ключом попадают в одну партицию (hash ключа), поэтому для
> упорядоченной обработки по сущности (пользователь, заказ) используют её ID как ключ. Между партициями порядка нет.

> 🎯 **Спросят на собесе:** Что такое consumer group?
> **Ответ:** Набор консьюмеров, делящих партиции топика: каждая партиция читается ровно одним консьюмером группы.
> Разные группы читают топик независимо (каждая со своими offset'ами). Консьюмеров больше, чем партиций, — лишние простаивают.

> 🎯 **Спросят на собесе:** Как надёжно записать в БД и отправить событие в Kafka?
> **Ответ:** Transactional Outbox: в той же транзакции, что и бизнес-изменение, пишем событие в таблицу `outbox`;
> отдельный процесс (поллер или CDC — Debezium) читает outbox и публикует в Kafka, помечая отправленные.
> Получается at-least-once, поэтому консьюмеры идемпотентны.
> **Типичная ошибка:** «отправить в Kafka внутри `@Transactional`» — если транзакция откатится после отправки, событие уже ушло.

> 🎯 **Спросят на собесе:** JWT vs сессии?
> **Ответ:** JWT — stateless, сервер не хранит состояние, легко масштабировать, подходит для межсервисного взаимодействия;
> но токен нельзя отозвать до `exp` без чёрного списка (т. е. без состояния), а payload виден всем.
> Сессии — легко отозвать, но нужен общий стор при масштабировании.

## Паттерны и принципы в фокусе

- **Database per service** и владелец схемы — кто хозяин миграций, кто только читает? (ТЗ прямо просит принять решение.)
  Scheduler читает пользователей и задачи — через БД backend-а или через API/события? Обоснуй компромисс.
- **Event-driven / Message contracts.** Контракт сообщения — это API: версионирование, обратная совместимость, общий модуль DTO или
  дублирование? (Shared library — источник связности.)
- **Request-Reply через брокер** (RPC scheduler ↔ summarization): correlation ID, reply topic, таймауты.
- **Idempotent Consumer.** Письмо не должно уйти дважды при повторной доставке.
- **Retry + Dead Letter Topic.** Что делать с «ядовитым» сообщением?
- **Transactional Outbox** для приветственного письма при регистрации.
- **Adapter** для LLM-провайдера (сменить OpenAI на GigaChat/YandexGPT — новый адаптер, не правка логики).
- **Twelve-Factor App**: конфиг через окружение, stateless-процессы, логи в stdout.

## Вехи

### Веха 7.1 — Архитектура на бумаге
Сдать до кода: `ARCHITECTURE.md` проекта:
- схема сервисов и потоков данных (Mermaid), топики, форматы сообщений (JSON-схемы/примеры), ключи сообщений;
- кто владеет какими данными и миграциями;
- дизайн недостающих эндпоинтов задач (создание, редактирование, отметка «сделано», удаление): пути, методы, коды, тела;
- монорепо (multi-module) или несколько репо — решение с аргументами.

Это ревьюится так же жёстко, как код. Ошибки архитектуры дороже всего исправлять потом.

### Веха 7.2 — Инфраструктура и backend: пользователи + JWT
Критерии приёмки:
- Compose: Postgres + Kafka (KRaft), healthchecks, `.env.example`.
- `POST /user` (регистрация сразу выдаёт токен в заголовке), `POST /auth/login`, `GET /user` — строго по ТЗ, 401/409 JSON.
- JWT валидируется в Security-фильтре, `STATELESS`, секрет/ключи из окружения, `exp` проверяется.
- Тесты: невалидная подпись, истёкший токен, отсутствие токена → 401.

### Веха 7.3 — Backend: задачи
Критерии приёмки:
- CRUD задач по твоему дизайну из 7.1; пользователь видит и меняет только свои задачи (проверка на уровне запроса к БД).
- Отметка «сделано» фиксирует время (для отчёта «сделано за сутки»); снятие отметки — корректно.
- Коллекция запросов Postman/IDEA HTTP Client в репо.
- Интеграционные тесты на Testcontainers.

### Веха 7.4 — Email sender + приветственное письмо
Вопросы до кода:
- Backend регистрирует пользователя и должен отправить событие. Что если Kafka недоступна в этот момент? Что если
  транзакция откатится?
- Email sender получил сообщение, отправил письмо, упал до коммита offset. Что будет при рестарте?

Критерии приёмки:
- Топик `EMAIL_SENDING_TASKS`, сообщение: получатель, тема, текст (по ТЗ).
- Outbox в backend (или обоснованная альтернатива с описанием компромисса).
- Идемпотентный консьюмер, retry с backoff, DLT для «ядовитых» сообщений.
- SMTP-креды только из окружения; для разработки — MailHog/Mailpit в compose (письма не уходят наружу).

### Веха 7.5 — Summarization-сервис (LLM)
Критерии приёмки:
- HTTP-клиент к LLM **без SDK** (как требует ТЗ): таймауты, обработка 429/5xx, ограничение длины входа.
- Системный промпт, формат ответа, примеры — в ресурсах, не хардкодом в Java.
- Защита от промпт-инъекции из пользовательских заголовков задач (хотя бы базовая: разделение данных и инструкций).
- Request-reply через Kafka: correlation ID, таймаут ожидания на стороне вызывающего.
- Тесты с моком LLM API (WireMock); ни одного реального вызова в тестах.

### Веха 7.6 — Scheduler: ежедневный отчёт
Вопросы до кода:
- 10 000 пользователей. Грузить всех в память? Отправлять 10 000 RPC-запросов одновременно?
- Приложение упало в середине рассылки. Что будет при следующем запуске? Кто-то получит два письма? Кто-то — ни одного?
- Полночь по какому часовому поясу?

Критерии приёмки:
- `@Scheduled` по cron, выборка пользователей батчами, отчёт за сутки: сделанные за 24 ч + количество несделанных.
- Защита от двойного запуска (ShedLock или обоснование).
- Время через `Clock` — тест «прошли сутки» без ожидания.

### Веха 7.7 — Docker, CI/CD, деплой
Критерии приёмки:
- Multi-stage Dockerfile на каждый сервис, `jre`-образ, non-root, образ < 300 МБ.
- GitHub Actions: на push в main — тесты, сборка, push образов в Docker Hub; креды — в GitHub Secrets.
- Деплой: VPS + `docker compose pull && up -d`, приложение доступно по IP/домену.
- `README.md`: архитектура, запуск локально одной командой, переменные окружения.

## Челленджи

- ★ **Structured logging + correlation ID** сквозь HTTP и Kafka (MDC, заголовки сообщений). Найти путь одного
  отчёта через 3 сервиса по логам.
- ★ **Refresh-токены** с ротацией и отзывом.
- ★★ **Outbox через Debezium (CDC)** вместо поллера.
- ★★ **Contract testing** (Spring Cloud Contract или Pact) для сообщений Kafka между сервисами.
- ★★★ **Observability**: OpenTelemetry-трейсинг сквозь HTTP и Kafka, Jaeger/Tempo в compose.
- ★★★ **Нагрузочный тест** (k6/Gatling): где узкое место? Масштабируй email-sender до 3 инстансов — что изменится
  в партициях и consumer group?

## За что будет 🔴

- Отправка в Kafka внутри транзакции БД без outbox и без осознанного компромисса.
- Неидемпотентный консьюмер (двойные письма при повторе), нет обработки ошибок консьюмера (бесконечный цикл ретраев).
- JWT: секрет в коде/репо, нет проверки `exp`, чувствительные данные в payload, `alg: none`-уязвимость.
- Общая БД, в которую пишут все сервисы без владельца; миграции в нескольких сервисах на одни таблицы.
- LLM-ключ/SMTP/Docker Hub креды в репозитории; LLM-вызовы в тестах.
- Scheduler, загружающий всех пользователей в память; двойной запуск при двух инстансах без защиты.
- Dockerfile с JDK и исходниками в финальном образе, запуск от root.
- Пользователь видит/меняет чужие задачи.

## Контрольные вопросы (защита)

1. Почему микросервисы? Где в твоём проекте они усложнили жизнь? Когда бы ты выбрал модульный монолит?
2. Kafka: topic, partition, offset, consumer group, rebalance. Как порядок и параллелизм связаны с партициями?
3. At-least-once vs exactly-once. Как ты обеспечил отсутствие двойных писем?
4. Transactional Outbox: схема, альтернативы (CDC), что с порядком событий?
5. Request-reply через Kafka vs HTTP между сервисами: плюсы, минусы, таймауты.
6. JWT: структура, подпись, почему не шифрование, как отозвать токен?
7. Как устроена stateless-аутентификация в твоём Spring Security-конфиге?
8. `@Scheduled` при горизонтальном масштабировании: проблема и решения.
9. Multi-stage Dockerfile: зачем, что в каждом этапе, как работает кеш слоёв?
10. CI/CD: что происходит от `git push` до обновлённого контейнера на сервере?

> 🎯 **Спросят на собесе:** Что такое идемпотентность и как сделать консьюмер идемпотентным?
> **Ответ:** Повторное выполнение операции даёт тот же результат, что и однократное. Для консьюмера: уникальный ID сообщения
> (или бизнес-ключ) + таблица обработанных ID с уникальным ограничением в той же транзакции, что и эффект; либо
> естественно идемпотентные операции (upsert, «установить статус X»).
