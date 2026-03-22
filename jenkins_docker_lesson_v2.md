# Установка образа Jenkins в Docker [#505337] v2

В этом уроке мы развернём Jenkins в контейнере Docker. Jenkins — это сервер автоматизации, который мы будем использовать для настройки CI/CD процессов. Мы создадим собственный Docker-образ с предустановленным Docker CLI, настроим постоянное хранилище данных и завершим установку через веб-интерфейс.

## Почему собственный Docker-образ?

Официальный образ `jenkins/jenkins` не содержит Docker CLI. Без него Jenkins не сможет собирать Docker-образы и запускать контейнеры. Можно было бы установить Docker CLI вручную внутри контейнера, но при пересоздании контейнера всё потеряется. Поэтому мы создадим свой образ на основе официального, в котором Docker CLI уже будет установлен.

## 1. Подготовка проекта

Создайте директорию для проекта и перейдите в неё:

```bash
mkdir -p ~/jenkins-server
cd ~/jenkins-server
```

## 2. Создание Dockerfile

Создайте файл `Dockerfile`:

```bash
nano Dockerfile
```

Содержимое:

```dockerfile
FROM jenkins/jenkins:lts-jdk21

USER root

RUN apt-get update && \
    apt-get install -y ca-certificates curl && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg \
      -o /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) \
      signed-by=/etc/apt/keyrings/docker.asc] \
      https://download.docker.com/linux/debian bookworm stable" | \
      tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && \
    apt-get install -y docker-ce-cli docker-compose-plugin && \
    rm -rf /var/lib/apt/lists/*

USER jenkins
```

Разберём ключевые моменты:

- `FROM jenkins/jenkins:lts-jdk21` — берём за основу официальный образ Jenkins с JDK 21.
  Используем тег `lts` (Long Term Support) — это стабильная версия с долгосрочной поддержкой.
- `USER root` — переключаемся на пользователя root, чтобы иметь права на установку пакетов.
- Блок `RUN` — добавляем официальный репозиторий Docker и устанавливаем из него `docker-ce-cli`
  (клиент Docker) и `docker-compose-plugin` (поддержка `docker compose`).
  Обратите внимание: мы устанавливаем именно **клиент** (`docker-ce-cli`), а не Docker Engine целиком.
  Контейнер Jenkins будет использовать Docker Engine хост-машины через примонтированный сокет.
- `rm -rf /var/lib/apt/lists/*` — очищаем кеш пакетов, чтобы не увеличивать размер образа.
- `USER jenkins` — возвращаемся на пользователя jenkins. Это важно для безопасности:
  Jenkins не должен работать от root.

## 3. Создание compose.yaml

Создайте файл `compose.yaml`:

```bash
nano compose.yaml
```

Содержимое:

```yaml
services:
  jenkins:
    build: .
    container_name: jenkins
    ports:
      - "9074:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    group_add:
      - ${DOCKER_GID}
    restart: unless-stopped

volumes:
  jenkins_home:
```

Разберём каждый параметр:

- `build: .` — собрать образ из Dockerfile в текущей директории.
- `container_name: jenkins` — имя контейнера. Удобно для команд вроде `docker logs jenkins`.
- `ports`:
    - `9074:8080` — веб-интерфейс Jenkins. Порт 8080 внутри контейнера маппится на 9074 хост-машины.
    - `50000:50000` — порт для подключения Jenkins-агентов (пригодится позже).
- `volumes`:
    - `jenkins_home:/var/jenkins_home` — именованный volume для данных Jenkins.
      При пересоздании контейнера все настройки, задания и плагины сохраняются.
    - `/var/run/docker.sock:/var/run/docker.sock` — монтируем Docker-сокет хост-машины
      внутрь контейнера. Это позволяет Jenkins выполнять Docker-команды на хосте.
- `group_add: ${DOCKER_GID}` — добавляет пользователя jenkins в группу docker хост-машины
  (по GID). Без этого Jenkins не сможет обращаться к Docker-сокету.
- `restart: unless-stopped` — контейнер автоматически перезапускается при сбое или
  перезагрузке сервера (если не был остановлен вручную).
- `volumes: jenkins_home:` — объявление именованного volume. Docker сам управляет
  его расположением на диске.

### Почему именованный volume, а не bind mount?

В оригинальном уроке используется bind mount: `-v /var/jenkins_home:/var/jenkins_home`.
Это требует ручного создания папки и настройки прав (UID 1000:1000, chmod 775).

Именованный volume (`jenkins_home:`) управляется Docker-ом автоматически — не нужно
создавать папки и разбираться с правами. Это рекомендуемый подход для хранения данных приложений.

## 4. Запуск Jenkins

Соберите образ и запустите контейнер:

```bash
DOCKER_GID=$(getent group docker | cut -d: -f3) docker compose up -d --build
```

Команда `getent group docker | cut -d: -f3` получает числовой GID группы docker
на хост-машине и передаёт его в compose.yaml через переменную окружения.

Проверьте, что контейнер запущен:

```bash
docker compose ps
```

Вывод должен показать контейнер jenkins в состоянии `Up`.

## 5. Начальная настройка Jenkins

Для первого входа в Jenkins нужен одноразовый пароль администратора. Получите его из логов:

```bash
docker compose logs jenkins
```

В выводе найдите блок:

```
Jenkins initial setup is required. An admin user has been created
and a password has been generated.
Please use the following password to proceed to installation:

2d7e8a5b29547465498a50266c69086747

This may also be found at:
/var/jenkins_home/secrets/initialAdminPassword
```

Откройте браузер и перейдите по адресу:

```
http://<IP_ВАШЕГО_СЕРВЕРА>:9074
```

Далее:

1. Введите пароль администратора из логов.
2. Выберите "Install suggested plugins" — Jenkins установит набор рекомендуемых плагинов.
3. Создайте учётную запись администратора (логин, пароль, email).
4. Подтвердите URL Jenkins (оставьте по умолчанию).

После завершения откроется главный интерфейс Jenkins.

## 6. Проверка Docker внутри Jenkins

Убедимся, что Jenkins может использовать Docker. Зайдите в контейнер:

```bash
docker compose exec jenkins bash
```

Выполните:

```bash
docker --version
docker compose version
```

Если обе команды вывели версии — всё настроено правильно. Выйдите из контейнера:

```bash
exit
```

## Полезные команды

```bash
# Просмотр логов в реальном времени
docker compose logs -f jenkins

# Остановка Jenkins
docker compose stop

# Запуск Jenkins
docker compose start

# Перезапуск Jenkins
docker compose restart

# Полное удаление контейнера (данные в volume сохраняются)
docker compose down

# Удаление контейнера вместе с данными (ОСТОРОЖНО!)
docker compose down -v
```

## Задание

1. Создайте проект с Dockerfile и compose.yaml по инструкции выше.
2. Запустите Jenkins и завершите начальную настройку через веб-интерфейс.
3. Убедитесь, что Docker CLI доступен внутри контейнера Jenkins.
4. Отправьте скриншот главного интерфейса Jenkins.
