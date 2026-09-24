# 🎯 Копилка вопросов к собеседованию

Сюда собирается всё, что помечено «🎯 Спросят на собесе» — в учебных модулях и в ревью твоего кода.
Вопросы из модулей даны ссылками (ответы там). Вопросы из ревью ментор дописывает целиком — **они ценнее всего**,
потому что выросли из твоих собственных ошибок.

Как пользоваться: перед собеседованием закрой ответы и проговори каждый вслух за 1–2 минуты.
Можно попросить ментора: *«устрой мок-собес по разделу Core/Collections/…»*.

Формат записи из ревью:
```
- **Вопрос** — [ревью](reviews/...) · кратко: ответ · ловушка: ...
```

---

## JVM и инструменты
- JDK vs JRE vs JVM — [00-setup](curriculum/00-setup.md)
- JIT и «прогрев» — [00-setup](curriculum/00-setup.md)
- classpath, `ClassNotFoundException` vs `NoClassDefFoundError` — [00-setup](curriculum/00-setup.md)
- LTS-версии Java — [00-setup](curriculum/00-setup.md)
- `mvn package` vs `install`; scope зависимостей, `provided`; разрешение конфликтов версий — [00-setup](curriculum/00-setup.md)

### Из ревью
_пока пусто_

## Java Core
- `Integer` кеш и `==` — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-1-статическая-типизация-примитивы-и-обёртки)
- `double` и деньги, `BigDecimal` — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-1-статическая-типизация-примитивы-и-обёртки), [03-currency-exchange](curriculum/03-currency-exchange.md)
- Иммутабельный класс; package-private vs `protected` — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-2-классы-объекты-инкапсуляция)
- `equals`/`hashCode`, мутабельные ключи — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-3-equals-hashcode-tostring-comparable)
- Абстрактный класс vs интерфейс; overload vs override — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-4-наследование-интерфейсы-абстрактные-классы-полиморфизм)
- Checked vs unchecked; `return` в `finally` — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-5-исключения)
- PECS, type erasure — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-7-generics)
- `Optional`: `orElse` vs `orElseGet`, где не использовать — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-8-records-enum-optional)
- Stream API: ленивость, промежуточные/терминальные — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-9-лямбды-и-stream-api)
- Иммутабельность `String`; `LocalDateTime` vs `Instant` — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-10-строки-ввод-вывод-время)

### Из ревью
_пока пусто_

## Коллекции
- Устройство `HashMap`, коллизии, resize — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-6-коллекции)
- `ArrayList` vs `LinkedList` — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-6-коллекции)
- `ConcurrentModificationException` — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-6-коллекции)
- `ConcurrentHashMap` vs `synchronizedMap` — [04-tennis-scoreboard](curriculum/04-tennis-scoreboard.md)

### Из ревью
_пока пусто_

## Многопоточность
- `volatile` и `count++` — [02-simulation](curriculum/02-simulation.md)
- `wait()` в цикле — [02-simulation](curriculum/02-simulation.md)
- `SecurityContext` в `ThreadLocal` — [06-cloud-file-storage](curriculum/06-cloud-file-storage.md)

### Из ревью
_пока пусто_

## ООП, SOLID, паттерны
- DI и конструкторная инъекция; SOLID на своём примере — [01-hangman](curriculum/01-hangman.md)
- Композиция vs наследование; LSP — [02-simulation](curriculum/02-simulation.md)
- Singleton как антипаттерн — [03-currency-exchange](curriculum/03-currency-exchange.md)
- Mock vs Stub vs Fake — [05-weather-viewer](curriculum/05-weather-viewer.md)

### Из ревью
_пока пусто_

## SQL, JDBC, транзакции
- `Statement` vs `PreparedStatement`; пул соединений — [03-currency-exchange](curriculum/03-currency-exchange.md)
- N+1 — [03-currency-exchange](curriculum/03-currency-exchange.md)
- Уровни изоляции — [03-currency-exchange](curriculum/03-currency-exchange.md)

### Из ревью
_пока пусто_

## Hibernate / JPA
- Состояния сущности; `LAZY` vs `EAGER` — [04-tennis-scoreboard](curriculum/04-tennis-scoreboard.md)

### Из ревью
_пока пусто_

## Servlets, Spring, Spring Boot, Security
- Жизненный цикл сервлета — [03-currency-exchange](curriculum/03-currency-exchange.md)
- `@Transactional` и self-invocation — [04-tennis-scoreboard](curriculum/04-tennis-scoreboard.md)
- Автоконфигурация Boot; цепочка фильтров Security — [06-cloud-file-storage](curriculum/06-cloud-file-storage.md)

### Из ревью
_пока пусто_

## Веб-безопасность
- BCrypt vs SHA-256; session fixation; `HttpOnly`/`SameSite` — [05-weather-viewer](curriculum/05-weather-viewer.md)
- JWT vs сессии — [07-task-tracker](curriculum/07-task-tracker.md)

### Из ревью
_пока пусто_

## Микросервисы, Kafka, DevOps
- Порядок в Kafka; consumer group — [07-task-tracker](curriculum/07-task-tracker.md)
- Transactional Outbox; идемпотентный консьюмер — [07-task-tracker](curriculum/07-task-tracker.md)

### Из ревью
_пока пусто_
