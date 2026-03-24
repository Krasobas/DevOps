# Что такое плагины в Gradle? [#505352]

Плагины в Gradle — это расширения, которые добавляют новую функциональность к сборочному процессу. Они упрощают настройку, управление зависимостями и автоматизацию различных задач, таких как компиляция, тестирование, упаковка и развёртывание. Плагины позволяют использовать готовые решения вместо ручного написания сложных конфигураций.

Gradle предоставляет два типа плагинов:

- **Плагины ядра (Core Plugins)** — встроенные в Gradle, такие как `java`, `application` и другие.
- **Пользовательские плагины** — создаются разработчиками для специфических задач и могут быть опубликованы в репозиториях.

## Наиболее частые плагины для Java-проектов

**java** — добавляет базовую поддержку для Java-проектов. Включает задачи компиляции (`compileJava`), тестирования (`test`) и упаковки в JAR-файлы (`jar`).

**java-library** — расширяет функциональность плагина `java` и оптимизирует управление зависимостями для библиотек, предоставляя концепцию `api`/`implementation` для зависимостей.

**application** — используется для проектов, которые создают исполняемые приложения. Позволяет указать точку входа через `mainClass`, автоматически создаёт задачи для упаковки приложения.

**maven-publish** — позволяет публиковать артефакты (например, JAR-файлы) в локальный или удалённый Maven-репозиторий.

**checkstyle** — помогает поддерживать код в соответствии со стандартами стиля. Выполняет статический анализ кода с использованием правил Checkstyle.

**spotbugs** — плагин для статического анализа кода с помощью SpotBugs (ранее FindBugs). Позволяет находить потенциальные ошибки в коде.

**jacoco** — интеграция с инструментом JaCoCo для измерения покрытия кода тестами. Генерирует отчёты о покрытии.

**spring-boot** — предоставляется Spring Boot для поддержки Spring Boot проектов. Упрощает управление зависимостями, настройку сборки и создание исполняемых JAR/WAR.

**kotlin** — если проект использует Kotlin, этот плагин добавляет поддержку компиляции и других задач для Kotlin-кода.

**shadow** — используется для создания «толстых» JAR-файлов, включающих все зависимости. Полезен для создания автономных приложений.

## Пример минимального build.gradle.kts для Java-проекта

```kotlin
plugins {
    java                 // Базовая поддержка Java
    application          // Поддержка приложений
    jacoco               // Отчёт о покрытии тестами
}

group = "com.example"
version = "1.0.0"

application {
    mainClass.set("com.example.Main")
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.apache.commons:commons-lang3:3.17.0")
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.4")
}

tasks.test {
    useJUnitPlatform() // Поддержка JUnit 5
    finalizedBy(tasks.jacocoTestReport)
}

tasks.jacocoTestReport {
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
}
```

> **Важно:** если вы используете `useJUnitPlatform()`, зависимость для тестов должна быть `org.junit.jupiter:junit-jupiter` (JUnit 5), а **не** `junit:junit` (JUnit 4). Эти две версии JUnit используют разные механизмы запуска и не взаимозаменяемы.

## Задание

В проекте `job4j_devops` необходимо добавить плагин SpotBugs для статического анализа кода. Этот плагин поможет находить потенциальные ошибки и улучшать качество вашего проекта.

1. Добавьте плагин SpotBugs в конфигурацию вашего `build.gradle.kts`.
2. Настройте генерацию HTML-отчёта о результатах анализа.
3. Убедитесь, что плагин запускается после выполнения задачи `test`.
4. Зафиксируйте изменения и отправьте ссылку на проверку.

**Ожидаемый результат:**

После выполнения команды `./gradlew spotbugsMain`:

- Должен быть сгенерирован отчёт в HTML-формате.
- Отчёт должен находиться в директории `build/reports/spotbugs`.

**Пример конфигурации SpotBugs:**

```kotlin
plugins {
    java
    application
    id("com.github.spotbugs") version "6.4.8"
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.apache.commons:commons-lang3:3.17.0")
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.4")
}

tasks.test {
    useJUnitPlatform()
}

tasks.spotbugsMain {
    reports.create("html") {
        required = true
        outputLocation.set(layout.buildDirectory.file("reports/spotbugs/spotbugs.html"))
    }
}

tasks.test {
    finalizedBy(tasks.spotbugsMain)
}
```
