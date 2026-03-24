# Скрипты и кастомизация в Gradle [#505353]

Gradle предоставляет гибкие механизмы для настройки и кастомизации процесса сборки, чтобы удовлетворять специфические требования проекта. Ниже описаны основные способы кастомизации:

## 1. Создание собственных задач

Gradle позволяет создавать собственные задачи для выполнения специфических действий, которых нет в стандартных плагинах.

Пример создания задачи:

```kotlin
tasks.register("hello") {
    group = "Custom Tasks" // Группа для удобства в `./gradlew tasks`
    description = "Prints a hello message"
    doLast {
        println("Hello from Gradle!")
    }
}
```

Объяснение:

- `tasks.register` создаёт задачу с ленивой инициализацией.
- `doLast` указывает действие, которое выполнится, когда задача будет вызвана.
- Добавление группы и описания делает задачу понятной в списке всех задач.

Вы можете комбинировать задачи:

```kotlin
tasks.register("cleanReports") {
    group = "Cleanup"
    description = "Deletes all report files"
    doLast {
        file("build/reports").deleteRecursively()
        println("Reports deleted.")
    }
}
```

## 2. Переиспользование кода через ext и buildSrc

### a) ext (extensions)

`ext` используется для определения переменных, которые могут быть переиспользованы в скриптах.

Пример с `ext` в Kotlin DSL:

```kotlin
extra["projectVersion"] = "1.0.0"
extra["commonsLangVersion"] = "3.17.0"

dependencies {
    implementation("org.apache.commons:commons-lang3:${extra["commonsLangVersion"]}")
}
```

Преимущества:

- Упрощает поддержку версий зависимостей.
- Удобно для небольших проектов.

> **Примечание:** для управления версиями зависимостей рекомендуется использовать каталоги версий (`libs.versions.toml`), как описано в уроке «Работа с зависимостями». `ext` и `buildSrc` больше подходят для переиспользования логики сборки.

### b) buildSrc

Для более сложных конфигураций Gradle рекомендует использовать директорию `buildSrc`, которая позволяет структурировать повторно используемый код.

Шаги:

1. Создайте папку `buildSrc` в корне проекта.
2. Добавьте файл Kotlin для описания логики, например `DependencyVersions.kt`:

```kotlin
object DependencyVersions {
    const val COMMONS_LANG = "3.17.0"
}
```

3. Используйте эту логику в `build.gradle.kts`:

```kotlin
dependencies {
    implementation("org.apache.commons:commons-lang3:${DependencyVersions.COMMONS_LANG}")
}
```

Преимущества:

- Удобно для крупных проектов с большим количеством повторяющегося кода.
- Автодополнение в IDE.

## 3. Настройка параметров проекта

### a) Передача параметров через командную строку

Вы можете передать параметры в проект с помощью `-P`:

```
./gradlew build -Penv=dev
```

Для их использования в `build.gradle.kts`:

```kotlin
val environment = project.findProperty("env") ?: "prod"
println("Building for environment: $environment")
```

### b) Профили (Environments)

Для разных окружений (например, `dev`, `test`, `prod`) можно создавать отдельные конфигурации:

```kotlin
val isDev = project.hasProperty("dev")
if (isDev) {
    println("Development environment detected!")
}
```

### c) Настройка JVM аргументов

Для настройки JVM параметров используйте:

```kotlin
tasks.withType<JavaExec> {
    jvmArgs("-Xms512m", "-Xmx1024m")
}
```

### d) Настройка путей

Вы можете кастомизировать пути к файлам и директориям:

```kotlin
layout.buildDirectory.set(file("customBuildDir"))
```

## Задание

Добавьте кастомную задачу для проверки размера JAR-файла.

**Условие:**

Добавьте в ваш `build.gradle.kts` кастомную задачу, которая проверяет размер сгенерированного JAR-файла и выводит предупреждение, если его размер превышает 5 MB.

**Требования:**

- Группа задачи: `verification`.
- Описание задачи: добавьте описание, чтобы она отображалась в списке `./gradlew tasks`.
- Логика проверки: после выполнения задачи `jar`, задача должна проверить размер файла `build/libs/имя-jar-файла.jar`. Если размер превышает 5 MB, выводите предупреждение: "JAR file exceeds the size limit of 5 MB". Если размер меньше или равен 5 MB, выводите сообщение: "JAR file is within the acceptable size limit".

**Ожидаемая реализация:**

```kotlin
tasks.register("checkJarSize") {
    group = "verification"
    description = "Checks the size of the generated JAR file."

    dependsOn("jar")

    doLast {
        val jarFile = file("build/libs/${project.name}-${project.version}.jar")
        if (jarFile.exists()) {
            val sizeInMB = jarFile.length() / (1024 * 1024)
            if (sizeInMB > 5) {
                println("WARNING: JAR file exceeds the size limit of 5 MB. Current size: ${sizeInMB} MB")
            } else {
                println("JAR file is within the acceptable size limit. Current size: ${sizeInMB} MB")
            }
        } else {
            println("JAR file not found. Please make sure the build process completed successfully.")
        }
    }
}
```

Загрузите изменения в репозиторий. Отправьте ссылку на проверку.
