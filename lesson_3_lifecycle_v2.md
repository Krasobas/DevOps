# Жизненный цикл сборки Gradle [#505351]

Жизненный цикл сборки в Gradle разделён на три ключевые фазы:

## 1. Initialization (Инициализация)

**Что происходит:**

- Определяется, какие проекты включены в сборку (особенно важно для многомодульных проектов).
- Создаётся объект `Project` для каждого включённого проекта.

Пример: в многомодульной сборке можно определить, какие модули участвуют в процессе:

```kotlin
// settings.gradle.kts
rootProject.name = "MyMultiModuleProject"
include("moduleA", "moduleB")
```

При запуске Gradle сначала будет определено, что есть `rootProject` и два дочерних проекта: `moduleA` и `moduleB`.

## 2. Configuration (Конфигурация)

**Что происходит:**

- Конфигурируются все проекты, а также задачи, которые в них определены.
- Все секции `build.gradle.kts` выполняются, чтобы настроить проекты.
- **Ключевой момент:** конфигурация выполняется до фактического выполнения задач.

Пример: можно настроить общие параметры для всех модулей и определить зависимости:

```kotlin
// build.gradle.kts (корневой проект)
subprojects {
    group = "com.example"
    version = "1.0"

    tasks.register("hello") {
        doLast {
            println("Hello from $project.name")
        }
    }
}
```

## 3. Execution (Исполнение)

**Что происходит:**

- Gradle определяет, какие задачи нужно выполнить, в каком порядке, и затем запускает их.
- Задачи могут иметь зависимости, определяемые методом `dependsOn`.

Пример: исполняются задачи с учётом их зависимости:

```kotlin
// build.gradle.kts
tasks.register("prepare") {
    doLast {
        println("Preparing...")
    }
}

tasks.register("buildApp") {
    dependsOn("prepare") // Задача buildApp зависит от prepare
    doLast {
        println("Building the application")
    }
}

tasks.register("deploy") {
    dependsOn("buildApp")
    doLast {
        println("Deploying the application")
    }
}
```

Команда для выполнения:

```
./gradlew deploy
```

Вывод:

```
> Task :prepare
Preparing...
> Task :buildApp
Building the application
> Task :deploy
Deploying the application
```

## Взаимодействие задач

Gradle предоставляет гибкие механизмы управления задачами.

### Зависимости между задачами

Используйте `dependsOn` для указания порядка выполнения задач.

```kotlin
tasks.register("taskA") {
    doLast {
        println("Executing Task A")
    }
}

tasks.register("taskB") {
    dependsOn("taskA")
    doLast {
        println("Executing Task B")
    }
}
```

При вызове `./gradlew taskB` сначала выполнится `taskA`, затем `taskB`.

### Условное выполнение

Используйте `onlyIf` для выполнения задачи при определённых условиях.

```kotlin
tasks.register("conditionalTask") {
    onlyIf {
        project.hasProperty("runCondition")
    }
    doLast {
        println("Conditional Task executed")
    }
}
```

Запуск:

```
./gradlew conditionalTask -PrunCondition
```

### Обработка общего состояния

Задачи могут обмениваться данными через объекты `ext`.

```kotlin
tasks.register("initializeData") {
    doLast {
        extensions.extraProperties["sharedValue"] = "Shared data"
        println("Data initialized")
    }
}

tasks.register("useData") {
    dependsOn("initializeData")
    doLast {
        val sharedValue = extensions.extraProperties["sharedValue"]
        println("Using data: $sharedValue")
    }
}
```

## Стандартные задачи Gradle для Java

Gradle для Java-проектов предоставляет набор стандартных задач, упрощающих процесс сборки, тестирования и развёртывания приложения. Эти задачи входят в состав плагина `java`.

### Основные задачи

**clean** — удаляет сгенерированные артефакты сборки из директории `build/`.

```
./gradlew clean
```

**compileJava** — компилирует исходный код Java в байт-код. Исходники по умолчанию находятся в папке `src/main/java`. Скомпилированные файлы помещаются в `build/classes/java/main`.

```
./gradlew compileJava
```

**processResources** — копирует ресурсы из папки `src/main/resources` в `build/resources/main`. Ресурсы могут быть такими файлами, как `.properties`, `.xml`, `.yml`, которые нужны приложению.

**classes** — выполняет `compileJava` и `processResources`. Создаёт готовую к использованию структуру классов и ресурсов для запуска или упаковки приложения.

```
./gradlew classes
```

**jar** — упаковывает скомпилированные классы и ресурсы в файл `.jar`. Файл `.jar` создаётся в папке `build/libs`.

Пример настройки в Kotlin DSL:

```kotlin
tasks.jar {
    manifest {
        attributes["Main-Class"] = "com.example.Main"
    }
}
```

```
./gradlew jar
```

**test** — выполняет тесты, находящиеся в папке `src/test/java`. По умолчанию использует JUnit или TestNG (если они добавлены в зависимости). Отчёты о тестах генерируются в `build/reports/tests`.

Пример добавления зависимости для тестов:

```kotlin
dependencies {
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.4")
}
```

```
./gradlew test
```

**check** — выполняет все проверки качества кода, включая тесты (включает задачу `test`). Полезно для интеграции с системами CI/CD.

```
./gradlew check
```

**build** — выполняет полный процесс сборки, включая `compileJava`, `processResources`, `classes`, `test`, `jar`. В результате создаётся готовый `.jar`-файл и выполняются тесты.

```
./gradlew build
```

### Пример Gradle файла для Java-проекта

```kotlin
plugins {
    id("java") // Плагин для работы с Java
}

group = "com.example"
version = "1.0"

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.apache.commons:commons-lang3:3.17.0")
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.4")
}

tasks.test {
    useJUnitPlatform() // Настройка для JUnit 5
}
```

### Стандартный рабочий процесс

Очистить проект:

```
./gradlew clean
```

Скомпилировать исходный код:

```
./gradlew compileJava
```

Запустить тесты:

```
./gradlew test
```

Собрать `.jar`-файл:

```
./gradlew build
```

### Дополнительные задачи

**javadoc** — генерирует документацию по исходному коду Java:

```
./gradlew javadoc
```

**dependencies** — отображает дерево зависимостей проекта:

```
./gradlew dependencies
```

**tasks** — показывает список всех доступных задач:

```
./gradlew tasks
```

## Задание

Добавьте в `build.gradle.kts` задачу для архивации JavaDoc.

```kotlin
tasks.register<Zip>("zipJavaDoc") {
    group = "documentation"
    description = "Packs the generated Javadoc into a zip archive"

    dependsOn("javadoc")

    from("build/docs/javadoc")
    archiveFileName.set("javadoc.zip")
    destinationDirectory.set(layout.buildDirectory.dir("archives"))
}
```

Опишем содержимое скрипта:

- `dependsOn("javadoc")` — указывает, что перед запуском задачи `zipJavaDoc` обязательно должна быть выполнена задача `javadoc`.
- `from("build/docs/javadoc")` — указывает, какие файлы будут упакованы в архив (в данном случае папка с Javadoc).
- `archiveFileName.set("javadoc.zip")` — устанавливает имя создаваемого архива.
- `destinationDirectory.set(...)` — указывает, где сохранить итоговый архив. В примере это `build/archives`.

Запустите задачу и поправьте ошибки. Ошибки будут связаны с отсутствием описания классов.

Загрузите изменения и отправьте ссылку на проверку.
