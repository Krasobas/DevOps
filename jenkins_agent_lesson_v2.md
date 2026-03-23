# Настройка агента | Постоянный агент через SSH [#505343] v2

В этом уроке мы подключим к Jenkins постоянного агента через SSH.
Агент — это отдельный контейнер с JDK 21, на котором Jenkins будет выполнять сборки.

## Зачем нужен агент?

Jenkins работает по архитектуре **Master-Agent**:

- **Master** (контроллер) — управляет задачами, хранит конфигурацию, предоставляет веб-интерфейс.
  Сам сборки выполнять не должен — это нагружает его и создаёт риски безопасности.
- **Agent** (агент) — выполняет сборки по заданию мастера. Агентов может быть несколько,
  с разными окружениями (разные версии JDK, OS, инструменты).

Мастер подключается к агенту по SSH, отправляет задание, агент выполняет его и возвращает результат.

## 1. Генерация SSH-ключей

SSH-ключи нужны для того, чтобы контейнер Jenkins (мастер) мог подключаться
к контейнеру агента без пароля.

Сгенерируем ключи на **хост-машине** и положим в volume, чтобы они не потерялись
при пересоздании контейнеров:

```bash
mkdir -p ~/jenkins-server/ssh-keys
ssh-keygen -t ed25519 -C "jenkins-agent" -f ~/jenkins-server/ssh-keys/jenkins_agent_key -N ""
```

Параметры:
- `-t ed25519` — современный алгоритм (быстрее и безопаснее RSA).
- `-C "jenkins-agent"` — комментарий для идентификации ключа.
- `-f ...` — путь к файлу ключа.
- `-N ""` — пустая парольная фраза (для автоматического подключения).

Будут созданы два файла:
- `jenkins_agent_key` — приватный ключ (его добавим в Jenkins).
- `jenkins_agent_key.pub` — публичный ключ (его передадим агенту).

## 2. Обновление compose.yaml

Перейдите в директорию проекта:

```bash
cd ~/jenkins-server
```

Обновите `compose.yaml`, добавив сервис агента и сеть:

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
      - ./ssh-keys:/var/jenkins_home/.ssh
    group_add:
      - ${DOCKER_GID}
    networks:
      - jenkins-net
    restart: unless-stopped

  agent1:
    image: jenkins/ssh-agent:jdk21
    container_name: agent1
    environment:
      - JENKINS_AGENT_SSH_PUBKEY=${JENKINS_SSH_PUBKEY}
    networks:
      - jenkins-net
    restart: unless-stopped

networks:
  jenkins-net:

volumes:
  jenkins_home:
```

Что изменилось по сравнению с предыдущим уроком:

- **`agent1`** — новый сервис. Использует официальный образ `jenkins/ssh-agent:jdk21`
  с предустановленным SSH-сервером и JDK 21.
- **`jenkins-net`** — общая Docker-сеть. Контейнеры в одной сети видят друг друга
  **по имени**: Jenkins обращается к агенту просто как `agent1`, а не по IP-адресу.
  Это критически важно — IP контейнера меняется при каждом перезапуске,
  а имя остаётся стабильным.
- **`./ssh-keys:/var/jenkins_home/.ssh`** — монтируем папку с SSH-ключами внутрь
  контейнера Jenkins, чтобы мастер мог подключаться к агенту.
- **`JENKINS_AGENT_SSH_PUBKEY`** — переменная окружения с публичным ключом.
  Образ `jenkins/ssh-agent` при запуске автоматически добавит этот ключ
  в `authorized_keys` пользователя `jenkins` внутри контейнера агента.

## 3. Запуск

Запустите обновлённый стек. Публичный ключ передаём через переменную окружения:

```bash
cd ~/jenkins-server

DOCKER_GID=$(getent group docker | cut -d: -f3) \
JENKINS_SSH_PUBKEY="$(cat ssh-keys/jenkins_agent_key.pub)" \
docker compose up -d --build
```

Проверьте, что оба контейнера запущены:

```bash
docker compose ps
```

Должны увидеть два контейнера (`jenkins` и `agent1`) в состоянии `Up`.

## 4. Проверка SSH-подключения

Прежде чем настраивать Jenkins, убедимся, что SSH-подключение работает.

Зайдите в контейнер Jenkins:

```bash
docker compose exec jenkins bash
```

Попробуйте подключиться к агенту по имени:

```bash
ssh -i /var/jenkins_home/.ssh/jenkins_agent_key jenkins@agent1
```

На вопрос о fingerprint ответьте `yes`. Если подключение прошло успешно —
вы увидите приглашение командной строки агента. Введите `exit` дважды,
чтобы вернуться на хост-машину.

## 5. Добавление ключа в Jenkins

Откройте Jenkins в браузере: `http://<IP_ВАШЕГО_СЕРВЕРА>:9074`

### Добавление Credentials

1. Перейдите в **Manage Jenkins** → **Credentials**.
2. Кликните на домен **(global)** → **Add Credentials**.
3. Заполните форму:

| Поле | Значение |
|------|----------|
| Kind | SSH Username with private key |
| Scope | Global |
| ID | jenkins-agent-ssh (или любой понятный идентификатор) |
| Username | jenkins |
| Private Key | Enter directly → вставьте содержимое приватного ключа |

Чтобы получить содержимое приватного ключа, выполните на хост-машине:

```bash
cat ~/jenkins-server/ssh-keys/jenkins_agent_key
```

Скопируйте **весь** вывод (включая строки `-----BEGIN` и `-----END`) и вставьте
в поле Private Key.

4. Нажмите **Create**.

## 6. Настройка Node в Jenkins

1. Перейдите в **Manage Jenkins** → **Nodes**.
2. Нажмите **New Node**.
3. Заполните форму:

| Поле | Значение |
|------|----------|
| Node name | agent1 |
| Type | Permanent Agent |

4. Нажмите **Create** и заполните настройки:

| Поле | Значение |
|------|----------|
| Remote root directory | /home/jenkins |
| Labels | docker jdk21 (через пробел — метки для Pipeline) |
| Usage | Use this node as much as possible |
| Launch method | Launch agents via SSH |
| Host | agent1 |
| Credentials | jenkins (тот, что создали выше) |
| Host Key Verification Strategy | Non verifying Verification Strategy |

**Host: `agent1`** — это имя контейнера, а не IP-адрес. Оно работает стабильно
благодаря Docker-сети `jenkins-net`. При перезапуске контейнера IP может измениться,
но имя останется прежним.

**Host Key Verification Strategy: Non verifying** — отключаем проверку ключа хоста.
Для продакшена это небезопасно, но для учебного окружения в закрытой Docker-сети допустимо.

**Labels: `docker jdk21`** — метки, по которым Pipeline сможет выбирать,
на каком агенте запускать сборку. Например: `agent { label 'jdk21' }`.

5. Нажмите **Save**.

## 7. Проверка

После сохранения Jenkins начнёт подключаться к агенту. Подождите несколько секунд.

1. Перейдите в **Manage Jenkins** → **Nodes**.
2. Агент `agent1` должен появиться в списке.
3. Кликните на `agent1` → **Log**. Убедитесь, что в логе нет ошибок
   и есть строка `Agent successfully connected and online`.

Если агент не подключается, проверьте:
- Оба контейнера запущены: `docker compose ps`
- SSH работает: `docker compose exec jenkins ssh -i /var/jenkins_home/.ssh/jenkins_agent_key jenkins@agent1`
- Credentials в Jenkins используют правильный приватный ключ
- В настройках Node указан Host `agent1`, а не IP-адрес

## Полезные команды

```bash
# Просмотр логов агента
docker compose logs -f agent1

# Перезапуск агента
docker compose restart agent1

# Зайти внутрь контейнера агента
docker compose exec agent1 bash

# Проверить версию Java на агенте
docker compose exec agent1 java -version
```

## Итоговая структура проекта

```
~/jenkins-server/
├── Dockerfile           # Образ Jenkins с Docker CLI
├── compose.yaml         # Jenkins + Agent
└── ssh-keys/
    ├── jenkins_agent_key      # Приватный ключ (не коммитим в git!)
    └── jenkins_agent_key.pub  # Публичный ключ
```

Если проект хранится в Git, добавьте ключи в `.gitignore`:

```bash
echo "ssh-keys/" >> .gitignore
```

## Задание

1. Сгенерируйте SSH-ключи для подключения агента.
2. Обновите compose.yaml, добавив сервис agent1.
3. Настройте Credentials и Node в Jenkins.
4. Убедитесь, что агент подключился (статус "online" в логе).
5. Отправьте скриншот списка Nodes в Jenkins.
