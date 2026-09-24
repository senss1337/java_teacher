# Шаг 0. Окружение: JDK, Maven, IntelliJ IDEA

**Зачем этот шаг.** В Python всё просто: поставил Python, создал `venv`, `pip install`, `python main.py`.
В Java между «написал код» и «программа работает» больше звеньев, и у каждого своё название.
Если не разобраться с ними сейчас, потом будут непонятные ошибки вида `ClassNotFoundException`
или «в IDEA работает, а из консоли нет». Шаг займёт 2–3 вечера.

**Что будет в конце:** Java 21 установлена, IntelliJ IDEA настроена, ты собрал и запустил первый проект
тремя способами (руками, через Maven, через IDEA) и понимаешь, что происходит на каждом этапе.

---

## 1. Как Java-программа запускается

### Простыми словами

Процессор не понимает ни Python, ни Java. Кто-то должен перевести код в понятные ему команды.

В **Python** ты пишешь `python main.py`. Интерпретатор читает файл, по-тихому компилирует его в байткод (`.pyc` в
`__pycache__`) и тут же исполняет. Ошибки типов ты увидишь, только когда дойдёшь до той строки.

В **Java** этих шагов два, и они явные:

1. **Компиляция.** Программа `javac` (Java Compiler) читает твои `.java`-файлы, **проверяет типы во всём коде сразу**
   и, если всё сходится, создаёт `.class`-файлы с **байткодом**. Если где-то в строке 500 передаёшь строку туда, где ждут число,
   программа не соберётся вообще.
2. **Запуск.** Программа `java` запускает **JVM** (Java Virtual Machine, виртуальную машину Java). JVM загружает `.class`-файлы
   и исполняет байткод.

```
 Ты пишешь        Компилятор              Файл с байткодом        Виртуальная машина
 Hello.java  ──>  javac Hello.java  ──>   Hello.class       ──>   java Hello
 (текст)          (проверка типов)        (не текст, не           (исполняет, горячий
                                           машинный код)           код ускоряет JIT)
```

**Зачем промежуточный байткод?** Он одинаковый для Windows, Linux и macOS. Скомпилировал один раз — запускаешь
на любой машине, где есть JVM. Это и есть знаменитое «write once, run anywhere».

**Что такое JIT.** JVM начинает с медленного построчного исполнения байткода (интерпретации) и параллельно считает,
какие методы вызываются чаще всего. Такие «горячие» методы она компилирует в настоящий машинный код, очень быстрый.
Отсюда «прогрев»: первые секунды или минуты Java-приложение работает медленнее, потом разгоняется. Это делает
**JIT-компилятор** (Just-In-Time — «точно вовремя»).

### Три аббревиатуры: JVM, JRE, JDK

Это просто вложенные «комплекты»:

- **JVM** — сама виртуальная машина, которая исполняет байткод.
- **JRE** (Java Runtime Environment) = JVM + стандартная библиотека (`String`, `List`, `HashMap`...). Этого достаточно,
  чтобы **запускать** Java-программы, например на сервере.
- **JDK** (Java Development Kit) = JRE + инструменты разработчика: компилятор `javac`, `jshell`, `jar`, отладчики.
  **Тебе нужен JDK.**

### Как в Python → как в Java

| В Python | В Java | Где аналогия ломается |
|----------|--------|-----------------------|
| CPython | JDK | В Java компиляция — отдельный обязательный шаг, и он проверяет типы |
| `pyenv` (несколько версий Python) | [SDKMAN!](https://sdkman.io/) | — |
| «Python Interpreter» в PyCharm | «Project SDK» в IDEA | В IDEA есть ещё **language level** — какие возможности языка разрешены |
| `venv` | **аналога нет** | См. ниже про classpath |
| `pip` + `pyproject.toml` | Maven + `pom.xml` | Maven ещё и компилирует, тестирует и упаковывает |
| `site-packages` | `~/.m2/repository` | Один общий кеш на все проекты, но конфликтов нет, потому что у каждого проекта свой classpath |
| `pytest` | `mvn test` | Тесты — встроенный этап сборки |
| `python` (REPL) | `jshell` | — |

### Classpath — почему в Java не нужен `venv`

`venv` в Python решает проблему: «проекту A нужна `requests 2.20`, проекту B — `2.31`, а `site-packages` один».

В Java такой проблемы нет, потому что JVM **не ищет библиотеки в глобальной папке**. При запуске ей явно передают
список: «вот папки и JAR-файлы, в которых лежат нужные классы». Этот список называется **classpath**.

```bash
# «Возьми классы из папки out и из файла gson.jar, запусти класс dev.me.App»
java -cp out:libs/gson-2.10.jar dev.me.App
```

Нет класса в classpath — JVM не найдёт его и упадёт с `ClassNotFoundException`. Руками этот список никто не пишет:
Maven собирает его сам из зависимостей в `pom.xml`, а IDEA берёт у Maven.

### Частые ошибки

- Поставить JRE вместо JDK: `java` есть, а `javac` нет.
- Иметь две Java на машине и не понимать, какую берёт терминал, какую Maven, какую IDEA. Разберём ниже.
- Держать класс не в той папке. В Java класс `dev.me.App` **обязан** лежать в `dev/me/App.java`, иначе не найдётся.

> 🎯 **Спросят на собесе:** Чем отличаются JDK, JRE и JVM?
> **Ответ:** JVM исполняет байткод (интерпретатор + JIT + сборщик мусора). JRE = JVM + стандартная библиотека, её достаточно для запуска.
> JDK = JRE + инструменты разработки (`javac`, `jar`, отладка, профилирование).
> **Типичная ошибка:** «JVM компилирует Java-код». Нет: исходник в байткод компилирует `javac`, а JVM компилирует байткод в машинный код (JIT).

> 🎯 **Спросят на собесе:** Что такое JIT и почему Java «прогревается»?
> **Ответ:** JVM стартует с интерпретации байткода, собирает статистику, горячие методы компилирует в оптимизированный
> машинный код (инлайнинг, escape analysis). До этого код работает медленнее.
> **Типичная ошибка:** путать JIT с AOT-компиляцией (GraalVM native-image), которая компилирует всё заранее.

> 🎯 **Спросят на собесе:** Что такое classpath и чем `ClassNotFoundException` отличается от `NoClassDefFoundError`?
> **Ответ:** Classpath — список директорий/JAR, где загрузчик классов ищет классы. `ClassNotFoundException` — checked-исключение
> при явной загрузке по имени (`Class.forName`), когда класса нет. `NoClassDefFoundError` — Error: при компиляции класс был,
> а в рантайме его нет (или он не смог инициализироваться).
> **Типичная ошибка:** считать их синонимами.

📚 **Читать** (по диагонали):
- CJ1, гл. 1 «An Introduction to Java» (история, «белая книга» — зачем JVM) и гл. 2 «The Java Programming Environment» (установка JDK, `javac`/`java`, `jshell`) — **весь шаг 0 идёт по ним**.
- [dev.java: Getting Started with Java](https://dev.java/learn/getting-started/) — официальный быстрый старт.
- [Oracle: The `java` Command](https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html) — пролистай, как задаётся classpath (`-cp`) и какие бывают режимы запуска.
- Для понимания JIT (по желанию): [dev.java: The JVM and Just-in-Time Compilation](https://dev.java/learn/jvm/) — раздел про JIT и HotSpot.

---

## 2. Установка JDK 21

### Какой JDK брать

Java — открытый проект **OpenJDK**. Из одного и того же исходного кода разные компании собирают свои
**дистрибутивы**: Eclipse Temurin, Amazon Corretto, Oracle JDK, Liberica, Zulu. По возможностям они почти одинаковы,
отличаются лицензией и сроком поддержки. Бери **Eclipse Temurin 21**: бесплатный, без лицензионных подвохов, стандарт де-факто.

**Почему 21?** Новая Java выходит каждые полгода, но в продакшене используют только **LTS**-версии
(Long-Term Support: 8, 11, 17, 21, 25), потому что их годами патчат. 21 — актуальная LTS со всеми современными фичами.

> 🎯 **Спросят на собесе:** Что такое LTS и почему в проде Java 17/21, а не 23?
> **Ответ:** Новая версия выходит каждые 6 месяцев. LTS-версии (8, 11, 17, 21, 25) получают многолетние патчи безопасности,
> промежуточные — только полгода. Прод живёт на LTS.

### macOS / Linux — через SDKMAN (рекомендую)

SDKMAN — это `pyenv` для Java: ставит несколько версий и переключает их.

```bash
curl -s "https://get.sdkman.io" | bash
# закрой и заново открой терминал
sdk list java | grep -i tem        # список доступных Temurin, ищи строку с 21
sdk install java 21.0.x-tem        # подставь точную версию из списка
sdk install maven                  # сразу и Maven
sdk current                        # что сейчас активно
```

SDKMAN сам настроит переменные `JAVA_HOME` и `PATH`. Переключить версию: `sdk use java <версия>`
(только в текущем окне терминала) или `sdk default java <версия>` (насовсем).

### Windows

- `winget install EclipseAdoptium.Temurin.21.JDK` или MSI-установщик с [adoptium.net](https://adoptium.net/).
  В установщике **отметь** *Set JAVA_HOME variable* и *Add to PATH*.
- Maven: `winget install Apache.Maven` или скачать архив, распаковать и добавить папку `bin` в `PATH`.

### Проверка — обязательно все четыре команды

```bash
java -version      # openjdk version "21.x" ... Temurin
javac -version     # javac 21.x  ← если команды нет, ты поставил JRE, а не JDK (или PATH смотрит не туда)
echo $JAVA_HOME    # на Windows: echo %JAVA_HOME%  — путь к папке JDK 21
mvn -version       # в выводе строка "Java version: 21..."
```

> ⚠️ **Типичная ловушка:** `java -version` показывает 21, а `mvn -version` показывает 17. Так бывает, потому что
> `java` в терминале находится через `PATH`, а Maven берёт JDK из переменной `JAVA_HOME`. Если они указывают на разные
> установки, получаешь две разные Java. Всегда проверяй обе команды.

---

## 3. Упражнение: Java без IDE (обязательно)

**Зачем.** IDEA делает всё это одной зелёной кнопкой. Но когда что-то сломается, ты должен понимать, что именно
IDE делала за тебя. Прогони руками один раз, это 30–40 минут.

Создай папку `projects/00-hello-manual/` и выполни по шагам:

1. **Самый простой запуск.** Создай `Hello.java`:
   ```java
   public class Hello {                               // имя класса = имя файла, это обязательно
       public static void main(String[] args) {       // точка входа, как if __name__ == "__main__"
           System.out.println("Args: " + String.join(", ", args));
       }
   }
   ```
   Выполни `javac Hello.java`: появится `Hello.class`. Открой его в текстовом редакторе и убедись, что это не текст.
   Запусти `java Hello a b c`. Обрати внимание: запускаешь **класс** `Hello`, а не файл `Hello.class`.

2. **Пакеты.** Добавь в начало файла `package dev.me.hello;` и перенеси файл в `src/dev/me/hello/Hello.java`.
   Скомпилируй с указанием папки для результата: `javac -d out src/dev/me/hello/Hello.java`.
   Посмотри, какая структура появилась в `out/`. Запусти: `java -cp out dev.me.hello.Hello`.
   Теперь попробуй `java Hello` и объясни себе, почему не работает.
   *Направление: где JVM ищет классы и под каким полным именем?*

3. **Запуск без компиляции.** `java src/dev/me/hello/Hello.java` — для одного файла Java умеет скомпилировать в памяти
   и сразу запустить. Удобно для скриптов, но для проектов не годится. Как думаешь, почему?

4. **JAR.** Собери исполняемый архив:
   `jar --create --file hello.jar --main-class dev.me.hello.Hello -C out .`
   и запусти его: `java -jar hello.jar x y`. Открой `hello.jar` как zip-архив и найди `META-INF/MANIFEST.MF`:
   там записано, какой класс запускать.

5. **Посмотри на байткод.** `javap -c -p out/dev/me/hello/Hello.class` — это «дизассемблер». Найди строку с вызовом
   `println` (`invokevirtual`). Понимать байткод не нужно, достаточно увидеть, что он существует.

6. **jshell.** Запусти `jshell` и попробуй:
   ```
   var list = List.of(1, 2, 3)
   list.add(4)                    // что случилось? почему?
   "abc".repeat(3)
   /vars
   /exit
   ```

---

## 4. Maven

### Простыми словами

Руками ты только что сделал 4 вещи: скомпилировал, собрал classpath, упаковал JAR, запустил. В настоящем проекте
к этому добавляются десятки библиотек со своими зависимостями, тесты и плагины. **Maven** делает всё это по описанию
из одного файла — `pom.xml` (Project Object Model).

Maven построен на **соглашениях** (convention over configuration): если класть файлы в стандартные папки,
настраивать почти ничего не надо.

```
my-project/
├── pom.xml                    ← описание проекта
├── src/main/java/             ← твой код (пакеты = подпапки)
├── src/main/resources/        ← ресурсы: конфиги, словари, SQL — попадают в JAR
├── src/test/java/             ← тесты (в JAR не попадают)
├── src/test/resources/        ← ресурсы только для тестов
└── target/                    ← всё, что сгенерировал Maven (в git не коммитим!)
```

### Как выглядит `pom.xml` — разбор по строкам

Это не решение задачи, а пример инструмента, поэтому показываю целиком. Не копируй бездумно: пойми каждую строку.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
  <modelVersion>4.0.0</modelVersion>          <!-- версия формата pom, всегда 4.0.0 -->

  <!-- Координаты проекта (GAV) — уникальный «адрес» артефакта -->
  <groupId>dev.myname</groupId>               <!-- «организация», обычно перевёрнутый домен -->
  <artifactId>hello</artifactId>              <!-- имя проекта -->
  <version>1.0-SNAPSHOT</version>             <!-- SNAPSHOT = «в разработке» -->
  <packaging>jar</packaging>                  <!-- что собирать: jar или war -->

  <properties>
    <maven.compiler.release>21</maven.compiler.release>        <!-- под какую Java компилировать -->
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <!-- BOM: один раз указываем версию JUnit, дальше версии не пишем -->
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.junit</groupId>
        <artifactId>junit-bom</artifactId>
        <version>5.11.0</version>             <!-- проверь актуальную на mvnrepository.com -->
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>                              <!-- «pip install», только декларативно -->
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>                     <!-- нужна только тестам, в JAR не попадёт -->
    </dependency>
  </dependencies>
</project>
```

Где искать библиотеки и их версии: [mvnrepository.com](https://mvnrepository.com/) или [central.sonatype.com](https://central.sonatype.com/).

### Scope — где нужна зависимость

- `compile` (по умолчанию) — нужна везде: при компиляции, тестах, в рантайме.
- `test` — только в тестах (JUnit, AssertJ, Mockito).
- `provided` — нужна для компиляции, но в рантайме её «предоставит» окружение. Пример из проекта 3: API сервлетов
  даст Tomcat, класть его в свой WAR нельзя, будет конфликт.
- `runtime` — для компиляции не нужна, нужна при запуске (например, JDBC-драйвер БД).

### Жизненный цикл: какие команды запускать

Maven работает **фазами**, которые идут строго по порядку. Запуск фазы выполняет все предыдущие:

```
validate → compile → test → package → verify → install → deploy
```

| Команда | Что происходит | Когда использовать |
|---------|----------------|--------------------|
| `mvn compile` | компилирует `src/main/java` в `target/classes` | проверить, что собирается |
| `mvn test` | + компилирует и запускает тесты | основная команда при разработке |
| `mvn package` | + упаковывает `target/*.jar` | нужен артефакт |
| `mvn verify` | + дополнительные проверки (интеграционные тесты и т. п.) | **перед сдачей вехи**, этой командой проверяю я |
| `mvn install` | + кладёт JAR в `~/.m2`, чтобы другие твои проекты могли его подключить | редко |
| `mvn clean` | удаляет `target/` | когда что-то «залипло»; можно `mvn clean verify` |

**Плагины** выполняют работу фаз: `maven-compiler-plugin` компилирует, `maven-surefire-plugin` запускает юнит-тесты,
`maven-jar-plugin` пакует. Версии плагинов тоже лучше указывать явно.

**Транзитивные зависимости.** Подключил библиотеку A, а ей нужна B — Maven скачает B сам. Посмотреть всё дерево:
`mvn dependency:tree`. Если двум библиотекам нужны разные версии третьей, Maven берёт ту, что **ближе к корню дерева**
(nearest wins), а не самую новую.

**Maven Wrapper** (`mvnw`) — скрипт в проекте, который скачивает конкретную версию Maven. Все, кто работает с проектом
(и CI), используют одну и ту же версию. Создаётся командой `mvn wrapper:wrapper`, дальше вместо `mvn` пишешь `./mvnw`.

### Частые ошибки

- JUnit 5 в проекте, а версия `maven-surefire-plugin` старая: тесты **молча не запускаются**, сборка зелёная.
  Всегда проверяй в выводе строку `Tests run: N`.
- `target/` в git.
- Версии зависимостей «на глаз» без BOM, в итоге конфликты.
- `source`/`target` вместо `release` в настройках компилятора. `release` дополнительно проверяет, что ты не используешь
  API стандартной библиотеки новее целевой версии.

📚 **Читать:**
- [Maven: The Complete Reference](https://books.sonatype.com/mvnref-book/reference/) (Sonatype, бесплатно онлайн): гл. 3 «The Project Object Model», гл. 4 «The Build Lifecycle» — по диагонали.
- [Introduction to the Dependency Mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html) — scope, транзитивность, «nearest wins». **Обязательно**.
- [Maven in 5 Minutes](https://maven.apache.org/guides/getting-started/maven-in-five-minutes.html), [Maven Wrapper](https://maven.apache.org/wrapper/).

> 🎯 **Спросят на собесе:** Чем `mvn package` отличается от `mvn install`?
> **Ответ:** `package` собирает артефакт в `target/`. `install` дополнительно кладёт его в локальный репозиторий `~/.m2`,
> чтобы другие локальные проекты могли подключить его как зависимость.

> 🎯 **Спросят на собесе:** Какие бывают scope зависимостей и зачем `provided`?
> **Ответ:** `compile`, `provided`, `runtime`, `test` (и служебные `system`, `import`). `provided` — нужна для компиляции,
> но в рантайме её предоставит окружение (servlet API в Tomcat), поэтому в WAR не кладётся.

> 🎯 **Спросят на собесе:** Как разрешается конфликт версий транзитивных зависимостей в Maven?
> **Ответ:** «Nearest wins»: побеждает версия, ближайшая к корню дерева, при равной глубине — объявленная первой.
> Управлять явно — через `<dependencyManagement>`/BOM или `<exclusions>`.
> **Типичная ошибка:** «берётся самая новая» — так работает Gradle, не Maven.

---

## 5. IntelliJ IDEA

### Какую редакцию ставить

- **Community** (бесплатная) — достаточно для проектов 0–3.
- **Ultimate** удобнее для Spring, баз данных, HTTP-запросов, Tomcat. Бесплатна для студентов
  ([JetBrains Education](https://www.jetbrains.com/community/education/)), есть триал. **Не обязательна**: всё можно
  сделать в Community + DBeaver + Postman.

Ставь через [JetBrains Toolbox](https://www.jetbrains.com/toolbox-app/): он сам обновляет IDE.

### Где в IDEA «интерпретатор» (самое важное)

В PyCharm одна настройка — интерпретатор. В IDEA их несколько, и они должны совпадать:

1. **Project SDK** — какой JDK использовать для компиляции и запуска.
   *File → Project Structure → Project → SDK*. Если JDK ещё нет, выбери *Add SDK → Download JDK → Temurin 21*,
   IDEA скачает его сама.
2. **Language level** — там же, строкой ниже. Какие возможности языка разрешены. Должно быть «21».
3. **Module SDK** — *Project Structure → Modules*: SDK для отдельного модуля. Обычно стоит *Project SDK*, не трогай.
4. **JDK для Maven** — *Settings → Build, Execution, Deployment → Build Tools → Maven → Runner → JRE*.
   Каким JDK IDEA запускает Maven. Поставь Project SDK.

**Главное правило: источник правды — `pom.xml`.** Когда IDEA открывает Maven-проект, она читает `pom.xml`
и выставляет language level по `maven.compiler.release`. Если в IDEA всё работает, а `./mvnw verify` в терминале
падает, прав терминал. Я на ревью собираю через `mvn`.

### Пошагово: открыть проект

1. *File → Open* → выбери **папку**, в которой лежит `pom.xml` (не сам файл и не *New Project*).
2. IDEA спросит *Trust project?* → *Trust*.
3. Справа появится вкладка **Maven**: там фазы жизненного цикла (двойной клик = запуск) и кнопка ⟳
   *Reload All Maven Projects*. Нажимай её каждый раз после правки `pom.xml`.
4. Открой класс с `main` → зелёный треугольник у метода → *Run*. IDEA создаст **Run Configuration**: её можно
   отредактировать (*Run → Edit Configurations*), задать аргументы программы и переменные окружения.
   **Секреты (пароли, API-ключи) передаются только так — через переменные окружения, не через код.**

### Отладчик — учись с первого дня

В Python ты, возможно, обходился `print`. В Java отладчик — основной инструмент, особенно когда дойдёшь до Spring.

- Клик по полю слева от номера строки ставит **breakpoint** (красную точку). Запусти через жука (*Debug*), а не *Run*.
- Программа остановится на точке. Внизу видны все переменные.
- *Step Over* (F8) — следующая строка; *Step Into* (F7) — зайти внутрь метода; *Step Out* (Shift+F8) — выйти.
- *Evaluate Expression* (Alt+F8) — выполнить любое выражение в текущем контексте.
- Правый клик по точке → *Condition*: останавливаться, только если условие истинно (например, `i == 500`).

### Навигация и рефакторинги — суперсила Java

Статическая типизация позволяет IDEA **точно** знать, где что используется. Поэтому рефакторинги в Java
безопасны, в отличие от Python. Выучи первыми (macOS — Cmd вместо Ctrl):

| Действие | Клавиши | Зачем |
|----------|---------|-------|
| Найти что угодно | Shift, Shift | класс, файл, настройка |
| Перейти к объявлению | Ctrl+B / Ctrl+клик | куда ведёт этот метод |
| Перейти к реализации | Ctrl+Alt+B | какие классы реализуют интерфейс |
| Где используется | Alt+F7 | кто вызывает метод |
| Переименовать | Shift+F6 | везде сразу, безопасно |
| Извлечь метод / переменную / константу | Ctrl+Alt+M / V / C | рефакторинг |
| Сгенерировать код | Alt+Insert (Cmd+N) | конструктор, геттеры, `equals/hashCode` |
| Отформатировать | Ctrl+Alt+L | стиль кода |
| Показать подсказки/исправления | Alt+Enter | главная клавиша IDEA |

**Генерацией кода** (конструкторы, `equals/hashCode`) пользоваться можно, но ты обязан понимать, что сгенерировано.
Например, `equals` по изменяемым полям у объекта, который лежит в `HashSet`, получит 🔴 на ревью.

**Инспекции.** Жёлтая подсветка в коде — это IDEA указывает на проблему, и часто это то, за что я поставлю 🟡.
Перед сдачей вехи: *Code → Inspect Code*.

**Настрой один раз:** *Settings → Tools → Actions on Save* → включи *Reformat code* и *Optimize imports*.
Плагины (*Settings → Plugins*): **SonarLint** (находит баги и плохие места), **.ignore**.

> ⚠️ **Важно:** на время обучения **выключи AI-автодополнение** (Full Line Code Completion, AI Assistant, Copilot).
> *Settings → Editor → General → Inline Completion*. Иначе ты будешь учить не Java, а клавишу Tab.

📚 **Читать:**
- [IntelliJ IDEA: Getting started](https://www.jetbrains.com/help/idea/getting-started.html) → разделы *Create your first Java application* и *SDKs* (про Project SDK и language level).
- [Maven support in IntelliJ IDEA](https://www.jetbrains.com/help/idea/maven-support.html).
- [Debug code](https://www.jetbrains.com/help/idea/debugging-code.html) — разделы *Breakpoints* и *Evaluate expressions*.
- Шпаргалка хоткеев: *Help → Keyboard Shortcuts PDF*. Распечатай, первые 2 недели держи рядом.

---

## 6. Прочие инструменты (ставить, когда понадобятся)

| Инструмент | Когда | Зачем |
|------------|-------|-------|
| Git + корневой `.gitignore` (уже есть в репо) | сразу | `target/`, `.idea/` не коммитим |
| Apache Tomcat 10.1 | проект 3 | сервер для сервлетов |
| Docker Desktop / Docker Engine | удобно с проекта 3 (Postgres), обязательно с проекта 6 | базы и сервисы без установки в систему |
| DBeaver (или DB-клиент в IDEA Ultimate) | проект 3 | смотреть таблицы и планы запросов |
| Postman / HTTPie / HTTP Client в IDEA | проект 3 | дёргать API руками |

---

## 7. Чекпоинт шага 0 → веха 0.1: `projects/00-hello/`

### Что нужно сделать
1. Создать Maven-проект `projects/00-hello/`: в IDEA *New Project → Java, Build system: Maven, JDK: 21*.
   Сгенерированный `pom.xml` приведи в порядок по образцу из раздела 4.
2. Подключить JUnit 5 (через BOM) и AssertJ в scope `test`, указать явную свежую версию `maven-surefire-plugin`.
3. Добавить Maven Wrapper: `mvn wrapper:wrapper`.
4. Написать класс `StringStats` в пакете `dev.<ник>.hello`: метод принимает строку и возвращает частоты символов,
   отсортированные по убыванию частоты, при равенстве — по символу.
5. Написать на него тесты, включая граничные случаи.
6. Сделать запуск через `java -jar`: у метода `main` есть класс, в `maven-jar-plugin` настроен `mainClass` в манифесте.
7. Написать `README.md` проекта: как собрать и как запустить.

### С чего начать
Создай проект в IDEA, открой `pom.xml` и сверь каждую строку с разбором выше. Потом напиши **один** простой тест
(например, `assertThat(1 + 1).isEqualTo(2)`) и добейся, чтобы `./mvnw test` в терминале показал `Tests run: 1`.
Только после этого переходи к `StringStats`.

### Вопросы для размышления
1. Что лежит в `target/classes`, а что в `target/test-classes`? Почему тестовые классы не попадают в JAR?
   *Направление: открой `target/` после `./mvnw package` и распакуй JAR.*
2. Откуда IDEA знает, какой JDK использовать? А Maven? А команда `java` в терминале?
   *Направление: раздел «Где в IDEA интерпретатор» и «Типичная ловушка» с `JAVA_HOME`.*
3. Что случится, если в `pom.xml` указать `release 21`, а запустить Maven на JDK 17? Попробуй, если есть SDKMAN.
4. Почему JUnit в scope `test`? Что изменится в JAR, если убрать scope?
5. Что вернуть для пустой строки и что делать с `null`? Реши и **обоснуй** в README.
   *Направление: что удобнее вызывающему — пустой результат или исключение? Когда `null` — это ошибка программиста?*

### Как проверить себя
- [ ] `./mvnw verify` проходит, в выводе `Tests run: N, Failures: 0`, где N > 0.
- [ ] `java -jar target/hello-1.0-SNAPSHOT.jar "hello world"` печатает статистику.
- [ ] `java -cp target/classes dev.<ник>.hello.<Main>` тоже работает.
- [ ] В git нет `target/` и `.idea/` (`git status` чистый после сборки).
- [ ] Тесты покрывают: обычную строку, пустую, одинаковые частоты (проверка сортировки по символу), твоё решение по `null`.

### Что прислать
Закоммить и напиши «сдаю веху 0.1».

## Контрольные вопросы шага 0

1. JDK vs JRE vs JVM. Что делает `javac`, а что JVM?
2. Что такое байткод, JIT, «прогрев»?
3. Что такое classpath? Как его задать? Как Maven формирует classpath для тестов?
4. Жизненный цикл Maven: фазы, плагины, `package` vs `install` vs `verify`.
5. Scope зависимостей, `provided`, транзитивные зависимости, «nearest wins».
6. Где в IDEA задаются SDK, language level, JDK для Maven? Почему источник правды — `pom.xml`?
