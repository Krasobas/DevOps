# Traefik: reverse proxy с автоматическим SSL

> **Стек:** Hetzner Cloud (Ubuntu 24.04), Docker, домен на Cloudflare
> **Подход:** production-grade. Всё, что написано ниже, можно брать и ставить в рабочую среду.

---

## Зачем нужен reverse proxy

На сервере работают Jenkins (порт 8080), Grafana (порт 3000), SonarQube (порт 9000). Без reverse proxy варианты доступа плохие:

- `http://168.119.xx.xx:8080` — запоминать порты, нет SSL
- Каждый сервис на 443 — невозможно, порт один
- nginx с ручными конфигами и Certbot — работает, но ручной труд при каждом изменении

Reverse proxy слушает 80/443 и по имени хоста (поддомену) решает, куда направить запрос:

```
jenkins.krasobas.com → Jenkins:8080
grafana.krasobas.com → Grafana:3000
sonar.krasobas.com   → SonarQube:9000
```

### Почему Traefik

**nginx** — статический. Добавил сервис → редактируй конфиг → перезагрузи nginx → запусти certbot.

**Traefik** — динамический. Следит за Docker, автоматически обнаруживает контейнеры. Поднял контейнер с labels → Traefik создал роут, получил SSL. Без перезагрузки, без правки конфигов.

> **Когда nginx лучше:** статичная инфраструктура из 2-3 сервисов, которые не меняются годами, и нужен тонкий контроль над кэшированием, rewrite rules, rate limiting на уровне location. Nginx быстрее в raw throughput. Но для Docker-среды, где контейнеры поднимаются и останавливаются — Traefik правильный выбор.

---

## 1. Подготовка

### Docker-сеть

```bash
docker network create proxy
```

Зачем отдельная сеть? Контейнеры в Docker изолированы друг от друга. Traefik должен «видеть» контейнеры, которые он проксирует. Общая сеть `proxy` — мост между ними.

> **Почему не дефолтная сеть `bridge`?** В дефолтной сети контейнеры общаются только по IP. В пользовательской сети работает встроенный DNS — контейнеры видят друг друга по имени (`jenkins`, `grafana`). Traefik использует именно это.

### Структура проекта

```bash
mkdir -p /opt/traefik
cd /opt/traefik
```

---

## 2. Docker Socket Proxy

Traefik нужен доступ к Docker API, чтобы обнаруживать контейнеры. Стандартный совет — примонтировать `/var/run/docker.sock` напрямую. Это плохо: Docker-сокет — это **полный root-доступ** к хосту. Если Traefik будет скомпрометирован (уязвимость, неправильная конфигурация), атакующий через сокет может создать привилегированный контейнер и получить контроль над сервером.

**Docker Socket Proxy** — лёгкий прокси (на базе HAProxy), который выставляет Docker API с ограниченными правами. Traefik получает доступ только на чтение, только к нужным endpoint'ам.

Это не паранойя — это стандартная практика. CVE на Traefik выходят регулярно, и ограничение blast radius — базовая гигиена.

### Что мы разрешаем Socket Proxy

```
CONTAINERS=1  — читать список контейнеров (Traefik нужно)
NETWORKS=1    — читать сети (Traefik нужно для маршрутизации)
SERVICES=0    — Swarm-сервисы (не используем)
TASKS=0       — Swarm-задачи (не используем)
POST=0        — запрещаем любые изменения (создание/удаление контейнеров)
```

Traefik через этот прокси может только **смотреть**, но не **менять**. Именно то, что нужно.

---

## 3. Конфигурация Traefik

У Traefik два уровня конфигурации:

**Статическая** (`traefik.yml`) — инфраструктура: порты, провайдеры, resolvers. Читается **один раз** при старте. Изменил — рестарт.

**Динамическая** (Docker labels) — маршруты: какой домен на какой контейнер. Читается **в реальном времени**. Добавил контейнер — Traefik подхватит за секунды.

### traefik.yml

```yaml
# ─── API и дашборд ───────────────────────────────────────────────
api:
  dashboard: true
  # Дашборд защитим через labels с Basic Auth.
  # Никогда не ставь insecure: true — это открывает дашборд без пароля.

# ─── Логирование ─────────────────────────────────────────────────
log:
  level: WARN
  # Уровни: DEBUG > INFO > WARN > ERROR
  # WARN — оптимальный баланс: видишь проблемы, не тонешь в логах.
  # При дебаге временно ставь DEBUG, но не забудь вернуть —
  # DEBUG генерирует гигабайты за сутки.

accessLog:
  filePath: /var/log/traefik/access.log
  bufferingSize: 100
  # Access log — кто, когда, куда ходил. Нужен для аудита и диагностики.
  # bufferingSize — пишет пачками по 100 записей, снижает нагрузку на диск.

# ─── Точки входа ─────────────────────────────────────────────────
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
          # permanent: true → HTTP 301 (постоянный редирект).
          # Браузер запомнит и больше не будет ходить на HTTP.
          # Без permanent — 302 (временный), браузер будет проверять каждый раз.
  websecure:
    address: ":443"
    http:
      tls:
        certResolver: letsencrypt
        # Дефолтный resolver для всех роутов на этом entryPoint.
        # Без него пришлось бы указывать certResolver в labels каждого контейнера.
    transport:
      respondingTimeouts:
        readTimeout: 60s
        writeTimeout: 60s
        idleTimeout: 180s
        # Таймауты предотвращают зависание соединений.
        # 60 секунд — достаточно для большинства сервисов.
        # Если Jenkins pipeline стримит лог дольше — увеличь writeTimeout.

# ─── Сертификаты ─────────────────────────────────────────────────
certificatesResolvers:
  letsencrypt:
    acme:
      email: твой-email@example.com       # ← ЗАМЕНИ! Алерты об истечении.
      storage: /acme.json
      httpChallenge:
        entryPoint: web
        # Traefik использует порт 80 для HTTP-01 challenge.
        # Let's Encrypt делает запрос → Traefik отвечает → сертификат выдан.

      # ──── STAGING (раскомментируй при первой настройке) ────
      # caServer: https://acme-staging-v02.api.letsencrypt.org/directory
      # Staging выдаёт невалидные сертификаты, но БЕЗ RATE LIMITS.
      # Когда убедишься, что всё работает:
      #   1. Закомментируй caServer обратно
      #   2. Очисти acme.json: truncate -s 0 /opt/traefik/acme.json
      #   3. docker compose restart traefik
      # Traefik получит настоящие сертификаты.

# ─── Провайдеры ──────────────────────────────────────────────────
providers:
  docker:
    endpoint: "tcp://socket-proxy:2375"
    # Подключаемся к Docker API через Socket Proxy, а не напрямую.
    # Socket Proxy ограничивает доступ только чтением — если Traefik
    # скомпрометирован, атакующий не сможет управлять контейнерами.
    exposedByDefault: false
    # КРИТИЧНО. По умолчанию Traefik публикует ВСЕ контейнеры.
    # Это значит, что случайно поднятый контейнер с базой данных
    # окажется доступен из интернета. Требуем явный label
    # "traefik.enable=true" для каждого сервиса.
    network: proxy
  file:
    directory: /etc/traefik/dynamic
    watch: true
    # File provider — для сервисов, которые работают ВНЕ Docker:
    # приложения на хосте, внешние серверы, legacy-сервисы.
    # watch: true — Traefik следит за изменениями в директории
    # и подхватывает новые конфиги без рестарта.
    # Кладёшь YAML-файл → роут появляется через секунды.
    # Удаляешь файл → роут исчезает.
```

### docker-compose.yml

```yaml
services:
  # ─── Docker Socket Proxy ──────────────────────────────────────
  socket-proxy:
    image: tecnativa/docker-socket-proxy:latest
    container_name: socket-proxy
    restart: unless-stopped
    environment:
      - CONTAINERS=1       # чтение контейнеров — нужно Traefik
      - NETWORKS=1         # чтение сетей — нужно для маршрутизации
      - SERVICES=0         # Swarm — не используем
      - TASKS=0            # Swarm — не используем
      - POST=0             # запрет на любые изменения
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - socket-proxy
    # Socket Proxy намеренно НЕ в сети proxy.
    # Он доступен только Traefik через отдельную сеть.
    # Ни один другой контейнер не может через него добраться до Docker API.

  # ─── Traefik ──────────────────────────────────────────────────
  traefik:
    image: traefik:v3.3
    container_name: traefik
    restart: unless-stopped
    depends_on:
      - socket-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./traefik.yml:/traefik.yml:ro
      - ./acme.json:/acme.json
      - ./dynamic:/etc/traefik/dynamic:ro
      - traefik_logs:/var/log/traefik
    networks:
      - proxy
      - socket-proxy
    # Traefik в двух сетях:
    #   proxy — для связи с контейнерами-сервисами
    #   socket-proxy — для связи с Docker Socket Proxy
    labels:
      # ─── Дашборд ────────────────────────────────────────────
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.krasobas.com`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.service=api@internal"

      # ─── Защита дашборда ─────────────────────────────────────
      - "traefik.http.routers.dashboard.middlewares=dashboard-auth,security-headers"
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$2y$$05$$ХЭШПАРОЛЯ"

      # ─── Security Headers (для всех сервисов) ───────────────
      - "traefik.http.middlewares.security-headers.headers.browserXssFilter=true"
      - "traefik.http.middlewares.security-headers.headers.contentTypeNosniff=true"
      - "traefik.http.middlewares.security-headers.headers.frameDeny=true"
      - "traefik.http.middlewares.security-headers.headers.stsIncludeSubdomains=true"
      - "traefik.http.middlewares.security-headers.headers.stsSeconds=31536000"
      - "traefik.http.middlewares.security-headers.headers.referrerPolicy=strict-origin-when-cross-origin"
      # Эти заголовки защищают от типовых атак:
      # - XSS-фильтр браузера
      # - Запрет MIME-type sniffing
      # - Запрет встраивания в iframe (clickjacking)
      # - HSTS: браузер запомнит, что сайт работает только по HTTPS
      # - Referrer Policy: не утекает полный URL при переходе на другой сайт

volumes:
  traefik_logs:

networks:
  proxy:
    external: true
  socket-proxy:
    # Внутренняя сеть, только между Traefik и Socket Proxy.
    # external: false (по умолчанию) — Compose создаст её сам.
```

### Генерация пароля

```bash
sudo apt install -y apache2-utils

# Замени admin и пароль на свои:
echo $(htpasswd -nbB admin 'MyStr0ngP@ss')
```

Результат: `admin:$2y$05$Wrs...`

Вставь в labels, **заменив каждый `$` на `$$`**:

```
admin:$2y$05$Wrs...  →  admin:$$2y$$05$$Wrs...
```

> **Почему `$$`?** Docker Compose интерпретирует `$` как переменную окружения (`$HOME`). Удвоение экранирует символ. Без этого пароль будет обрезан, и ты не сможешь залогиниться — потратишь час на дебаг.

> **Почему `-nbB`?** Флаг `-B` — bcrypt. Без него htpasswd использует MD5-apr1, который слабее. Traefik поддерживает оба, но bcrypt — стандарт.

### Подготовка файлов

```bash
cd /opt/traefik

# acme.json — хранилище сертификатов (приватные ключи внутри!)
touch acme.json
chmod 600 acme.json
# Traefik откажется запускаться, если права шире 600.

# Директория для динамических конфигов (file provider)
mkdir -p dynamic
```

> **acme.json должен быть пустым.** Не клади туда `{}` или `[]` — Traefik заполнит сам. Любой невалидный контент вызовет ошибку парсинга.

---

## 4. Запуск и проверка

```bash
cd /opt/traefik
docker compose up -d
docker logs traefik -f
```

Что должно быть в логах:
- `Configuration loaded from file: /traefik.yml`
- `Starting provider *docker.Provider`
- `Obtained certificate` для traefik.krasobas.com

### Типичные ошибки

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `permission denied` на acme.json | Права не 600 | `chmod 600 acme.json` |
| `unable to obtain ACME certificate` | DNS не обновился | `dig домен @8.8.8.8`, подожди |
| `challenge failed` | Порт 80 закрыт или Cloudflare Proxied | Проверь firewall Hetzner и тучку |
| `too many certificates` | Rate limit | Раскомментируй caServer (staging) |
| `connection refused` к socket-proxy | Socket Proxy не запустился | `docker logs socket-proxy` |

> **Firewall Hetzner:** в Cloud Console есть встроенный firewall. Если включён — открой порты 80 и 443 для входящих. Без 80-го Let's Encrypt не пройдёт challenge. Это частая причина «всё настроил, а сертификат не выдаётся».

---

## 5. Подключение Jenkins

Каждый сервис — свой каталог, свой docker-compose. Это не просто аккуратность: разные lifecycle, независимый `docker compose up/down`, чистые логи.

```bash
mkdir -p /opt/jenkins
```

### /opt/jenkins/docker-compose.yml

```yaml
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    volumes:
      - jenkins_data:/var/jenkins_home
    expose:
      - "8080"
    # ↑ expose, а не ports. Принципиальная разница:
    # ports: "8080:8080" — открывает порт В ИНТЕРНЕТ, мимо Traefik.
    # Любой может зайти http://IP:8080 напрямую, без SSL, без авторизации.
    # expose — порт виден только внутри Docker-сети.
    # К Jenkins можно попасть ТОЛЬКО через Traefik.
    networks:
      - proxy
    environment:
      - JENKINS_OPTS=--httpPort=8080
      # Jenkins нужно знать, на каком порту слушать.
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jenkins.rule=Host(`jenkins.krasobas.com`)"
      - "traefik.http.routers.jenkins.entrypoints=websecure"
      - "traefik.http.services.jenkins.loadbalancer.server.port=8080"
      - "traefik.http.routers.jenkins.middlewares=security-headers@docker"
      # ↑ Подключаем security headers, определённые в docker-compose Traefik.
      # @docker говорит: «возьми middleware из Docker-провайдера».

volumes:
  jenkins_data:
  # Именованный volume. Данные Jenkins (джобы, плагины, credentials)
  # переживут пересоздание контейнера. Без volume — всё пропадёт
  # при docker compose down/up.

networks:
  proxy:
    external: true
```

### Разбор ключевых решений

**`expose` vs `ports`** — одна из самых частых ошибок в production. Люди ставят Traefik, настраивают SSL, радуются — а сервис торчит голым на прямом порту мимо всей защиты. Проверяй себя:

```bash
# После запуска — НИ ОДИН сервисный порт не должен быть виден:
sudo ss -tlnp | grep -E ':(8080|3000|9000)'
# Пусто — правильно.
# Если что-то показало — где-то ports вместо expose.
```

**`loadbalancer.server.port=8080`** — Traefik должен знать внутренний порт контейнера. Если в Dockerfile один `EXPOSE` — Traefik может угадать. Но явно указать надёжнее: не зависишь от того, как собран образ.

**Почему нет `tls.certresolver` в labels?** Потому что мы настроили дефолтный resolver в `traefik.yml` (`entryPoints.websecure.http.tls.certResolver`). Все роуты на `websecure` автоматически получают сертификат.

### Запуск

```bash
cd /opt/jenkins
docker compose up -d

# Начальный пароль:
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Открой https://jenkins.krasobas.com — Jenkins с валидным SSL.

> **Jenkins стартует 1-3 минуты** при первом запуске (скачивает плагины, инициализирует базу). Следи через `docker logs jenkins -f`.

> **После начальной настройки:** Manage Jenkins → System → Jenkins URL → `https://jenkins.krasobas.com/`. Без этого webhook'и из GitHub будут содержать неправильный callback URL, и автоматические билды не запустятся.

---

## 6. Шаблон для любого сервиса

Четыре замены — и сервис доступен по HTTPS:

```yaml
services:
  СЕРВИС:                                              # 1. имя
    image: ОБРАЗ:latest                                # 2. образ
    container_name: СЕРВИС
    restart: unless-stopped
    expose:
      - "ПОРТ"                                         # 3. внутренний порт
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.СЕРВИС.rule=Host(`ПОДДОМЕН.krasobas.com`)"  # 4. поддомен
      - "traefik.http.routers.СЕРВИС.entrypoints=websecure"
      - "traefik.http.services.СЕРВИС.loadbalancer.server.port=ПОРТ"
      - "traefik.http.routers.СЕРВИС.middlewares=security-headers@docker"

networks:
  proxy:
    external: true
```

DNS не нужно трогать — wildcard `*` покрывает все поддомены.

---

## 7. Готовые примеры

### Grafana (grafana.krasobas.com)

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    volumes:
      - grafana_data:/var/lib/grafana
    expose:
      - "3000"
    networks:
      - proxy
    environment:
      - GF_SERVER_ROOT_URL=https://grafana.krasobas.com
      # Grafana должна знать свой публичный URL.
      # Без этого OAuth, embed-ссылки и API будут ломаться.
      - GF_SECURITY_ADMIN_PASSWORD=ЗАМЕНИ_НА_СИЛЬНЫЙ_ПАРОЛЬ
      # Дефолтный пароль admin/admin. Grafana попросит сменить при входе,
      # но лучше задать сразу через переменную — один шаг вместо двух.
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.krasobas.com`)"
      - "traefik.http.routers.grafana.entrypoints=websecure"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"
      - "traefik.http.routers.grafana.middlewares=security-headers@docker"

volumes:
  grafana_data:

networks:
  proxy:
    external: true
```

### SonarQube (sonar.krasobas.com)

SonarQube использует Elasticsearch, который требует увеличенный лимит виртуальной памяти. **Настрой это на хосте перед запуском:**

```bash
# Применить сейчас:
sudo sysctl -w vm.max_map_count=524288

# Сохранить на перезагрузку:
echo "vm.max_map_count=524288" | sudo tee -a /etc/sysctl.conf
```

> **Почему именно 524288?** Это рекомендация SonarQube. Elasticsearch маппит файлы в память (mmap), и дефолтный лимит Linux (65530) слишком мал. Без этой настройки Elasticsearch упадёт при старте.

```yaml
services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    restart: unless-stopped
    volumes:
      - sonar_data:/opt/sonarqube/data
      - sonar_logs:/opt/sonarqube/logs
      - sonar_extensions:/opt/sonarqube/extensions
    expose:
      - "9000"
    networks:
      - proxy
    environment:
      - SONAR_WEB_CONTEXT=/
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.sonar.rule=Host(`sonar.krasobas.com`)"
      - "traefik.http.routers.sonar.entrypoints=websecure"
      - "traefik.http.services.sonar.loadbalancer.server.port=9000"
      - "traefik.http.routers.sonar.middlewares=security-headers@docker"

volumes:
  sonar_data:
  sonar_logs:
  sonar_extensions:

networks:
  proxy:
    external: true
```

> **Память:** SonarQube + Elasticsearch хотят минимум 2 GB RAM. На сервере с 4 GB это ощутимо. Следи через `docker stats`. Если OOM-killer убивает контейнеры — нужен сервер побольше или ограничение через `deploy.resources.limits.memory`.

---

## 8. Middlewares: встроенная защита

Middlewares — обработчики между Traefik и сервисом. Security headers мы уже определили глобально. Вот ещё полезные:

### Rate Limiting

```yaml
labels:
  - "traefik.http.middlewares.rate-limit.ratelimit.average=100"
  - "traefik.http.middlewares.rate-limit.ratelimit.burst=50"
  - "traefik.http.middlewares.rate-limit.ratelimit.period=1s"
  - "traefik.http.routers.myservice.middlewares=rate-limit,security-headers@docker"
```

100 запросов/сек в среднем, burst до 50 — защита от простого DDoS и брутфорса.

### IP Whitelist

```yaml
labels:
  - "traefik.http.middlewares.ip-whitelist.ipallowlist.sourcerange=YOUR_IP/32,SECOND_IP/32"
  - "traefik.http.routers.jenkins.middlewares=ip-whitelist,security-headers@docker"
```

Доступ только с определённых IP. Для Jenkins и админских панелей — разумная мера.

> **Если у тебя динамический IP:** используй VPN с фиксированным IP (WireGuard на этом же сервере или отдельном) и вайтлисть IP VPN. Это надёжнее, чем Basic Auth, для критичных сервисов вроде Jenkins.

### Комбинирование middlewares

```yaml
- "traefik.http.routers.jenkins.middlewares=ip-whitelist,rate-limit,security-headers@docker"
```

Порядок имеет значение: запрос проходит через middlewares слева направо.

---

## 9. File Provider: сервисы вне Docker

Docker labels работают только для Docker-контейнеров. Но не всё работает в Docker: приложение запущенное напрямую на хосте, база данных на отдельном сервере, legacy-система коллеги на другом IP. Для таких случаев — file provider.

### Как это работает

Traefik следит за директорией `/etc/traefik/dynamic` (мы настроили `watch: true`). Кладёшь YAML-файл — роут появляется. Удаляешь — исчезает. Без рестарта Traefik, без перезагрузки чего-либо.

Каждый файл в директории — самостоятельный конфиг. Можно один файл на все роуты, можно по файлу на сервис. По файлу на сервис — чище: удалил файл = убрал роут, никакого мусора.

### Пример 1: приложение на хосте

Допустим, ты запустил Node.js-приложение напрямую на хосте (без Docker) на порту 3000:

```bash
# /opt/traefik/dynamic/node-app.yml

http:
  routers:
    node-app:
      rule: "Host(`app.krasobas.com`)"
      entryPoints:
        - websecure
      service: node-app
      tls:
        certResolver: letsencrypt
      middlewares:
        - security-headers

  services:
    node-app:
      loadBalancer:
        servers:
          - url: "http://HOST_IP:3000"
          # HOST_IP — не 127.0.0.1!
          # Traefik работает в контейнере, и localhost для него —
          # это сам контейнер, а не хост. Используй:
          #   - IP хоста в сети Docker: обычно 172.17.0.1
          #   - Или реальный IP сервера
          # Узнать IP хоста из контейнера:
          #   docker exec traefik ip route | grep default | awk '{print $3}'

  middlewares:
    security-headers:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        frameDeny: true
        stsIncludeSubdomains: true
        stsSeconds: 31536000
        referrerPolicy: "strict-origin-when-cross-origin"
```

> **Ключевой момент — `url: "http://HOST_IP:3000"`:** Traefik внутри Docker-контейнера. `localhost` и `127.0.0.1` указывают на сам контейнер, а не на хост. Чтобы достучаться до приложения на хосте, используй IP хоста в Docker-сети. Обычно это `172.17.0.1` (дефолтный bridge gateway), но лучше проверить:
> ```bash
> docker exec traefik ip route | grep default | awk '{print $3}'
> ```

### Пример 2: внешний сервер

Сервис работает на другом сервере (другой IP):

```bash
# /opt/traefik/dynamic/external-api.yml

http:
  routers:
    external-api:
      rule: "Host(`api.krasobas.com`)"
      entryPoints:
        - websecure
      service: external-api
      tls:
        certResolver: letsencrypt

  services:
    external-api:
      loadBalancer:
        servers:
          - url: "http://10.0.0.50:8080"
        # Traefik проксирует запросы на другой сервер.
        # Сертификат Let's Encrypt получает Traefik (домен указывает на него),
        # а до бэкенда трафик идёт по HTTP внутри приватной сети.
        # Если бэкенд в публичной сети — используй https:// и добавь:
        # serversTransport: skip-verify@file  (если у бэкенда самоподписанный серт)
```

### Пример 3: балансировка между несколькими инстансами

```bash
# /opt/traefik/dynamic/balanced-app.yml

http:
  routers:
    balanced-app:
      rule: "Host(`app.krasobas.com`)"
      entryPoints:
        - websecure
      service: balanced-app
      tls:
        certResolver: letsencrypt

  services:
    balanced-app:
      loadBalancer:
        servers:
          - url: "http://10.0.0.10:3000"
          - url: "http://10.0.0.11:3000"
          - url: "http://10.0.0.12:3000"
        healthCheck:
          path: /health
          interval: 10s
          timeout: 3s
        # Traefik распределяет запросы между тремя серверами.
        # healthCheck: если сервер не отвечает на /health — Traefik
        # временно исключает его из ротации. Когда оживёт — вернёт.
```

### Docker Labels vs File Provider: когда что

| Ситуация | Подход |
|----------|--------|
| Сервис в Docker на этом сервере | Docker Labels |
| Приложение на хосте (без Docker) | File Provider |
| Сервис на другом сервере | File Provider |
| Legacy-система, которую нельзя трогать | File Provider |
| Балансировка между несколькими серверами | File Provider |
| Временный роут для тестирования | File Provider (создал файл — удалил файл) |

Оба провайдера работают **одновременно**. Docker Labels для контейнеров, File Provider для всего остального. Traefik объединяет роуты из обоих источников в единую таблицу маршрутизации.

> **Важно:** имена роутеров и сервисов должны быть уникальны **глобально** — между Docker Labels и File Provider тоже. Если в labels роутер называется `jenkins`, не создавай роутер `jenkins` в YAML-файле — будет конфликт. Traefik предупредит об этом в логах, но один из роутов перестанет работать.

> **Middlewares в File Provider** — независимы от Docker. Middleware `security-headers`, определённый в labels Traefik (через `@docker`), **не виден** из файлового конфига. Для file provider нужно определить свои middlewares прямо в YAML-файле (как показано в примере 1). Это частая ошибка: пишешь `middlewares: security-headers@docker` в YAML-файле, а Traefik не находит его и молча отбрасывает.

---

## 10. Бэкап

### Что бэкапить и почему

```bash
#!/bin/bash
# /opt/scripts/backup.sh

BACKUP_DIR="/opt/backups/$(date +%Y-%m-%d)"
mkdir -p "$BACKUP_DIR"

# 1. acme.json — сертификаты и приватные ключи.
# Потеря = перевыпуск всех сертификатов разом.
# Если поддоменов много — можно упереться в rate limit Let's Encrypt
# (50 сертификатов/неделю на домен).
cp /opt/traefik/acme.json "$BACKUP_DIR/acme.json"
chmod 600 "$BACKUP_DIR/acme.json"

# 2. Jenkins — джобы, плагины, настройки, credentials, pipeline history.
# Невосполнимо. Это часы/дни работы.
docker run --rm \
  -v jenkins_data:/data:ro \
  -v "$BACKUP_DIR":/backup \
  alpine tar czf /backup/jenkins_data.tar.gz -C /data .

# 3. Grafana — дашборды, datasources, алерты.
docker run --rm \
  -v grafana_data:/data:ro \
  -v "$BACKUP_DIR":/backup \
  alpine tar czf /backup/grafana_data.tar.gz -C /data .

# 4. Docker Compose файлы, конфиги и динамические роуты
tar czf "$BACKUP_DIR/configs.tar.gz" \
  /opt/traefik/docker-compose.yml \
  /opt/traefik/traefik.yml \
  /opt/traefik/dynamic/ \
  /opt/jenkins/docker-compose.yml

# Удаление бэкапов старше 30 дней
find /opt/backups -maxdepth 1 -type d -mtime +30 -exec rm -rf {} +

echo "Backup completed: $BACKUP_DIR"
```

```bash
chmod +x /opt/scripts/backup.sh

# Cron: ежедневно в 3:00
(crontab -l 2>/dev/null; echo "0 3 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1") | crontab -
```

> **Идеально** — отправлять бэкапы на внешнее хранилище (Hetzner Storage Box, S3, rsync на другой сервер). Бэкап на том же сервере защищает от случайного удаления, но не от отказа диска.

### Восстановление

```bash
# Восстановить Jenkins из бэкапа:
docker compose -f /opt/jenkins/docker-compose.yml down
docker run --rm \
  -v jenkins_data:/data \
  -v /opt/backups/2026-03-28:/backup \
  alpine sh -c "rm -rf /data/* && tar xzf /backup/jenkins_data.tar.gz -C /data"
docker compose -f /opt/jenkins/docker-compose.yml up -d
```

---

## 11. Диагностика

```bash
# Логи Traefik:
docker logs traefik -f

# Логи конкретного сервиса:
docker logs jenkins -f

# Контейнеры, которые видит Traefik:
docker ps --filter "label=traefik.enable=true" \
  --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Потребление ресурсов:
docker stats --no-stream

# Проверить сертификат:
echo | openssl s_client -connect jenkins.krasobas.com:443 \
  -servername jenkins.krasobas.com 2>/dev/null | \
  openssl x509 -noout -subject -dates

# Убедиться, что порты не торчат наружу:
sudo ss -tlnp | grep -E ':(8080|3000|9000)'
# Должно быть пусто!

# Access log (последние запросы):
docker exec traefik tail -20 /var/log/traefik/access.log
```

### Частые проблемы

**502 Bad Gateway** — Traefik нашёл роут, но контейнер недоступен:
- Контейнер стартует → подожди, `docker logs`
- Неправильный `loadbalancer.server.port`
- Контейнер не в сети `proxy`

**404 Not Found** — Traefik не нашёл роут:
- Нет `traefik.enable=true`
- Опечатка в `Host()`
- Контейнер не запущен

**Сертификат не выдаётся:**
- Cloudflare в Proxied (нужен DNS only)
- Порт 80 закрыт в firewall Hetzner Cloud
- DNS не обновился (`dig домен @8.8.8.8`)
- Rate limit → используй staging

---

## 12. Итоговая архитектура

```
                    Интернет
                       │
                       ▼
              ┌─────────────────┐
              │   Cloudflare    │
              │   DNS only      │
              │ *.krasobas.com  │
              │  → Hetzner IP   │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          │     Hetzner VPS         │
          │                         │
          │  ┌───────────────────┐  │
          │  │     Traefik       │  │
          │  │  :80 → :443       │  │
          │  │  auto SSL         │  │
          │  │  security headers │  │
          │  └──┬──────────┬──┘    │
          │     │          │       │
          │  Docker     File       │
          │  Labels     Provider   │
          │     │          │       │
          │  ┌──┘    ┌─────┘       │
          │  ▼       ▼             │
          │ Jenkins  Node.js app   │
          │ :8080    :3000 (хост)  │
          │ Grafana  External API  │
          │ :3000    (другой IP)   │
          │ SonarQube              │
          │ :9000                  │
          │                         │
          │  ┌───────────────┐      │
          │  │ Socket Proxy  │      │
          │  │ (read-only    │      │
          │  │  Docker API)  │      │
          │  └───────────────┘      │
          └─────────────────────────┘
```

### Чеклист нового сервиса

1. **DNS** — ничего (wildcard `*`)
2. **Хост (если нужно)** — `sysctl`, пакеты, etc.
3. **docker-compose.yml:**
   - `expose`, **никогда** `ports`
   - Сеть `proxy` (external: true)
   - Labels: `traefik.enable`, `Host()`, `entrypoints`, `port`, middlewares
4. `docker compose up -d`
5. **Проверить:**
   - `docker logs traefik` → `Obtained certificate`
   - `ss -tlnp | grep :ПОРТ` → пусто
   - Открыть https://поддомен.krasobas.com → работает
6. **Добавить в бэкап-скрипт** volume нового сервиса
