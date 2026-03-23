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
cd ~/jenkins-server
make ssh-keys
```

Эта команда создаст два файла в папке `ssh-keys/`:
- `jenkins_agent_key` — приватный ключ (его добавим в Jenkins).
- `jenkins_agent_key.pub` — публичный ключ (его передадим агенту).

Под капотом выполняется:

```bash
ssh-keygen -t ed25519 -C "jenkins-agent" -f ssh-keys/jenkins_agent_key -N ""
```

- `-t ed25519` — современный алгоритм (быстрее и безопаснее RSA).
- `-C "jenkins-agent"` — комментарий для идентификации ключа.
- `-N ""` — пустая парольная фраза (для автоматического подключения).

## 2. Обновление compose.yaml

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

## 3. Makefile

Команда запуска требует передачи переменных окружения `DOCKER_GID` и `JENKINS_SSH_PUBKEY`.
Чтобы не запоминать их и не ошибиться, вынесем все операции в `Makefile`:

```makefile
# Jenkins CI/CD Server Management
# Usage: make [target]

DOCKER_GID := $(shell getent group docker | cut -d: -f3)
JENKINS_SSH_PUBKEY := $(shell cat ssh-keys/jenkins_agent_key.pub 2>/dev/null)

export DOCKER_GID
export JENKINS_SSH_PUBKEY

# ── Lifecycle ───────────────────────────────────────────

.PHONY: up down start stop restart build

up: ## Build and start all services
	docker compose up -d --build

down: ## Stop and remove containers (data preserved in volumes)
	docker compose down

start: ## Start stopped containers
	docker compose start

stop: ## Stop running containers
	docker compose stop

restart: ## Restart all services
	docker compose restart

build: ## Rebuild images without starting
	docker compose build

# ── Setup ───────────────────────────────────────────────

.PHONY: ssh-keys

ssh-keys: ## Generate SSH keys for Jenkins agent
	mkdir -p ssh-keys
	ssh-keygen -t ed25519 -C "jenkins-agent" -f ssh-keys/jenkins_agent_key -N ""
	@echo "SSH keys generated in ssh-keys/"

# ── Info & Logs ─────────────────────────────────────────

.PHONY: ps logs logs-jenkins logs-agent

ps: ## Show running containers
	docker compose ps

logs: ## Tail logs from all services
	docker compose logs -f

logs-jenkins: ## Tail Jenkins logs
	docker compose logs -f jenkins

logs-agent: ## Tail agent logs
	docker compose logs -f agent1

# ── Shell Access ────────────────────────────────────────

.PHONY: shell-jenkins shell-agent

shell-jenkins: ## Open shell in Jenkins container
	docker compose exec jenkins bash

shell-agent: ## Open shell in agent container
	docker compose exec agent1 bash

# ── Checks ──────────────────────────────────────────────

.PHONY: check-ssh check-docker

check-ssh: ## Test SSH connection from Jenkins to agent
	docker compose exec jenkins ssh -i /var/jenkins_home/.ssh/jenkins_agent_key -o StrictHostKeyChecking=no jenkins@agent1 "echo 'SSH OK'"

check-docker: ## Verify Docker CLI inside Jenkins
	docker compose exec jenkins docker --version
	docker compose exec jenkins docker compose version

# ── Danger Zone ─────────────────────────────────────────

.PHONY: nuke

nuke: ## Remove everything including volumes (ALL DATA LOST)
	@echo "This will delete ALL Jenkins data. Press Ctrl+C to cancel."
	@sleep 3
	docker compose down -v

# ── Help ────────────────────────────────────────────────

.PHONY: help

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-15s\033[0m %s\n", $$1, $$2}'

.DEFAULT_GOAL := help
```

Ключевые моменты:

- **Переменные вычисляются автоматически** — `DOCKER_GID` и `JENKINS_SSH_PUBKEY`
  определяются в шапке Makefile. Не нужно ничего запоминать и копировать вручную.
- **`make` без аргументов** выводит справку — список всех доступных команд с описанием.
- **`.PHONY`** — указывает make, что эти таргеты не являются файлами.
  Без этого, если в директории окажется файл с именем `up`, make решит,
  что "файл уже существует" и не выполнит команду.

## 4. Запуск

```bash
cd ~/jenkins-server
make up
```

Проверьте, что оба контейнера запущены:

```bash
make ps
```

Должны увидеть два контейнера (`jenkins` и `agent1`) в состоянии `Up`.

## 5. Проверка SSH-подключения

Прежде чем настраивать Jenkins, убедимся, что SSH-подключение работает:

```bash
make check-ssh
```

Если всё в порядке, увидите вывод `SSH OK`.

Если нужно отладить вручную:

```bash
make shell-jenkins
ssh -i /var/jenkins_home/.ssh/jenkins_agent_key jenkins@agent1
```

На вопрос о fingerprint ответьте `yes`. После успешного подключения введите `exit` дважды.

## 6. Добавление ключа в Jenkins

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

## 7. Настройка Node в Jenkins

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

**Host: `agent1`** — имя контейнера, а не IP-адрес. Работает стабильно
благодаря Docker-сети `jenkins-net`. При перезапуске контейнера IP может измениться,
но имя останется прежним.

**Host Key Verification Strategy: Non verifying** — отключаем проверку ключа хоста.
Для продакшена это небезопасно, но для учебного окружения в закрытой Docker-сети допустимо.

**Labels: `docker jdk21`** — метки, по которым Pipeline сможет выбирать,
на каком агенте запускать сборку. Например: `agent { label 'jdk21' }`.

5. Нажмите **Save**.

## 8. Проверка

После сохранения Jenkins начнёт подключаться к агенту. Подождите несколько секунд.

1. Перейдите в **Manage Jenkins** → **Nodes**.
2. Агент `agent1` должен появиться в списке.
3. Кликните на `agent1` → **Log**. Убедитесь, что в логе нет ошибок
   и есть строка `Agent successfully connected and online`.

Если агент не подключается, проверьте:
- Оба контейнера запущены: `make ps`
- SSH работает: `make check-ssh`
- Credentials в Jenkins используют правильный приватный ключ
- В настройках Node указан Host `agent1`, а не IP-адрес

## Краткая справка по командам

```bash
make              # показать все доступные команды
make up           # собрать и запустить всё
make down         # остановить (данные сохраняются)
make ps           # статус контейнеров
make logs         # логи всех сервисов
make logs-agent   # логи агента
make check-ssh    # проверить SSH между Jenkins и агентом
make check-docker # проверить Docker CLI внутри Jenkins
make shell-jenkins # зайти в контейнер Jenkins
make shell-agent  # зайти в контейнер агента
make nuke         # удалить всё включая данные (осторожно!)
```

## Итоговая структура проекта

```
~/jenkins-server/
├── Dockerfile              # Образ Jenkins с Docker CLI
├── Makefile                # Управление проектом
├── compose.yaml            # Jenkins + Agent
├── .gitignore              # Исключения для Git
└── ssh-keys/
    ├── jenkins_agent_key       # Приватный ключ (не коммитим!)
    └── jenkins_agent_key.pub   # Публичный ключ
```

Если проект хранится в Git, добавьте ключи в `.gitignore`:

```bash
echo "ssh-keys/" >> .gitignore
```

## Задание

1. Сгенерируйте SSH-ключи: `make ssh-keys`.
2. Обновите compose.yaml, добавив сервис agent1.
3. Запустите стек: `make up`.
4. Проверьте SSH: `make check-ssh`.
5. Настройте Credentials и Node в Jenkins.
6. Убедитесь, что агент подключился (статус "online" в логе).
7. Отправьте скриншот списка Nodes в Jenkins.
