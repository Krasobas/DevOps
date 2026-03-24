# Отладка и диагностика сборки Gradle [#NEW]

В Maven, когда в логах не хватает зависимости, вы идёте в Maven Central, находите нужную библиотеку, копируете блок `<dependency>` и вставляете в `pom.xml`. В Gradle путь аналогичный, но есть нюансы. В этом уроке разберём, как читать ошибки Gradle, где что искать и как думать при отладке.

## Карта проекта: где что лежит

В Maven всё в одном файле `pom.xml`. В Gradle конфигурация разбита на несколько файлов. Важно понимать, за что отвечает каждый.

```
project/
├── settings.gradle.kts      ← имя проекта, список модулей
├── build.gradle.kts          ← плагины, зависимости, задачи
├── gradle.properties         ← параметры JVM и оптимизации
└── gradle/
    └── libs.versions.toml    ← каталог версий (необязательный)
```

**Аналогии с Maven:**

| Maven                          | Gradle                              |
|--------------------------------|-------------------------------------|
| `pom.xml` → `<dependencies>`  | `build.gradle.kts` → `dependencies` |
| `pom.xml` → `<plugins>`       | `build.gradle.kts` → `plugins`      |
| `pom.xml` → `<properties>`    | `gradle/libs.versions.toml` или `gradle.properties` |
| `pom.xml` → `<modules>`       | `settings.gradle.kts` → `include()` |
| `mvn` / `mvnw`                | `gradle` / `./gradlew`              |
| Maven Central (search.maven.org) | Maven Central (search.maven.org) — тот же репозиторий |

## Два способа подключать зависимости

В Gradle зависимости можно указывать двумя способами. Оба ведут к одному и тому же результату.

### Способ 1: Напрямую (как в Maven)

Прямо в `build.gradle.kts`, по координатам `group:artifact:version`:

```kotlin
dependencies {
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.4")
}
```

Это самый простой путь. Вы идёте на https://search.maven.org, ищете библиотеку, копируете координаты и вставляете. Точно так же, как в Maven, только вместо XML — строка в скобках.

### Способ 2: Через каталог версий (libs.versions.toml)

Когда проект растёт, версии удобнее хранить в одном месте — файле `gradle/libs.versions.toml`:

```toml
[versions]
junit = "5.11.4"

[libraries]
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit" }
```

А в `build.gradle.kts` ссылаетесь через `libs.`:

```kotlin
dependencies {
    testImplementation(libs.junit.jupiter)
}
```

**Как это связано:** `libs.junit.jupiter` — это не магия, а ссылка на запись `junit-jupiter` в секции `[libraries]` файла `libs.versions.toml`. Gradle при старте читает этот файл и создаёт объект `libs` с полями для каждой библиотеки.

Правило преобразования имени: дефисы и подчёркивания в `libs.versions.toml` становятся точками в `build.gradle.kts`:

```
junit-jupiter  →  libs.junit.jupiter
assertj-core   →  libs.assertj.core
spring-boot-starter-web  →  libs.spring.boot.starter.web
```

**Когда что использовать:**

- Маленький проект, одна-две зависимости — пишите напрямую в `build.gradle.kts`.
- Проект с множеством зависимостей, или многомодульный проект — используйте `libs.versions.toml`.
- Если `gradle init` сгенерировал `libs.versions.toml` — работайте с ним, не смешивайте подходы.

## Как читать ошибки Gradle

### Ошибка 1: «Could not find» — нет зависимости

```
> Could not resolve all files for configuration ':testCompileClasspath'.
   > Could not find org.junit.jupiter:junit-jupiter:5.11.4.
```

**Что случилось:** Gradle не может скачать библиотеку. Это аналог Maven-ошибки `Could not find artifact`.

**Что делать:**

1. Проверьте координаты — опечатка в group, artifact или version.
2. Проверьте блок `repositories` — есть ли `mavenCentral()`.
3. Проверьте интернет-соединение.
4. Если используете `libs.versions.toml` — проверьте, что `module` и `version.ref` указаны верно.

### Ошибка 2: «Failed to load JUnit Platform» — нет launcher

```
> Failed to load JUnit Platform. Please ensure that all JUnit Platform
  dependencies are available on the test's runtime classpath,
  including the JUnit Platform launcher.
```

**Что случилось:** Gradle нашёл тесты, но не может их запустить. Начиная с Gradle 9.x, зависимость `junit-platform-launcher` необходимо подключать явно.

**Что делать:**

```kotlin
dependencies {
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.4")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher:1.11.4")
}
```

`testRuntimeOnly` означает, что эта зависимость нужна только при запуске тестов, но не при компиляции. Launcher — это движок, который обнаруживает и запускает тесты. В Gradle 8.x он подтягивался автоматически, в 9.x — нет.

### Ошибка 3: «no tests to execute» — тесты не найдены

```
> There are test sources present and no filters are applied,
  but the test task did not discover any tests to execute.
```

**Что случилось:** В папке `src/test/java` есть файлы, но Gradle не распознал их как тесты.

**Что делать:**

Проверьте, совпадает ли версия JUnit в зависимостях и в коде тестов:

- Если в тестах `import org.junit.jupiter.api.Test` (JUnit 5) — нужен `useJUnitPlatform()` в блоке `tasks.test` и зависимость `junit-jupiter`.
- Если в тестах `import org.junit.Test` (JUnit 4) — **не** нужен `useJUnitPlatform()`, а нужна зависимость `junit:junit:4.13.2`.

Самая частая ошибка — смешать JUnit 4 зависимости с JUnit 5 настройкой или наоборот.

### Ошибка 4: «Task not found» — задача не существует

```
> Task 'run' not found in root project 'elementary'.
```

**Что случилось:** Вы вызываете задачу, которой нет. Задачи в Gradle создаются плагинами. Нет плагина — нет задачи.

**Что делать:**

Посмотрите, какие задачи доступны:

```
./gradlew tasks
```

Основные плагины и их задачи:

| Плагин         | Что добавляет                                       |
|----------------|-----------------------------------------------------|
| `java`         | `compileJava`, `test`, `jar`, `build`, `clean`      |
| `java-library` | То же что `java` + конфигурации `api`/`implementation` |
| `application`  | Всё из `java` + `run`, `installDist`, `distZip`    |

Задача `run` есть только у плагина `application`. Если проект — библиотека (нет метода `main`, который нужно запускать), используйте `java-library` и запускайте только `./gradlew build`.

### Ошибка 5: «Cannot resolve symbol» — IDE не видит зависимости

IDE (IntelliJ IDEA) подсвечивает импорты красным, хотя `./gradlew build` проходит.

**Что делать:**

1. В IntelliJ: File → Invalidate Caches → перезапуск.
2. Нажмите кнопку «Reload Gradle» (иконка со стрелками в панели Gradle).
3. Или из терминала: `./gradlew --refresh-dependencies`.

### Ошибка 6: «Unresolved reference: libs» — не найден каталог версий

```
> Unresolved reference: libs
```

**Что случилось:** `build.gradle.kts` ссылается на `libs.something`, но файл `gradle/libs.versions.toml` не существует или имеет ошибку.

**Что делать:**

1. Проверьте, что файл `gradle/libs.versions.toml` существует.
2. Проверьте формат файла — секции `[versions]` и `[libraries]` обязательны.
3. Проверьте, что имя библиотеки в `[libraries]` соответствует тому, что вы пишете после `libs.` (с заменой дефисов на точки).

## Алгоритм отладки: пошаговый подход

Когда сборка упала, действуйте по порядку:

**Шаг 1. Прочитайте ошибку.** Ключевая информация — в строке после `> `. Gradle пишет много, но суть обычно в одном предложении.

**Шаг 2. Определите тип проблемы:**

- Не найдена зависимость → проверьте координаты и `repositories`.
- Не найдена задача → проверьте плагины.
- Тесты не запускаются → проверьте соответствие JUnit версии.
- Ошибка компиляции → проверьте версию Java в `toolchain`.

**Шаг 3. Получите больше деталей.** Добавьте `--info` или `--stacktrace`:

```
./gradlew build --info
./gradlew build --stacktrace
```

**Шаг 4. Проверьте дерево зависимостей.** Полезно, когда непонятно, что реально подключено:

```
./gradlew dependencies --configuration testRuntimeClasspath
```

Это покажет все зависимости для тестов с транзитивными. Аналог `mvn dependency:tree`.

**Шаг 5. Очистите кэш.** Иногда помогает начать с чистого состояния:

```
./gradlew clean build --no-configuration-cache
```

## Конфигурации зависимостей: когда какую использовать

В Maven есть `scope` (`compile`, `test`, `provided`, `runtime`). В Gradle — конфигурации:

| Maven scope  | Gradle конфигурация  | Когда использовать                        |
|--------------|----------------------|-------------------------------------------|
| `compile`    | `implementation`     | Основные зависимости проекта              |
| `test`       | `testImplementation` | Зависимости только для тестов             |
| `provided`   | `compileOnly`        | Есть при компиляции, нет в runtime (Lombok) |
| `runtime`    | `runtimeOnly`        | Нет при компиляции, есть в runtime (драйвер БД) |
| —            | `testRuntimeOnly`    | Нужно только при запуске тестов (launcher) |

Правило простое: если зависимость нужна в коде (import) — `implementation` или `testImplementation`. Если нужна только при запуске — `runtimeOnly` или `testRuntimeOnly`.

## Полезные команды для диагностики

Список всех доступных задач:

```
./gradlew tasks
```

Дерево зависимостей:

```
./gradlew dependencies
```

Версия Gradle, Java, ОС:

```
./gradlew --version
```

Подробный лог сборки:

```
./gradlew build --info
```

Сборка с профилем производительности:

```
./gradlew build --profile
```

Принудительное обновление зависимостей:

```
./gradlew build --refresh-dependencies
```
