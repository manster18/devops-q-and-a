# Web — Вопросы и ответы для собеседований (Middle/Senior DevOps)

## Содержание

### HTTP / HTTPS
1. [Что происходит от момента ввода URL до получения ответа?](#1-что-происходит-от-момента-ввода-url-до-получения-ответа)
2. [В чём разница между HTTP/1.1, HTTP/2 и HTTP/3?](#2-в-чём-разница-между-http11-http2-и-http3)
3. [Как работает TLS Handshake?](#3-как-работает-tls-handshake)
4. [Что такое HTTP Keep-Alive и зачем он нужен?](#4-что-такое-http-keep-alive-и-зачем-он-нужен)
5. [В чём разница между 301 и 302 редиректом? Когда использовать каждый?](#5-в-чём-разница-между-301-и-302-редиректом-когда-использовать-каждый)
6. [Что такое CORS и как его правильно настроить?](#6-что-такое-cors-и-как-его-правильно-настроить)

### Nginx
7. [Как работает Nginx и чем он отличается от Apache?](#7-как-работает-nginx-и-чем-он-отличается-от-apache)
8. [Как настроить Nginx как reverse proxy?](#8-как-настроить-nginx-как-reverse-proxy)
9. [Как настроить SSL/TLS в Nginx?](#9-как-настроить-ssltls-в-nginx)
10. [Как настроить кэширование в Nginx?](#10-как-настроить-кэширование-в-nginx)
11. [Как настроить rate limiting в Nginx?](#11-как-настроить-rate-limiting-в-nginx)
12. [Как анализировать и дебажить проблемы через логи Nginx?](#12-как-анализировать-и-дебажить-проблемы-через-логи-nginx)

### Балансировка нагрузки
13. [Какие алгоритмы балансировки нагрузки существуют?](#13-какие-алгоритмы-балансировки-нагрузки-существуют)
14. [В чём разница между L4 и L7 балансировщиком?](#14-в-чём-разница-между-l4-и-l7-балансировщиком)
15. [Что такое health check и как его настроить?](#15-что-такое-health-check-и-как-его-настроить)

### DNS
16. [Как работает DNS-резолюция? Полный путь запроса.](#16-как-работает-dns-резолюция-полный-путь-запроса)
17. [Какие типы DNS-записей существуют и для чего они нужны?](#17-какие-типы-dns-записей-существуют-и-для-чего-они-нужны)
18. [Что такое TTL в DNS и как правильно его выставлять?](#18-что-такое-ttl-в-dns-и-как-правильно-его-выставлять)

### SSL/TLS и безопасность
19. [Что такое SNI и зачем он нужен?](#19-что-такое-sni-и-зачем-он-нужен)
20. [Как работает Let's Encrypt и протокол ACME?](#20-как-работает-lets-encrypt-и-протокол-acme)
21. [Что такое HTTP Security Headers и какие из них важнейшие?](#21-что-такое-http-security-headers-и-какие-из-них-важнейшие)
22. [Что такое DDoS и каковы основные методы защиты?](#22-что-такое-ddos-и-каковы-основные-методы-защиты)

---

## HTTP / HTTPS

### 1. Что происходит от момента ввода URL до получения ответа?

Это классический вопрос, проверяющий глубину понимания сетевого стека. Разберём по шагам на примере `https://example.com/page`.

**Шаг 1 — DNS-резолюция**
Браузер проверяет кэши по порядку: собственный кэш браузера → кэш ОС → `/etc/hosts` → запрос к локальному DNS-резолверу (обычно из DHCP) → рекурсивная резолюция вплоть до авторитетного NS-сервера. Получаем IP-адрес.

**Шаг 2 — TCP Handshake**
Устанавливается TCP-соединение с сервером (IP + порт 443):
```
Client → SYN         → Server
Client ← SYN-ACK     ← Server
Client → ACK         → Server
```

**Шаг 3 — TLS Handshake**
Поверх TCP устанавливается защищённый канал (подробнее в вопросе 3). Это добавляет 1-2 RTT (round-trip time).

**Шаг 4 — HTTP-запрос**
Браузер отправляет HTTP-запрос:
```
GET /page HTTP/2
Host: example.com
Accept: text/html,application/xhtml+xml
Accept-Encoding: gzip, br
Cookie: session_id=abc123
```

**Шаг 5 — Обработка на сервере**
Запрос проходит через: CDN → Load Balancer → Web-сервер (Nginx) → Application → Database (если нужно).

**Шаг 6 — HTTP-ответ**
Сервер возвращает ответ со статусом, заголовками и телом:
```
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Cache-Control: public, max-age=3600
```

**Шаг 7 — Рендеринг**
Браузер парсит HTML, находит ресурсы (CSS, JS, изображения), делает дополнительные запросы (уже по существующему соединению благодаря Keep-Alive/multiplexing в HTTP/2).

**Итого задержки (типичные цифры):**
- DNS: 20-120 мс (первый раз, потом из кэша ~0)
- TCP handshake: 1 RTT (~20-100 мс в зависимости от расстояния)
- TLS handshake: 1-2 RTT
- Первый байт ответа (TTFB): зависит от сервера

> Именно поэтому важны: геораспределённые DNS (Anycast), CDN близко к пользователю, TLS 1.3 (сокращает handshake до 1 RTT), HTTP/2 (мультиплексирование).

---

### 2. В чём разница между HTTP/1.1, HTTP/2 и HTTP/3?

**HTTP/1.1** (1997)
- Текстовый протокол
- Одно соединение = один запрос в один момент (HOL blocking — Head-of-Line blocking)
- Браузер открывает 6-8 параллельных соединений как workaround
- Keep-Alive позволяет переиспользовать соединение для нескольких запросов последовательно

**HTTP/2** (2015)
- Бинарный протокол (не читаемый человеком)
- **Мультиплексирование:** много запросов в одном TCP-соединении одновременно
- **Header compression (HPACK):** заголовки сжимаются и кэшируются
- **Server Push:** сервер может сам отправить ресурсы (CSS, JS) до того, как браузер их запросит
- **Stream приоритизация**
- Проблема: HOL blocking на уровне TCP — потеря одного пакета блокирует все потоки

```
HTTP/1.1:  [req1]----[req2]----[req3]    (последовательно)
HTTP/2:    [req1]
           [req2]    (параллельно в одном TCP-соединении)
           [req3]
```

**HTTP/3** (2022)
- Работает поверх **QUIC** (UDP вместо TCP)
- Решает HOL blocking на транспортном уровне: потеря пакета блокирует только один поток, а не всё соединение
- **0-RTT / 1-RTT**: быстрое установление соединения (для повторных визитов — 0 RTT)
- TLS 1.3 встроен в QUIC изначально
- Лучше работает при нестабильных соединениях (мобильные сети, переключение между WiFi и LTE)

| Характеристика | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Транспорт | TCP | TCP | QUIC (UDP) |
| Мультиплексирование | нет | да | да |
| Сжатие заголовков | нет | HPACK | QPACK |
| HOL blocking | на уровне HTTP | на уровне TCP | нет |
| TLS | опционально | обязателен (де-факто) | встроен |

**Проверить поддержку HTTP/2 / HTTP/3:**
```bash
curl -I --http2 https://example.com
curl -I --http3 https://example.com

# В Nginx включить HTTP/2:
# listen 443 ssl http2;

# HTTP/3 (Nginx 1.25+):
# listen 443 quic reuseport;
# add_header Alt-Svc 'h3=":443"; ma=86400';
```

---

### 3. Как работает TLS Handshake?

TLS обеспечивает **аутентификацию** (ты разговариваешь с правильным сервером), **конфиденциальность** (шифрование) и **целостность** (данные не изменены).

**TLS 1.2 Handshake (2 RTT):**

```
Client                              Server
  |                                   |
  |--- ClientHello -----------------> |  # версии TLS, cipher suites, случайное число
  |                                   |
  |<-- ServerHello ------------------|  # выбранный cipher suite, случайное число
  |<-- Certificate ------------------|  # публичный ключ сервера + цепочка CA
  |<-- ServerHelloDone --------------|
  |                                   |
  |  [Проверяем сертификат:           |
  |   - подпись CA валидна?           |
  |   - срок действия?                |
  |   - CN/SAN совпадает с хостом?]   |
  |                                   |
  |--- ClientKeyExchange -----------> |  # pre-master secret (шифрован публичным ключом сервера)
  |--- ChangeCipherSpec ------------> |
  |--- Finished --------------------> |
  |                                   |
  |<-- ChangeCipherSpec -------------|
  |<-- Finished --------------------|
  |                                   |
  |=== Зашифрованный HTTP ========== |
```

**TLS 1.3 Handshake (1 RTT):**
Значительно упрощён — убраны слабые алгоритмы, согласование происходит быстрее:
```
Client                              Server
  |--- ClientHello -----------------> |  # + KeyShare (публичная часть DH)
  |<-- ServerHello, Certificate -----|  # + KeyShare сервера, Finished
  |                                   |
  |--- Finished --------------------> |
  |=== Зашифрованный HTTP ========== |
```

**0-RTT в TLS 1.3:** при повторном соединении клиент может сразу отправить данные, используя сохранённый session ticket. Но это уязвимо к replay-атакам, поэтому используется осторожно.

**Проверить TLS-конфигурацию:**
```bash
# Полная информация о handshake
openssl s_client -connect example.com:443 -servername example.com

# Проверить поддерживаемые версии и cipher suites
nmap --script ssl-enum-ciphers -p 443 example.com

# Онлайн: ssllabs.com/ssltest
```

**Ключевые понятия для собеседования:**
- **Certificate Chain:** сертификат сервера → промежуточный CA → корневой CA (доверенный)
- **Cipher Suite:** набор алгоритмов, например `TLS_AES_256_GCM_SHA384` (TLS 1.3) или `ECDHE-RSA-AES256-GCM-SHA384` (TLS 1.2)
- **Perfect Forward Secrecy (PFS):** использование ephemeral ключей (ECDHE) — компрометация приватного ключа сервера не раскроет прошлые сессии

---

### 4. Что такое HTTP Keep-Alive и зачем он нужен?

По умолчанию в HTTP/1.0 каждый запрос открывал новое TCP-соединение: `3-way handshake → запрос → ответ → закрытие соединения`. При загрузке страницы с десятками ресурсов это катастрофически медленно.

**Keep-Alive** (persistent connection) позволяет переиспользовать одно TCP-соединение для нескольких HTTP-запросов/ответов.

```
# HTTP/1.1 — Keep-Alive включён по умолчанию
Connection: keep-alive
Keep-Alive: timeout=5, max=100
# timeout — сколько секунд держать соединение открытым
# max — максимум запросов через это соединение
```

**Настройка в Nginx:**
```nginx
# Таймаут keep-alive для клиентских соединений
keepalive_timeout 65;

# Keep-alive соединения к upstream (backend)
upstream backend {
    server 127.0.0.1:8080;
    keepalive 32;  # пул из 32 persistent соединений к backend
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";  # убрать "close" для keepalive к upstream
    }
}
```

**Почему важно для DevOps:**
- Без `keepalive` к upstream — каждый запрос создаёт новое TCP-соединение к backend. При высоком RPS это тысячи лишних handshake в секунду и быстрое исчерпание портов (`TIME_WAIT` соединения).
- С `keepalive 32` — Nginx держит пул из 32 постоянных соединений, переиспользуя их.

> В HTTP/2 проблема решена архитектурно — один мультиплексируемый поток делает Keep-Alive бессмысленным на уровне HTTP (но TCP-соединение всё равно одно).

---

### 5. В чём разница между 301 и 302 редиректом? Когда использовать каждый?

| Код | Название | Кэшируется? | Метод | Когда использовать |
|---|---|---|---|---|
| **301** | Moved Permanently | Да (браузером, CDN) | GET (принудительно) | Постоянный переезд: HTTP → HTTPS, смена домена, canonical URL |
| **302** | Found (Temporary) | Нет | Сохраняется | Временный редирект: A/B тест, maintenance page, геолокация |
| **303** | See Other | Нет | Всегда GET | После POST-запроса (PRG pattern — Post/Redirect/Get) |
| **307** | Temporary Redirect | Нет | Сохраняется (строго) | Временный редирект с сохранением метода (POST остаётся POST) |
| **308** | Permanent Redirect | Да | Сохраняется (строго) | Постоянный редирект с сохранением метода |

**Критическая разница 301 vs 302 на практике:**

При 301 браузер **кэширует** редирект навсегда (пока не очистить кэш). Это значит:
- CDN тоже закэширует и начнёт отдавать 301 из кэша
- Если ты ошибся с 301 и хочешь его откатить — пользователи со старым кэшем будут видеть старый редирект ещё долго

```nginx
# Nginx — правильный редирект HTTP → HTTPS (301, постоянный)
server {
    listen 80;
    server_name example.com;
    return 301 https://example.com$request_uri;
}

# Временный редирект на maintenance (302)
location / {
    return 302 /maintenance.html;
}
```

**Важно про SEO:** поисковики передают "PageRank" только через 301. Если нужно сохранить позиции при смене домена — только 301.

**Совет:** если не уверен — используй 302. Потом всегда можно поменять на 301. Обратное — болезненно.

---

### 6. Что такое CORS и как его правильно настроить?

**CORS (Cross-Origin Resource Sharing)** — механизм браузерной безопасности, ограничивающий HTTP-запросы к другому origin (домену, протоколу или порту).

**Политика одного источника (Same-Origin Policy):** браузер блокирует JavaScript-запросы к `api.example.com` если страница загружена с `app.example.com` — это разные origins.

**Как работает CORS:**

1. **Simple requests** (GET/POST с простыми заголовками): браузер добавляет заголовок `Origin` к запросу. Сервер должен ответить с `Access-Control-Allow-Origin`.

2. **Preflight (предварительный запрос):** для "сложных" запросов (PUT/DELETE, кастомные заголовки, JSON body) браузер сначала отправляет `OPTIONS`-запрос, чтобы спросить разрешение:
```
OPTIONS /api/users HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: Authorization, Content-Type
```
Сервер должен ответить:
```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400  # кэшировать preflight на 24 часа
```

**Настройка в Nginx:**
```nginx
map $http_origin $cors_origin {
    default "";
    "https://app.example.com"   $http_origin;
    "https://admin.example.com" $http_origin;
}

server {
    location /api/ {
        # Preflight
        if ($request_method = 'OPTIONS') {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
            add_header Access-Control-Allow-Headers "Authorization, Content-Type, X-Requested-With" always;
            add_header Access-Control-Max-Age 86400;
            add_header Content-Length 0;
            return 204;
        }

        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Credentials true always;

        proxy_pass http://backend;
    }
}
```

**Частые ошибки:**
- `Access-Control-Allow-Origin: *` + `Access-Control-Allow-Credentials: true` — так нельзя. При credentials нужен конкретный origin.
- Не добавлен заголовок в ответы с кодом 4xx/5xx (нужен `always` в директиве `add_header`).
- Забыт обработчик `OPTIONS`.

---

## Nginx

### 7. Как работает Nginx и чем он отличается от Apache?

**Apache** использует **процессо-ориентированную или поточную** модель: на каждое соединение создаётся процесс (prefork MPM) или поток (worker MPM). При 10 000 соединений — 10 000 потоков/процессов → огромный расход памяти. Это называется "C10K problem".

**Nginx** использует **event-driven, асинхронную, неблокирующую** архитектуру:
- Один master-процесс: читает конфиг, управляет воркерами
- N worker-процессов (обычно = числу CPU ядер): каждый обрабатывает тысячи соединений через event loop (epoll на Linux)
- Нет блокирующих вызовов — пока один запрос ждёт ответа от backend, воркер обрабатывает другие запросы

```
nginx:
Master process
├── Worker process (core 0) → [conn1, conn2, conn3, ..., conn10000]  # event loop
├── Worker process (core 1) → [conn1, conn2, conn3, ..., conn10000]
├── Worker process (core 2) → [...]
└── Cache manager process
```

**Практические отличия:**

| Параметр | Nginx | Apache |
|---|---|---|
| Архитектура | Event-driven | Process/Thread-based |
| Статические файлы | Очень быстро | Медленнее |
| Динамический контент | Через proxy_pass (нет mod_php) | mod_php, mod_wsgi inline |
| Память при нагрузке | Практически не растёт | Растёт линейно |
| `.htaccess` | Не поддерживается | Да |
| Конфигурация | Централизованная | Можно в директориях (.htaccess) |

**Когда всё ещё выбирают Apache:**
- Shared hosting (нужен `.htaccess`)
- Legacy PHP-приложения с mod_php
- Нужна per-directory конфигурация

**Структура конфига Nginx:**
```nginx
# /etc/nginx/nginx.conf
worker_processes auto;          # число воркеров = число CPU
worker_rlimit_nofile 65535;     # лимит файловых дескрипторов

events {
    worker_connections 4096;    # соединений на воркер
    use epoll;                  # механизм мультиплексирования
    multi_accept on;            # принимать несколько соединений за раз
}

http {
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

---

### 8. Как настроить Nginx как reverse proxy?

**Reverse proxy** принимает запросы от клиентов и перенаправляет их к backend-серверам, скрывая их за собой.

```nginx
upstream backend {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    # SSL (см. вопрос 9)
    ssl_certificate     /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    location / {
        proxy_pass http://backend;

        # Передаём реальный IP клиента
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Таймауты
        proxy_connect_timeout  5s;
        proxy_send_timeout     60s;
        proxy_read_timeout     60s;

        # Буферизация ответа от backend
        proxy_buffering on;
        proxy_buffer_size          4k;
        proxy_buffers              8 16k;
        proxy_busy_buffers_size    32k;

        # Keep-alive к backend
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    # Статику отдаём напрямую, не дёргая backend
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2)$ {
        root /var/www/static;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

**Важные заголовки:**
- `X-Real-IP` — реальный IP клиента (иначе backend видит IP Nginx)
- `X-Forwarded-For` — цепочка прокси (может быть несколько)
- `X-Forwarded-Proto` — оригинальный протокол (http/https)

**На backend (например, Django/Express) нужно доверять этим заголовкам:**
```python
# Django settings.py
ALLOWED_HOSTS = ['example.com']
USE_X_FORWARDED_HOST = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

---

### 9. Как настроить SSL/TLS в Nginx?

```nginx
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # Сертификат и ключ
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Поддерживаемые версии (только TLS 1.2 и 1.3)
    ssl_protocols TLSv1.2 TLSv1.3;

    # Cipher suites (современная конфигурация)
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;  # в TLS 1.3 клиент выбирает, это нормально

    # DH параметры (для DHE cipher suites)
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;
    # Генерация: openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048

    # Session cache (ускоряет повторные TLS handshake)
    ssl_session_cache   shared:SSL:10m;  # 10 МБ ≈ 40 000 сессий
    ssl_session_timeout 1d;
    ssl_session_tickets off;  # выключить для лучшей forward secrecy

    # OCSP Stapling (сервер сам проверяет отзыв сертификата и прикладывает ответ)
    ssl_stapling        on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    resolver 8.8.8.8 8.8.4.4 valid=300s;

    # HSTS (браузеры будут обращаться только по HTTPS)
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    location / {
        proxy_pass http://backend;
    }
}

# Редирект HTTP → HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

**Проверка конфигурации:**
```bash
nginx -t                     # проверить синтаксис
nginx -T                     # вывести итоговый конфиг

# Проверить сертификат
openssl s_client -connect example.com:443 -servername example.com | openssl x509 -text -noout

# Срок действия
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# Онлайн-проверка: https://www.ssllabs.com/ssltest/
```

---

### 10. Как настроить кэширование в Nginx?

Nginx может кэшировать ответы от upstream, чтобы не дёргать backend при каждом запросе.

```nginx
http {
    # Определяем зону кэша
    # levels=1:2 — структура директорий (для эффективной работы ФС)
    # keys_zone=my_cache:10m — 10 МБ для хранения ключей (≈80 000 записей)
    # max_size=1g — максимум 1 ГБ данных на диске
    # inactive=60m — удалить кэш если не запрашивался 60 минут
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m use_temp_path=off;

    server {
        location / {
            proxy_pass http://backend;
            proxy_cache my_cache;

            # Ключ кэша (по чему различаем записи)
            proxy_cache_key "$scheme$request_method$host$request_uri";

            # Кэшировать ответы 200 и 301 на 1 час, 404 на 1 минуту
            proxy_cache_valid 200 301 1h;
            proxy_cache_valid 404 1m;

            # Отдавать устаревший кэш если backend недоступен
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;

            # Только один запрос к backend при обновлении кэша (cache lock)
            proxy_cache_lock on;

            # Добавить заголовок для отладки: HIT/MISS/BYPASS/EXPIRED
            add_header X-Cache-Status $upstream_cache_status;
        }

        # Не кэшировать определённые пути
        location /api/user/ {
            proxy_pass http://backend;
            proxy_no_cache 1;
            proxy_cache_bypass 1;
        }
    }
}
```

**Управление кэшем через заголовки backend:**
```
Cache-Control: no-store          # никогда не кэшировать
Cache-Control: no-cache          # кэшировать, но всегда валидировать
Cache-Control: public, max-age=3600  # кэшировать 1 час
Cache-Control: private           # только клиентский кэш (не прокси)
Vary: Accept-Encoding            # разные версии кэша для разных кодировок
```

**Очистить кэш:**
```bash
# Вручную — удалить файлы
find /var/cache/nginx -type f -delete

# Через Nginx Plus (коммерческий) — purge
# В open source через модуль ngx_cache_purge
```

**Кэширование статики (отдельный механизм):**
```nginx
location ~* \.(css|js|png|jpg|woff2|svg)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    # immutable = браузер не будет делать conditional request даже при force-reload
}
```

---

### 11. Как настроить rate limiting в Nginx?

Rate limiting защищает от брутфорса, DDoS, злоупотреблений API.

```nginx
http {
    # Зона для rate limiting по IP
    # $binary_remote_addr — IP в бинарном виде (экономит память)
    # zone=req_limit:10m — 10 МБ памяти (~160 000 IP-адресов)
    # rate=10r/s — максимум 10 запросов в секунду с одного IP
    limit_req_zone $binary_remote_addr zone=req_limit:10m rate=10r/s;

    # Отдельная зона для login endpoint (жёстче)
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=1r/s;

    # Rate limit по соединениям (не запросам)
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    server {
        location / {
            # burst=20 — разрешить всплеск до 20 запросов (в очередь)
            # nodelay — не задерживать burst-запросы, обрабатывать сразу
            limit_req zone=req_limit burst=20 nodelay;
            limit_conn conn_limit 10;  # макс 10 одновременных соединений с IP

            proxy_pass http://backend;
        }

        location /api/login {
            limit_req zone=login_limit burst=5;
            # без nodelay — запросы сверх rate будут задержаны (не отклонены сразу)

            proxy_pass http://backend;
        }
    }
}
```

**Как работает burst:**
```
rate=10r/s, burst=20, nodelay:
- Одновременно пришло 30 запросов
- 20 обрабатываются сразу (burst)
- 10 → 429 Too Many Requests

rate=10r/s, burst=20 (без nodelay):
- 10 обрабатываются сразу
- 20 ставятся в очередь и обрабатываются со скоростью 10/сек
- Клиент ждёт до 2 секунд
```

**Настройка кода ответа и логирования:**
```nginx
limit_req_status 429;         # вернуть 429, а не дефолтный 503
limit_conn_status 429;

# Логировать rate limit события (по умолчанию уровень error — многовато)
limit_req_log_level warn;
limit_conn_log_level warn;
```

**Rate limiting по API-ключу (не по IP):**
```nginx
# Если API-ключ передаётся в заголовке
limit_req_zone $http_x_api_key zone=api_limit:10m rate=100r/s;
```

---

### 12. Как анализировать и дебажить проблемы через логи Nginx?

**Формат access.log по умолчанию:**
```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
```

**Расширенный формат для дебага (рекомендую добавить):**
```nginx
log_format detailed '$remote_addr - [$time_local] "$request" $status '
                    '$body_bytes_sent $request_time '
                    'upstream: $upstream_addr $upstream_status $upstream_response_time '
                    '"$http_referer" "$http_user_agent"';
```

**Полезные команды для анализа:**
```bash
# Топ-10 IP по количеству запросов
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Топ запрашиваемые URL
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Все ошибки 5xx
grep '" 5' /var/log/nginx/access.log | tail -50

# Медленные запросы (>1 секунды) — если логируешь $request_time
awk '$NF > 1 {print $NF, $0}' /var/log/nginx/access.log | sort -rn | head -20

# Количество запросов в минуту (пиковые моменты)
awk '{print $4}' /var/log/nginx/access.log | cut -d: -f1-3 | uniq -c

# Статусы ответов
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

**Debug logging (временно!):**
```nginx
# В server или location блоке
error_log /var/log/nginx/debug.log debug;
# ВНИМАНИЕ: debug-логи огромны, включать только временно
```

**Включить логирование заголовков запроса/ответа:**
```nginx
# В location блоке для конкретного endpoint
log_format headers '$remote_addr "$request" $status '
                   'req_headers: $http_authorization $http_content_type '
                   'resp_headers: $sent_http_content_type $sent_http_cache_control';
```

**Анализ с помощью GoAccess (визуальный реалтайм-анализатор):**
```bash
goaccess /var/log/nginx/access.log -o /var/www/report.html --log-format=COMBINED
# Или в реальном времени:
tail -f /var/log/nginx/access.log | goaccess - --log-format=COMBINED
```

---

## Балансировка нагрузки

### 13. Какие алгоритмы балансировки нагрузки существуют?

**1. Round Robin (по умолчанию в Nginx)**
Запросы распределяются по очереди: 1→2→3→1→2→3...
```nginx
upstream backend {
    server 10.0.0.1;
    server 10.0.0.2;
    server 10.0.0.3;
}
```
Хорош для однородных серверов с одинаковой мощностью.

**2. Weighted Round Robin**
Серверам разного "веса" достаётся пропорционально больше трафика:
```nginx
upstream backend {
    server 10.0.0.1 weight=3;  # получит 3/5 трафика
    server 10.0.0.2 weight=2;  # получит 2/5 трафика
}
```
Используют при постепенном выкатывании новой версии (canary deployment).

**3. Least Connections**
Запрос идёт на сервер с наименьшим числом активных соединений. Хорош при разной длительности обработки запросов:
```nginx
upstream backend {
    least_conn;
    server 10.0.0.1;
    server 10.0.0.2;
}
```

**4. IP Hash**
Клиент всегда попадает на один и тот же сервер (по хешу IP). Используется для **sticky sessions** — когда сессия пользователя хранится на конкретном backend:
```nginx
upstream backend {
    ip_hash;
    server 10.0.0.1;
    server 10.0.0.2;
}
```
Минус: при выходе из строя одного сервера часть пользователей теряет сессии. Лучше хранить сессии в Redis.

**5. Hash (по произвольному ключу)**
```nginx
upstream backend {
    hash $request_uri consistent;  # consistent = consistent hashing
    server 10.0.0.1;
    server 10.0.0.2;
}
```
`consistent` — алгоритм consistent hashing: при добавлении/удалении сервера перераспределяется минимум ключей. Удобно для кэширующих прокси.

**6. Random (Nginx Plus)**
Случайный выбор из N серверов с наименьшим числом соединений.

**Дополнительные параметры серверов:**
```nginx
upstream backend {
    server 10.0.0.1 max_fails=3 fail_timeout=30s;
    # max_fails=3 — после 3 неудачных попыток сервер помечается как unavailable
    # fail_timeout=30s — на 30 секунд, потом снова пробуем

    server 10.0.0.2 backup;
    # backup — используется только когда все основные недоступны

    server 10.0.0.3 down;
    # down — постоянно отключён (например, на обслуживании)
}
```

---

### 14. В чём разница между L4 и L7 балансировщиком?

Цифры относятся к уровням модели OSI.

**L4 (Transport Layer) балансировщик:**
- Работает с TCP/UDP-потоками, не понимает содержимого
- Видит только: IP-адреса источника/назначения, порты, TCP-флаги
- Очень быстрый, низкий overhead
- Не может: различать URL, читать HTTP-заголовки, обрабатывать SSL

```
Клиент → L4 LB → Backend
         (только IP:port routing)
```

**Примеры L4:** AWS NLB, HAProxy (в TCP mode), Linux IPVS (LVS), Nginx (stream module).

```nginx
# Nginx как L4 балансировщик (stream module)
stream {
    upstream mysql_backend {
        server 10.0.0.1:3306;
        server 10.0.0.2:3306;
    }

    server {
        listen 3306;
        proxy_pass mysql_backend;
    }
}
```

**L7 (Application Layer) балансировщик:**
- Понимает HTTP/HTTPS-протокол полностью
- Может: маршрутизировать по URL, заголовкам, cookies, методу
- Умеет: терминировать SSL, изменять запросы, кэшировать, сжимать
- Выше overhead (разбирает каждый запрос)

```
Клиент → L7 LB (SSL termination) → Backend
         (URL-based routing, заголовки, cookies)
```

**Примеры L7:** Nginx (http module), AWS ALB, HAProxy (HTTP mode), Traefik, Envoy.

```nginx
# Nginx как L7: маршрутизация по URL
upstream api_backend  { server 10.0.0.1:8080; }
upstream web_backend  { server 10.0.0.2:8080; }
upstream static_cdn   { server 10.0.0.3:8080; }

server {
    location /api/    { proxy_pass http://api_backend; }
    location /static/ { proxy_pass http://static_cdn; }
    location /        { proxy_pass http://web_backend; }
}
```

| Критерий | L4 | L7 |
|---|---|---|
| Скорость | Выше | Ниже (overhead на парсинг) |
| SSL termination | Нет | Да |
| Content-based routing | Нет | Да |
| DDoS защита | Базовая | Продвинутая |
| Протоколы | Любые TCP/UDP | HTTP, gRPC, WebSocket |

---

### 15. Что такое health check и как его настроить?

**Health check** — механизм проверки доступности и работоспособности backend-серверов, чтобы балансировщик не слал трафик на упавший сервер.

**Пассивные health checks (Nginx open source):**
Nginx замечает проблему по реальным запросам — если ответ не пришёл или вернулся 502/504:
```nginx
upstream backend {
    server 10.0.0.1 max_fails=3 fail_timeout=30s;
    server 10.0.0.2 max_fails=3 fail_timeout=30s;
}
```
Минус: пока Nginx "понимает" что сервер упал — несколько реальных запросов пользователей уже получили ошибку.

**Активные health checks (Nginx Plus или модуль ngx_http_healthcheck):**
Nginx сам периодически опрашивает backend:
```nginx
# Только Nginx Plus
upstream backend {
    zone backend 64k;
    server 10.0.0.1;
    server 10.0.0.2;
}

server {
    location / {
        proxy_pass http://backend;
        health_check interval=5s fails=3 passes=2 uri=/health;
        # interval  — проверять каждые 5 секунд
        # fails=3   — пометить unhealthy после 3 неудач подряд
        # passes=2  — вернуть в ротацию после 2 успехов подряд
        # uri       — URL для проверки
    }
}
```

**Что должен возвращать endpoint `/health`:**
```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "status": "ok",
  "db": "ok",
  "redis": "ok",
  "version": "1.2.3"
}
```
Если хотя бы один компонент недоступен — возвращать 503.

**Уровни health checks:**
- **Shallow check:** просто пинг порта — TCP-соединение устанавливается. Не проверяет работоспособность приложения.
- **Deep check:** HTTP-запрос к `/health`, который проверяет соединение с БД, кэшем, внешними зависимостями.

**В Kubernetes:**
```yaml
livenessProbe:
  httpGet:
    path: /health/live   # жив ли процесс? нет — перезапустить Pod
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready  # готов принимать трафик? нет — убрать из Service
    port: 8080
  periodSeconds: 5
```

---

## DNS

### 16. Как работает DNS-резолюция? Полный путь запроса.

Разберём что происходит при первом запросе к `api.example.com`.

```
1. Браузер/ОС проверяет кэш (DNS cache)
         ↓ промах
2. Запрос к локальному Stub Resolver (127.0.0.53 или 127.0.0.1:53)
         ↓
3. Stub Resolver → Recursive Resolver (8.8.8.8 или провайдерский DNS)
   Recursive Resolver проверяет свой кэш → промах
         ↓
4. Recursive Resolver → Root Name Server (.)
   Ответ: "я не знаю, но для .com спроси 192.5.6.30"
         ↓
5. Recursive Resolver → TLD Name Server (.com)
   Ответ: "я не знаю, но для example.com спроси ns1.example.com (205.251.196.1)"
         ↓
6. Recursive Resolver → Authoritative Name Server (ns1.example.com)
   Ответ: "api.example.com → 93.184.216.34, TTL=300"
         ↓
7. Recursive Resolver кэширует ответ (на TTL секунд)
   Возвращает IP браузеру/ОС
         ↓
8. ОС кэширует ответ
   Браузер кэширует ответ
```

**Важные термины:**
- **Recursive Resolver** — делает всю работу за клиента (рекурсивно обходит серверы). Обычно предоставляется ISP или публичный (8.8.8.8, 1.1.1.1).
- **Authoritative NS** — единственный источник истины для зоны. Отвечает финальным ответом.
- **Root Servers** — 13 групп серверов (a.root-servers.net...m.root-servers.net), реально сотни серверов по всему миру через Anycast.

**Инструменты диагностики:**
```bash
# Полная трассировка резолюции
dig +trace api.example.com

# Запрос к конкретному NS
dig @8.8.8.8 api.example.com

# Посмотреть кэшированный TTL
dig api.example.com | grep -i ttl

# Reverse DNS (PTR запись)
dig -x 93.184.216.34

# Проверить NS-серверы зоны
dig NS example.com

# Проверить SOA (Start of Authority)
dig SOA example.com
```

---

### 17. Какие типы DNS-записей существуют и для чего они нужны?

| Тип | Назначение | Пример |
|---|---|---|
| **A** | Домен → IPv4 | `example.com. → 93.184.216.34` |
| **AAAA** | Домен → IPv6 | `example.com. → 2606:2800:220:1:248:1893:25c8:1946` |
| **CNAME** | Псевдоним → другое имя | `www → example.com.` |
| **MX** | Mail server | `example.com. MX 10 mail.example.com.` |
| **TXT** | Произвольный текст | SPF, DKIM, DMARC, верификация Google |
| **NS** | Авторитетные серверы зоны | `example.com. NS ns1.example.com.` |
| **PTR** | IP → домен (reverse DNS) | `34.216.184.93.in-addr.arpa. → example.com.` |
| **SRV** | Сервис + порт + протокол | `_https._tcp.example.com. SRV 0 5 443 web.example.com.` |
| **SOA** | Начало зоны, параметры | Serial, Refresh, Retry, Expire, TTL |
| **CAA** | Какие CA могут выпускать сертификаты | `example.com. CAA 0 issue "letsencrypt.org"` |

**CNAME — важные ограничения:**
- CNAME нельзя использовать для apex домена (root domain, `example.com` без www). Только A/AAAA/ALIAS.
- Не может сосуществовать с другими записями (нельзя одновременно иметь CNAME и MX для одного имени).

**TXT-записи для email-безопасности:**
```dns
# SPF — кто может слать почту от имени домена
example.com. TXT "v=spf1 ip4:192.0.2.0/24 include:_spf.google.com ~all"

# DKIM — цифровая подпись писем
selector._domainkey.example.com. TXT "v=DKIM1; k=rsa; p=MIIBIjANBgkq..."

# DMARC — политика при нарушении SPF/DKIM
_dmarc.example.com. TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"
```

**SRV-запись** (используется в Kubernetes, Consul, SIP):
```
_service._proto.name. TTL class SRV priority weight port target
_http._tcp.example.com. 300 IN SRV 10 5 8080 backend.example.com.
```

---

### 18. Что такое TTL в DNS и как правильно его выставлять?

**TTL (Time To Live)** — время в секундах, на которое DNS-резолверы кэшируют запись. Пока TTL не истёк — резолвер отвечает из кэша, не обращаясь к авторитетному NS.

```bash
dig example.com A
# example.com.   300   IN   A   93.184.216.34
#                ^^^— TTL 300 секунд = 5 минут
```

**Стратегии TTL:**

| Сценарий | Рекомендуемый TTL | Причина |
|---|---|---|
| Статичные записи (редко меняются) | 3600-86400 (1 час — 1 день) | Меньше нагрузки на NS, быстрее резолюция |
| Обычные продакшн-записи | 300-3600 (5 мин — 1 час) | Баланс между нагрузкой и гибкостью |
| Планируемый failover | 60-300 (1-5 минут) | Быстрое переключение при аварии |
| Активная миграция | 60 (1 минута) | Максимальная гибкость |

**Правильная стратегия при смене IP (Zero-downtime DNS migration):**

```
За неделю до миграции:
1. Снизить TTL с 86400 до 300

За несколько минут до миграции:
2. Убедиться что все кэши обновились (ждём старый TTL)
3. Поменять A-запись на новый IP

После миграции:
4. Убедиться что всё работает
5. Вернуть TTL до 3600+
```

> Если поменять IP не снизив TTL заранее — часть пользователей будет идти на старый сервер ещё сутки (пока кэш не протухнет).

**DNS propagation — что это и почему это миф:**
"DNS propagation" технически означает лишь ожидание истечения TTL. Нет никакого "глобального распространения" — просто у разных резолверов разные моменты кэширования. Реальное время = TTL старой записи.

---

## SSL/TLS и безопасность

### 19. Что такое SNI и зачем он нужен?

**SNI (Server Name Indication)** — расширение TLS, которое позволяет клиенту указать **имя хоста** в процессе TLS handshake, ещё до того как зашифрованный канал установлен.

**Проблема без SNI:**
На одном IP-адресе может быть несколько доменов (virtual hosting). Nginx должен выбрать правильный сертификат ещё до чтения HTTP-заголовка `Host` — потому что заголовок уже в зашифрованном теле запроса. Без SNI сервер не знает какой сертификат подавать.

**С SNI:**
Клиент в `ClientHello` (ещё в открытом виде, до шифрования) добавляет:
```
Extension: server_name
  Type: host_name
  Name: api.example.com
```
Сервер видит имя хоста и выбирает правильный сертификат.

```nginx
# Nginx автоматически использует SNI
# Разные сертификаты для разных доменов на одном IP:
server {
    listen 443 ssl;
    server_name example.com;
    ssl_certificate /etc/ssl/example.com.crt;
}

server {
    listen 443 ssl;
    server_name api.example.com;
    ssl_certificate /etc/ssl/api.example.com.crt;
}
```

**Проверить SNI поддержку:**
```bash
# Запрос с указанием SNI
openssl s_client -connect 93.184.216.34:443 -servername api.example.com

# Без SNI (сервер отдаст дефолтный сертификат)
openssl s_client -connect 93.184.216.34:443 -noservername
```

**Практическое следствие для DevOps:** SNI означает что можно хостить множество HTTPS-сайтов на одном IP — это нормальная практика. Проблема возникает только со старыми клиентами (Windows XP / IE 6 — не поддерживают SNI), но в 2024-м это не актуально.

---

### 20. Как работает Let's Encrypt и протокол ACME?

**Let's Encrypt** — бесплатный Certificate Authority (CA), автоматизирующий выпуск и обновление DV (Domain Validation) SSL-сертификатов через протокол **ACME (Automatic Certificate Management Environment)**.

**Как ACME подтверждает владение доменом:**

**HTTP-01 challenge:**
```
1. ACME-клиент (certbot) запрашивает сертификат для example.com
2. Let's Encrypt генерирует token: "abc123xyz"
3. certbot создаёт файл: /.well-known/acme-challenge/abc123xyz
   с содержимым "abc123xyz.fingerprint_of_your_key"
4. Let's Encrypt делает HTTP GET: http://example.com/.well-known/acme-challenge/abc123xyz
5. Если контент совпадает — домен подтверждён, сертификат выпускается
```

**DNS-01 challenge** (нужен для wildcard-сертификатов `*.example.com`):
```
1. certbot запрашивает wildcard для *.example.com
2. Let's Encrypt: "создай TXT-запись _acme-challenge.example.com = token"
3. certbot добавляет TXT через DNS API провайдера
4. Let's Encrypt проверяет DNS TXT-запись
5. Сертификат выпускается
```

**Certbot — самый популярный ACME-клиент:**
```bash
# Установка
apt install certbot python3-certbot-nginx

# Выпустить и настроить автоматически (изменит nginx конфиг)
certbot --nginx -d example.com -d www.example.com

# Только выпустить (без изменения конфига)
certbot certonly --nginx -d example.com

# Wildcard через DNS (нужен плагин для провайдера)
certbot certonly --dns-cloudflare --dns-cloudflare-credentials ~/.secrets/cloudflare.ini \
  -d example.com -d "*.example.com"

# Проверить автообновление (сертификаты валидны 90 дней, обновляются при <30 днях)
certbot renew --dry-run

# Статус сертификатов
certbot certificates
```

**Автообновление через systemd timer (или cron):**
```bash
# Certbot при установке добавляет systemd timer
systemctl status certbot.timer

# Или через cron
echo "0 3 * * * root certbot renew --quiet --deploy-hook 'systemctl reload nginx'" > /etc/cron.d/certbot
```

**Важные ограничения Let's Encrypt:**
- 50 сертификатов на домен в неделю
- 5 дублирующихся сертификатов в неделю
- Срок действия: 90 дней (намеренно — стимулирует автоматизацию)
- Только DV (Domain Validation) — OV и EV не выпускает

---

### 21. Что такое HTTP Security Headers и какие из них важнейшие?

HTTP Security Headers — заголовки в HTTP-ответе, которые говорят браузеру как защищать пользователей. Настраиваются на уровне web-сервера.

**HSTS (HTTP Strict Transport Security):**
```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```
Браузер запомнит: для этого домена всегда использовать HTTPS. Даже если пользователь наберёт `http://` — браузер сам перенаправит на HTTPS, без обращения к серверу. Защищает от SSL-stripping атак.

**CSP (Content Security Policy):**
```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self' https://cdn.example.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; frame-ancestors 'none'" always;
```
Указывает браузеру откуда разрешено загружать ресурсы (JS, CSS, images, fonts). Основная защита от XSS-атак. Сложно настроить для legacy-приложений.

**X-Frame-Options** (старый, замещён CSP `frame-ancestors`):
```nginx
add_header X-Frame-Options "DENY" always;
# или SAMEORIGIN — разрешить только с того же домена
```
Защита от Clickjacking — запрет встраивать страницу в `<iframe>`.

**X-Content-Type-Options:**
```nginx
add_header X-Content-Type-Options "nosniff" always;
```
Запрет браузеру угадывать Content-Type (MIME sniffing). Если сервер отдаёт `text/plain` — браузер не будет исполнять это как JS.

**Referrer-Policy:**
```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```
Контролирует какой URL передаётся в заголовке `Referer` при переходах. Защищает от утечки приватных URL.

**Permissions-Policy (бывший Feature-Policy):**
```nginx
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```
Запрещает использование браузерных API (геолокация, камера, микрофон).

**Пример итоговой конфигурации Nginx:**
```nginx
# Добавить в http {} или server {} блок
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

**Проверить заголовки:**
```bash
curl -I https://example.com
# или онлайн: https://securityheaders.com
```

---

### 22. Что такое DDoS и каковы основные методы защиты?

**DDoS (Distributed Denial of Service)** — атака с тысяч/миллионов источников, цель которой — исчерпать ресурсы цели (полосу пропускания, CPU, соединения, память) и сделать её недоступной.

**Типы DDoS-атак:**

**Volumetric (объёмные):** цель — забить канал трафиком.
- UDP Flood, ICMP Flood, DNS Amplification (атакующий шлёт маленький запрос, ответ огромный)
- Измеряется в Gbps. Защита только у upstream-провайдера или специализированных сервисов (Cloudflare, Akamai).

**Protocol attacks:** эксплуатируют особенности протоколов.
- SYN Flood: отправляет тысячи SYN без завершения handshake, переполняя таблицу half-open соединений.
- Измеряется в pps (packets per second).

**Application Layer (L7):** атаки на уровне приложения.
- HTTP Flood: легитимные HTTP GET/POST запросы в огромном количестве.
- Slowloris: открывает много соединений, передаёт данные очень медленно, исчерпывая worker'ов.
- Труднее обнаружить (запросы выглядят как легитимные).

**Защита:**

**1. Rate Limiting (уже разобрали в вопросе 11)**

**2. Защита от SYN Flood:**
```bash
# SYN Cookies — ядро отвечает на SYN без сохранения состояния
sysctl net.ipv4.tcp_syncookies=1

# Уменьшить очередь half-open соединений
sysctl net.ipv4.tcp_max_syn_backlog=4096
```

**3. Защита от Slowloris в Nginx:**
```nginx
client_body_timeout    10s;   # таймаут на чтение тела запроса
client_header_timeout  10s;   # таймаут на чтение заголовков
keepalive_timeout      10s;   # время keep-alive соединения
send_timeout           10s;   # таймаут на отправку ответа
```

**4. Geo-блокировка (Nginx GeoIP модуль):**
```nginx
geoip2 /etc/nginx/GeoLite2-Country.mmdb {
    $geoip2_data_country_code country iso_code;
}

map $geoip2_data_country_code $allowed_country {
    default yes;
    CN  no;   # блокировать Китай
    RU  no;
}

server {
    if ($allowed_country = no) {
        return 403;
    }
}
```

**5. CDN с DDoS-защитой (Cloudflare, AWS Shield):**
- Поглощают volumetric атаки на уровне сети провайдера
- Имеют WAF (Web Application Firewall) для L7 атак
- Скрывают реальный IP сервера (origin IP должен быть закрыт — только Cloudflare IP в firewall)

**6. Connection limits:**
```nginx
limit_conn_zone $binary_remote_addr zone=addr:10m;
limit_conn addr 20;  # максимум 20 одновременных соединений с одного IP
```

**7. Мониторинг и алертинг:**
```bash
# Смотреть количество SYN_RECV (признак SYN flood)
ss -s | grep SYN

# Топ IP по соединениям в реальном времени
watch -n 2 'ss -tn state established | awk '\''{print $5}'\'' | cut -d: -f1 | sort | uniq -c | sort -rn | head -20'

# Мониторить rate запросов в Nginx-логах
tail -f /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -5
```
