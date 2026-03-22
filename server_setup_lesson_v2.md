# Настройка Linux-сервера для CI/CD [#505341] v2

В этом уроке мы подготовим сервер для работы с CI/CD. Вместо локальной виртуальной машины
мы арендуем облачный VPS — это даст опыт, максимально приближённый к реальной работе:
публичный IP, SSH-доступ извне, настоящая сетевая среда.

## 1. Что такое сервер?

Сервер — это компьютер, предназначенный для хранения, обработки данных и предоставления
сервисов другим устройствам (клиентам) через сеть. Доступ к серверу осуществляется
удалённо по протоколу SSH (Secure Shell).

### Типы серверов

**Выделенный сервер (Dedicated)** — физический сервер, который полностью предоставляется
в ваше распоряжение. Подходит для задач, требующих максимальной производительности.
Стоимость: от €30-50/мес.

**Облачный сервер (VPS/VDS)** — виртуальная машина на физическом сервере. На одном
физическом сервере работает несколько виртуальных. Подходит для большинства задач
разработки и CI/CD. Стоимость: от €3-5/мес.

Для обучения и большинства рабочих задач достаточно облачного сервера (VPS).

## 2. Аренда VPS

Мы будем использовать **Hetzner Cloud** — немецкий провайдер с отличным соотношением
цена/качество. Это один из самых популярных хостингов среди разработчиков в мире.

### Почему Hetzner?

- Низкие цены: от €2.99/мес за 2 vCPU, 4 GB RAM, 40 GB NVMe
- NVMe-диски, DDoS-защита, IPv4/IPv6, файрвол — всё включено без доплат
- 20 TB трафика в месяц
- Дата-центры в Германии и Финляндии (Tier III)
- Почасовой биллинг — платите только за время использования
- Оплата любой картой Visa/Mastercard

### Создание сервера

1. Зарегистрируйтесь на [hetzner.com](https://www.hetzner.com).
2. Перейдите в **Cloud Console** → создайте новый проект.
3. Внутри проекта нажмите **Create Server** и выберите:

| Параметр | Значение |
|----------|----------|
| Location | Falkenstein (FSN) или Nuremberg (NBG) |
| Image | Ubuntu 24.04 |
| Type | Shared Resources → Cost Optimized → **x86 (Intel/AMD)** |
| Plan | **CX23**: 2 vCPU / 4 GB RAM / 40 GB NVMe |
| SSH Key | Добавьте ваш публичный ключ (см. следующий раздел) |
| Networking | IPv4 + IPv6 (по умолчанию) |
| Backups | Для обучения не нужно |
| Name | Любое, например `ci-cd-server` |

4. Нажмите **Create & Buy**.

Сервер будет готов через 30-60 секунд. IP-адрес появится в панели управления.

## 3. SSH-ключи

SSH-ключи — это пара криптографических ключей для безопасной аутентификации.
Приватный ключ хранится на вашем компьютере, публичный — на сервере.
Это безопаснее и удобнее, чем пароль.

### Генерация SSH-ключа

Если у вас ещё нет SSH-ключа, сгенерируйте его на **локальной машине**:

```bash
ssh-keygen -t ed25519
```

Нажмите Enter на все вопросы (можно без пароля для учебного сервера).

Будут созданы два файла:
- `~/.ssh/id_ed25519` — приватный ключ (никому не показывайте!)
- `~/.ssh/id_ed25519.pub` — публичный ключ (его добавляете на сервер)

Посмотрите публичный ключ:

```bash
cat ~/.ssh/id_ed25519.pub
```

Именно это содержимое нужно вставить при создании сервера в Hetzner (поле SSH Key).

## 4. Подключение к серверу

После создания сервера подключитесь по SSH:

```bash
ssh root@<IP_АДРЕС_СЕРВЕРА>
```

При первом подключении система спросит, доверяете ли вы этому серверу:

```
The authenticity of host '...' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no)?
```

Введите `yes`. Это нормально при первом подключении — ваш компьютер запоминает
отпечаток сервера и в дальнейшем будет проверять его автоматически.

### Настройка SSH config (опционально)

Чтобы не вводить IP каждый раз, добавьте алиас на **локальной машине**.
Откройте файл `~/.ssh/config`:

```bash
nano ~/.ssh/config
```

Добавьте:

```
Host hetzner
    HostName <IP_АДРЕС_СЕРВЕРА>
    User leha
```

Теперь подключение к серверу — одна команда:

```bash
ssh hetzner
```

SSH config также позволяет настроить ключи, порт, проброс портов и другие параметры —
это пригодится при работе с Jenkins.

## 5. Первоначальная настройка сервера

### Обновление системы

Первым делом обновите систему:

```bash
apt update && apt upgrade -y
```

`apt update` — обновляет список доступных пакетов из репозиториев.
`apt upgrade` — устанавливает новые версии уже установленных пакетов.

### Что такое репозиторий пакетов?

В Windows вы скачиваете `.exe` с сайта и запускаете. В Linux всё иначе — есть
централизованные хранилища программ, называемые **репозиториями**. Это серверы в интернете,
где лежат тысячи программ в виде готовых пакетов.

Когда вы пишете `apt install htop`, система идёт в репозиторий, скачивает программу
и устанавливает её. Аналогия из Java-мира: это как Maven-репозиторий. Без `<repository>`
в `pom.xml` Maven не знает, откуда качать артефакт. Здесь то же самое: без репозитория
apt не знает, откуда ставить программу.

У Ubuntu из коробки есть свои репозитории со стандартными программами. Но некоторые
программы (например, Docker) распространяются через собственные репозитории —
их нужно добавлять отдельно.

## 6. Создание пользователя

Работать под root — плохая практика. Одна ошибка может сломать всю систему.
Создадим отдельного пользователя:

### Создание пользователя и настройка прав

```bash
adduser leha
```

Придумайте пароль, остальные поля (Full Name и т.д.) можно пропустить через Enter.

Дайте пользователю права на выполнение команд от root через sudo:

```bash
usermod -aG sudo leha
```

Флаг `-aG` означает "append to Group" — добавить в группу, не удаляя из других.
Группа `sudo` даёт право выполнять команды через `sudo`.

### Копирование SSH-ключа

Чтобы новый пользователь мог подключаться по SSH без пароля:

```bash
mkdir -p /home/leha/.ssh
cp /root/.ssh/authorized_keys /home/leha/.ssh/
chown -R leha:leha /home/leha/.ssh
chmod 700 /home/leha/.ssh
chmod 600 /home/leha/.ssh/authorized_keys
```

Разберём команды:
- `mkdir -p` — создаёт директорию (флаг `-p` — создать родительские, если не существуют).
- `cp` — копирует файл с авторизованными ключами от root к новому пользователю.
- `chown -R leha:leha` — меняет владельца файлов на leha (рекурсивно для всей папки).
- `chmod 700` — права доступа: только владелец может читать, писать, входить в папку.
- `chmod 600` — только владелец может читать и писать файл.

SSH строго проверяет права на эти файлы — если они слишком открытые, подключение будет отклонено.

### Проверка

Подключитесь как новый пользователь (из нового терминала на локальной машине):

```bash
ssh leha@<IP_АДРЕС_СЕРВЕРА>
```

Убедитесь, что sudo работает:

```bash
sudo whoami
```

Должно вывести `root`.

## 7. Установка Docker

Docker нет в стандартных репозиториях Ubuntu. Чтобы его установить, нужно сначала
добавить официальный репозиторий Docker в систему.

### Подготовка репозитория

```bash
# Устанавливаем утилиты для работы с HTTPS и скачивания файлов
sudo apt-get update
sudo apt-get install -y ca-certificates curl

# Создаём папку для хранения GPG-ключей репозиториев
sudo install -m 0755 -d /etc/apt/keyrings

# Скачиваем публичный GPG-ключ Docker
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Добавляем репозиторий Docker в систему
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**Зачем GPG-ключ?** Docker Inc. подписывает свои пакеты криптографическим ключом.
При каждом `apt update` система проверяет подпись — это гарантирует, что пакеты
не подменены злоумышленником. Без ключа apt откажется ставить пакеты из этого репозитория.

### Установка Docker Engine

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

Что устанавливаем:
- `docker-ce` — Docker Engine (сам движок).
- `docker-ce-cli` — CLI-клиент (команда `docker`).
- `containerd.io` — среда выполнения контейнеров.
- `docker-buildx-plugin` — расширенный инструмент сборки образов.
- `docker-compose-plugin` — плагин Docker Compose v2 (команда `docker compose`).

### Настройка доступа без sudo

По умолчанию Docker работает от root. Чтобы ваш пользователь мог выполнять
Docker-команды без `sudo`, добавьте его в группу `docker`:

```bash
sudo groupadd docker        # создать группу (может уже существовать)
sudo usermod -aG docker $USER
```

Применить изменения (перелогиниться):

```bash
newgrp docker
```

### Проверка

```bash
docker --version
docker compose version
docker run hello-world
```

Все три команды должны выполниться без ошибок и без `sudo`.
Команда `docker run hello-world` скачает тестовый образ и выведет приветственное сообщение —
это подтверждает, что Docker Engine работает корректно.

## 8. Настройка окружения (опционально)

Этот раздел не обязателен, но делает работу с сервером приятнее и продуктивнее.

### Установка современных CLI-инструментов

```bash
sudo apt install -y zsh git curl wget htop btop eza bat fd-find \
  ripgrep neofetch unzip fontconfig
```

Что ставим и зачем:
- **zsh** — продвинутый шелл вместо стандартного bash. Автодополнение, плагины, темы.
- **git** — система контроля версий, понадобится для CI/CD.
- **curl / wget** — скачивание файлов. curl для API-запросов, wget для файлов.
- **htop** — интерактивный мониторинг процессов (замена `top`).
- **btop** — мониторинг с графиками CPU, RAM, сети, дисков.
- **eza** — современная замена `ls` с иконками и цветами.
- **bat** — замена `cat` с подсветкой синтаксиса и нумерацией строк.
- **fd-find** — быстрая замена `find` с простым синтаксисом.
- **ripgrep** — быстрая замена `grep`, умеет игнорировать `.gitignore`.
- **neofetch** — красивая сводка о системе при входе в терминал.

### Установка Oh My Zsh

Oh My Zsh — фреймворк для управления конфигурацией zsh: темы, плагины, алиасы.

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

На вопрос "Do you want to change your default shell to zsh?" ответьте **Y**.

### Плагины для Zsh

```bash
# Автодополнение команд из истории (серый текст-подсказка)
git clone https://github.com/zsh-users/zsh-autosuggestions \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

# Подсветка синтаксиса команд (правильные — зелёные, ошибки — красные)
git clone https://github.com/zsh-users/zsh-syntax-highlighting \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

### Тема Powerlevel10k

Красивый информативный промпт с иконками, git-статусом и временем выполнения команд.

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k
```

### Настройка ~/.zshrc

Обновите тему и плагины:

```bash
sed -i 's/ZSH_THEME="robbyrussell"/ZSH_THEME="powerlevel10k\/powerlevel10k"/' ~/.zshrc
sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions zsh-syntax-highlighting docker docker-compose)/' ~/.zshrc
```

Добавьте алиасы в конец файла:

```bash
cat >> ~/.zshrc << 'EOF'

# Modern CLI aliases
alias ls='eza --icons --group-directories-first'
alias ll='eza -la --icons --group-directories-first'
alias lt='eza --tree --icons --level=2'
alias cat='batcat --style=plain'
alias fd='fdfind'
alias top='btop'
alias ..='cd ..'
alias ...='cd ../..'
EOF
```

Применить изменения:

```bash
source ~/.zshrc
```

### Шрифт для Powerlevel10k

Для корректного отображения иконок в терминале нужен Nerd Font.
Установите его на **локальной машине** (не на сервере):

```bash
mkdir -p ~/.local/share/fonts
cd ~/.local/share/fonts
curl -fLO https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.tar.xz
tar -xf JetBrainsMono.tar.xz
fc-cache -fv
```

После установки выберите шрифт **JetBrainsMono Nerd Font** в настройках вашего терминала
и переподключитесь к серверу. Powerlevel10k запустит мастер настройки автоматически.

Для macOS: шрифт можно установить через Homebrew:

```bash
brew install --cask font-jetbrains-mono-nerd-font
```

Для Windows: скачайте шрифт с
[github.com/ryanoasis/nerd-fonts/releases](https://github.com/ryanoasis/nerd-fonts/releases),
распакуйте и установите через правый клик → "Установить для всех пользователей".

## Задание

1. Арендуйте VPS-сервер (Hetzner Cloud или аналог).
2. Подключитесь по SSH.
3. Создайте отдельного пользователя (не root).
4. Установите Docker и Docker Compose.
5. Выполните команды `docker --version` и `docker compose version`.
6. Отправьте скриншот терминала с результатом.
