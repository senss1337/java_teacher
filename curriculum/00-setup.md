# Шаг 0. Окружение: JDK, Maven, IntelliJ IDEA

Цель шага: понять, **как Java-код превращается в работающую программу**, и настроить окружение так,
чтобы в дальнейшем ничего не «магичило». В Python ты выбираешь интерпретатор и `venv`. В Java всё устроено иначе,
и если это не понять сейчас, потом будешь месяцами страдать от `ClassNotFoundException` и
«в IDEA работает, а из консоли нет».

---

## 1. Модель: откуда берётся «интерпретатор»

### Путь кода

```
Hello.java  --javac-->  Hello.class (байткод)  --java (JVM)-->  машинный код
  исходник   компилятор   платформонезависимый     интерпретатор + JIT
```

- **`javac`** — компилятор. Проверяет типы, выдаёт **байткод** (`.class`). Ошибки типов ловятся тут, а не в рантайме.
- **JVM** (`java`) загружает `.class`-файлы, сначала **интерпретирует** байткод, а горячие методы
  **JIT-компилирует** в машинный код (HotSpot: C1/C2). Поэтому Java «прогревается», первые запросы медленнее.
- **JAR** — zip-архив с `.class` + `META-INF/MANIFEST.MF`. **WAR** — то же для веб-контейнеров (Tomcat).

### JDK / JRE / JVM

| Что | Состав | Кому нужно |
|-----|--------|------------|
| **JVM** | виртуальная машина: загрузчик классов, GC, JIT | всем, кто запускает |
| **JRE** | JVM + стандартная библиотека | тем, кто только запускает (на сервере) |
| **JDK** | JRE + `javac`, `jshell`, `jar`, `javadoc`, `jcmd`, `jconsole`... | разработчику, т. е. тебе |

С Java 11 отдельный JRE официально не распространяется, но у вендоров (Temurin) он есть — пригодится для Docker-образов.

### Параллели с Python — и где они ломаются

| Python | Java | Где аналогия врёт |
|--------|------|-------------------|
| CPython | JDK (HotSpot JVM) | В Python исходник исполняется «напрямую» (на деле тоже через байткод `.pyc`), в Java обязательный явный шаг компиляции с проверкой типов |
| `pyenv` | [SDKMAN!](https://sdkman.io/) | — |
| выбор интерпретатора в PyCharm | Project SDK в IDEA | Кроме SDK есть ещё **language level** — какую версию синтаксиса разрешать |
| `venv` | **нет аналога** | Изоляция в Java — это **classpath**: список JAR-ов, передаваемый JVM при запуске. Глобального `site-packages` нет |
| `pip` + `requirements.txt` / `pyproject.toml` | Maven + `pom.xml` | Maven не только качает зависимости, но и **собирает** проект (компиляция, тесты, упаковка) |
| `site-packages` | `~/.m2/repository` | Один кеш на все проекты, версии лежат рядом, конфликтов нет, т. к. classpath у каждого проекта свой |
| `python -m pytest` | `mvn test` | Тесты — фаза жизненного цикла сборки |
| REPL `python` | `jshell` | — |

> 🎯 **Спросят на собесе:** Чем отличаются JDK, JRE и JVM?
> **Ответ:** JVM исполняет байткод (интерпретатор + JIT + GC). JRE = JVM + стандартная библиотека, достаточна для запуска.
> JDK = JRE + инструменты разработки (`javac`, `jar`, отладка, профилирование).
> **Типичная ошибка:** «JVM компилирует Java-код» — нет, компилирует `javac` в байткод, JVM компилирует байткод в машинный код (JIT).

> 🎯 **Спросят на собесе:** Что такое JIT и почему Java «прогревается»?
> **Ответ:** JVM стартует с интерпретации байткода, собирает профиль, горячие методы компилирует в оптимизированный
> машинный код (инлайнинг, escape analysis). До этого код работает медленнее.
> **Типичная ошибка:** путать JIT с AOT (GraalVM native-image).

> 🎯 **Спросят на собесе:** Что такое classpath и откуда `ClassNotFoundException` vs `NoClassDefFoundError`?
> **Ответ:** classpath — список директорий/JAR, где ClassLoader ищет классы. `ClassNotFoundException` — checked, при явной
> загрузке по имени (`Class.forName`) класса нет. `NoClassDefFoundError` — Error: класс был при компиляции, но его нет
> (или не инициализировался) в рантайме.
> **Типичная ошибка:** считать их синонимами.

📚 **Читать** (по диагонали):
- CJ1, гл. 1 «An Introduction to Java» (история, «белая книга» — зачем JVM) и гл. 2 «The Java Programming Environment» (установка JDK, `javac`/`java`, `jshell`) — **весь шаг 0 по ним**.
- [dev.java: Getting Started with Java](https://dev.java/learn/getting-started/) — официальный быстрый старт.
- [Oracle: The `java` Command](https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html) — пролистай, как задаётся classpath (`-cp`) и какие бывают режимы запуска.
- Для понимания JIT (по желанию): [dev.java: The JVM and Just-in-Time Compilation](https://dev.java/learn/jvm/) — раздел про JIT и HotSpot.

---

## 2. Установка JDK 21

Берём **Eclipse Temurin 21 (LTS)**. Дистрибутивы (Temurin, Amazon Corretto, Oracle OpenJDK, Liberica, Zulu)
собираются из одного исходника OpenJDK, отличаются лицензией, сроками поддержки и доп. сборками.
Oracle JDK для коммерции имеет свои условия — для учёбы и работы бери Temurin/Corretto.

> 🎯 **Спросят на собесе:** Что такое LTS и почему в проде Java 17/21, а не 23?
> **Ответ:** Новая версия выходит каждые 6 месяцев; LTS (8, 11, 17, 21, 25) получают многолетние патчи безопасности.
> Прод живёт на LTS.

### macOS / Linux — через SDKMAN (рекомендую)

```bash
curl -s "https://get.sdkman.io" | bash
# перезапусти терминал
sdk list java | grep -i tem        # посмотреть доступные Temurin
sdk install java 21.0.x-tem        # подставь актуальную версию из списка
sdk install maven
sdk current                        # что сейчас активно
```

SDKMAN сам выставит `JAVA_HOME` и `PATH`. Переключение версий: `sdk use java <версия>` (в текущем шелле)
или `sdk default java <версия>`.

### Windows

- `winget install EclipseAdoptium.Temurin.21.JDK` (или MSI с [adoptium.net](https://adoptium.net/)) — в установщике отметь
  *Set JAVA_HOME* и *Add to PATH*.
- Maven: `winget install Apache.Maven` или распаковать архив и добавить `bin` в `PATH`.

### Проверка

```bash
java -version      # openjdk version "21.x" ... Temurin
javac -version     # javac 21.x  ← если нет, у тебя JRE вместо JDK или PATH смотрит не туда
echo $JAVA_HOME    # Windows: echo %JAVA_HOME%
mvn -version       # в выводе должна быть та же Java 21
```

> ⚠️ **Типичная ловушка:** `java -version` показывает 21, а `mvn -version` — 17. Maven берёт JDK из `JAVA_HOME`,
> а не из `PATH`. Всегда проверяй обе команды.

---

## 3. Упражнение: Java без IDE (обязательно)

Сделай руками, чтобы понимать, что IDE делает за тебя:

1. Напиши класс `Hello` с `main`, выводящий аргументы командной строки.
2. `javac Hello.java` → посмотри на `Hello.class`. Запусти `java Hello a b c`.
3. Положи класс в пакет `dev.me.hello` (директории должны соответствовать пакету!). Скомпилируй с `-d out`,
   запусти с `-cp out`. Разберись, почему `java Hello` больше не работает.
4. Запусти однофайловую программу без явной компиляции: `java Hello.java` (JEP 330). Чем это отличается от п. 2?
5. Собери JAR: `jar --create --file hello.jar --main-class dev.me.hello.Hello -C out .`, запусти `java -jar hello.jar`.
6. `javap -c -p out/dev/me/hello/Hello.class` — посмотри на байткод. Найди вызов `println`.
7. `jshell`: поиграй с `var`, `List.of(1,2,3)`, `"abc".repeat(3)`, `/vars`, `/methods`, `/exit`.

---

## 4. Maven

### Что нужно понимать

- **Convention over configuration**: `src/main/java`, `src/main/resources`, `src/test/java`, `src/test/resources`, `target/`.
- **Координаты**: `groupId:artifactId:version` (GAV). Зависимость = координаты + `scope`
  (`compile` по умолчанию, `test`, `provided` — даст контейнер, например servlet-api в Tomcat, `runtime`).
- **Жизненный цикл**: `validate → compile → test → package → verify → install → deploy`.
  Запуск фазы выполняет все предыдущие. `mvn package` = скомпилировать + прогнать тесты + собрать JAR/WAR.
- **Плагины** выполняют работу фаз: `maven-compiler-plugin`, `maven-surefire-plugin` (юнит-тесты),
  `maven-failsafe-plugin` (интеграционные).
- **Транзитивные зависимости** и конфликты версий: `mvn dependency:tree`. Управление версиями — `<dependencyManagement>`, BOM.
- **Maven Wrapper** (`mvnw`) — фиксирует версию Maven в проекте, как `poetry.lock` для самого инструмента. Используй его в проектах.

Что прочитать: [Maven in 5 Minutes](https://maven.apache.org/guides/getting-started/maven-in-five-minutes.html),
[Introduction to the Build Lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html),
[Baeldung: Maven Guide](https://www.baeldung.com/maven).

В `pom.xml` вместо `source`/`target` используй `<maven.compiler.release>21</maven.compiler.release>` —
разберись сам, почему `release` правильнее (подсказка: проверка API стандартной библиотеки).

📚 **Читать:**
- [Maven: The Complete Reference](https://books.sonatype.com/mvnref-book/reference/) (Sonatype, бесплатно онлайн): гл. 3 «The Project Object Model», гл. 4 «The Build Lifecycle» — по диагонали.
- [Introduction to the Dependency Mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html) — scope, транзитивность, «nearest wins». **Обязательно**.
- [Maven Wrapper](https://maven.apache.org/wrapper/).

> 🎯 **Спросят на собесе:** Чем `mvn package` отличается от `mvn install`?
> **Ответ:** `package` собирает артефакт в `target/`. `install` дополнительно кладёт его в локальный репозиторий `~/.m2`,
> чтобы другие локальные проекты могли его подключить как зависимость.

> 🎯 **Спросят на собесе:** Какие бывают scope зависимостей и зачем `provided`?
> **Ответ:** `compile`, `provided`, `runtime`, `test`, `system`, `import`. `provided` — нужна для компиляции, но в рантайме её
> предоставит окружение (servlet API в Tomcat), поэтому в WAR не кладётся.

> 🎯 **Спросят на собесе:** Как разрешается конфликт версий транзитивных зависимостей в Maven?
> **Ответ:** «Nearest wins» — побеждает ближайшая к корню дерева версия, при равной глубине — объявленная первой.
> Управлять явно через `<dependencyManagement>`/BOM или `<exclusions>`.
> **Типичная ошибка:** «берётся самая новая» — это Gradle, не Maven.

---

## 5. IntelliJ IDEA

### Какую редакцию

- **Community** (бесплатная) — достаточно для проектов 0–3 (чистая Java, Maven, сервлеты с внешним Tomcat).
- **Ultimate** — удобнее для Spring, баз данных, HTTP-клиента, Tomcat-интеграции. Бесплатна для студентов
  ([JetBrains Education](https://www.jetbrains.com/community/education/)), есть триал. Не обязательна:
  всё можно сделать в Community + DBeaver + Postman.

Установка — через [JetBrains Toolbox](https://www.jetbrains.com/toolbox-app/) (обновления, несколько версий).

### Где «интерпретатор» (самое важное)

| Настройка | Где | Что значит |
|-----------|-----|------------|
| **Project SDK** | *File → Project Structure → Project → SDK* | Какой JDK используется для компиляции/запуска. Аналог «Python Interpreter» в PyCharm. Кнопка *Add SDK → Download JDK* скачает Temurin сама |
| **Language level** | там же | Какие фичи синтаксиса разрешены. Должно совпадать с `maven.compiler.release` |
| **Module SDK** | *Project Structure → Modules* | Может переопределить SDK для модуля (обычно *Project default*) |
| **Maven JDK** | *Settings → Build, Execution, Deployment → Build Tools → Maven → Runner → JRE* | Каким JDK IDEA запускает Maven. Должен быть тот же 21 |
| **Gradle/Maven import** | *Settings → Build Tools → Maven → Importing* | Автоперезагрузка `pom.xml` |

Правило: **источник правды — `pom.xml`**. IDEA при импорте Maven-проекта выставит language level из него.
Если в IDEA что-то работает, а `mvn verify` в терминале — нет, прав терминал. Ревью я делаю через `mvn`.

📚 **Читать:**
- [IntelliJ IDEA: Getting started](https://www.jetbrains.com/help/idea/getting-started.html) → разделы *Create your first Java application* и *SDKs* (про Project SDK и language level).
- [Maven support in IntelliJ IDEA](https://www.jetbrains.com/help/idea/maven-support.html).
- [Debug code](https://www.jetbrains.com/help/idea/debugging-code.html) — разделы *Breakpoints* и *Evaluate expressions*.
- Шпаргалка хоткеев: *Help → Keyboard Shortcuts PDF*. Распечатай, первые 2 недели держи рядом.

### Что освоить в IDEA

- **Открытие проекта**: *Open* → выбрать папку с `pom.xml` (не *New Project* поверх существующего).
- **Окно Maven** (справа): lifecycle-фазы, перезагрузка проекта (иконка ⟳ после правки `pom.xml`).
- **Run Configurations**: запуск `main`, аргументы, переменные окружения (сюда, а не в код, кладутся секреты).
- **Отладчик** — обязательно: breakpoint, conditional breakpoint, *Evaluate Expression* (Alt+F8), *Step Into/Over/Out*,
  *Drop Frame*. В Java отладчик — основной инструмент понимания чужого кода (особенно Spring).
- **Навигация**: Go to Class/File/Symbol (Ctrl/Cmd+N, Shift+Shift), Go to Declaration/Implementation
  (Ctrl/Cmd+B, Ctrl/Cmd+Alt+B), Find Usages (Alt+F7), File Structure (Ctrl/Cmd+F12), Type Hierarchy (Ctrl+H).
- **Рефакторинги**: Rename (Shift+F6), Extract Method/Variable/Constant/Field (Ctrl/Cmd+Alt+M/V/C/F), Inline, Change Signature,
  Move. Используй их вместо ручной правки — это Java-суперсила по сравнению с Python.
- **Генерация кода** (Alt+Insert / Cmd+N): конструкторы, getters, `equals/hashCode`, `toString`.
  Пользоваться можно, но **ты обязан понимать, что сгенерировано**. За `equals` по мутабельным полям в `HashSet` будет 🔴.
- **Инспекции**: жёлтые подсветки — это часто то, за что я буду ставить 🟡. *Code → Inspect Code* перед сдачей вехи.
- **Форматирование**: *Code → Reformat Code* (Ctrl/Cmd+Alt+L), *Optimize Imports*. Включи *Actions on Save* → reformat + optimize imports.
- **Плагины**: SonarLint (ловит баги и code smells), CheckStyle-IDEA (опционально), .ignore.

> ⚠️ **Важно:** на время обучения **отключи AI-автодополнение** (Full Line Code Completion, AI Assistant, Copilot и т. п.).
> *Settings → Editor → General → Inline Completion*. Иначе ты будешь учить не Java, а Tab.

---

## 6. Прочие инструменты (ставить по мере надобности)

| Инструмент | Когда понадобится | Зачем |
|------------|-------------------|-------|
| Git + корневой `.gitignore` | сразу | `target/`, `.idea/` не коммитим |
| Apache Tomcat 10.1 | проект 3 | Сервлет-контейнер (Jakarta EE 10, `jakarta.servlet`) |
| Docker (Desktop/Engine) + Compose | проекты 3–5 для Postgres — удобно; проекты 6–7 — обязательно | Базы и сервисы без установки в систему |
| DBeaver (или DB-клиент IDEA Ultimate) | проект 3 | Смотреть на таблицы, планы запросов |
| Postman / HTTPie / IDEA HTTP Client | проект 3 | Дёргать API |

---

## 7. Чекпоинт шага 0 → `projects/00-hello/`

Создай Maven-проект (через IDEA *New Project → Maven* или `mvn archetype:generate`, но потом приведи `pom.xml` в порядок сам):

**Критерии приёмки:**
- `pom.xml`: Java 21 через `maven.compiler.release`, JUnit 5 (через BOM `junit-bom`), AssertJ, актуальный
  `maven-surefire-plugin` (с JUnit 5 старый surefire тесты молча не находит — проверь, что тесты реально запускаются).
- Maven Wrapper (`mvn wrapper:wrapper`), сборка через `./mvnw verify` проходит.
- Один класс в пакете `dev.<ник>.hello` с нетривиальным методом (например, `StringStats`: частоты символов строки,
  сортированные по убыванию частоты, при равенстве — по символу) и тестами на него, включая граничные случаи
  (пустая строка, `null` — реши, как на него реагировать, и **обоснуй**).
- `README.md` проекта: как собрать и запустить.
- Запуск из терминала через `java -cp target/classes ...` и через `java -jar` (для этого настрой manifest в
  `maven-jar-plugin`).

**Что прислать:** «сдаю веху 0.1».

**Вопросы, на которые надо ответить до сдачи:**
1. Что лежит в `target/classes` и `target/test-classes`? Почему тестовые классы не попадают в JAR?
2. Откуда IDEA знает, какой JDK использовать? А Maven? А `java` в терминале?
3. Что случится, если в `pom.xml` указать `release 21`, а запустить `mvn` на JDK 17?
4. Почему JUnit в scope `test`?

## Контрольные вопросы шага 0

- JDK vs JRE vs JVM. Что делает `javac`, а что — JVM?
- Что такое байткод, JIT, «прогрев»?
- Что такое classpath? Как его задать? Как Maven формирует classpath для тестов?
- Жизненный цикл Maven: фазы, плагины, `package` vs `install` vs `verify`.
- Scope зависимостей, `provided`, транзитивные зависимости, «nearest wins».
- Где в IDEA задаётся SDK, language level, JDK для Maven? Почему источник правды — `pom.xml`?
