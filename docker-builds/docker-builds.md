# Сборка приложений в Docker — Лучшие практики

Краткий справочник: как правильно упаковывать приложения разных стеков в Docker.
Общий принцип для всех — **multi-stage build**: собираем в одном образе, запускаем в минимальном.

---

## Содержание

1. [Go](#1-go)
2. [Node.js](#2-nodejs)
3. [Python](#3-python)
4. [Java (Maven / Gradle)](#4-java-maven--gradle)
5. [.NET](#5-net)
6. [PHP (Laravel / Symfony)](#6-php-laravel--symfony)
7. [Frontend (React / Vue / Angular)](#7-frontend-react--vue--angular)
8. [Rust](#8-rust)
9. [Универсальные правила и антипаттерны](#9-универсальные-правила-и-антипаттерны)

---

## 1. Go

Go компилируется в статический бинарь — идеальный кандидат для образа `scratch` (буквально ноль ОС).

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.22-alpine AS builder

WORKDIR /build

# Сначала зависимости — кэш слоя живёт долго
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# CGO_ENABLED=0  — статическая линковка (нет зависимости от libc)
# -ldflags "-w -s" — убрать debug symbols (~30% меньше)
# -trimpath      — убрать пути к исходникам из бинаря
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -trimpath -o /app/server ./cmd/server

# ──────────────────────────────────────────────
FROM scratch

# Нужны только если приложение делает HTTPS-запросы наружу
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Timezone data (если нужны time.LoadLocation)
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

COPY --from=builder /app/server /server

# Непривилегированный пользователь в scratch: задаём UID напрямую
USER 65534:65534

EXPOSE 8080
ENTRYPOINT ["/server"]
```

**Результат:** образ ~5–15 МБ. Нет shell, нет пакетного менеджера — минимальная attack surface.

> Если нужен shell для healthcheck — используй `FROM gcr.io/distroless/static-debian12` вместо `scratch`.

---

## 2. Node.js

Ключевые проблемы: `node_modules` (~сотни МБ) и секреты в environment при сборке.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS deps

WORKDIR /app

# Только файлы зависимостей — кэш не инвалидируется при изменении кода
COPY package.json package-lock.json ./

# npm ci: строго по lock-файлу, чище чем npm install
RUN npm ci --only=production

# ──────────────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci  # все зависимости (включая devDependencies для сборки)

COPY . .
RUN npm run build

# ──────────────────────────────────────────────
FROM node:20-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production

# Только prod зависимости из deps стадии
COPY --from=deps    /app/node_modules ./node_modules
# Только артефакт сборки
COPY --from=builder /app/dist         ./dist
COPY package.json ./

# Никогда не запускать от root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

**`.dockerignore`** — обязателен:
```
node_modules
npm-debug.log
.git
.env*
dist
coverage
*.test.ts
```

**Для TypeScript / Монорепозитория (pnpm):**
```dockerfile
FROM node:20-alpine AS builder
RUN corepack enable && corepack prepare pnpm@latest --activate
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml ./
COPY packages/api/package.json ./packages/api/
RUN pnpm install --frozen-lockfile
COPY . .
RUN pnpm --filter api build
```

---

## 3. Python

Главные задачи: маленький образ, детерминированные зависимости, нет `.pyc` мусора в образе.

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS builder

WORKDIR /app

# Системные зависимости для компиляции (psycopg2, Pillow, etc.)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev gcc && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

# --user: ставим в ~/.local (легко скопировать в финальный образ)
# --no-cache-dir: не хранить кэш pip
RUN pip install --user --no-cache-dir -r requirements.txt

# ──────────────────────────────────────────────
FROM python:3.12-slim AS runner

WORKDIR /app

# Только runtime зависимости (не gcc и libpq-dev)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 && \
    rm -rf /var/lib/apt/lists/*

# Копируем установленные пакеты из builder
COPY --from=builder /root/.local /root/.local

COPY . .

# Убрать .pyc файлы и не писать в __pycache__ в runtime
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/root/.local/bin:$PATH"

RUN useradd --create-home appuser
USER appuser

EXPOSE 8000
CMD ["gunicorn", "app.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

**Для FastAPI / async приложений:**
```dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

**Poetry вместо pip:**
```dockerfile
FROM python:3.12-slim AS builder

RUN pip install poetry==1.8.0
ENV POETRY_NO_INTERACTION=1 \
    POETRY_VENV_IN_PROJECT=1

WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN poetry install --only=main --no-root

COPY . .

# ──────────────────────────────────────────────
FROM python:3.12-slim AS runner
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app .
ENV PATH="/app/.venv/bin:$PATH"
CMD ["gunicorn", "app.wsgi:application"]
```

---

## 4. Java (Maven / Gradle)

JVM-образы большие по природе. Цель: убрать JDK из финального образа, оставить только JRE. С Java 11+ можно сделать кастомный минимальный JRE через `jlink`.

**Maven:**
```dockerfile
# syntax=docker/dockerfile:1
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /build

# Сначала только pom.xml — скачать зависимости в кэш
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Потом исходники
COPY src ./src
RUN mvn package -DskipTests -B

# ──────────────────────────────────────────────
# jlink: создать минимальный JRE только с нужными модулями
FROM eclipse-temurin:21-jdk AS jre-builder

RUN $JAVA_HOME/bin/jlink \
    --module-path "$JAVA_HOME/jmods" \
    --add-modules java.base,java.logging,java.sql,java.naming,java.desktop,java.management,java.security.jgss,java.instrument \
    --strip-debug \
    --no-man-pages \
    --no-header-files \
    --compress=2 \
    --output /custom-jre

# ──────────────────────────────────────────────
FROM debian:12-slim AS runner

COPY --from=jre-builder /custom-jre /opt/jre

ENV JAVA_HOME=/opt/jre \
    PATH="/opt/jre/bin:$PATH"

WORKDIR /app
COPY --from=builder /build/target/app.jar ./app.jar

RUN useradd -r -u 1001 appuser
USER appuser

EXPOSE 8080

# JVM-флаги для контейнера:
# UseContainerSupport — автоопределение лимитов cgroups (Java 10+)
# MaxRAMPercentage    — использовать 75% от limit (не хардкодить -Xmx!)
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-XX:+UseG1GC", \
    "-jar", "app.jar"]
```

**Gradle:**
```dockerfile
FROM gradle:8.5-jdk21 AS builder
WORKDIR /build
COPY build.gradle settings.gradle ./
COPY gradle ./gradle
# Скачать зависимости отдельно (кэш)
RUN gradle dependencies --no-daemon
COPY src ./src
RUN gradle bootJar --no-daemon
```

**Spring Boot + Layered JARs (лучший кэш слоёв):**
```dockerfile
FROM eclipse-temurin:21-jre AS runner
WORKDIR /app
COPY --from=builder /build/target/app.jar app.jar

# Spring Boot Layertools: разбить JAR на слои
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre
WORKDIR /app
# Порядок от редко меняемых к часто — максимальный cache reuse
COPY --from=runner /app/dependencies/           ./
COPY --from=runner /app/spring-boot-loader/     ./
COPY --from=runner /app/snapshot-dependencies/  ./
COPY --from=runner /app/application/            ./

ENTRYPOINT ["java", "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", "org.springframework.boot.loader.JarLauncher"]
```

**Результат:** builder ~800 МБ → runner с custom JRE ~80–120 МБ.

---

## 5. .NET

```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS builder

WORKDIR /src

# Только .csproj для restore (кэш NuGet)
COPY src/MyApp/*.csproj ./src/MyApp/
RUN dotnet restore src/MyApp/MyApp.csproj

COPY . .

# publish: self-contained=false (используем runtime образ)
# PublishSingleFile — один бинарь
RUN dotnet publish src/MyApp/MyApp.csproj \
    -c Release \
    -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# ──────────────────────────────────────────────
# Используем ASP.NET runtime (не SDK!)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runner

WORKDIR /app

# Не запускать от root
RUN useradd -u 1000 appuser
USER appuser

COPY --from=builder /app/publish .

EXPOSE 8080

# Переменная окружения вместо хардкода порта
ENV ASPNETCORE_URLS=http://+:8080 \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_GCConserveMemory=9

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Self-contained (нет зависимости от dotnet runtime на хосте):**
```dockerfile
RUN dotnet publish -c Release -o /app/publish \
    --runtime linux-x64 \
    --self-contained true \
    /p:PublishSingleFile=true \
    /p:PublishTrimmed=true

# Финальный образ — просто debian:slim
FROM debian:12-slim
COPY --from=builder /app/publish/MyApp /app/MyApp
ENTRYPOINT ["/app/MyApp"]
```

---

## 6. PHP (Laravel / Symfony)

```dockerfile
# syntax=docker/dockerfile:1

# Stage 1: Composer зависимости
FROM composer:2.7 AS composer

WORKDIR /app
COPY composer.json composer.lock ./

# --no-dev: без dev зависимостей
# --no-scripts: не запускать post-install скрипты (нет БД в сборке)
# --no-autoloader: сгенерируем позже с оптимизацией
RUN composer install \
    --no-dev \
    --no-scripts \
    --no-progress \
    --prefer-dist

COPY . .
RUN composer dump-autoload --optimize --classmap-authoritative

# Stage 2: Node для assets (Vite / Mix)
FROM node:20-alpine AS assets

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: финальный образ
FROM php:8.3-fpm-alpine AS runner

# PHP расширения (через docker-php-ext-install)
RUN apk add --no-cache libpq-dev libzip-dev && \
    docker-php-ext-install pdo pdo_pgsql zip opcache && \
    docker-php-ext-enable opcache

# OPcache для production
RUN { \
    echo 'opcache.enable=1'; \
    echo 'opcache.revalidate_freq=0'; \
    echo 'opcache.validate_timestamps=0'; \
    echo 'opcache.max_accelerated_files=20000'; \
    echo 'opcache.memory_consumption=256'; \
    echo 'opcache.interned_strings_buffer=16'; \
    echo 'opcache.fast_shutdown=1'; \
} > /usr/local/etc/php/conf.d/opcache.ini

WORKDIR /var/www

COPY --from=composer /app/vendor    ./vendor
COPY --from=composer /app           .
COPY --from=assets   /app/public/build ./public/build

RUN chown -R www-data:www-data /var/www && chmod -R 755 storage bootstrap/cache

USER www-data

EXPOSE 9000
CMD ["php-fpm"]
```

> PHP-FPM запускается за Nginx. Nginx отдаёт статику сам, PHP-FPM обрабатывает только `.php`:

```nginx
# nginx конфиг для Laravel
location ~ \.php$ {
    fastcgi_pass php-fpm:9000;
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    include fastcgi_params;
}
```

---

## 7. Frontend (React / Vue / Angular)

SPA-приложения компилируются в статические файлы. Финальный образ — Nginx с этими файлами.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS builder

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

# Build args для окружения (не секреты — они попадут в JS bundle!)
ARG VITE_API_URL=https://api.example.com
ENV VITE_API_URL=$VITE_API_URL

RUN npm run build

# ──────────────────────────────────────────────
FROM nginx:1.25-alpine AS runner

# Кастомный nginx конфиг (SPA: все маршруты → index.html)
COPY nginx.conf /etc/nginx/nginx.conf

# Статические файлы из builder
COPY --from=builder /app/dist /usr/share/nginx/html

# Права
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chmod -R 755 /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**`nginx.conf` для SPA (React Router / Vue Router):**
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip
    gzip on;
    gzip_types text/css application/javascript application/json image/svg+xml;

    # Кэш статики с хешем в имени (Vite/webpack генерирует)
    location ~* \.(js|css|png|jpg|svg|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # SPA fallback: любой маршрут → index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Не кэшировать index.html (браузер всегда проверяет обновления)
    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }
}
```

**Важно:** переменные окружения во frontend должны быть известны на этапе сборки (`ARG`). Для runtime конфигурации используй либо `/config.json` который генерируется при старте контейнера, либо подстановку в `index.html`:

```dockerfile
# entrypoint.sh — подставить ENV переменные в runtime
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
CMD ["/entrypoint.sh"]
```

```bash
#!/bin/sh
# entrypoint.sh — заменить плейсхолдеры в собранном JS
envsubst '${API_URL} ${APP_VERSION}' < /usr/share/nginx/html/env-config.js.template \
  > /usr/share/nginx/html/env-config.js
exec nginx -g "daemon off;"
```

---

## 8. Rust

Rust компилирует статический бинарь — аналогично Go, идеально для `scratch` или `distroless`.

```dockerfile
# syntax=docker/dockerfile:1

# Кэш зависимостей: создаём фиктивный main.rs
FROM rust:1.77-slim AS builder

WORKDIR /build

# Установить musl target для статической линковки
RUN rustup target add x86_64-unknown-linux-musl && \
    apt-get update && apt-get install -y --no-install-recommends musl-tools && \
    rm -rf /var/lib/apt/lists/*

# Хак для кэша зависимостей: собрать только Cargo.toml без исходников
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release --target x86_64-unknown-linux-musl
RUN rm src/main.rs

# Теперь реальные исходники
COPY src ./src
# touch чтобы cargo пересобрал (иначе использует кэш с dummy main)
RUN touch src/main.rs
RUN cargo build --release --target x86_64-unknown-linux-musl

# ──────────────────────────────────────────────
FROM scratch

COPY --from=builder /build/target/x86_64-unknown-linux-musl/release/myapp /myapp

USER 65534:65534
EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

**Результат:** образ ~5–20 МБ в зависимости от приложения.

> Используй `cargo-chef` для ещё более эффективного кэширования зависимостей в CI.

---

## 9. Универсальные правила и антипаттерны

### Правила (применимы к любому стеку)

**Порядок слоёв — от стабильных к изменчивым:**
```dockerfile
# ✅ Правильно
COPY package.json ./          # меняется редко → кэш живёт долго
RUN npm ci                    # пересобирается только если package.json изменился
COPY src/ ./src/              # меняется часто → кэш инвалидируется, но только для этого слоя

# ❌ Неправильно
COPY . .                      # любое изменение кода → пересборка npm ci
RUN npm ci
```

**Exec-форма CMD/ENTRYPOINT — всегда:**
```dockerfile
# ✅ Приложение = PID 1, получает SIGTERM → graceful shutdown
ENTRYPOINT ["node", "server.js"]

# ❌ /bin/sh = PID 1, SIGTERM не доходит до node → kill -9 через 30 сек
ENTRYPOINT node server.js
```

**Один процесс — один контейнер:**
```dockerfile
# ❌ Supervisord для nginx + app в одном контейнере → плохо
# ✅ Отдельные контейнеры + docker-compose / Kubernetes
```

**Непривилегированный пользователь везде:**
```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

**Read-only filesystem где возможно:**
```bash
docker run --read-only --tmpfs /tmp myapp
```

### Сравнительная таблица базовых образов

| Образ | Размер | Shell | Пакетный менеджер | Когда использовать |
|---|---|---|---|---|
| `ubuntu:22.04` | ~70 МБ | bash | apt | Разработка, когда нужно много утилит |
| `debian:12-slim` | ~30 МБ | bash | apt | Общий production выбор |
| `alpine:3.19` | ~5 МБ | sh (busybox) | apk | Минимальный размер, musl libc |
| `distroless/base` | ~20 МБ | Нет | Нет | Безопасность, есть glibc |
| `distroless/static` | ~2 МБ | Нет | Нет | Статические бинари (Go, Rust) |
| `scratch` | 0 МБ | Нет | Нет | Статические бинари, максимум безопасности |

> **Осторожно с Alpine для Java/Python:** musl libc может давать неожиданное поведение. Для JVM используй `-slim` варианты на glibc.

### Антипаттерны

```dockerfile
# ❌ Устанавливать и не чистить кэш в одном RUN
RUN apt-get install nginx
RUN rm -rf /var/lib/apt/lists/*
# Кэш всё равно останется в предыдущем слое!

# ✅ Всё в одном RUN
RUN apt-get update && apt-get install -y nginx && rm -rf /var/lib/apt/lists/*

# ❌ Хранить секреты в образе
ARG DB_PASSWORD
RUN echo $DB_PASSWORD > /app/.env  # навсегда в слое!

# ✅ Передавать секреты при запуске
# docker run -e DB_PASSWORD=secret myapp
# или Docker/K8s Secrets

# ❌ Хардкодить версии зависимостей как latest
FROM node:latest
FROM python:latest

# ✅ Фиксировать версии
FROM node:20.11-alpine
FROM python:3.12.2-slim

# ❌ Запускать как root без необходимости
# (по умолчанию если не указан USER)

# ❌ Большой build context без .dockerignore
# Отправляет node_modules (500 МБ) демону перед сборкой

# ❌ Отдельный RUN для каждой команды установки
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
# Три отдельных слоя = три раза overhead
```

### Проверка образа перед публикацией

```bash
# Размер и слои
docker image history myapp:latest
dive myapp:latest        # интерактивный анализ слоёв

# Уязвимости
trivy image myapp:latest
docker scout cves myapp:latest

# Запуск без root
docker run --user 1000:1000 --read-only --cap-drop ALL myapp:latest

# Проверить что приложение = PID 1
docker run myapp:latest ps aux
# должно быть: PID 1 — твоё приложение, не sh/bash
```
