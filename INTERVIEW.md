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
- `double` и деньги, `BigDecimal` — [00-java-for-pythonists](curriculum/00-java-for-pythonists.md#тема-1-статическая-типизация-примитивы-и-обёртки), [03-currency-exchange](curriculum/03-currency-exchange.md#1-bigdecimal--деньги-без-потерь)
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
- `volatile` и `count++`; `wait()` в цикле — [М2](curriculum/02-0-bridge-oop-design.md#часть-5-потоки-минимум-для-симуляции)
- `SecurityContext` в `ThreadLocal` — [М6](curriculum/06-0-bridge-boot-docker-security.md)
- `@Scheduled` на нескольких инстансах — [07-task-tracker](curriculum/07-task-tracker.md)

### Из ревью
_пока пусто_

## ООП, SOLID, паттерны
- DI и конструкторная инъекция; SOLID на своём примере — [01-hangman](curriculum/01-hangman.md)
- Композиция vs наследование; LSP — [М2](curriculum/02-0-bridge-oop-design.md)
- Singleton как антипаттерн — [М3](curriculum/03-0-bridge-web-jdbc.md)
- Mock vs Stub vs Fake — [05-weather-viewer](curriculum/05-weather-viewer.md)

### Из ревью
_пока пусто_

## SQL, JDBC, транзакции
- `Statement` vs `PreparedStatement`; пул соединений; N+1; уровни изоляции — [М3](curriculum/03-0-bridge-web-jdbc.md)

### Из ревью
_пока пусто_

## Hibernate / JPA
- Состояния сущности; `LAZY` vs `EAGER` — [М4](curriculum/04-0-bridge-spring-hibernate.md)

### Из ревью
_пока пусто_

## Servlets, Spring, Spring Boot, Security
- Жизненный цикл сервлета — [М3](curriculum/03-0-bridge-web-jdbc.md)
- `@Transactional` и self-invocation — [М4](curriculum/04-0-bridge-spring-hibernate.md)
- Автоконфигурация Boot; цепочка фильтров Security — [М6](curriculum/06-0-bridge-boot-docker-security.md)
- Отдать большой файл без загрузки в память — [06-cloud-file-storage](curriculum/06-cloud-file-storage.md)

### Из ревью
_пока пусто_

## Веб-безопасность
- BCrypt vs SHA-256; session fixation; `HttpOnly`/`SameSite` — [05-weather-viewer](curriculum/05-weather-viewer.md)
- JWT vs сессии — [М7](curriculum/07-0-bridge-kafka-microservices.md)

### Из ревью
_пока пусто_

## Микросервисы, Kafka, DevOps
- Порядок в Kafka; consumer group; Transactional Outbox; идемпотентный консьюмер — [М7](curriculum/07-0-bridge-kafka-microservices.md)

### Из ревью
_пока пусто_
