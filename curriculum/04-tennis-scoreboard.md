# Проект 4. Табло теннисного матча

**ТЗ:** [zhukovsd — Табло теннисного матча](https://zhukovsd.github.io/java-backend-learning-course/projects/tennis-scoreboard/)
REST API: создать матч (`POST /matches`, в ответ UUID), начислить очко (`POST /matches/{uuid}/point`),
получить счёт (`GET /matches/{uuid}`), список завершённых матчей с пагинацией и фильтром по имени (`GET /matches?page=&player_name=`).
Счёт: очки 0/15/30/40/AD, «больше-меньше», геймы, сеты до 2 побед, тай-брейк до 7 при 6:6.
Текущие матчи — в **потокобезопасной коллекции в памяти**, завершённые — в Postgres через **Hibernate**.
Стек из ТЗ: Spring MVC, Hibernate, Postgres, JUnit 5, деплой в Tomcat.
Есть [фронтенд](https://github.com/zhukovsd/tennis-scoreboard-frontend) — статические файлы кладутся в проект.

## Цели обучения

- **Богатая доменная модель**: сложная логика живёт в объектах домена, а не в «сервисе на 500 строк».
- Три разных «модели» в одном проекте: **доменная модель**, **JPA-сущность**, **DTO** — и чёткие границы между ними.
- Hibernate/JPA: маппинг, `SessionFactory`/`EntityManager`, жизненный цикл сущности, транзакции, fetch-стратегии, N+1, пагинация.
- Spring MVC: `DispatcherServlet`, `@RestController`, DI-контейнер, бины, `@ControllerAdvice`.
- Юнит-тестирование сложной логики, TDD.
- Конкурентный доступ к изменяемому состоянию в памяти.

## Что изучить до старта

| Тема | Обрати внимание |
|------|-----------------|
| Фаулер: Domain Model vs Transaction Script, [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html) | поведение рядом с данными |
| Value Object, иммутабельность, `record` | счёт гейма — Value Object? |
| [State](https://refactoring.guru/ru/design-patterns/state) | обычный гейм vs тай-брейк; матч идёт vs завершён |
| JPA/Hibernate: `@Entity`, `@Id`/`@GeneratedValue`, `@ManyToOne`, `FetchType`, `@Column(nullable, length, unique)`, `@Table(indexes)` | по умолчанию `@ManyToOne` — **EAGER**. Это ловушка |
| Hibernate: `Session`, `Transaction`, состояния сущности (transient/managed/detached/removed), dirty checking, first-level cache | почему «сессия на операцию» — антипаттерн |
| JPQL/HQL, `JOIN FETCH`, `setFirstResult/setMaxResults`, count-запрос | пагинация + `JOIN FETCH` коллекций = беда (in-memory pagination). Здесь `@ManyToOne` — ок, но разберись почему |
| Spring Core: IoC-контейнер, `@Configuration`, `@Bean`, `@Component`, скоупы бинов, конструкторная инъекция | бины — синглтоны по умолчанию → то же требование к потокобезопасности, что у сервлетов |
| Spring MVC: `DispatcherServlet`, `@RestController`, `@RequestBody`, `@PathVariable`, `@RequestParam`, `@ControllerAdvice`, `WebApplicationInitializer` | без Spring Boot — собираешь конфигурацию сам |
| Spring `@Transactional`: как работает (прокси), propagation, почему не работает на private и при self-invocation | Hibernate `SessionFactory` + Spring `HibernateTransactionManager` или JPA `EntityManagerFactory` |
| `ConcurrentHashMap`: `compute`, `computeIfPresent`, атомарность составных операций; `synchronized` на объекте матча | CHM потокобезопасна для **мапы**, но не для **объектов в ней** |
| JUnit 5: `@ParameterizedTest`, `@Nested`, AssertJ | тест-кейсы строят счёт — нужны удобные хелперы в тестах |

📚 **Читать:**
- PoEAA, гл. 9: Domain Model vs Transaction Script; гл. 11 «Object-Relational Behavioral Patterns» (Unit of Work, Identity Map, Lazy Load — это то, что делает Hibernate); гл. 18: Value Object.
- Фаулер, [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html) (статья, 5 минут).
- HFDP, гл. 10 «The State Pattern» или [GURU: Состояние](https://refactoring.guru/ru/design-patterns/state).
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html): разделы Domain Model, Bootstrap, Persistence Context, Fetching, Transactions — по диагонали.
- HPJP, часть II «JPA and Hibernate»: главы про маппинг связей (Relationships), Flushing, **Fetching** (N+1, `JOIN FETCH`, пагинация) — Fetching обязательно.
- Блог Михалчи: [N+1 query problem](https://vladmihalcea.com/n-plus-1-query-problem/), [Best way to map @ManyToOne](https://vladmihalcea.com/manytoone-jpa-hibernate/).
- SIA, гл. 1 «Getting started with Spring» (IoC, DI, конфигурация) и гл. 2 «Developing web applications» (Spring MVC). Всё, что про Boot, мысленно переводи в ручную конфигурацию.
- [Spring Framework Reference: Core — The IoC Container](https://docs.spring.io/spring-framework/reference/core/beans.html) (Introduction, Bean Scopes, Java-based Container Configuration), [Web MVC — DispatcherServlet](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html), [Transaction Management — Declarative](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html) (особенно раздел про прокси).
- JCIP, гл. 4 «Composing Objects», гл. 5 «Building Blocks» (concurrent-коллекции) — перед вехой 4.4.
- UTPP, гл. 4 «The four pillars of a good unit test» и гл. 6 «Styles of unit testing» — для TDD вехи 4.1.
- Кент Бек, «Экстремальное программирование: разработка через тестирование» (*TDD by Example*), часть I — по желанию, если TDD в новинку.

> 🎯 **Спросят на собесе:** Состояния сущности в Hibernate/JPA?
> **Ответ:** Transient (новый объект, не связан с сессией), Managed/Persistent (в контексте персистентности, изменения
> отслеживаются dirty checking и сбрасываются при flush), Detached (сессия закрыта, изменения не отслеживаются),
> Removed (помечен на удаление).

> 🎯 **Спросят на собесе:** Почему `@Transactional` не срабатывает при вызове метода из того же класса?
> **Ответ:** Spring оборачивает бин в прокси; транзакционная логика — в прокси. Вызов `this.method()` идёт мимо прокси.
> То же для `private`/`final` методов (CGLIB не может переопределить).

> 🎯 **Спросят на собесе:** `FetchType.LAZY` vs `EAGER`, какие значения по умолчанию?
> **Ответ:** `@ManyToOne`/`@OneToOne` — EAGER, `@OneToMany`/`@ManyToMany` — LAZY. Рекомендация — всё LAZY,
> а нужные связи подгружать явно (`JOIN FETCH`, `@EntityGraph`) в конкретных запросах.

## Паттерны и принципы в фокусе

- **Rich Domain Model.** `Match` → `Set` → `Game`/`TieBreak`: каждый знает свои правила и когда он завершён.
  Сервис только достаёт матч, говорит «очко игроку X» и сохраняет результат.
- **State / полиморфизм.** Гейм и тай-брейк — разные правила подсчёта. Один класс с `if (isTieBreak)` повсюду → 🔴 (SRP).
- **Value Object + `enum`.** Очки в гейме — не `int` 0..4, а тип с доменным смыслом (LOVE, FIFTEEN, ...).
- **Single Source of Truth.** «Завершён ли гейм», «кто победитель» — **вычисляются**, а не хранятся отдельным полем
  рядом с данными, из которых выводятся.
- **Repository/DAO + разделение моделей.** Домен ≠ entity ≠ DTO. Доменный игрок — это не JPA-сущность `Player`.
- **Исключения на границе.** `HibernateException` не должен долететь до контроллера. Трансляция в свои исключения.

## Вехи

### Веха 4.1 — Доменная модель подсчёта очков (TDD, без фреймворков)
**Цель:** чистая Java-модель, без Spring, без Hibernate, без HTTP. Это самая важная веха проекта.

Вопросы до кода:
- Выпиши все переходы счёта: 40:40 → AD → deuce/победа; 5:5 → 6:5 → 7:5; 6:6 → тай-брейк; тай-брейк 6:6 → нужна разница 2.
- Какие классы и какова ответственность каждого? Кто знает про «разницу в 2»?
- Как внешний код узнаёт счёт, не имея возможности его изменить?
- Что происходит при попытке начислить очко в завершённом матче?

Критерии приёмки:
- **Сначала тесты** (TDD): `@Nested` группы на гейм, «больше-меньше», сет, тай-брейк, матч. 30+ кейсов.
  Все сценарии из ТЗ + придуманные тобой граничные.
- Нет сеттеров, нет публичных изменяемых коллекций. Поведенческие методы вместо `setPoints`.
- Гейм и тай-брейк — разные классы (или стратегии) с общим контрактом.
- Нет хранимого вычисляемого состояния (`isFinished`, `winner` как поля).

### Веха 4.2 — Каркас Spring MVC + Hibernate + Postgres
Критерии приёмки:
- Spring MVC без Boot: Java-конфиг, `DispatcherServlet` регистрируется через `WebApplicationInitializer`, WAR в Tomcat.
- Postgres в Docker (`docker run` или compose), креды — через переменные окружения/внешний файл, **не** в git.
- Сущности `Player`, `Match` по ТЗ: `nullable=false`, `length`, уникальный индекс на имя игрока, внешние ключи.
- `SessionFactory`/`EntityManagerFactory` — один на приложение, создаётся при старте, закрывается при остановке.

### Веха 4.3 — Создание матча и хранилище текущих матчей
Вопросы до кода:
- Где хранятся текущие матчи? Какой ключ? Кто генерирует UUID?
- Игрок с таким именем уже есть в БД — что делать? Как избежать гонки «проверить → вставить»?
- Что проверяется валидацией (одинаковые имена, пустые, длина) и где? Совпадают ли ограничения валидации и БД?

Критерии приёмки:
- `POST /matches` по ТЗ, 400 при ошибках валидации с `{"message": ...}`.
- Хранилище матчей — `ConcurrentHashMap` (или обоснованная альтернатива) внутри отдельного компонента.
- Контроллер тонкий: DTO in → сервис → DTO out.

### Веха 4.4 — Очки и счёт: `POST /matches/{uuid}/point`, `GET /matches/{uuid}`
Вопросы до кода:
- Два запроса начисляют очко одному матчу одновременно. Что сломается? Как защитить **объект матча**?
- Как доменный счёт превращается в DTO из ТЗ (`"points": "40"`/`"AD"`, `tieBreakPoints`)? Где живёт этот маппинг?

Критерии приёмки:
- Формат ответа строго по ТЗ (включая `null`-поля в тай-брейке и `winnerName`).
- 404 для неизвестного UUID; начисление в завершённом матче — осмысленная ошибка.
- Нет гонки при одновременных начислениях — **докажи тестом** (многопоточный тест на `ExecutorService` + `CountDownLatch`).

### Веха 4.5 — Сохранение завершённого матча
Критерии приёмки:
- По завершении матч сохраняется в БД **в одной транзакции** и удаляется из памяти. Порядок операций обоснован
  (что если БД недоступна?).
- В БД: победитель — один из игроков, игроки разные (`CHECK`-ограничения).

### Веха 4.6 — Список завершённых матчей: `GET /matches?page=&player_name=`
Вопросы до кода:
- Сколько SQL-запросов уходит на страницу из 10 матчей? Проверь, включив логирование SQL. Должно быть 2 (данные + count).
- Где считается `offset` из номера страницы — в DAO или сервисе?
- Поиск по имени: точное совпадение или подстрока? Регистр? Какой индекс это поддержит?

Критерии приёмки:
- Пагинация на уровне SQL (`LIMIT/OFFSET`), стабильная сортировка, `totalPages`.
- Нет N+1 (подтверждено логом SQL в описании вехи).
- Фронтенд из ТЗ работает со всеми 4 страницами.

### Веха 4.7 — Деплой
Критерии приёмки:
- WAR в Tomcat на VPS + Postgres (в Docker или системный), креды вне репозитория.
- Пройден [чеклист из ТЗ](https://zhukovsd.github.io/java-backend-learning-course/projects/tennis-scoreboard/#чеклист-для-самопроверки) —
  он большой и очень конкретный, пройди **каждый** пункт и отметь в README.

## Челленджи

- ★ **Параметры матча.** Best of 3 / best of 5, тай-брейк в решающем сете или нет — через конфигурацию матча, без `if` по всей модели.
- ★ **Логирование SQL** через `datasource-proxy` или p6spy: в тестах DAO проверяй **количество** запросов (защита от N+1 регрессий).
- ★★ **Оптимистическая блокировка.** Представь, что матчи хранятся в БД (несколько инстансов приложения). Как защитить
  от гонок? `@Version`, `OptimisticLockException`, retry.
- ★★ **Keyset-пагинация** вместо `OFFSET` для списка матчей. Когда `OFFSET` становится проблемой?
- ★★★ **Property-based тесты** (jqwik): случайные последовательности очков → инварианты (сумма сетов победителя = 2,
  матч не продолжается после победы и т. п.).
- ★★★ **Архитектурные тесты** (ArchUnit): контроллеры не импортируют entity, домен не зависит от Spring/Hibernate.

## За что будет 🔴

- Анемичная модель: вся логика подсчёта в сервисе.
- Один класс на гейм и тай-брейк с флагами. Очки как `int` 0..4.
- Сеттеры на доменных объектах, хранимое вычисляемое состояние.
- JPA-сущности в доменной модели, в DTO, в контроллерах. Lombok `@Data` на сущностях.
- `HashMap` для текущих матчей; гонка при начислении очков.
- N+1, пагинация в памяти, нет сортировки, `offset` считается в DAO.
- Сессия Hibernate на каждую операцию; `SessionFactory` создаётся больше одного раза или не закрывается.
- Креды БД в репозитории.
- Недостаточно тестов подсчёта очков (особенно AD и тай-брейк).

## Контрольные вопросы (защита)

1. Rich vs anemic domain model. Покажи, где в твоём коде поведение, а где данные.
2. Почему домен, entity и DTO — три разные вещи? Что сломается, если их слить?
3. Как Hibernate понимает, что сущность изменилась (dirty checking)? Что такое first-level cache?
4. N+1: как обнаружить и как лечить (3 способа)?
5. `LAZY`/`EAGER`, `LazyInitializationException` — когда возникает и как правильно лечить (и почему Open Session in View — не решение)?
6. Как работает `@Transactional` в Spring? Propagation `REQUIRED` vs `REQUIRES_NEW`?
7. Бины Spring: скоупы, жизненный цикл, почему синглтон-бин должен быть stateless?
8. `ConcurrentHashMap` — что гарантирует и чего нет? Как ты защитил объект матча?
9. `DispatcherServlet`: путь запроса до твоего контроллера (HandlerMapping, HandlerAdapter, MessageConverter).
10. Пагинация: `OFFSET` vs keyset.

> 🎯 **Спросят на собесе:** Чем `ConcurrentHashMap` отличается от `Collections.synchronizedMap`?
> **Ответ:** `synchronizedMap` блокирует всю мапу на каждую операцию. CHM — неблокирующее чтение и блокировка на уровне
> бакета при записи (CAS + synchronized на узле в Java 8+), атомарные `compute*`/`merge`, слабо согласованные итераторы
> без `ConcurrentModificationException`. `null` ключи и значения в CHM запрещены.
