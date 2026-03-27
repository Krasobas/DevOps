# Liquibase | Миграция баз данных — Обновлённые уроки (2026)

---

## Урок 1. Liquibase в Spring Boot

В этом уроке мы подключим к проекту `job4j_devops` миграцию базы данных с использованием Liquibase.

### Зачем вообще миграции?

Прежде чем писать код — поймите проблему. Без инструмента миграций вы получаете хаос: один разработчик добавил колонку, другой — нет. На проде одна схема, на тесте — другая. Кто-то руками выполнил `ALTER TABLE` и забыл. Через месяц никто не помнит, что было сделано.

Liquibase (и его конкурент Flyway) решает это: каждое изменение схемы — это версионированный скрипт, который выполняется ровно один раз и отслеживается в служебной таблице.

> **Лайфхак от бэкенд-разработчика:** Относитесь к миграциям как к коммитам в Git. Каждый changeset — это атомарное изменение. Один changeset = одно логическое изменение. Не мешайте `CREATE TABLE` и `INSERT` данных в одном скрипте.

### Подключение зависимостей

Откройте файл `build.gradle.kts` и добавьте зависимости:

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.liquibase:liquibase-core")
    runtimeOnly("org.postgresql:postgresql")
}
```

**Важно:** Мы НЕ указываем версии `liquibase-core` и `postgresql`. Spring Boot 4.x управляет версиями через BOM (Bill of Materials). Это значит, что Spring Boot уже протестировал совместимость этих библиотек друг с другом. На март 2026 года Spring Boot 4.0.4 подтягивает Liquibase 5.x и PostgreSQL JDBC 42.7.10.

> **Антипаттерн:** В оригинальном уроке было написано: "всегда указывайте версии библиотек". Для Spring Boot это ВРЕДНЫЙ совет. Если вы укажете `liquibase-core:4.30.0` вручную, вы сломаете совместимость с тем, что ожидает Spring Boot. Вы должны доверять BOM. Исключение — если вам нужна конкретная версия из-за бага. В этом случае вы осознанно переопределяете и документируете причину.

> **Когда версии указывать НУЖНО:** Для зависимостей, которые НЕ входят в Spring Boot BOM. Например, сторонние библиотеки, которые Spring Boot не знает.

### Настройка подключения

Создайте файл `src/main/resources/application.yaml` (не `.properties` — yaml читабельнее и поддерживает вложенность):

```yaml
# src/main/resources/application.yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/job4j_devops
    username: postgres
    password: password
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.xml
```

> **Предупреждение от DevOps:** Никогда не храните реальные пароли в `application.yaml`, который попадает в Git. Здесь указаны значения для **локальной разработки**. Для CI/CD и продакшена пароли приходят через переменные окружения, Kubernetes Secrets или Spring Cloud Config. Мы разберём это в уроке про dotEnv.

> **Лайфхак:** `spring.datasource.driver-class-name` можно не указывать. Spring Boot определяет драйвер автоматически по JDBC URL. Меньше конфигурации — меньше мест для ошибок.

### Создание changelog

Создайте мастер-файл `src/main/resources/db/changelog/db.changelog-master.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        https://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <include file="changeset/001_ddl_create_tables.sql"
             relativeToChangelogFile="true"/>
</databaseChangeLog>
```

**Что изменилось по сравнению с оригинальным уроком:**

1. **XSD-версия: `dbchangelog-latest.xsd`** вместо `dbchangelog-3.8.xsd`. Версия 3.8 вышла в 2019 году. Использование `latest` означает, что IDE будет подсказывать все актуальные атрибуты и элементы. Liquibase сам определяет свою версию при выполнении.

2. **`https://` вместо `http://`** в `schemaLocation`. Мелочь, но безопасность начинается с привычек.

3. **Убран атрибут `author`** из тега `<include>`. Автор указывается внутри changeset, а не в include. В оригинальном уроке это вводило в заблуждение — атрибут `author` в теге `<include>` не является стандартным атрибутом Liquibase и игнорируется.

### Первый changeset

Создайте файл `src/main/resources/db/changelog/changeset/001_ddl_create_tables.sql`:

```sql
--liquibase formatted sql
--changeset job4j:001_create_users_table

CREATE TABLE users (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username   VARCHAR(255) NOT NULL
);
```

**Ключевые отличия от оригинала:**

1. **`BIGINT GENERATED ALWAYS AS IDENTITY`** вместо `SERIAL`. Тип `SERIAL` — это legacy из PostgreSQL 8.x. Современный стандарт SQL — `GENERATED ALWAYS AS IDENTITY`. Он безопаснее: нельзя случайно вставить значение вручную и сломать последовательность. `BIGINT` вместо `INT` — потому что на продакшене 2 миллиарда записей может не хватить, а менять тип потом — это боль.

2. **`NOT NULL` на `username`**. Всегда думайте о constraint'ах с первой миграции. Добавить `NOT NULL` к таблице с миллионом строк, где есть NULL — это отдельный changeset, lock таблицы, и потенциальный downtime.

3. **`VARCHAR(255)` вместо `VARCHAR(2000)`**. 2000 символов для username — это подозрительно. Думайте о том, что пишете в миграции, как о контракте. Если бизнес-логика допускает 255 — так и пишите.

4. **Changeset ID: `001_create_users_table`** — информативный и уникальный. В оригинале ID был `create_users_table` без номера. Номер помогает понять порядок.

### Запуск и проверка

Создайте базу данных `job4j_devops` в PostgreSQL и запустите приложение. В логах вы увидите:

```
liquibase.changelog : ChangeSet db/changelog/changeset/001_ddl_create_tables.sql::001_create_users_table::job4j ran successfully
liquibase.util      : UPDATE SUMMARY
liquibase.util      : Run:                          1
liquibase.util      : Previously run:               0
liquibase.util      : Total change sets:            1
```

> **Лайфхак от DevOps:** После запуска загляните в таблицу `databasechangelog` в PostgreSQL. Посмотрите, какие поля там есть. Это ваш «журнал миграций». Понимание этой таблицы спасёт вас при траблшутинге.

### Liquibase vs Flyway: когда что выбрать

В продакшн-проектах вы встретите два инструмента: Liquibase и Flyway. Кратко:

**Flyway** — проще: SQL-файлы с префиксом `V1__`, `V2__`. Нет XML. Меньше возможностей.

**Liquibase** — мощнее: rollback, preconditions, контексты, форматы (SQL, XML, YAML, JSON), `diff` между БД. Но сложнее в настройке.

Для учебных проектов и большинства production-проектов оба подходят. Liquibase чаще встречается в enterprise с множеством окружений (dev/test/stage/prod).

### Задание

1. Подключите Liquibase к проекту `job4j_devops` (Spring Boot 4.x).
2. Создайте changeset с таблицей `users` (по образцу выше).
3. Убедитесь, что при запуске приложения changeset применяется успешно.
4. Загрузите изменения в репозиторий и отправьте ссылку.

---

## Урок 2. Liquibase в Gradle

В предыдущем уроке Liquibase запускался при старте Spring Boot приложения. Это удобно для разработки, но в CI/CD пайплайне вы хотите разделить этапы: сначала мигрируем БД, потом деплоим приложение. Если миграция упала — деплой не начинается. Это принцип «fail fast».

### Зачем отдельный Gradle-плагин?

Представьте: у вас микросервисная архитектура, 5 сервисов, 3 базы данных. Миграции должны запускаться до деплоя, независимо от приложения. Gradle-плагин позволяет выполнить `./gradlew update` как отдельный шаг в Jenkins/GitLab CI.

> **Лайфхак от DevOps:** В зрелых командах миграции часто выделены в **отдельный репозиторий** или отдельный Gradle-модуль. Это позволяет DBA ревьюить миграции отдельно от кода приложения.

### Подключение плагина

Откройте `build.gradle.kts`:

```kotlin
// build.gradle.kts

plugins {
    id("org.liquibase.gradle") version "3.1.0"
}

buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.liquibase:liquibase-core:5.0.2")
    }
}
```

> **Внимание:** Блок `buildscript` обязателен для плагина Liquibase. Начиная с версии 3.0.0, плагин требует, чтобы `liquibase-core` был в classpath и при подключении плагина, и при выполнении задач.

### Runtime-зависимости

Добавьте зависимости, которые Liquibase будет использовать при выполнении:

```kotlin
dependencies {
    liquibaseRuntime("org.liquibase:liquibase-core:5.0.2")
    liquibaseRuntime("org.postgresql:postgresql:42.7.10")
    liquibaseRuntime("info.picocli:picocli:4.7.6")
    liquibaseRuntime("org.apache.commons:commons-lang3:3.17.0")
    liquibaseRuntime("ch.qos.logback:logback-core:1.5.16")
    liquibaseRuntime("ch.qos.logback:logback-classic:1.5.16")
}
```

**Что изменилось по сравнению с оригиналом:**

1. **Liquibase 5.0.2** (март 2026) вместо 4.30.0. Liquibase 5.0 — мажорный релиз: модульная архитектура, минимальное требование Java 17+, улучшенная безопасность.

2. **`commons-lang3`** — обязательная зависимость с Liquibase 4.30.0+. Без неё — `ClassNotFoundException` при выполнении любой команды. Это частая ловушка.

3. **Убран `javax.xml.bind:jaxb-api`**. Это javax-namespace из Java 8 эпохи. Liquibase 5.x на Java 17+ не нуждается в этой зависимости. Если вы видите `javax.*` в зависимостях 2026 года — это почти всегда ошибка. Jakarta namespace (`jakarta.*`) пришёл на замену.

4. **PostgreSQL JDBC 42.7.10** — актуальная версия на март 2026.

### Конфигурация профиля

```kotlin
liquibase {
    activities.register("main") {
        this.arguments = mapOf(
            "logLevel"      to "info",
            "url"           to "jdbc:postgresql://localhost:5432/job4j_devops",
            "username"      to "postgres",
            "password"      to "password",
            "changelogFile" to "src/main/resources/db/changelog/db.changelog-master.xml"
        )
    }
    runList = "main"
}
```

> **Лайфхак:** Не используйте `classpath` в аргументах Gradle-плагина. Путь к changelog указывайте относительно корня проекта. Это отличается от Spring Boot, где используется `classpath:` prefix.

### Основные команды

После настройки вам доступны Gradle-задачи. Всегда используйте `./gradlew` (wrapper), а не системный `gradle`:

```bash
./gradlew update              # Применить все непримененные changesets
./gradlew validate            # Проверить корректность changelog (делайте перед commit!)
./gradlew status              # Показать непримененные changesets
./gradlew updateSql           # Показать SQL БЕЗ выполнения (dry-run)
./gradlew tag -PliquibaseTag=v1.0  # Пометить текущее состояние
```

> **Предупреждение:** Команда `./gradlew update` — это необратимая операция (если нет rollback-секции). Перед выполнением на любом окружении кроме localhost, **сначала** запустите `./gradlew updateSql` и прочитайте, что будет выполнено.

> **Лайфхак от DevOps:** В CI/CD пайплайне всегда запускайте `./gradlew validate` перед `./gradlew update`. Это ловит синтаксические ошибки в changelog до того, как они попадут на реальную БД.

### Почему changeset не выполнится повторно

Если changeset уже был применён, Liquibase пропустит его. Это работает через проверку `id + author + filename` в таблице `databasechangelog`. Контрольная сумма (MD5) используется для обнаружения **изменений** в уже выполненном скрипте.

### Задание

1. Настройте Gradle-плагин Liquibase 3.1.0 в проекте.
2. Выполните `./gradlew validate` и `./gradlew update`.
3. Загрузите изменения в репозиторий и отправьте ссылку.

---

## Урок 3. Структура и написание скриптов миграций

### Архитектура changelog

Liquibase использует иерархическую структуру: один мастер-файл (`db.changelog-master.xml`) ссылается на отдельные файлы с changesets. Мастер-файл — это оглавление книги. Он не содержит SQL, а только ссылки.

```
src/main/resources/db/changelog/
├── db.changelog-master.xml          # мастер-файл (оглавление)
└── changeset/
    ├── 001_ddl_create_users.sql     # DDL: создание таблиц
    ├── 002_ddl_add_email_to_users.sql
    ├── 003_dml_insert_default_roles.sql  # DML: начальные данные
    └── 004_ddl_create_orders.sql
```

### Правила именования файлов

Формат: `NNN_тип_описание.sql`

- **NNN** — порядковый номер (001, 002, ...). Гарантирует правильный порядок выполнения.
- **Тип**: `ddl` (структура: CREATE, ALTER, DROP) или `dml` (данные: INSERT, UPDATE, DELETE).
- **Описание**: что делает скрипт, в snake_case.

Примеры хороших имён:

```
001_ddl_create_users.sql
002_ddl_create_orders.sql
003_ddl_add_email_to_users.sql
004_dml_insert_default_roles.sql
005_ddl_create_index_users_email.sql
```

Примеры плохих имён:

```
migration.sql            # Что за миграция? Номер?
changes.sql              # Какие changes?
fix.sql                  # Что фиксим?
001_create_table.sql     # Какую таблицу?
```

### Анатомия changeset

```sql
--liquibase formatted sql
--changeset job4j:003_add_email_to_users

ALTER TABLE users ADD COLUMN email VARCHAR(320) UNIQUE;

--rollback ALTER TABLE users DROP COLUMN email;
```

Обязательные элементы:
- `--liquibase formatted sql` — маркер формата (первая строка файла).
- `--changeset author:id` — автор и уникальный ID.

> **Лайфхак:** Используйте название проекта или команды как `author` (например, `job4j`), а не имя конкретного разработчика. Люди уходят из проектов, автор в Liquibase — это идентификатор, а не credits.

### Золотое правило: changeset неизменяем

После того как changeset выполнен **хотя бы на одном окружении** — его содержимое нельзя менять. Liquibase вычисляет MD5-хеш файла и хранит его в таблице `databasechangelog`. При следующем запуске хеш проверяется. Если он изменился — Liquibase остановится с ошибкой.

Это как коммит в Git: вы не меняете старые коммиты, вы создаёте новые.

**Нужно исправить миграцию?** Создайте новый changeset:

```sql
--liquibase formatted sql
--changeset job4j:005_fix_username_length

ALTER TABLE users ALTER COLUMN username TYPE VARCHAR(100);

--rollback ALTER TABLE users ALTER COLUMN username TYPE VARCHAR(255);
```

### Дополнительные параметры changeset

Параметры указываются после `--changeset` через пробел:

```sql
--changeset job4j:006_insert_test_data context:dev runOnChange:true
```

Наиболее полезные:

- **`context`** — выполнять только в указанном окружении. Например, `context:dev` — тестовые данные только для разработки.
- **`runOnChange:true`** — перевыполнять при изменении (полезно для views и stored procedures).
- **`failOnError:false`** — не останавливаться при ошибке (осторожно! обычно не нужно).
- **`dbms:postgresql`** — только для PostgreSQL.

> **Предупреждение от бэкенд-разработчика:** `failOnError:false` — это запах. Если вы его используете, значит, вы не уверены в своём скрипте. Почти всегда лучше исправить скрипт, чем подавлять ошибку.

### Preconditions — защита от ошибок

Liquibase позволяет задать условия, при которых changeset выполняется. Это мощный инструмент, который в оригинальных уроках не упоминался:

```sql
--liquibase formatted sql
--changeset job4j:007_add_phone_to_users
--preconditions onFail:MARK_RAN
--precondition-sql-check expectedResult:0 SELECT COUNT(*) FROM information_schema.columns WHERE table_name='users' AND column_name='phone'

ALTER TABLE users ADD COLUMN phone VARCHAR(20);
```

Этот changeset проверит: если колонка `phone` уже существует — пометит changeset как выполненный, но не выполнит SQL. Это делает миграции **идемпотентными** — безопасными для повторного запуска.

### Задание

1. Добавьте changeset `002_ddl_alter_users_add_columns.sql`, который добавляет колонки `first_arg`, `second_arg`, `result` (все `INTEGER`) в таблицу `users`.
2. Добавьте rollback-секцию к changeset.
3. Не забудьте включить новый changeset в `db.changelog-master.xml`.
4. Загрузите изменения в репозиторий и отправьте ссылку.

---

## Урок 4. Liquibase: Rollback

### Защита от разрушительных изменений

Liquibase защищает вашу БД двумя способами:

1. **Контрольные суммы** — нельзя изменить уже выполненный changeset.
2. **Rollback** — можно компенсировать изменения.

### Таблица DATABASECHANGELOG

При первом запуске Liquibase создаёт две служебные таблицы:

- **`databasechangelog`** — журнал всех выполненных changesets.
- **`databasechangeloglock`** — блокировка, предотвращающая одновременное выполнение миграций.

Ключевые поля `databasechangelog`:

- **`id`** + **`author`** + **`filename`** — уникальный идентификатор changeset.
- **`md5sum`** — контрольная сумма. Если она изменилась — Liquibase остановится.
- **`exectype`** — статус: `EXECUTED`, `FAILED`, `SKIPPED`, `MARK_RAN`.
- **`deployment_id`** — группирует changesets, выполненные за один запуск.

### Эксперимент: изменение выполненного changeset

Откройте `001_ddl_create_tables.sql`, переименуйте `username` в `user_name`, выполните:

```bash
./gradlew update
```

Результат — ошибка:

```
Validation Failed:
  1 changesets check sum
  db/changelog/changeset/001_ddl_create_tables.sql::001_create_users_table::job4j
  was: 9:768d088239f67e217a96fc085b75d811
  but is now: 9:29728504ee2796fb2448afa3d13014ce
```

Скрипт не выполнился. Это правильное поведение.

> **Предупреждение:** Команда `clearCheckSums` существует, но использовать её — это как делать `git push --force` на main. Используйте только при подключении Liquibase к существующей БД (baseline) или при реальном повреждении данных в `databasechangelog`.

### Написание rollback

Добавим changeset с rollback-секцией:

Файл: `changeset/003_ddl_add_created_at.sql`

```sql
--liquibase formatted sql
--changeset job4j:003_add_created_at

ALTER TABLE users ADD COLUMN created_at TIMESTAMP WITH TIME ZONE DEFAULT now();

--rollback ALTER TABLE users DROP COLUMN created_at;
```

> **Важно:** `TIMESTAMP WITH TIME ZONE` вместо `WITHOUT TIME ZONE`. Это практика из реальных production систем. Без time zone вы получите проблемы, когда серверы в разных часовых поясах. Всегда используйте `timestamptz` (алиас `TIMESTAMP WITH TIME ZONE`) для хранения моментов времени.

### Rollback — это компенсация, а не отмена

Rollback **не отменяет** изменения как `git revert`. Он выполняет **компенсирующий** SQL. Если вы добавили колонку — rollback её удаляет. Если вы вставили данные — rollback должен их удалить.

**Но:** если вы удалили данные (`DROP TABLE`, `DELETE`) — rollback НЕ восстановит данные. Он может воссоздать структуру таблицы, но не данные в ней. Поэтому бэкап перед миграцией на production — обязателен.

### Команды rollback

```bash
# Откатить 1 последний changeset
./gradlew rollbackCount -PliquibaseCount=1

# Откатить до тега
./gradlew rollback -PliquibaseTag=v1.0

# Откатить до даты
./gradlew rollbackToDate -PliquibaseDate="2026-03-01 00:00:00"

# Показать SQL отката (dry-run)
./gradlew rollbackCountSql -PliquibaseCount=1
```

> **Лайфхак от DevOps:** Всегда запускайте `rollbackCountSql` перед `rollbackCount`. Прочитайте SQL, убедитесь, что он делает то, что вы ожидаете. На production это обязательно.

> **Лайфхак от бэкенд-разработчика:** При разработке на localhost rollback — ваш лучший друг. Цикл: написал changeset → выполнил → нашёл ошибку → откатил → исправил → выполнил снова. Это намного быстрее, чем пересоздавать базу.

### Когда rollback не нужен

Для простых `CREATE TABLE` Liquibase автоматически генерирует rollback (`DROP TABLE`). Rollback-секция обязательна для:

- `ALTER TABLE` (добавление/удаление/изменение колонок)
- `INSERT`/`UPDATE`/`DELETE` (DML-операции)
- Создание индексов, views, functions

### Теги: снапшоты состояния БД

Перед деплоем ставьте тег:

```bash
./gradlew tag -PliquibaseTag=release-2.1.0
```

Если деплой пошёл не так:

```bash
./gradlew rollback -PliquibaseTag=release-2.1.0
```

> **Практика из больших компаний:** В CI/CD пайплайне тег ставится автоматически перед каждым `update`. Имя тега = версия приложения или SHA коммита.

### Задание

1. Создайте changeset `004_ddl_add_updated_at.sql`, который добавляет колонку `updated_at TIMESTAMP WITH TIME ZONE` в таблицу `users`.
2. Добавьте rollback-секцию.
3. Выполните `./gradlew update`.
4. Выполните `./gradlew rollbackCount -PliquibaseCount=1`.
5. Убедитесь, что колонка исчезла.
6. Загрузите скрипт в репозиторий и оставьте ссылку.

---

## Урок 5. Настройки окружения | dotEnv

### Окружения (environments)

В профессиональной разработке ваш код проходит через несколько окружений (стендов):

1. **Local (localhost)** — ваша машина. Здесь всё ломается, и это нормально.
2. **Dev / Development** — общий сервер разработки. Код попадает сюда автоматически из main-ветки.
3. **Staging** — копия production. Финальная проверка перед выкладкой.
4. **Production** — пользователи. Ошибки здесь = реальные деньги / репутация.

> **Лайфхак от DevOps:** В крупных компаниях окружений может быть 6-8 (dev, qa, integration, staging, canary, production, dr). Но принцип один: каждое окружение имеет свои credentials и адреса, а код — одинаковый.

Каждое окружение имеет свою БД с разными credentials. Как передать их в Liquibase, не хардкодя в `build.gradle.kts`?

### Подход 1: Плагин dotEnv (для локальной разработки)

Подключите плагин в `build.gradle.kts`:

```kotlin
plugins {
    id("co.uzzu.dotenv.gradle") version "4.0.0"
}
```

Создайте файлы настроек:

**`env/.env.local`** (для localhost):
```properties
DB_URL=jdbc:postgresql://localhost:5432/job4j_devops
DB_USERNAME=postgres
DB_PASSWORD=password
```

**`env/.env.develop`** (для dev-сервера):
```properties
DB_URL=jdbc:postgresql://dev-db.internal:5432/job4j_devops
DB_USERNAME=app_user
DB_PASSWORD=strong_password_here
```

Укажите путь по умолчанию в `gradle.properties`:

```properties
dotenv.filename=env/.env.local
```

Используйте в конфигурации Liquibase:

```kotlin
liquibase {
    activities.register("main") {
        this.arguments = mapOf(
            "logLevel"      to "info",
            "url"           to env.DB_URL.value,
            "username"      to env.DB_USERNAME.value,
            "password"      to env.DB_PASSWORD.value,
            "changelogFile" to "src/main/resources/db/changelog/db.changelog-master.xml"
        )
    }
    runList = "main"
}
```

Переключение окружения:

```bash
# Локальная разработка (по умолчанию)
./gradlew update

# Dev-окружение
./gradlew update -P"dotenv.filename"="env/.env.develop"
```

> **КРИТИЧЕСКОЕ ПРЕДУПРЕЖДЕНИЕ:** Файлы `.env` содержат пароли. Они НИКОГДА не должны попадать в Git.

Добавьте в `.gitignore`:

```
# .gitignore
env/.env.*
!env/.env.example
```

Создайте **шаблон** `env/.env.example` (без реальных паролей):

```properties
DB_URL=jdbc:postgresql://localhost:5432/job4j_devops
DB_USERNAME=postgres
DB_PASSWORD=changeme
```

Этот шаблон попадает в Git и служит документацией: какие переменные нужны.

### Подход 2: Системные переменные окружения (для CI/CD)

В Jenkins, GitLab CI, GitHub Actions переменные задаются через UI или Secrets:

```bash
# В Jenkins credentials или GitLab CI Variables
DB_URL=jdbc:postgresql://prod-db:5432/job4j_devops
DB_USERNAME=app_service
DB_PASSWORD=<из vault>
```

Gradle умеет читать системные переменные напрямую:

```kotlin
liquibase {
    activities.register("main") {
        this.arguments = mapOf(
            "url"      to (System.getenv("DB_URL") ?: env.DB_URL.value),
            "username" to (System.getenv("DB_USERNAME") ?: env.DB_USERNAME.value),
            "password" to (System.getenv("DB_PASSWORD") ?: env.DB_PASSWORD.value),
            // ...
        )
    }
}
```

Логика: если есть системная переменная (CI/CD) — используем её. Нет — fallback на `.env` файл (локальная разработка).

> **Лайфхак от DevOps:** В production-окружениях пароли хранятся в HashiCorp Vault, AWS Secrets Manager или Kubernetes Secrets. Они инжектятся как переменные окружения в момент запуска. Файлы `.env` — это инструмент разработчика, а не production-решение.

### Подход 3: Spring Profiles (для приложения)

Для самого Spring Boot приложения (не для Gradle-плагина) используйте Spring Profiles:

```yaml
# application.yaml (общие настройки)
spring:
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.xml

---
# application-local.yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/job4j_devops
    username: postgres
    password: password

---
# application-dev.yaml
spring:
  datasource:
    url: jdbc:postgresql://dev-db:5432/job4j_devops
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

Запуск:
```bash
SPRING_PROFILES_ACTIVE=local ./gradlew bootRun
```

### Задание

1. Подключите плагин dotEnv для Gradle.
2. Создайте файлы `env/.env.local` и `env/.env.develop`.
3. Создайте `env/.env.example` и добавьте в Git.
4. Добавьте `env/.env.*` (кроме `.example`) в `.gitignore`.
5. Измените конфигурацию Liquibase-плагина для использования переменных из dotEnv.
6. Убедитесь, что `./gradlew profile` (создайте такую задачу) выводит разные URL для разных окружений.
7. Загрузите изменения в репозиторий и отправьте ссылку.

---

## Урок 6. Обновление базы данных в CI/CD pipeline

### Архитектура стенда

Напоминание о нашем окружении:

- Хост-машина (Debian/Ubuntu).
- Docker-контейнер с Jenkins.
- Docker-контейнер agent-jdk21 (сборка проекта).
- PostgreSQL — для базы данных.

### PostgreSQL: в Docker, а не на хосте

В оригинальном уроке PostgreSQL устанавливался прямо на хост-машину. В 2026 году это антипаттерн для dev/test окружений. PostgreSQL должен быть в Docker:

```yaml
# compose.yaml (добавьте к существующему)
services:
  db:
    image: postgres:17
    container_name: job4j-postgres
    environment:
      POSTGRES_DB: job4j_devops
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD:-password}
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - ci-network

volumes:
  pgdata:

networks:
  ci-network:
    driver: bridge
```

> **Почему Docker, а не установка на хост?**
> 
> 1. **Воспроизводимость** — `docker compose up` и у всех одинаковая БД. Не надо гуглить "как установить PostgreSQL на Ubuntu 24.04".
> 2. **Изоляция** — нужна вторая БД? Добавь сервис в compose. Не надо разбираться с кластерами PostgreSQL на хосте.
> 3. **Чистый старт** — `docker compose down -v && docker compose up` = свежая БД за 5 секунд.
> 4. **Безопасность** — нет необходимости открывать PostgreSQL на `0.0.0.0` и добавлять `host all all 0.0.0.0/0 md5` в pg_hba.conf (как предлагалось в оригинальном уроке). Docker-сеть изолирует доступ.

> **Предупреждение от DevOps:** Оригинальный урок содержал опасную конфигурацию: `listen_addresses = '*'` + `host job4j_devops postgres 0.0.0.0/0 md5`. Это открывает PostgreSQL для подключений с **любого IP-адреса в интернете**. На VPS с публичным IP это прямое приглашение для brute-force атак. Docker-сеть решает эту проблему: контейнеры общаются через внутреннюю сеть, PostgreSQL не выставлен наружу.

### Настройка Jenkins

Файл с credentials для агента:

```bash
# /var/agent-jdk21/env/.env.develop
DB_URL=jdbc:postgresql://db:5432/job4j_devops
DB_USERNAME=postgres
DB_PASSWORD=password
```

> **Обратите внимание:** hostname `db` — это имя сервиса в `compose.yaml`. Docker DNS резолвит его в IP-адрес контейнера PostgreSQL. Не `localhost`, не IP-адрес — имя сервиса. Это работает, когда агент Jenkins и PostgreSQL находятся в одной Docker-сети.

### Jenkinsfile с этапом миграции

```groovy
pipeline {
    agent { label 'agent-jdk21' }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/your-org/job4j_devops.git'
            }
        }

        stage('Validate Migrations') {
            steps {
                sh './gradlew validate -P"dotenv.filename"="/var/agent-jdk21/env/.env.develop"'
            }
        }

        stage('Update DB') {
            steps {
                sh './gradlew update -P"dotenv.filename"="/var/agent-jdk21/env/.env.develop"'
            }
        }

        stage('Build & Test') {
            steps {
                sh './gradlew check -P"dotenv.filename"="/var/agent-jdk21/env/.env.develop"'
            }
        }
    }

    post {
        failure {
            // Telegram уведомление об ошибке
        }
        always {
            junit '**/build/test-results/test/*.xml'
        }
    }
}
```

**Ключевые отличия от оригинала:**

1. **Этап `Validate Migrations` перед `Update DB`** — ловит ошибки в changelog до выполнения SQL.

2. **Все Gradle-вызовы используют одинаковый `dotenv.filename`** — не забудьте, иначе тесты не смогут подключиться к БД.

3. **`./gradlew` вместо `gradle`** — wrapper гарантирует одинаковую версию Gradle на всех окружениях. Без wrapper Jenkins может использовать системный Gradle другой версии и сломать сборку.

### Передача credentials в Spring Boot тесты

В `build.gradle.kts`:

```kotlin
tasks.named<Test>("test") {
    systemProperty("spring.datasource.url", env.DB_URL.value)
    systemProperty("spring.datasource.username", env.DB_USERNAME.value)
    systemProperty("spring.datasource.password", env.DB_PASSWORD.value)
}
```

> **Продвинутый лайфхак: Testcontainers** — ещё лучший подход для тестов. Вместо зависимости от внешней БД, Testcontainers поднимает PostgreSQL в Docker **на время теста** и убивает после:
>
> ```kotlin
> // build.gradle.kts
> testImplementation("org.springframework.boot:spring-boot-testcontainers")
> testImplementation("org.testcontainers:postgresql")
> ```
>
> ```java
> @SpringBootTest
> @Testcontainers
> class UserRepositoryTest {
>     @Container
>     static PostgreSQLContainer<?> pg =
>         new PostgreSQLContainer<>("postgres:17");
>
>     @DynamicPropertySource
>     static void props(DynamicPropertyRegistry r) {
>         r.add("spring.datasource.url", pg::getJdbcUrl);
>         r.add("spring.datasource.username", pg::getUsername);
>         r.add("spring.datasource.password", pg::getPassword);
>     }
> }
> ```
>
> С Testcontainers тесты не зависят от dev-БД. Каждый прогон — чистая БД. Миграции Liquibase применяются автоматически при старте Spring контекста.

### Чеклист безопасности для CI/CD с БД

- [ ] Пароли в `.env` файлах, не в коде
- [ ] `.env` файлы в `.gitignore`
- [ ] PostgreSQL не слушает на `0.0.0.0` (используйте Docker-сеть)
- [ ] `pg_hba.conf` без `0.0.0.0/0`
- [ ] Jenkins credentials хранятся в Credentials Store, не в plaintext
- [ ] `./gradlew validate` запускается перед `./gradlew update`
- [ ] Бэкап БД перед миграцией на staging/production

### Задание

1. Добавьте PostgreSQL в `compose.yaml` как Docker-сервис.
2. Создайте файл `/var/agent-jdk21/env/.env.develop` с credentials.
3. Добавьте этапы `Validate Migrations` и `Update DB` в Jenkinsfile.
4. Убедитесь, что pipeline проходит полностью: валидация → миграция → тесты.
5. Загрузите изменения в репозиторий и оставьте ссылки на коммиты.

---

## Сводка изменений: что обновлено и почему

### Версии зависимостей (март 2026)

| Зависимость | Было (оригинал) | Стало | Почему |
|---|---|---|---|
| Liquibase Core | 4.30.0 | 5.0.2 (или Spring Boot BOM) | Мажорный релиз, Java 17+, модульность |
| Gradle Plugin | 3.0.1 | 3.1.0 | Поддержка Gradle 9 |
| PostgreSQL JDBC | 42.7.4 | 42.7.10 (или Spring Boot BOM) | Фикс безопасности, регрессий |
| XSD | dbchangelog-3.8.xsd | dbchangelog-latest.xsd | 3.8 — 2019 год |
| picocli | 4.6.1 | 4.7.6 | Актуальная версия |
| commons-lang3 | отсутствовал | 3.17.0 | Обязательная зависимость с LB 4.30+ |
| javax.xml.bind | 2.3.1 | убран | javax → jakarta, не нужен на Java 17+ |
| Spring Boot | 3.x (подразумевалось) | 4.0.4 | Актуальная стабильная версия |

### Исправленные ошибки и антипаттерны

| Проблема | Было | Стало |
|---|---|---|
| Версии в Spring Boot | "Всегда указывайте версии" | Доверяйте BOM, не переопределяйте |
| SERIAL | `id SERIAL PRIMARY KEY` | `BIGINT GENERATED ALWAYS AS IDENTITY` |
| Пароли в коде | Хардкод в application.properties | dotEnv + .gitignore + .env.example |
| PostgreSQL setup | Установка на хост + 0.0.0.0/0 | Docker-контейнер + Docker-сеть |
| Команды Gradle | `gradle update` | `./gradlew update` (wrapper) |
| XSD | http:// + 3.8 | https:// + latest |
| Timestamp | WITHOUT TIME ZONE | WITH TIME ZONE |
| author в include | `author="Petr Arsentev"` в include | author только в changeset |
| driver-class-name | Указан явно | Убран (автоопределение) |
| Validate | Не упоминался | Обязательный этап перед update |


# Урок 7. Liquibase в production: чеклист выживания

В предыдущих уроках мы настроили Liquibase, научились писать changesets, откатывать их и интегрировать в CI/CD. Всё это работает красиво на localhost с пустой базой и одним разработчиком.

Production — это другой мир. Там таблицы на сотни миллионов строк, два инстанса приложения работают одновременно, DBA звонит в 3 часа ночи, и команда из 8 человек мержит миграции в один main.

Этот урок — коллекция граблей, на которые наступали десятки команд до вас. Каждый пункт — это чей-то потерянный выходной.

---

## 1. Залипший DATABASECHANGELOGLOCK

### Что происходит

Liquibase перед выполнением миграций ставит блокировку в таблице `databasechangeloglock`. Это защита от одновременного запуска двух миграций. После завершения — блокировка снимается.

Но если процесс убит принудительно (Ctrl+C, OOM killer, рестарт контейнера, таймаут Jenkins) — блокировка остаётся. Следующий запуск видит `LOCKED = TRUE` и ждёт. Бесконечно.

### Как выглядит

```
Waiting for changelog lock....
Waiting for changelog lock....
Waiting for changelog lock....
```

Разработчик ждёт 5 минут, решает что "зависло", убивает процесс. Запускает заново. Та же картина. Паника.

### Диагностика

```sql
SELECT locked, lockgranted, lockedby FROM databasechangeloglock;
```

Если `locked = true`, а `lockgranted` было вчера — это залипший lock.

### Решение

Вариант 1 — через Liquibase:

```bash
./gradlew releaseLocks -P"dotenv.filename"="env/.env.develop"
```

Вариант 2 — через SQL (если Gradle тоже завис на lock):

```sql
UPDATE databasechangeloglock
SET locked = false, lockgranted = null, lockedby = null
WHERE id = 1;
```

### Правило

Используйте только когда вы **уверены**, что никакая другая миграция не выполняется прямо сейчас. Если сомневаетесь — проверьте через `pg_stat_activity`:

```sql
SELECT pid, state, query, query_start
FROM pg_stat_activity
WHERE datname = 'job4j_devops'
  AND query ILIKE '%databasechangelog%';
```

> **Лайфхак от DevOps:** В Jenkins-пайплайне установите таймаут на этап миграции. Если миграция не завершилась за 10 минут — что-то не так:
>
> ```groovy
> stage('Update DB') {
>     options {
>         timeout(time: 10, unit: 'MINUTES')
>     }
>     steps {
>         sh './gradlew update ...'
>     }
> }
> ```

---

## 2. ALTER TABLE на большой таблице = downtime

### Проблема

Вы пишете простой changeset:

```sql
ALTER TABLE orders ADD COLUMN status VARCHAR(50) NOT NULL DEFAULT 'pending';
```

На localhost с 10 записями — 5 миллисекунд. На production с 80 миллионами строк — зависит от версии PostgreSQL и от того, что именно вы делаете.

### Что блокирует, а что нет (PostgreSQL 11+)

**Мгновенно (не перезаписывает строки):**

```sql
-- Добавить колонку с DEFAULT (PG 11+): default хранится в метаданных
ALTER TABLE orders ADD COLUMN status VARCHAR(50) DEFAULT 'pending';

-- Удалить колонку: помечает как невидимую, не чистит данные сразу
ALTER TABLE orders DROP COLUMN old_field;

-- Убрать NOT NULL: только метаданные
ALTER TABLE orders ALTER COLUMN status DROP NOT NULL;
```

**Медленно и опасно (full table scan / lock):**

```sql
-- Добавить NOT NULL на существующую колонку с NULL-значениями:
-- PG проверяет КАЖДУЮ строку
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;

-- Изменить тип колонки: перезапись каждой строки
ALTER TABLE orders ALTER COLUMN price TYPE NUMERIC(12,2);

-- Добавить UNIQUE constraint: сканирует всю таблицу
ALTER TABLE orders ADD CONSTRAINT uq_order_ref UNIQUE (reference);
```

### Безопасный паттерн для больших таблиц

Вместо одного changeset — три, разнесённых во времени:

```sql
--liquibase formatted sql

--changeset job4j:050_add_status_column
-- Шаг 1: добавляем колонку без NOT NULL (мгновенно)
ALTER TABLE orders ADD COLUMN status VARCHAR(50) DEFAULT 'pending';
--rollback ALTER TABLE orders DROP COLUMN status;
```

Между деплоями: бэкфилл (заполнение существующих строк). Это делается **не через Liquibase**, а скриптом приложения или отдельным процессом, батчами:

```sql
-- Выполняется вручную или через приложение, НЕ через changeset
UPDATE orders SET status = 'completed'
WHERE status IS NULL AND id BETWEEN 1 AND 100000;
-- Повторяйте для следующих батчей
```

После бэкфилла:

```sql
--changeset job4j:051_add_status_not_null
-- Шаг 3: теперь все строки заполнены — ставим NOT NULL
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;
--rollback ALTER TABLE orders ALTER COLUMN status DROP NOT NULL;
```

> **Предупреждение от бэкенд-разработчика:** `UPDATE` на 80 миллионах строк одним запросом — это не просто медленно. Это генерирует WAL-логи размером с саму таблицу, забивает дисковое пространство, и может убить репликацию. Всегда батчами.

### Как узнать размер таблицы

Перед любым ALTER на production — проверяйте:

```sql
SELECT
    relname AS table,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
    n_live_tup AS row_estimate
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

Если таблица больше 1 ГБ или 10 миллионов строк — планируйте миграцию отдельно. Обсуждайте с командой. Тестируйте на staging с реальным объёмом данных.

---

## 3. Backward-compatible миграции

### Зачем

Если у вас blue-green deployment, rolling update, или canary release — в какой-то момент времени работают **две версии** приложения. Обе читают и пишут в одну и ту же БД.

Это значит: миграция не может ломать старую версию приложения.

### Запрещённые операции (без подготовки)

| Операция | Почему опасно |
|---|---|
| `RENAME COLUMN` | Старый код обращается по старому имени |
| `DROP COLUMN` | Старый код SELECT'ит эту колонку |
| `DROP TABLE` | Старый код обращается к таблице |
| `ALTER TYPE` (сужение) | `VARCHAR(200)` → `VARCHAR(50)`: старый код пишет длинные строки |
| `ADD NOT NULL` (без default) | Старый код INSERT'ит без этого поля |

### Паттерн: Expand → Migrate → Contract

Любое ломающее изменение разбивается на три деплоя:

**Деплой 1 — Expand (расширяем):**

Добавляем новое, не убираем старое. Код пишет в оба места.

```sql
--changeset job4j:060_expand_add_login
ALTER TABLE users ADD COLUMN login VARCHAR(255);
```

Код приложения (v2):
```java
// Пишем в оба поля
user.setUsername(value);
user.setLogin(value);
// Читаем из нового, если заполнено; иначе из старого
String name = user.getLogin() != null ? user.getLogin() : user.getUsername();
```

**Деплой 2 — Migrate (мигрируем данные):**

Заполняем новую колонку из старой. Переключаем чтение.

```sql
--changeset job4j:061_backfill_login
UPDATE users SET login = username WHERE login IS NULL;
```

Код приложения (v3):
```java
// Читаем только из login
// Пишем только в login
// username больше не трогаем
```

**Деплой 3 — Contract (убираем старое):**

Старая версия больше нигде не работает. Безопасно удалять.

```sql
--changeset job4j:062_contract_drop_username
ALTER TABLE users DROP COLUMN username;
```

> **Да, это три деплоя вместо одного.** Да, это медленно. Нет, быстрее не получится без downtime. Все крупные компании делают именно так. Netflix, Uber, Spotify — у всех один и тот же паттерн.

---

## 4. Конфликты миграций в команде

### Проблема

Два разработчика создают миграцию с порядковым номером `005`:

- Ветка feature-A: `005_ddl_create_payments.sql`
- Ветка feature-B: `005_ddl_create_notifications.sql`

Оба мержат в main. Liquibase не упадёт (он не смотрит на номера файлов), но порядок может быть непредсказуемым, а в `db.changelog-master.xml` будет конфликт мержа.

### Решение 1: Timestamp в имени файла

Вместо порядковых номеров — timestamp:

```
20260327_1400_ddl_create_payments.sql
20260327_1530_ddl_create_notifications.sql
```

Конфликты имён практически невозможны. Порядок выполнения определяется по времени создания.

### Решение 2: includeAll вместо include

Замените перечисление каждого файла на автоподхват:

```xml
<databaseChangeLog ...>
    <includeAll path="changeset/"
                relativeToChangelogFile="true"/>
</databaseChangeLog>
```

`includeAll` подбирает все файлы в директории по алфавитному порядку. С timestamp-именами — это хронологический порядок.

Минус: вы теряете явный контроль над порядком. Для большинства проектов это приемлемый трейдофф.

### Решение 3: Ревью миграций как отдельный checklist

На code review для PR с миграциями проверяйте:

- Есть ли rollback-секция?
- Нет ли конфликта имён с другими ветками?
- Обратно ли совместимо изменение?
- Есть ли индексы для новых колонок, по которым будет WHERE/JOIN?
- Не мешает ли DDL и DML в одном changeset?
- На больших таблицах: был ли замерен размер? Нужен ли `CONCURRENTLY`?

> **Лайфхак:** Создайте шаблон PR-описания для миграций. Разработчик заполняет чеклист перед ревью. Это дисциплинирует и ловит 80% проблем.

---

## 5. DDL и DML: никогда в одном changeset

### Почему это важно

PostgreSQL выполняет DDL внутри транзакции. Но если вы смешиваете DDL и DML, и DML падает — возникает неприятная ситуация.

```sql
--changeset job4j:070_create_and_seed_roles
CREATE TABLE roles (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);
INSERT INTO roles (name) VALUES ('ADMIN'), ('USER'), ('ADMIN');
-- Дубликат 'ADMIN' → UNIQUE violation → транзакция откатывается
-- Но changeset помечен как FAILED
```

Таблица `roles` не создана (транзакция откатилась). Changeset помечен как `FAILED`. При повторном запуске Liquibase попытается выполнить changeset заново — и всё заработает (если вы исправили дубликат). Но в некоторых СУБД или при `runInTransaction:false` — таблица уже создана, и повторный `CREATE TABLE` упадёт.

### Правильная структура

```sql
-- Файл: 070_ddl_create_roles.sql
--liquibase formatted sql
--changeset job4j:070_create_roles
CREATE TABLE roles (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);
--rollback DROP TABLE roles;
```

```sql
-- Файл: 071_dml_seed_roles.sql
--liquibase formatted sql
--changeset job4j:071_seed_roles
INSERT INTO roles (name) VALUES ('ADMIN'), ('USER');
--rollback DELETE FROM roles WHERE name IN ('ADMIN', 'USER');
```

Если `INSERT` упадёт — `CREATE TABLE` уже выполнен и зафиксирован. Исправляете INSERT, запускаете снова — только INSERT выполняется.

---

## 6. Подключение Liquibase к существующей БД (baseline)

### Ситуация

Проект работает полгода. БД создавалась руками или через SQL-скрипты. Миграций нет. Вы решили внедрить Liquibase.

Проблема: Liquibase видит changesets с `CREATE TABLE users` и пытается их выполнить. Таблица уже существует. Ошибка.

### Решение: changelogSync

Шаг 1. Создайте changelog с текущей схемой БД. Можно вручную, можно сгенерировать:

```bash
./gradlew generateChangelog \
    -PliquibaseChangelogFile=src/main/resources/db/changelog/changeset/000_baseline.xml
```

Это создаст XML-файл с текущей схемой (CREATE TABLE, индексы, constraints).

Шаг 2. Добавьте baseline в `db.changelog-master.xml`.

Шаг 3. Выполните `changelogSync` — это пометит ВСЕ changesets как выполненные **без фактического выполнения SQL**:

```bash
./gradlew changelogSync
```

Теперь Liquibase "знает" текущее состояние БД. Новые changesets будут выполняться как обычно.

> **Предупреждение:** Запускайте `changelogSync` только на той БД, схема которой соответствует вашему baseline changelog. Если схема отличается — вы получите расхождение между реальной БД и тем, что Liquibase считает выполненным.

### Альтернатива: preconditions

Если вы не хотите делать `changelogSync`, добавьте preconditions к baseline changesets:

```sql
--liquibase formatted sql
--changeset job4j:000_create_users
--preconditions onFail:MARK_RAN
--precondition-sql-check expectedResult:0 SELECT COUNT(*) FROM information_schema.tables WHERE table_name='users' AND table_schema='public'

CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username VARCHAR(255) NOT NULL
);
```

Если таблица уже существует — changeset будет помечен как `MARK_RAN` без выполнения. Если нет — создаст таблицу. Работает на любой БД без ручного `changelogSync`.

---

## 7. Индексы: CONCURRENTLY и runInTransaction

### Проблема

```sql
CREATE INDEX idx_orders_customer ON orders(customer_id);
```

Обычный `CREATE INDEX` берёт `SHARE` lock на таблицу. Пока индекс строится — никакие `INSERT`, `UPDATE`, `DELETE` не проходят. На таблице в 50 миллионов строк — это минуты блокировки.

### Решение: CONCURRENTLY

```sql
--liquibase formatted sql
--changeset job4j:080_add_index_orders_customer runInTransaction:false
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders(customer_id);
--rollback DROP INDEX IF EXISTS idx_orders_customer;
```

`CONCURRENTLY` строит индекс без блокировки записи. Но у него есть ограничение: **нельзя выполнять внутри транзакции**. Поэтому обязателен `runInTransaction:false`.

### Ловушка: CONCURRENTLY может "зависнуть"

`CREATE INDEX CONCURRENTLY` ждёт завершения всех текущих транзакций. Если есть долгая транзакция (забытый `BEGIN` в psql, зависший процесс) — индекс будет ждать бесконечно.

Проверяйте перед созданием индекса:

```sql
SELECT pid, state, query_start, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < now() - interval '5 minutes';
```

### Ловушка: CONCURRENTLY может создать INVALID индекс

Если `CREATE INDEX CONCURRENTLY` упадёт (нехватка памяти, deadlock) — индекс останется в состоянии `INVALID`. Он не используется планировщиком, но занимает место и замедляет INSERT.

Проверка:

```sql
SELECT indexrelid::regclass, indisvalid
FROM pg_index
WHERE NOT indisvalid;
```

Решение: удалить и пересоздать.

```sql
DROP INDEX CONCURRENTLY idx_orders_customer;
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders(customer_id);
```

---

## 8. Мониторинг и диагностика

### Три запроса, которые расскажут всё

Когда миграция сломалась — не надо читать весь лог. Начните с этих запросов:

```sql
-- 1. Последние выполненные changesets
SELECT id, author, filename, dateexecuted, exectype, orderexecuted
FROM databasechangelog
ORDER BY orderexecuted DESC
LIMIT 10;

-- 2. Есть ли FAILED changesets?
SELECT id, author, filename, dateexecuted, description
FROM databasechangelog
WHERE exectype = 'FAILED';

-- 3. Кто держит lock?
SELECT locked, lockgranted, lockedby
FROM databasechangeloglock;
```

### Проверка в CI/CD pipeline

Добавьте этап верификации после миграции:

```groovy
stage('Verify Migration') {
    steps {
        script {
            def failed = sh(
                script: '''
                    PGPASSWORD=$DB_PASSWORD psql -h db -U postgres -d job4j_devops \
                        -t -A -c "SELECT COUNT(*) FROM databasechangelog WHERE exectype = 'FAILED'"
                ''',
                returnStdout: true
            ).trim()
            if (failed != '0') {
                error "Found ${failed} FAILED migration(s)!"
            }
        }
    }
}
```

### Алерт на долгие миграции

Если миграция выполняется дольше 30 секунд — это повод разобраться:

```sql
SELECT id, filename,
       dateexecuted,
       deployment_id,
       description
FROM databasechangelog
WHERE dateexecuted > now() - interval '1 hour'
ORDER BY dateexecuted DESC;
```

---

## 9. Безопасное удаление таблиц и колонок

### Правило трёх недель

Никогда не делайте `DROP TABLE` или `DROP COLUMN` сразу. Используйте паттерн "deprecate → verify → drop":

**Неделя 1: переименование**

```sql
--changeset job4j:090_deprecate_legacy_payments
ALTER TABLE legacy_payments RENAME TO _deprecated_legacy_payments_20260327;
```

Если какой-то забытый сервис, cron-job или отчёт обращался к этой таблице — он упадёт. Вы узнаете об этом из алертов, а не из потери данных.

**Неделя 2-3: наблюдение**

Мониторьте ошибки в логах приложения. Если за две недели никто не пожаловался — таблица действительно не используется.

**Неделя 3+: удаление**

```sql
--changeset job4j:095_drop_deprecated_legacy_payments
DROP TABLE _deprecated_legacy_payments_20260327;
```

> **Лайфхак:** Перед DROP сделайте `pg_dump` только этой таблицы. Если через месяц окажется, что данные нужны — восстановите из дампа, а не из полного бэкапа.

### Для колонок — тот же принцип

```sql
-- Шаг 1: перестаём читать колонку в коде (деплой v1)
-- Шаг 2: перестаём писать в колонку (деплой v2)

-- Шаг 3: ставим DEFAULT NULL, убираем NOT NULL
--changeset job4j:091_relax_old_status
ALTER TABLE orders ALTER COLUMN old_status DROP NOT NULL;
ALTER TABLE orders ALTER COLUMN old_status SET DEFAULT NULL;

-- Шаг 4: через 2 недели — удаляем
--changeset job4j:096_drop_old_status
ALTER TABLE orders DROP COLUMN old_status;
```

---

## 10. Чеклист: перед выполнением миграции на production

Распечатайте и повесьте рядом с монитором:

```
□  Миграция протестирована на staging с production-like данными
□  ./gradlew validate — прошёл без ошибок
□  ./gradlew updateSql — прочитан и проверен сгенерированный SQL
□  Есть rollback-секция для каждого changeset
□  rollback протестирован: update → rollback → update работает
□  Бэкап БД сделан ДО миграции
□  Тег поставлен: ./gradlew tag -PliquibaseTag=<release-version>
□  ALTER на большие таблицы: размер проверен, CONCURRENTLY если нужно
□  DDL и DML в разных changesets
□  Нет ломающих изменений (RENAME/DROP) без expand-contract
□  Команда оповещена о предстоящей миграции
□  Мониторинг и алерты включены
□  План отката готов и задокументирован
```

---

## Итого

| Грабли | Как избежать |
|---|---|
| Залипший lock | `releaseLocks` + таймаут в Jenkins |
| ALTER на большой таблице | Проверять размер, разбивать на этапы, `CONCURRENTLY` |
| Сломал старую версию приложения | Expand → Migrate → Contract |
| Конфликт номеров миграций | Timestamp в имени + `includeAll` |
| DDL + DML в одном changeset | Разделять: структура отдельно, данные отдельно |
| Внедрение в существующий проект | `changelogSync` или preconditions |
| CREATE INDEX заблокировал таблицу | `CONCURRENTLY` + `runInTransaction:false` |
| Не понятно что сломалось | 3 диагностических SQL-запроса |
| DROP уничтожил нужные данные | Rename → 2 недели → Drop |
| Миграция упала на production | Чеклист перед деплоем |

---

## Задание

Это задание — симуляция реальной production-ситуации:

1. Создайте таблицу `orders` с колонками `id`, `user_id (FK)`, `amount`, `created_at`.
2. Вставьте 5 тестовых записей (отдельный changeset с `context:dev`).
3. Добавьте колонку `status` с DEFAULT — без NOT NULL.
4. Создайте индекс на `user_id` с `CONCURRENTLY` и `runInTransaction:false`.
5. Выполните `./gradlew update`.
6. Откатите 2 последних changeset через `rollbackCount`.
7. Убедитесь, что индекс и колонка `status` исчезли, а таблица `orders` на месте.
8. Выполните `./gradlew update` повторно — всё должно примениться снова.
9. Загрузите результат в репозиторий.