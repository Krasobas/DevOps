# Создание проекта на Gradle [#505349]

В этом уроке мы установим Gradle, создадим каркас проекта со сборкой Gradle, а также рассмотрим основные компоненты скрипта сборки и её этапы.

## Установка Gradle

Для установки Gradle рекомендуется использовать менеджер версий SDKMAN. Он позволяет устанавливать и переключаться между версиями Java, Gradle, Maven и других инструментов JVM-экосистемы без использования `sudo` и без конфликтов с системными пакетами.

### Установка через SDKMAN (рекомендуемый способ)

Работает на Linux и macOS. Требуется `bash` или `zsh` и `curl`.

```
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install gradle
```

Проверка:

```
gradle --version
```

SDKMAN также позволяет установить конкретную версию:

```
sdk install gradle 8.14
```

Посмотреть доступные версии:

```
sdk list gradle
```

### Ручная установка

Если вы не хотите использовать SDKMAN, можно установить Gradle вручную.

**Linux / macOS:**

```
wget https://services.gradle.org/distributions/gradle-8.14-bin.zip
unzip gradle-8.14-bin.zip -d /opt/gradle
```

Добавьте в `~/.bashrc` (или `~/.zshrc`):

```
export PATH="/opt/gradle/gradle-8.14/bin:$PATH"
```

Примените изменения:

```
source ~/.bashrc
```

**Windows:**

1. Скачайте архив с https://gradle.org/releases/
2. Распакуйте, например, в `C:\gradle\gradle-8.14`
3. Добавьте `C:\gradle\gradle-8.14\bin` в переменную окружения `PATH`

### Примечание о Gradle Wrapper

После создания проекта через `gradle init` в нём появится файл `gradlew` (и `gradlew.bat` для Windows). Это Gradle Wrapper — скрипт, который запускает Gradle нужной версии без необходимости глобальной установки.

Поэтому глобально Gradle нужно установить только один раз — для создания первого проекта. После этого в рабочих проектах используйте `./gradlew` (Linux/macOS) или `gradlew.bat` (Windows).

## Создание проекта

Создайте папку для проекта:

**Linux / macOS:**

```
mkdir -p ~/projects/hello_gradle && cd ~/projects/hello_gradle
```

**Windows:**

```
mkdir c:\projects\hello_gradle && cd c:\projects\hello_gradle
```

Выполните команду создания проекта:

```
gradle init
```

Выберите следующие параметры при инициализации:

- Тип проекта: Application
- Язык: Java
- Язык для настройки сборки: Kotlin DSL
- Тестовая библиотека: JUnit Jupiter

Пример инициализации проекта:

```
gradle init

Select type of build to generate:
  1: Application
  2: Library
  3: Gradle plugin
  4: Basic (build structure only)
Enter selection (default: Application) [1..4] 1

Select implementation language:
  1: Java
  2: Kotlin
  3: Groovy
  4: Scala
  5: C++
  6: Swift
Enter selection (default: Java) [1..6] 1

Enter target Java version (min: 7, default: 21):

Project name (default: hello_gradle):

Select application structure:
  1: Single application project
  2: Application and library project
Enter selection (default: Single application project) [1..2] 1

Select build script DSL:
  1: Kotlin
  2: Groovy
Enter selection (default: Kotlin) [1..2] 1

Select test framework:
  1: JUnit 4
  2: TestNG
  3: Spock
  4: JUnit Jupiter
Enter selection (default: JUnit Jupiter) [1..4] 4

Generate build using new APIs and behavior (some features may change in the next minor release)? (default: no) [yes, no] yes


> Task :init
Learn more about Gradle by exploring our Samples at https://docs.gradle.org/8.14/samples/sample_building_java_applications.html

BUILD SUCCESSFUL in 21s
```

## Структура проекта

После создания проекта структура будет выглядеть так:

```
│   .gitattributes
│   .gitignore
│   gradle.properties
│   gradlew
│   gradlew.bat
│   settings.gradle.kts
│
├───app
│   │   build.gradle.kts
│   │
│   └───src
│       ├───main
│       │   ├───java
│       │   │   └───org
│       │   │       └───example
│       │   │               App.java
│       │   │
│       │   └───resources
│       └───test
│           ├───java
│           │   └───org
│           │       └───example
│           │               AppTest.java
│           │
│           └───resources
└───gradle
    │   libs.versions.toml
    │
    └───wrapper
            gradle-wrapper.jar
```

## Gradlew

`gradlew` нужен для того, чтобы запускать Gradle в проекте, даже если Gradle не установлен на вашем компьютере. Это рекомендуемый способ запуска Gradle в проектах.

Такой же механизм поддерживается и Maven. Главный элемент в Maven — это файл `pom.xml`. В Gradle — это файл `settings.gradle.kts`.

Файл `settings.gradle.kts` используется для настройки проекта и управления его модулями. Это основной файл конфигурации, который Gradle загружает на этапе инициализации сборки.

```kotlin
rootProject.name = "hello_gradle"
include("app")
```

Метод `include` используется для подключения модулей. Каждый модуль должен содержать в корне файл `build.gradle.kts`.

Файл `build.gradle.kts` — это основной файл сборки Gradle, написанный на Kotlin DSL. Он используется для конфигурации сборочного процесса вашего проекта, включая подключение зависимостей, плагинов, настроек задач и других параметров.

```kotlin
plugins {
    application
}

repositories {
    mavenCentral()
}

dependencies {
    implementation(libs.guava)
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

application {
    mainClass = "org.example.App"
}
```

Разберём элементы указанного скрипта `build.gradle.kts`:

### 1. plugins

```kotlin
plugins {
    application
}
```

Раздел для подключения плагинов, расширяющих функциональность Gradle.

`application` — плагин, предназначенный для создания и запуска Java-приложений. Он добавляет задачи для упаковки приложения в JAR-файл и запуска класса `main`.

### 2. repositories

```kotlin
repositories {
    mavenCentral()
}
```

Указывает репозитории, откуда Gradle будет загружать зависимости.

`mavenCentral()` — подключает репозиторий Maven Central, один из наиболее популярных хостингов библиотек Java.

### 3. dependencies

```kotlin
dependencies {
    implementation(libs.guava)
}
```

Определяет зависимости проекта, то есть подключаемые библиотеки.

`implementation` — указывает, что библиотека необходима для выполнения приложения, но её внутренние API не видны пользователям вашего кода.

`libs.guava` — использует зависимость `guava` из Gradle Version Catalog, которая управляется через файл `libs.versions.toml`.

### 4. java

```kotlin
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

Раздел для настройки JVM и компиляции Java.

`toolchain` — позволяет указать версию Java, которая должна использоваться для компиляции и выполнения кода.

`languageVersion = JavaLanguageVersion.of(21)` — указывает использование Java 21. Gradle автоматически загрузит подходящий JDK, если он не установлен локально.

### 5. application

```kotlin
application {
    mainClass = "org.example.App"
}
```

Конфигурация плагина `application`.

`mainClass` — указывает путь к основному классу, содержащему метод `main`. Этот класс будет запускаться при выполнении команды `./gradlew run`.

`org.example.App` — полное имя класса (FQCN — Fully Qualified Class Name), включая пакет `org.example` и класс `App`.

## Основные этапы сборки

1. **Инициализация (Initialization Phase):** Gradle считывает `settings.gradle.kts` и устанавливает имя и структуру проекта.

2. **Конфигурация (Configuration Phase):** Gradle анализирует `build.gradle.kts`, загружает зависимости, плагины и определяет задачи.

3. **Исполнение (Execution Phase):** Выполняются задачи, такие как `build`, `test` и `run`, в зависимости от команды.

## Основные команды для работы с проектом

Сборка проекта (создаёт JAR-файл и запускает тесты):

```
./gradlew build
```

Запуск приложения (выполняет Main класс):

```
./gradlew run
```

Запуск тестов:

```
./gradlew test
```

Очистка проекта:

```
./gradlew clean
```

> На Windows вместо `./gradlew` используйте `gradlew.bat`.

## Задание

1. Возьмите проект https://github.com/peterarsentev/job4j_elementary

2. Создайте новый репозиторий `job4j_elementary_gradle`.

3. Замените сборку Maven на Gradle.

**Подсказка:** если выполнить `gradle init` в директории, где уже есть `pom.xml`, Gradle обнаружит его и предложит сконвертировать проект автоматически:

```
Found a Maven build. Generate a Gradle build from this? (default: yes) [yes, no]
```

Gradle прочитает `pom.xml` и перенесёт зависимости и настройки в `build.gradle.kts`. После конвертации проверьте результат — автоматическая конвертация не всегда переносит всё корректно. В частности, убедитесь что:

- В блоке `tasks.test` есть `useJUnitPlatform()` (если тесты используют JUnit 5).
- В зависимостях есть `testRuntimeOnly("org.junit.platform:junit-platform-launcher:...")` — в Gradle 9.x это требуется явно.
- Сборка проходит: `./gradlew build`.

После успешной сборки файл `pom.xml` можно удалить.

4. Загрузите изменения в репозиторий и отправьте ссылку на фиксацию изменений.
