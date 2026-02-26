# Docker — Вопросы и ответы для собеседований (Middle/Senior DevOps)

## Содержание

### Архитектура и основы
1. [Как устроен Docker изнутри? Из чего состоит архитектура?](#1-как-устроен-docker-изнутри-из-чего-состоит-архитектура)
2. [В чём разница между образом (image) и контейнером?](#2-в-чём-разница-между-образом-image-и-контейнером)
3. [Как работает файловая система Docker? Что такое слои и Union FS?](#3-как-работает-файловая-система-docker-что-такое-слои-и-union-fs)
4. [Чем контейнер отличается от виртуальной машины?](#4-чем-контейнер-отличается-от-виртуальной-машины)

### Dockerfile
5. [Каковы лучшие практики написания Dockerfile?](#5-каковы-лучшие-практики-написания-dockerfile)
6. [Что такое multi-stage build и зачем он нужен?](#6-что-такое-multi-stage-build-и-зачем-он-нужен)
7. [В чём разница между CMD и ENTRYPOINT?](#7-в-чём-разница-между-cmd-и-entrypoint)
8. [В чём разница между COPY и ADD?](#8-в-чём-разница-между-copy-и-add)
9. [Что такое .dockerignore и почему он важен?](#9-что-такое-dockerignore-и-почему-он-важен)

### Сеть
10. [Какие типы сетей есть в Docker и чем они отличаются?](#10-какие-типы-сетей-есть-в-docker-и-чем-они-отличаются)
11. [Как контейнеры общаются между собой и с внешним миром?](#11-как-контейнеры-общаются-между-собой-и-с-внешним-миром)

### Хранилище данных
12. [В чём разница между volume, bind mount и tmpfs?](#12-в-чём-разница-между-volume-bind-mount-и-tmpfs)
13. [Как правильно управлять данными в Docker?](#13-как-правильно-управлять-данными-в-docker)

### Запуск и отладка
14. [Как дебажить работающий контейнер?](#14-как-дебажить-работающий-контейнер)
15. [Как работают сигналы и graceful shutdown в контейнерах?](#15-как-работают-сигналы-и-graceful-shutdown-в-контейнерах)
16. [Как настроить health check для контейнера?](#16-как-настроить-health-check-для-контейнера)
17. [Как ограничить ресурсы контейнера?](#17-как-ограничить-ресурсы-контейнера)

### Безопасность
18. [Какие основные риски безопасности Docker и как их снизить?](#18-какие-основные-риски-безопасности-docker-и-как-их-снизить)
19. [Что такое rootless Docker и зачем он нужен?](#19-что-такое-rootless-docker-и-зачем-он-нужен)

### Docker Compose и реестры
20. [Как работает Docker Compose? Ключевые возможности.](#20-как-работает-docker-compose-ключевые-возможности)
21. [Как работать с Docker Registry? Как организовать собственный реестр?](#21-как-работать-с-docker-registry-как-организовать-собственный-реестр)
22. [Как оптимизировать размер Docker-образа?](#22-как-оптимизировать-размер-docker-образа)

---

## Архитектура и основы

### 1. Как устроен Docker изнутри? Из чего состоит архитектура?

Docker — клиент-серверная система. При запуске `docker run` происходит гораздо больше, чем кажется.

```
┌──────────────────────────────────────────────────────────┐
│                    Docker CLI (client)                    │
│              docker run / build / pull ...               │
└──────────────────────┬───────────────────────────────────┘
                       │ REST API (unix:///var/run/docker.sock)
                       │ или TCP (2375/2376)
┌──────────────────────▼───────────────────────────────────┐
│                    dockerd (Docker Daemon)                │
│        Управляет образами, сетями, томами, API           │
└──────────────────────┬───────────────────────────────────┘
                       │ gRPC
┌──────────────────────▼───────────────────────────────────┐
│                  containerd                               │
│    Управляет жизненным циклом контейнеров,               │
│    snapshot'ами, образами, namespace'ами                 │
└──────────────────────┬───────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────┐
│               containerd-shim + runc                     │
│   runc — OCI runtime, создаёт namespace'ы, cgroups,     │
│   запускает процесс. После старта отсоединяется.        │
│   shim — держит stdin/stdout, отвечает за сигналы.      │
└──────────────────────────────────────────────────────────┘
```

**Ключевые компоненты:**

- **Docker CLI** — просто клиент, который отправляет команды через REST API.
- **dockerd** — высокоуровневый демон, принимает API-запросы, управляет image'ами, сетями, volumes. Разговаривает с containerd.
- **containerd** — низкоуровневый runtime, управляет контейнерами независимо от Docker. Используется напрямую в Kubernetes.
- **runc** — OCI-совместимый runtime, который непосредственно вызывает Linux kernel API: `clone()` для namespace'ов, `cgroups` для ресурсов. Запускается, создаёт контейнер и завершается.
- **containerd-shim** — остаётся живым между runc и containerd. Позволяет перезапустить dockerd без убийства контейнеров.

**Механизмы ядра Linux, которые делают контейнеры возможными:**
- **Namespaces** — изоляция: pid, net, mnt, uts, ipc, user. Каждый контейнер видит свои процессы, сеть, точки монтирования.
- **cgroups (control groups)** — ограничение ресурсов: CPU, память, I/O, количество процессов.
- **Capabilities** — гранулярные права root. Вместо полного root можно дать только нужные права (например, `CAP_NET_BIND_SERVICE` для прослушивания порта <1024).
- **seccomp** — фильтрация системных вызовов.
- **Union FS (overlay2)** — слоёная файловая система для образов.

---

### 2. В чём разница между образом (image) и контейнером?

**Image (образ)** — неизменяемый (read-only) шаблон, набор слоёв файловой системы + метаданные (команды, переменные окружения, порты). Аналог класса в ООП или ISO-файла для ВМ.

**Container (контейнер)** — запущенный экземпляр образа. Это образ + тонкий записываемый слой поверх (copy-on-write). Аналог объекта в ООП или запущенной ВМ.

```
Image (read-only слои):
  Layer 3: ADD app.py /app/         [только чтение]
  Layer 2: RUN pip install flask    [только чтение]
  Layer 1: FROM python:3.11-slim    [только чтение]

Container = Image + Container Layer:
  Container Layer: /tmp/file, logs  [запись]  ← данные живут здесь
  Layer 3: ADD app.py /app/         [только чтение]
  Layer 2: RUN pip install flask    [только чтение]
  Layer 1: FROM python:3.11-slim    [только чтение]
```

Из одного образа можно запустить сотни контейнеров — они все разделяют read-only слои образа (экономия памяти и диска), у каждого свой тонкий writable layer.

```bash
# Образы
docker images
docker image ls
docker pull nginx:1.25
docker image inspect nginx:1.25
docker image history nginx:1.25   # посмотреть слои

# Контейнеры
docker ps          # запущенные
docker ps -a       # все, включая остановленные
docker run -d --name web nginx:1.25
docker inspect web

# Разница в состояниях контейнера:
# created → running → paused / stopped / exited / dead
```

**Что теряется при удалении контейнера:**
Всё, что было записано в container layer — логи, временные файлы, изменения в ФС — теряется. Для сохранения данных нужны volumes.

---

### 3. Как работает файловая система Docker? Что такое слои и Union FS?

**Union File System** — позволяет монтировать несколько директорий поверх друг друга, представляя их как одну единую ФС. Docker использует **overlay2** (по умолчанию на современных Linux).

**Анатомия overlay2:**
```
/var/lib/docker/overlay2/
├── <layer-id-1>/   # нижний слой (FROM базового образа)
├── <layer-id-2>/   # следующий слой (RUN apt install ...)
├── <layer-id-3>/   # ещё один слой (COPY . /app)
└── <container-id>/
    ├── lower    → ссылки на все read-only слои
    ├── upper/   → writable layer контейнера
    ├── work/    → служебная директория overlay
    └── merged/  → итоговая смонтированная ФС (видит контейнер)
```

**Механизм Copy-on-Write (CoW):**
Когда контейнер изменяет файл из read-only слоя — файл **копируется** в writable layer и там изменяется. Оригинал в слое образа остаётся нетронутым.

```bash
# Посмотреть реальные слои на диске
ls /var/lib/docker/overlay2/

# Посмотреть изменённые файлы в контейнере (diff writable layer)
docker diff <container_name>
# A - добавлен, C - изменён, D - удалён

# Пример:
docker run -it ubuntu bash -c "echo hello > /tmp/test.txt"
docker diff <container_id>
# C /tmp
# A /tmp/test.txt
```

**Почему порядок инструкций в Dockerfile важен:**
Каждая инструкция `RUN`, `COPY`, `ADD` создаёт новый слой. Docker кэширует слои — если слой не изменился, он берётся из кэша:

```dockerfile
# Плохо: COPY всего проекта — изменение любого файла инвалидирует
# кэш и пересобирает всё включая pip install
COPY . /app
RUN pip install -r requirements.txt

# Хорошо: сначала копируем только requirements — pip install
# пересобирается только если изменился requirements.txt
COPY requirements.txt /app/
RUN pip install -r /app/requirements.txt
COPY . /app
```

---

### 4. Чем контейнер отличается от виртуальной машины?

| Характеристика | Контейнер | Виртуальная машина |
|---|---|---|
| Изоляция | Namespace'ы ядра Linux | Гипервизор (аппаратная виртуализация) |
| Ядро ОС | Разделяемое с хостом | Собственное ядро |
| Старт | Секунды / миллисекунды | Минуты |
| Размер | МБ (образ) | ГБ (диск ВМ) |
| Overhead | Минимальный (~1-3%) | Значительный (10-20%) |
| Изоляция безопасности | Слабее (escape возможен) | Сильнее |
| Portability | Образ запускается где угодно с Docker | Зависит от гипервизора |

```
ВМ:                              Контейнеры:
┌─────────────────────┐          ┌──────┐ ┌──────┐ ┌──────┐
│   App A   │  App B  │          │ App A│ │ App B│ │ App C│
├───────────┼─────────┤          ├──────┤ ├──────┤ ├──────┤
│  Guest OS │Guest OS │          │Libs A│ │Libs B│ │Libs C│
├───────────┴─────────┤          └──────┴─┴──────┴─┴──────┘
│       Hypervisor    │          ┌──────────────────────────┐
├─────────────────────┤          │      Docker Engine        │
│      Host OS        │          ├──────────────────────────┤
├─────────────────────┤          │      Host OS Kernel       │
│      Hardware       │          ├──────────────────────────┤
└─────────────────────┘          │        Hardware           │
                                 └──────────────────────────┘
```

**Когда выбирать контейнер:** микросервисы, CI/CD пайплайны, быстрый деплой, горизонтальное масштабирование.

**Когда выбирать ВМ:** жёсткие требования к изоляции безопасности (разные клиенты на одном хосте), Windows/macOS на Linux-хосте, legacy-приложения, требующие отдельного ядра.

**Гибрид (kata containers):** контейнер запускается внутри лёгкой ВМ (microVM), совмещая быстрый старт и сильную изоляцию. Используется в облаках для мультиарендности.

---

## Dockerfile

### 5. Каковы лучшие практики написания Dockerfile?

**1. Использовать минимальный базовый образ:**
```dockerfile
# Плохо: ubuntu — 70 МБ, много лишнего
FROM ubuntu:22.04

# Лучше: alpine — 5 МБ
FROM python:3.11-alpine

# Ещё лучше для Go/статических бинарей: нет ОС вообще
FROM scratch
```

**2. Использовать конкретные теги, не `latest`:**
```dockerfile
# Плохо: не воспроизводимо
FROM nginx:latest

# Хорошо: фиксированная версия
FROM nginx:1.25.3-alpine
```

**3. Объединять RUN команды для уменьшения числа слоёв:**
```dockerfile
# Плохо: 3 слоя, промежуточные файлы apt-cache остаются в layer 1
RUN apt-get update
RUN apt-get install -y curl git
RUN rm -rf /var/lib/apt/lists/*

# Хорошо: 1 слой, кэш удалён в том же слое
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      git && \
    rm -rf /var/lib/apt/lists/*
```

**4. Правильный порядок слоёв (от редко меняемых к часто):**
```dockerfile
FROM node:20-alpine

WORKDIR /app

# Сначала — зависимости (меняются редко, кэш живёт долго)
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Потом — код (меняется часто, инвалидирует кэш)
COPY src/ ./src/
```

**5. Запускать приложение не от root:**
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

**6. Использовать WORKDIR вместо cd:**
```dockerfile
# Плохо
RUN mkdir /app && cd /app && ...

# Хорошо
WORKDIR /app
```

**7. Явно указывать EXPOSE (документация):**
```dockerfile
EXPOSE 8080
```
Это только документация — не открывает порт реально. Реальный проброс — через `-p` при `docker run`.

**8. Использовать `--no-install-recommends` для apt:**
```dockerfile
RUN apt-get install -y --no-install-recommends nginx
```

**9. Не хранить секреты в образе:**
```dockerfile
# ПЛОХО: пароль навсегда остаётся в слое образа!
RUN echo "DB_PASSWORD=secret" >> /app/.env

# Хорошо: передавать через переменные окружения при запуске
# docker run -e DB_PASSWORD=secret myapp
# или через Docker secrets / Kubernetes secrets
```

**10. Добавить HEALTHCHECK:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

---

### 6. Что такое multi-stage build и зачем он нужен?

**Проблема:** для сборки приложения нужны инструменты (компилятор, build-зависимости, тестовые фреймворки), которые не нужны в production-образе. Если включить их — образ разрастается до сотен МБ.

**Multi-stage build** позволяет в одном Dockerfile описать несколько `FROM` стадий. Финальный образ содержит только то, что явно скопировано из предыдущих стадий.

**Пример для Go-приложения:**
```dockerfile
# Стадия 1: сборка (builder)
FROM golang:1.21-alpine AS builder

WORKDIR /build

# Сначала зависимости (кэш)
COPY go.mod go.sum ./
RUN go mod download

# Потом код
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o server ./cmd/server

# Стадия 2: финальный минимальный образ
FROM scratch

# Только сертификаты (для HTTPS-запросов из приложения)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Только собранный бинарь
COPY --from=builder /build/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

**Результат:**
- Builder образ: ~350 МБ (Go toolchain + исходники)
- Финальный образ: ~10 МБ (только бинарь)

**Пример для Node.js:**
```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist ./dist
COPY --from=deps /app/node_modules ./node_modules
COPY package.json .

USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

**Выборочная сборка конкретной стадии (полезно для тестов в CI):**
```bash
# Собрать только до стадии builder (например, для запуска тестов)
docker build --target builder -t myapp:test .
docker run myapp:test go test ./...

# Собрать финальный образ
docker build -t myapp:latest .
```

---

### 7. В чём разница между CMD и ENTRYPOINT?

Оба определяют что запускать при старте контейнера. Разница — в том, насколько это можно переопределить.

**CMD** — команда по умолчанию. Легко переопределяется при `docker run`:
```dockerfile
FROM ubuntu
CMD ["echo", "Hello, World!"]
```
```bash
docker run myimage              # → echo "Hello, World!"
docker run myimage ls -la       # → ls -la (CMD полностью заменён)
```

**ENTRYPOINT** — точка входа, которую сложнее переопределить (только через `--entrypoint`). Аргументы `docker run` добавляются к ENTRYPOINT:
```dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
```
```bash
docker run myimage              # → echo (пустой вывод)
docker run myimage "Hello!"     # → echo "Hello!" → выведет Hello!
docker run --entrypoint ls myimage -la  # переопределить ENTRYPOINT
```

**Совместное использование (самый распространённый паттерн):**
```dockerfile
ENTRYPOINT ["nginx"]          # всегда запускаем nginx
CMD ["-g", "daemon off;"]     # аргументы по умолчанию, можно переопределить
```
```bash
docker run nginx-image                   # → nginx -g "daemon off;"
docker run nginx-image -t                # → nginx -t (проверить конфиг)
```

**Shell vs Exec форма — критически важное отличие:**
```dockerfile
# Shell форма — запускается через /bin/sh -c "..."
# PID 1 будет /bin/sh, а не ваше приложение!
# SIGTERM не дойдёт до приложения → нет graceful shutdown
CMD python app.py
ENTRYPOINT python app.py

# Exec форма — запускается напрямую, ваше приложение = PID 1
# SIGTERM доходит до приложения → graceful shutdown работает
CMD ["python", "app.py"]
ENTRYPOINT ["python", "app.py"]
```

> **На собеседовании:** exec-форма обязательна для production. Shell-форма — частая причина проблем с graceful shutdown в Kubernetes.

---

### 8. В чём разница между COPY и ADD?

**COPY** — просто копирует файлы/директории с хоста в образ. Ничего лишнего.

**ADD** — делает то же самое + дополнительное поведение:
- Автоматически распаковывает `.tar`, `.tar.gz`, `.tar.bz2`, `.tar.xz` архивы
- Умеет скачивать файлы по URL (но это плохая практика)

```dockerfile
# ADD — автоматически распакует архив
ADD app.tar.gz /app/

# COPY — просто скопирует архив как есть
COPY app.tar.gz /app/

# ADD по URL (не рекомендуется!)
ADD https://example.com/file.tar.gz /tmp/
# Лучше использовать RUN curl:
RUN curl -fsSL https://example.com/file.tar.gz | tar xz -C /tmp/
```

**Правило:** всегда использовать `COPY`, если не нужна распаковка архива. `ADD` с URL — антипаттерн (файл не кэшируется между сборками, нельзя верифицировать).

---

### 9. Что такое .dockerignore и почему он важен?

`.dockerignore` — файл, аналогичный `.gitignore`, перечисляющий файлы и директории, которые **не должны** попадать в build context при `docker build`.

**Почему это важно:**

При выполнении `docker build .` Docker CLI отправляет весь **build context** (директорию `.`) демону dockerd. Если в проекте есть `node_modules` (500 МБ), `.git`, логи — они все передаются перед началом сборки. Это медленно и увеличивает образ если использовать `COPY . .`.

**Пример `.dockerignore`:**
```
# Git
.git
.gitignore

# Зависимости (будут установлены внутри образа)
node_modules
vendor/

# Тесты и документация
*.test.js
tests/
docs/
*.md

# Логи и временные файлы
*.log
tmp/
.cache/

# Среда разработки
.env
.env.local
*.env.*
.idea/
.vscode/
*.swp

# Docker-файлы (не нужны внутри образа)
Dockerfile
docker-compose*.yml
.dockerignore

# CI/CD
.github/
.gitlab-ci.yml
```

**Практическая демонстрация влияния:**
```bash
# Посмотреть что войдёт в build context
docker build --no-cache -t test . 2>&1 | head -3
# Sending build context to Docker daemon  1.234GB  ← без .dockerignore
# Sending build context to Docker daemon  45.2MB   ← с .dockerignore
```

---

## Сеть

### 10. Какие типы сетей есть в Docker и чем они отличаются?

```bash
docker network ls
# NETWORK ID   NAME      DRIVER    SCOPE
# abc123       bridge    bridge    local
# def456       host      host      local
# ghi789       none      null      local
```

**bridge (по умолчанию)**
Виртуальный мост `docker0` на хосте. Контейнеры получают IP из подсети `172.17.0.0/16`. Общаются между собой по IP, с внешним миром — через NAT (iptables MASQUERADE).

```bash
# Пользовательский bridge (рекомендуется вместо дефолтного)
docker network create --driver bridge --subnet 172.20.0.0/16 mynet

# Ключевое преимущество пользовательского bridge:
# контейнеры могут общаться по имени (DNS резолюция по имени контейнера)
docker run -d --name db --network mynet postgres
docker run -d --name app --network mynet myapp
# app может обращаться к db по имени: psql -h db
```

**host**
Контейнер использует сетевой стек хоста напрямую. Нет изоляции, нет NAT, максимальная производительность.
```bash
docker run --network host nginx
# nginx слушает на портах хоста напрямую, без -p
```
Используется для: высоконагруженных сетевых приложений, мониторинга сети, когда нужен прямой доступ к сетевым интерфейсам хоста.

**none**
Полное отсутствие сети. Только loopback (`lo`). Для изолированных задач (batch-обработка файлов).
```bash
docker run --network none myapp
```

**overlay**
Многохостовая сеть для Docker Swarm и Kubernetes. Позволяет контейнерам на разных хостах общаться как будто они в одной сети. Использует VXLAN для инкапсуляции.
```bash
docker network create --driver overlay --attachable myoverlay
```

**macvlan**
Контейнер получает собственный MAC-адрес и выглядит как физическое устройство в сети. Используется когда нужен прямой доступ к физической сети (без NAT).

---

### 11. Как контейнеры общаются между собой и с внешним миром?

**Контейнер → Интернет:**
Исходящий трафик проходит через iptables MASQUERADE (NAT). Docker автоматически добавляет правила:
```bash
iptables -t nat -L POSTROUTING -n
# MASQUERADE  all  172.17.0.0/16  0.0.0.0/0  # трафик из docker-подсети → NAT
```

**Внешний мир → Контейнер (проброс портов):**
```bash
docker run -p 8080:80 nginx
# Docker добавляет правило: трафик на порт 8080 хоста → 172.17.0.2:80 контейнера
```
```bash
iptables -t nat -L DOCKER -n
# DNAT  tcp  0.0.0.0/0  0.0.0.0/0  tcp dpt:8080 to:172.17.0.2:80
```

**Контейнер → Хост:**
С помощью специального DNS-имени `host.docker.internal` (Docker Desktop) или через IP шлюза:
```bash
# Узнать IP шлюза (адрес хоста в сети контейнера)
docker inspect <container> | grep Gateway
# "Gateway": "172.17.0.1"

# Или через host-gateway (Docker 20.10+)
docker run --add-host=host.docker.internal:host-gateway myapp
```

**Контейнер → Контейнер (пользовательская bridge-сеть):**
```bash
docker network create appnet
docker run -d --name postgres --network appnet postgres:15
docker run -d --name app --network appnet -e DB_HOST=postgres myapp
# app резолвит 'postgres' → 172.20.0.2 через встроенный DNS Docker (127.0.0.11)
```

**Диагностика сетевых проблем:**
```bash
# Посмотреть сети контейнера
docker inspect <container> --format '{{json .NetworkSettings.Networks}}' | jq

# Зайти в контейнер и пинговать
docker exec -it myapp sh -c "ping -c 3 db"

# Посмотреть правила iptables которые создал Docker
iptables -t nat -L -n -v
iptables -L DOCKER-USER -n -v

# Сниффинг трафика контейнера
nsenter -t $(docker inspect -f '{{.State.Pid}}' myapp) -n tcpdump -i eth0 -n
```

---

## Хранилище данных

### 12. В чём разница между volume, bind mount и tmpfs?

| Характеристика | Volume | Bind Mount | tmpfs |
|---|---|---|---|
| Расположение | `/var/lib/docker/volumes/` | Любой путь на хосте | Только в памяти |
| Управление | Docker | Пользователь | Docker |
| Портабельность | Да | Нет (путь зависит от хоста) | Нет |
| Производительность | Хорошая | Хорошая (на Linux) | Максимальная |
| Данные после rm | Сохраняются | Сохраняются | Удаляются |
| Backup / restore | Через Docker API | cp файлов | Нет смысла |

**Volume:**
```bash
# Создать именованный volume
docker volume create pgdata

# Использование в run
docker run -d \
  -v pgdata:/var/lib/postgresql/data \
  postgres:15

# Или через --mount (более явный синтаксис)
docker run -d \
  --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
  postgres:15

# Инспект
docker volume inspect pgdata
docker volume ls
```

**Bind Mount:**
```bash
# Монтировать директорию с хоста (абсолютный путь обязателен)
docker run -d \
  -v /home/user/project:/app \
  -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  myapp

# Удобно для разработки: изменения в коде сразу видны в контейнере
# ro — только чтение
```

**tmpfs:**
```bash
# Хранить данные только в RAM (секреты, временные файлы)
docker run -d \
  --mount type=tmpfs,destination=/tmp,tmpfs-size=100m \
  myapp
```

**Когда что использовать:**
- **Volume** — базы данных, persistent application data в production
- **Bind Mount** — конфиги, разработка (hot reload кода), sharing логов с хостом
- **tmpfs** — секреты в памяти, высокочастотный I/O временных данных

---

### 13. Как правильно управлять данными в Docker?

**Backup volume:**
```bash
# Создать архив данных из volume
docker run --rm \
  -v pgdata:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/pgdata-backup.tar.gz -C /source .

# Восстановить
docker run --rm \
  -v pgdata:/target \
  -v $(pwd):/backup \
  alpine tar xzf /backup/pgdata-backup.tar.gz -C /target
```

**Очистка неиспользуемых volume'ов:**
```bash
# Показать dangling volumes (не привязаны ни к одному контейнеру)
docker volume ls -f dangling=true

# Удалить dangling volumes
docker volume prune

# Полная очистка (осторожно!)
docker system prune --volumes
```

**Передача данных между контейнерами (shared volume):**
```bash
docker volume create shared_data

docker run -d --name writer \
  -v shared_data:/data \
  alpine sh -c "while true; do date >> /data/log.txt; sleep 1; done"

docker run -d --name reader \
  -v shared_data:/data:ro \
  alpine sh -c "tail -f /data/log.txt"
```

**Важный нюанс — инициализация volume при первом запуске:**
Если volume пустой и монтируется в директорию образа с файлами — Docker скопирует содержимое этой директории в volume. При повторном запуске уже нет — volume непустой, данные из образа игнорируются. Это частая причина путаницы с конфигами.

---

## Запуск и отладка

### 14. Как дебажить работающий контейнер?

**`docker logs` — первый шаг:**
```bash
# Последние 100 строк лога
docker logs --tail 100 myapp

# Follow (как tail -f)
docker logs -f myapp

# С временными метками
docker logs -t myapp

# Логи за последний час
docker logs --since 1h myapp

# Логи между временными метками
docker logs --since "2024-01-15T10:00:00" --until "2024-01-15T11:00:00" myapp
```

**`docker exec` — зайти внутрь:**
```bash
# Интерактивный shell (если есть bash/sh)
docker exec -it myapp bash
docker exec -it myapp sh   # для alpine и минимальных образов

# Выполнить команду без входа
docker exec myapp cat /etc/hosts
docker exec myapp env | grep DB_

# Запустить от другого пользователя
docker exec -it -u root myapp bash
```

**Если в образе нет shell (scratch/distroless):**
```bash
# Использовать nsenter — войти в namespace'ы контейнера с хоста
PID=$(docker inspect -f '{{.State.Pid}}' myapp)
nsenter -t $PID -m -u -i -n -p -- sh

# Или ephemeral debug container (Docker 20.10+)
docker debug myapp   # экспериментально

# Kubernetes-подход: временный debug sidecar
# docker run -d --pid=container:myapp --network=container:myapp \
#   --volumes-from myapp nicolaka/netshoot
```

**Инспектирование состояния:**
```bash
# Всё о контейнере (JSON)
docker inspect myapp

# Конкретные поля
docker inspect -f '{{.State.Status}}' myapp
docker inspect -f '{{.NetworkSettings.IPAddress}}' myapp
docker inspect -f '{{json .HostConfig.PortBindings}}' myapp | jq

# Статистика ресурсов в реальном времени
docker stats myapp
docker stats --no-stream myapp  # один снимок

# Изменённые файлы относительно базового образа
docker diff myapp

# Запущенные процессы
docker top myapp
```

**Копирование файлов:**
```bash
# Из контейнера на хост
docker cp myapp:/var/log/app/app.log ./app.log

# С хоста в контейнер
docker cp ./config.yaml myapp:/app/config.yaml
```

**Запуск остановленного контейнера с переопределённой точкой входа:**
```bash
# Контейнер крашится при старте — запустить с shell вместо entrypoint
docker run -it --entrypoint sh myapp:broken
# Или
docker run -it --entrypoint /bin/bash myapp:broken
```

---

### 15. Как работают сигналы и graceful shutdown в контейнерах?

**Проблема:** при `docker stop` Docker отправляет `SIGTERM` процессу с PID 1. Если приложение не обрабатывает SIGTERM — через `--stop-timeout` (по умолчанию 10 сек) Docker отправляет `SIGKILL`, убивая контейнер принудительно. Незавершённые запросы теряются.

**Частая ошибка — shell форма в CMD/ENTRYPOINT:**
```dockerfile
# Плохо: /bin/sh -c "node app.js" получает SIGTERM
# но /bin/sh по умолчанию не пересылает сигналы дочерним процессам
ENTRYPOINT node app.js

# Хорошо: node app.js = PID 1, напрямую получает SIGTERM
ENTRYPOINT ["node", "app.js"]
```

**Вторая частая ошибка — shell скрипт как entrypoint без `exec`:**
```bash
#!/bin/sh
# Плохо: скрипт = PID 1, приложение = дочерний процесс
# SIGTERM дойдёт до скрипта, но не до приложения
/app/server --config /etc/app/config.yaml

# Хорошо: exec заменяет текущий процесс (shell) на приложение
# теперь приложение = PID 1
exec /app/server --config /etc/app/config.yaml
```

**Приложение должно обрабатывать SIGTERM:**
```python
# Python пример
import signal
import sys

def graceful_shutdown(signum, frame):
    print("Received SIGTERM, shutting down gracefully...")
    # Завершить текущие запросы, закрыть соединения с БД
    server.shutdown()
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
```

**Настройка таймаута graceful shutdown:**
```bash
# Дать контейнеру 30 секунд на завершение
docker stop --time 30 myapp

# В Docker Compose
services:
  app:
    stop_grace_period: 30s
```

**`tini` — правильный init для контейнеров:**
```dockerfile
# tini решает проблему зомби-процессов и корректно пересылает сигналы
FROM node:20-alpine
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "app.js"]
```

> Начиная с Docker 1.13 можно использовать `docker run --init` — Docker автоматически добавит tini как PID 1.

---

### 16. Как настроить health check для контейнера?

**В Dockerfile:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Параметры:
# --interval     — интервал между проверками (default: 30s)
# --timeout      — таймаут одной проверки (default: 30s)
# --start-period — grace period при старте (не считать неудачи) (default: 0s)
# --retries      — N неудач подряд → unhealthy (default: 3)
```

**При `docker run`:**
```bash
docker run -d \
  --health-cmd="curl -f http://localhost:8080/health || exit 1" \
  --health-interval=30s \
  --health-timeout=5s \
  --health-start-period=10s \
  --health-retries=3 \
  myapp
```

**Отключить healthcheck (если задан в образе):**
```bash
docker run --no-healthcheck myapp
```

**Посмотреть состояние:**
```bash
docker inspect --format='{{.State.Health.Status}}' myapp
# starting → healthy / unhealthy

docker inspect --format='{{json .State.Health}}' myapp | jq
# Покажет историю последних проверок с output
```

**Правила хорошего health endpoint:**
```python
# Должен проверять реальную работоспособность, а не просто возвращать 200
@app.route('/health')
def health():
    checks = {
        'database': check_db(),
        'cache': check_redis(),
    }
    status = 'ok' if all(v == 'ok' for v in checks.values()) else 'degraded'
    code = 200 if status == 'ok' else 503
    return jsonify({'status': status, 'checks': checks}), code
```

**В Docker Compose (зависимости по healthcheck):**
```yaml
services:
  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  app:
    image: myapp
    depends_on:
      db:
        condition: service_healthy  # ждать пока db не станет healthy
```

---

### 17. Как ограничить ресурсы контейнера?

**CPU:**
```bash
# --cpus — дробное число ядер (рекомендуется)
docker run --cpus="1.5" myapp           # максимум 1.5 ядра

# --cpu-shares — относительный вес (default 1024)
docker run --cpu-shares=512 myapp       # вдвое меньший приоритет

# --cpuset-cpus — привязать к конкретным ядрам
docker run --cpuset-cpus="0,2" myapp    # только ядра 0 и 2
```

**Память:**
```bash
# --memory — жёсткий лимит RAM
docker run --memory="512m" myapp

# --memory-swap — лимит RAM + swap (должен быть >= --memory)
docker run --memory="512m" --memory-swap="1g" myapp    # 512 МБ swap
docker run --memory="512m" --memory-swap="512m" myapp  # swap отключён
docker run --memory="512m" --memory-swap="-1" myapp    # swap не ограничен

# --memory-reservation — "мягкий" лимит (рекомендуемое потребление)
docker run --memory="1g" --memory-reservation="512m" myapp
```

**I/O:**
```bash
# Ограничить IOPS
docker run --device-read-iops=/dev/sda:1000 \
           --device-write-iops=/dev/sda:500 myapp

# Ограничить пропускную способность
docker run --device-read-bps=/dev/sda:50mb \
           --device-write-bps=/dev/sda:25mb myapp
```

**Посмотреть текущее потребление:**
```bash
# Реальное время
docker stats

# Детали из cgroups
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.usage_in_bytes
```

**В Docker Compose:**
```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '1.5'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
```

> **Важно:** `deploy.resources` в Compose работает только в Swarm mode. Для обычного Compose используй `mem_limit` и `cpus` на уровне сервиса (Compose v2):
```yaml
services:
  app:
    mem_limit: 512m
    cpus: 1.5
```

---

## Безопасность

### 18. Какие основные риски безопасности Docker и как их снизить?

**1. Запуск контейнеров от root**

По умолчанию процессы внутри контейнера запускаются от `root` (UID 0). Если есть уязвимость, позволяющая выбраться из контейнера — атакующий получает root на хосте.

```dockerfile
# В Dockerfile создавать непривилегированного пользователя
RUN addgroup -S app && adduser -S app -G app
USER app
```

```bash
# Проверить от кого запущен процесс
docker exec myapp whoami
docker exec myapp id
```

**2. Privileged mode — избегать**
```bash
# НИКОГДА в production:
docker run --privileged myapp
# Контейнер получает доступ ко всем устройствам хоста и всем capabilities
```

**3. Capabilities — давать минимально необходимые**
```bash
# Убрать все capabilities, добавить только нужные
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
# NET_BIND_SERVICE — слушать порты <1024
# NET_ADMIN — управление сетью
# SYS_PTRACE — нужен для strace, gdb
```

**4. Read-only filesystem**
```bash
# Сделать корневую ФС контейнера read-only
docker run --read-only \
  --tmpfs /tmp:size=100m \    # разрешить запись во временные директории
  --tmpfs /var/run \
  myapp
```

**5. Не доверять образам из публичных реестров**
```bash
# Использовать Docker Content Trust (подпись образов)
export DOCKER_CONTENT_TRUST=1
docker pull nginx:1.25   # только подписанные образы

# Сканировать образы на уязвимости
docker scout cves nginx:1.25       # встроенный инструмент Docker
trivy image nginx:1.25             # популярный open source сканер
grype nginx:1.25
```

**6. Не монтировать Docker socket в контейнер**
```bash
# ОПАСНО: полный контроль над Docker-хостом
docker run -v /var/run/docker.sock:/var/run/docker.sock myapp
# Эквивалентно root-доступу к хосту
```

**7. Ограничить системные вызовы через seccomp**
```bash
# Docker по умолчанию применяет seccomp профиль, блокирующий ~40 syscall'ов
# Можно задать собственный:
docker run --security-opt seccomp=/path/to/profile.json myapp

# Или отключить (не рекомендуется):
docker run --security-opt seccomp=unconfined myapp
```

**8. Сканирование на секреты в образах:**
```bash
# Проверить что секреты не попали в слои образа
docker history --no-trunc myapp | grep -i "secret\|password\|token"
trufflehog docker --image myapp
```

---

### 19. Что такое rootless Docker и зачем он нужен?

**Стандартный Docker:**
- `dockerd` запускается от `root`
- Создание namespace'ов, cgroups, mount — всё через root
- Риск: уязвимость в Docker daemon = компрометация хоста

**Rootless Docker:**
Docker daemon и контейнеры запускаются **полностью от обычного пользователя**, без root на хосте.

```bash
# Установка rootless Docker
dockerd-rootless-setuptool.sh install

# Запуск
systemctl --user start docker
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock

# Проверка
docker info | grep 'rootless'
```

**Ограничения rootless Docker:**
- Нельзя прокидывать порты < 1024 (без `sysctl net.ipv4.ip_unprivileged_port_start=80`)
- `--network host` не поддерживается
- Некоторые storage drivers недоступны
- Performance storage чуть хуже (fuse-overlayfs вместо native overlay2)

**Альтернатива — Podman:**
```bash
# Podman — rootless by default, совместим с Docker CLI
podman run -d nginx

# Нет демона! Каждый контейнер — прямой дочерний процесс пользователя
# Совместим с Docker CLI (можно сделать alias docker=podman)
```

---

## Docker Compose и реестры

### 20. Как работает Docker Compose? Ключевые возможности.

Docker Compose — инструмент для определения и запуска многоконтейнерных приложений. Вся конфигурация описывается в `docker-compose.yml` (или `compose.yml`).

**Базовый пример (production-ready):**
```yaml
# compose.yml
name: myapp

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runner             # multi-stage target
    image: myapp:${APP_VERSION:-latest}
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
      - DB_NAME=${DB_NAME}
      - DB_PASSWORD=${DB_PASSWORD}
    env_file:
      - .env.production
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - app_logs:/app/logs
    networks:
      - frontend
      - backend
    deploy:
      resources:
        limits:
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: postgres:15-alpine
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --save 60 1 --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s

  nginx:
    image: nginx:1.25-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      - app
    networks:
      - frontend

volumes:
  pgdata:
  redis_data:
  app_logs:

networks:
  frontend:
  backend:
    internal: true   # нет доступа в интернет из backend-сети
```

**Основные команды:**
```bash
# Запустить в фоне
docker compose up -d

# Пересобрать образы перед запуском
docker compose up -d --build

# Масштабировать сервис
docker compose up -d --scale app=3

# Посмотреть статус
docker compose ps
docker compose top

# Логи
docker compose logs -f app
docker compose logs --tail 50

# Остановить (контейнеры остаются)
docker compose stop

# Остановить и удалить контейнеры (volumes сохраняются)
docker compose down

# Удалить всё включая volumes (ОСТОРОЖНО!)
docker compose down -v

# Выполнить команду в сервисе
docker compose exec app sh
docker compose run --rm app python manage.py migrate
```

**Override файлы (разные конфиги для окружений):**
```bash
# compose.yml              — базовая конфигурация
# compose.override.yml     — автоматически применяется поверх (dev defaults)
# compose.prod.yml         — production overrides

# Production запуск:
docker compose -f compose.yml -f compose.prod.yml up -d
```

---

### 21. Как работать с Docker Registry? Как организовать собственный реестр?

**Публичный DockerHub:**
```bash
# Аутентификация
docker login
docker login -u myuser -p mypassword

# Push
docker tag myapp:latest myuser/myapp:1.0.0
docker push myuser/myapp:1.0.0

# Pull
docker pull myuser/myapp:1.0.0
```

**Приватный реестр (self-hosted) — Distribution (registry:2):**
```bash
# Запустить приватный реестр
docker run -d \
  -p 5000:5000 \
  --name registry \
  -v /data/registry:/var/lib/registry \
  registry:2

# Push в локальный реестр
docker tag myapp:latest localhost:5000/myapp:1.0.0
docker push localhost:5000/myapp:1.0.0

# Pull
docker pull localhost:5000/myapp:1.0.0
```

**С авторизацией и TLS (production-ready):**
```bash
# Генерация htpasswd
docker run --rm httpd:2.4-alpine htpasswd -Bbn user password > /data/registry/htpasswd

docker run -d \
  -p 443:5000 \
  -v /data/registry:/var/lib/registry \
  -v /etc/registry/certs:/certs \
  -v /data/registry/htpasswd:/auth/htpasswd \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```

**Популярные self-hosted решения:**
- **Harbor** — enterprise-grade, поддерживает сканирование уязвимостей (Trivy встроен), RBAC, репликацию
- **Nexus Repository** — универсальный (Docker, Maven, npm, PyPI)
- **GitLab Container Registry** — встроен в GitLab
- **AWS ECR / GCR / ACR** — managed cloud реестры

**Управление тегами и очистка:**
```bash
# Посмотреть все теги образа в реестре
curl -s http://localhost:5000/v2/myapp/tags/list | jq

# Удалить старые образы (через garbage collect)
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml

# Автоматическая очистка в Harbor / ECR через политики хранения
```

---

### 22. Как оптимизировать размер Docker-образа?

**1. Выбор базового образа:**
```dockerfile
# python:3.11         — 900 МБ
# python:3.11-slim    — 130 МБ (без лишних пакетов)
# python:3.11-alpine  — 50 МБ (musl libc, минимальный)
# gcr.io/distroless   — только runtime, нет shell (безопаснее)
```

**2. Multi-stage builds (уже разобрано в вопросе 6)**

**3. Удаление кэша пакетных менеджеров в том же слое:**
```dockerfile
# apt
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# apk (alpine)
RUN apk add --no-cache curl

# pip
RUN pip install --no-cache-dir -r requirements.txt
```

**4. Использовать `.dockerignore`** (вопрос 9)

**5. Squash слоёв (осторожно, теряем кэш):**
```bash
docker build --squash -t myapp .   # экспериментальная функция
# Объединяет все слои в один (помогает если удаляете файлы в разных RUN)
```

**6. Анализ размера образа:**
```bash
# История слоёв с размерами
docker image history myapp

# Детальный анализ (установить dive)
dive myapp
# Показывает какой слой сколько весит и что в нём находится
```

**7. Не устанавливать dev-зависимости:**
```dockerfile
# Python
RUN pip install --no-cache-dir -r requirements.txt  # только prod зависимости

# Node.js
RUN npm ci --only=production
```

**8. Статически скомпилированный бинарь + scratch:**
```dockerfile
# Go — итоговый образ 5-10 МБ
FROM golang:1.21 AS builder
RUN CGO_ENABLED=0 go build -ldflags="-w -s" -o app .

FROM scratch
COPY --from=builder /app /app
ENTRYPOINT ["/app"]
```

**Практический результат оптимизации (пример Python-приложения):**

| Подход | Размер |
|---|---|
| `FROM python:3.11` + всё подряд | ~1.2 ГБ |
| `FROM python:3.11-slim` + .dockerignore | ~200 МБ |
| `FROM python:3.11-slim` + multi-stage + оптимизация | ~120 МБ |
| `FROM python:3.11-alpine` + оптимизация | ~80 МБ |
| `FROM distroless/python3` + multi-stage | ~60 МБ |
