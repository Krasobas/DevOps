# Работа с зависимостями в Gradle [#505350]

Gradle предоставляет мощный инструмент для управления зависимостями проекта. Зависимости — это сторонние библиотеки или модули, которые проект использует в процессе разработки.

Если вы работали с Maven, то привыкли искать зависимости на https://search.maven.org и вставлять блок `<dependency>` в `pom.xml`. В Gradle процесс тот же самый — вы ищете библиотеку на Maven Central, берёте её координаты и подключаете. Репозиторий один и тот же, меняется только синтаксис.

## 1. Определение зависимостей в Gradle

Зависимости определяются в файле `build.gradle.kts`. Есть два способа указать зависимость — напрямую или через каталог версий.

### Способ 1: Напрямую в build.gradle.kts

Самый простой способ. Вы указываете координаты библиотеки прямо в блоке `dependencies`:

```kotlin
dependencies {
    implementation("com.google.code.gson:gson:2.11.0")
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.4")
}
```

Структура записи — `"group:artifact:version"`:

- `group` — группа или организация, выпускающая библиотеку.
- `artifact` — название артефакта (библиотеки).
- `version` — номер версии библиотеки.

Это аналог Maven-записи:

```xml
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.11.0</version>
</dependency>
```

### Способ 2: Через каталог версий (libs.versions.toml)

Когда зависимостей становится много, удобнее хранить их версии в одном месте. Для этого создаётся файл `gradle/libs.versions.toml`:

```toml
[versions]
gson = "2.11.0"
junit = "5.11.4"

[libraries]
gson = { module = "com.google.code.gson:gson", version.ref = "gson" }
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit" }
```

А в `build.gradle.kts` вместо координат пишете ссылку через `libs.`:

```kotlin
dependencies {
    implementation(libs.gson)
    testImplementation(libs.junit.jupiter)
}
```

**Как это работает:** при запуске Gradle читает файл `libs.versions.toml` и создаёт объект `libs`. Каждая запись в секции `[libraries]` становится полем этого объекта. Дефисы в именах заменяются на точки:

```
gson           →  libs.gson
junit-jupiter  →  libs.junit.jupiter
assertj-core   →  libs.assertj.core
```

**Когда что использовать:**

- Маленький проект, пара зависимостей — пишите напрямую в `build.gradle.kts`.
- Много зависимостей или многомодульный проект — используйте `libs.versions.toml`.
- Если `gradle init` сгенерировал `libs.versions.toml` — работайте с ним.

> **Важно:** не смешивайте подходы в одном проекте. Если есть `libs.versions.toml` — добавляйте новые зависимости через него, а не напрямую.

## 2. Репозитории для зависимостей

Gradle использует репозитории для загрузки зависимостей. Репозитории указываются в блоке `repositories`:

```kotlin
repositories {
    mavenCentral() // Популярный репозиторий Maven
    google()       // Репозиторий Google (используется в Android)
}
```

Основные репозитории:

- `mavenCentral()` — стандартный репозиторий Maven. Тот самый, который использует Maven.
- `google()` — содержит зависимости, связанные с Android SDK.
- Кастомные репозитории — можно добавлять свои, используя URL:

```kotlin
maven {
    url = uri("https://jitpack.io")
}
```

## 3. Управление версиями зависимостей

Помимо каталога версий и прямого указания, версии можно задавать ещё несколькими способами.

**С использованием переменных:**

```kotlin
val retrofitVersion = "2.9.0"

dependencies {
    implementation("com.squareup.retrofit2:retrofit:$retrofitVersion")
}
```

**Через файл `gradle.properties`:**

```properties
retrofitVersion=2.9.0
```

В `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.squareup.retrofit2:retrofit:${project.properties["retrofitVersion"]}")
}
```

Каталоги версий (`libs.versions.toml`) — рекомендуемый способ в современных проектах. Переменные и `gradle.properties` — более старые подходы, которые вы можете встретить в существующих проектах.

## 4. Конфигурации зависимостей

Gradle предлагает несколько конфигураций для зависимости, каждая из которых определяет, где и как библиотека будет использоваться.

Аналогия с Maven: в Maven есть `<scope>` — `compile`, `test`, `provided`, `runtime`. В Gradle вместо scope используются конфигурации:

| Maven scope  | Gradle конфигурация  | Когда использовать                        |
|--------------|----------------------|-------------------------------------------|
| `compile`    | `implementation`     | Основные зависимости проекта              |
| `test`       | `testImplementation` | Зависимости только для тестов             |
| `provided`   | `compileOnly`        | Есть при компиляции, нет в runtime (Lombok) |
| `runtime`    | `runtimeOnly`        | Нет при компиляции, есть в runtime (драйвер БД) |
| —            | `testRuntimeOnly`    | Нужно только при запуске тестов            |
| —            | `api`                | Как `implementation`, но экспортирует зависимость для потребителей модуля |

Пример с конфигурациями:

```kotlin
dependencies {
    implementation("org.springframework:spring-core:6.2.2")          // Для компиляции и выполнения
    compileOnly("org.projectlombok:lombok:1.18.36")                  // Только для компиляции
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.4")     // Для тестов
    testRuntimeOnly("org.junit.platform:junit-platform-launcher:1.11.4") // Для запуска тестов
    annotationProcessor("org.projectlombok:lombok:1.18.36")          // Для аннотаций
}
```

Правило: если зависимость нужна в коде (вы пишете `import`) — `implementation` или `testImplementation`. Если нужна только при запуске — `runtimeOnly` или `testRuntimeOnly`.

## Задание

1. Откройте проект `job4j_devops`.
2. Перенесите настройки версий с использованием каталогов версий `libs.versions.toml`.
3. Загрузите изменения в репозиторий. Отправьте ссылку на изменения на проверку.
