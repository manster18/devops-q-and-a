# Базы данных — Вопросы и ответы для собеседований (Middle/Senior DevOps)

## Содержание

### Фундаментальные концепции
1. [Что такое ACID? Разберём каждое свойство.](#1-что-такое-acid-разберём-каждое-свойство)
2. [Что такое CAP теорема? Как она применяется на практике?](#2-что-такое-cap-теорема-как-она-применяется-на-практике)
3. [SQL vs NoSQL — когда что выбирать?](#3-sql-vs-nosql--когда-что-выбирать)

### SQL и реляционные БД
4. [Как работают индексы? Типы индексов в PostgreSQL.](#4-как-работают-индексы-типы-индексов-в-postgresql)
5. [Что такое уровни изоляции транзакций?](#5-что-такое-уровни-изоляции-транзакций)
6. [Как анализировать и оптимизировать медленные запросы?](#6-как-анализировать-и-оптимизировать-медленные-запросы)
7. [Что такое шардирование и партиционирование? В чём разница?](#7-что-такое-шардирование-и-партиционирование-в-чём-разница)

### HA и репликация
8. [Как работает репликация в PostgreSQL?](#8-как-работает-репликация-в-postgresql)
9. [Что такое connection pooling и зачем он нужен? PgBouncer.](#9-что-такое-connection-pooling-и-зачем-он-нужен-pgbouncer)
10. [Как организовать High Availability для PostgreSQL? Patroni.](#10-как-организовать-high-availability-для-postgresql-patroni)
11. [Что такое RPO и RTO? Стратегии резервного копирования.](#11-что-такое-rpo-и-rto-стратегии-резервного-копирования)

### Redis
12. [Как работает Redis? Структуры данных и персистентность.](#12-как-работает-redis-структуры-данных-и-персистентность)
13. [Redis Sentinel vs Redis Cluster — в чём разница?](#13-redis-sentinel-vs-redis-cluster--в-чём-разница)
14. [Паттерны использования Redis в production.](#14-паттерны-использования-redis-в-production)

### MongoDB
15. [Как работает репликация в MongoDB? Replica Set.](#15-как-работает-репликация-в-mongodb-replica-set)
16. [Как работает шардирование в MongoDB?](#16-как-работает-шардирование-в-mongodb)

### Elasticsearch
17. [Как устроен Elasticsearch? Индексы, шарды, реплики.](#17-как-устроен-elasticsearch-индексы-шарды-реплики)

### Эксплуатация и мониторинг
18. [Какие ключевые метрики нужно мониторить в PostgreSQL?](#18-какие-ключевые-метрики-нужно-мониторить-в-postgresql)
19. [Как работает VACUUM в PostgreSQL и зачем он нужен?](#19-как-работает-vacuum-в-postgresql-и-зачем-он-нужен)

---

## Фундаментальные концепции

### 1. Что такое ACID? Разберём каждое свойство.

**ACID** — набор свойств, гарантирующих надёжность транзакций в реляционных БД.

**A — Atomicity (Атомарность)**
Транзакция выполняется целиком или не выполняется совсем. Нет промежуточных состояний.

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 1000 WHERE id = 1;  -- списать
  UPDATE accounts SET balance = balance + 1000 WHERE id = 2;  -- зачислить
COMMIT;
-- Если второй UPDATE упадёт — первый тоже откатится. Деньги не исчезнут.
```

**C — Consistency (Согласованность)**
Транзакция переводит БД из одного консистентного состояния в другое. Все ограничения (CHECK, FK, UNIQUE) должны соблюдаться до и после.

```sql
-- Если balance < 0 нарушает CHECK constraint — транзакция откатится
ALTER TABLE accounts ADD CONSTRAINT positive_balance CHECK (balance >= 0);
```

**I — Isolation (Изолированность)**
Параллельные транзакции не видят промежуточных результатов друг друга. Степень изоляции настраивается (см. вопрос 5).

```sql
-- Transaction A                  -- Transaction B
BEGIN;                             BEGIN;
SELECT balance FROM accounts       -- видит 1000, а не 0 (пока A не COMMIT)
  WHERE id = 1;  -- = 1000
UPDATE accounts SET balance = 0
  WHERE id = 1;
                                   SELECT balance FROM accounts
                                     WHERE id = 1;  -- ?
COMMIT;
```

**D — Durability (Долговечность)**
После успешного COMMIT данные сохранены навсегда, даже при сбое питания. Реализуется через WAL (Write-Ahead Log) — изменения сначала пишутся в журнал, потом на диск.

```
WAL (Write-Ahead Log):
  1. Записать изменение в WAL файл (fsync)
  2. Ответить клиенту "COMMIT OK"
  3. Позже асинхронно записать в основные файлы данных
  При сбое — воспроизвести WAL при следующем старте
```

**Важно для DevOps:** именно WAL используется для репликации PostgreSQL. Standby-сервер применяет WAL-записи от primary.

---

### 2. Что такое CAP теорема? Как она применяется на практике?

**CAP теорема** (Brewer, 2000) гласит: в распределённой системе одновременно можно гарантировать только два из трёх свойств:

**C — Consistency (Согласованность)**
Все узлы видят одинаковые данные в один момент времени. Любое чтение возвращает последнюю запись.

**A — Availability (Доступность)**
Каждый запрос получает ответ (не обязательно с последними данными).

**P — Partition Tolerance (Устойчивость к разделению)**
Система продолжает работать при потере/задержке сообщений между узлами.

**Ключевой момент:** P (partition tolerance) в реальных распределённых системах — не выбор, а требование. Сетевые разделения (partitions) неизбежны. Поэтому выбор реально стоит между **CP** и **AP**:

```
CP (Consistency + Partition Tolerance):
  При partition — стать недоступным, но не отдавать устаревшие данные.
  Примеры: HBase, MongoDB (при strict read concern), ZooKeeper, etcd
  Сценарий: финансовые транзакции, системы бронирования

AP (Availability + Partition Tolerance):
  При partition — отвечать, но данные могут быть устаревшими.
  Примеры: Cassandra, CouchDB, DynamoDB (eventual consistency)
  Сценарий: счётчики просмотров, корзина покупок, DNS

CA (Consistency + Availability) — только на одном узле (нет partition):
  Примеры: одиночный PostgreSQL, MySQL
```

**PACELC — уточнённая теорема** (практичнее CAP):
```
Если есть Partition — выбирай между A и C.
Else (без partition) — выбирай между Latency и Consistency.

DynamoDB: PA/EL (availability при partition, low latency в норме)
PostgreSQL: PC/EC (consistency при partition, consistency в норме)
```

**Eventual Consistency — что это значит на практике:**
```
Cassandra, write на Node 1:
  t=0: Node1=v2, Node2=v1, Node3=v1  ← inconsistent
  t=1: Node1=v2, Node2=v2, Node3=v1  ← propagating
  t=2: Node1=v2, Node2=v2, Node3=v2  ← eventually consistent

Читая в момент t=1 с Node3 — получишь устаревшее v1.
```

---

### 3. SQL vs NoSQL — когда что выбирать?

**Реляционные БД (SQL):** PostgreSQL, MySQL, Oracle, SQL Server

Сильные стороны:
- ACID гарантии, сложные транзакции
- Мощный язык запросов (JOIN, агрегации, оконные функции)
- Строгая схема (schema enforcement)
- Зрелая экосистема, большой опыт у команд

**Когда выбирать SQL:**
- Финансовые системы, транзакции, инвентарь
- Сложные связи между сущностями
- Запросы заранее неизвестны (ad-hoc analytics)
- GDPR/compliance требует строгой целостности

---

**Документные (Document):** MongoDB, CouchDB, Firestore

```json
{
  "_id": "user_123",
  "name": "Alice",
  "address": {
    "city": "Berlin",
    "zip": "10115"
  },
  "orders": [
    {"id": "ord_1", "total": 99.99},
    {"id": "ord_2", "total": 149.00}
  ]
}
```
Плюсы: гибкая схема (schema-less), хорошо для иерархических данных, горизонтальное масштабирование.
Применение: CMS, каталоги товаров, профили пользователей.

---

**Key-Value:** Redis, DynamoDB, Riak

Простейшая структура: key → value. Максимальная производительность.
Применение: кэш, сессии, счётчики, pub/sub, очереди задач.

---

**Wide-Column (Columnar):** Cassandra, HBase, Bigtable

Оптимизированы под write-heavy нагрузку, временные ряды. Данные хранятся по колонкам.
Применение: IoT метрики, логи, time-series данные, рекомендательные системы.

---

**Graph:** Neo4j, Amazon Neptune, JanusGraph

Узлы и рёбра. Эффективны для связанных данных.
Применение: социальные сети, системы рекомендаций, fraud detection, knowledge graphs.

---

**Search:** Elasticsearch, OpenSearch, Solr

Полнотекстовый поиск с релевантностью, агрегации.
Применение: поиск по сайту, аналитика логов (ELK стек), APM.

---

**Правило выбора:**
```
1. Нужны ли сложные транзакции (деньги, заказы)?       → PostgreSQL
2. Нужен ли быстрый кэш / сессии?                      → Redis
3. Иерархические документы, гибкая схема?               → MongoDB
4. Огромный write-throughput, time-series?              → Cassandra / InfluxDB
5. Полнотекстовый поиск?                               → Elasticsearch
6. Графовые связи?                                      → Neo4j
7. Не знаешь? → Начни с PostgreSQL. Он умеет почти всё.
```

---

## SQL и реляционные БД

### 4. Как работают индексы? Типы индексов в PostgreSQL.

**Индекс** — структура данных, ускоряющая поиск строк. Без индекса — sequential scan (перебор всей таблицы). С индексом — index scan (поиск по дереву/хешу).

**B-Tree (по умолчанию)** — сбалансированное дерево. Поиск O(log n). Работает с `=`, `<`, `>`, `BETWEEN`, `LIKE 'prefix%'`, `ORDER BY`:

```sql
-- Обычный индекс
CREATE INDEX idx_users_email ON users(email);

-- Уникальный индекс
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Составной индекс (порядок столбцов важен!)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);
-- Ускорит: WHERE user_id = 5 ORDER BY created_at DESC
-- Ускорит: WHERE user_id = 5 AND created_at > '2024-01-01'
-- НЕ ускорит: WHERE created_at > '2024-01-01' (нет user_id в начале)

-- Partial index — индексировать только подмножество строк
CREATE INDEX idx_active_users ON users(email) WHERE deleted_at IS NULL;
-- Маленький, быстрый, для запросов WHERE deleted_at IS NULL

-- Expression index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
-- Ускорит: WHERE LOWER(email) = 'alice@example.com'
```

**Hash Index** — только для `=`. Быстрее B-Tree для точного совпадения, но не поддерживает диапазоны:
```sql
CREATE INDEX idx_sessions_token ON sessions USING HASH (token);
```

**GIN (Generalized Inverted Index)** — для составных типов: массивы, JSONB, полнотекстовый поиск:
```sql
-- Индекс для JSONB
CREATE INDEX idx_products_attrs ON products USING GIN(attributes);
-- Ускорит: WHERE attributes @> '{"color": "red"}'

-- Полнотекстовый поиск
CREATE INDEX idx_articles_fts ON articles USING GIN(to_tsvector('english', title || ' ' || body));
-- Ускорит: WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('postgresql')
```

**GiST (Generalized Search Tree)** — геометрические типы, range типы, полнотекстовый поиск:
```sql
-- Геопространственные запросы (PostGIS)
CREATE INDEX idx_locations_coords ON locations USING GIST(coordinates);
-- Ускорит: WHERE coordinates && ST_MakeEnvelope(...)

-- Range типы
CREATE INDEX idx_bookings_period ON bookings USING GIST(period);
-- Ускорит: WHERE period && '[2024-01-01, 2024-01-31]'::daterange
```

**BRIN (Block Range Index)** — для очень больших таблиц с коррелированными данными (временные метки в append-only таблицах):
```sql
-- Очень маленький индекс для таблицы логов
CREATE INDEX idx_logs_created ON logs USING BRIN(created_at);
-- Работает когда данные физически отсортированы по created_at (append-only)
```

**Когда индексы НЕ помогают:**
```sql
-- Функция на индексированной колонке — индекс не используется!
WHERE UPPER(email) = 'ALICE@EXAMPLE.COM'  -- нет индекса на UPPER(email)

-- Неселективное условие (много строк)
WHERE status = 'active'  -- если 90% строк active — seq scan быстрее

-- Маленькие таблицы — seq scan быстрее из-за overhead индекса
```

---

### 5. Что такое уровни изоляции транзакций?

Уровни изоляции контролируют какие аномалии могут возникать при параллельных транзакциях.

**Аномалии параллельного доступа:**

| Аномалия | Описание |
|---|---|
| **Dirty Read** | Читать незакоммиченные изменения другой транзакции |
| **Non-Repeatable Read** | Повторное чтение той же строки возвращает разные данные (другая транзакция изменила и закоммитила) |
| **Phantom Read** | Повторный запрос с условием возвращает другое число строк (другая транзакция добавила/удалила строки) |
| **Serialization Anomaly** | Результат параллельных транзакций не эквивалентен никакому последовательному выполнению |

**Уровни изоляции SQL стандарта:**

| Уровень | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | Возможен | Возможен | Возможен |
| READ COMMITTED | Защищён | Возможен | Возможен |
| REPEATABLE READ | Защищён | Защищён | Возможен |
| SERIALIZABLE | Защищён | Защищён | Защищён |

**В PostgreSQL:**
- `READ UNCOMMITTED` = `READ COMMITTED` (грязное чтение не допускается)
- `REPEATABLE READ` защищает и от phantom reads (через MVCC snapshot)
- `SERIALIZABLE` использует SSI (Serializable Snapshot Isolation) — без блокировок

```sql
-- Установить уровень для транзакции
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
...
COMMIT;

-- Или сразу
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- По умолчанию для всех соединений
ALTER SYSTEM SET default_transaction_isolation = 'read committed';
```

**READ COMMITTED (default в PostgreSQL)** — каждое SELECT видит данные зафиксированные на момент начала этого SELECT:
```sql
-- T1                          -- T2
BEGIN;
SELECT balance FROM acc        -- = 1000
  WHERE id = 1;

                               BEGIN;
                               UPDATE acc SET balance = 500 WHERE id = 1;
                               COMMIT;

SELECT balance FROM acc        -- = 500 (!)  Non-Repeatable Read
  WHERE id = 1;
COMMIT;
```

**MVCC (Multi-Version Concurrency Control)** — PostgreSQL не блокирует строки для чтения. Вместо этого хранит несколько версий строки (xmin, xmax — номера транзакций). Читатели видят "snapshot" на момент начала своей транзакции.

---

### 6. Как анализировать и оптимизировать медленные запросы?

**Шаг 1 — найти медленные запросы:**
```sql
-- Включить логирование медленных запросов
-- postgresql.conf:
-- log_min_duration_statement = 1000  # логировать запросы > 1 сек

-- pg_stat_statements — статистика выполнения запросов
CREATE EXTENSION pg_stat_statements;

SELECT query,
       calls,
       total_exec_time / 1000 AS total_sec,
       mean_exec_time / 1000  AS mean_sec,
       rows,
       shared_blks_hit,       -- попадания в кэш
       shared_blks_read        -- чтения с диска
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

**Шаг 2 — EXPLAIN ANALYZE:**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2024-01-01'
  AND o.status = 'completed'
GROUP BY u.id, u.name
ORDER BY order_count DESC
LIMIT 10;
```

**Пример вывода EXPLAIN ANALYZE:**
```
Limit  (cost=12500.00..12500.03 rows=10) (actual time=250.123..250.130 rows=10)
  -> Sort  (cost=12500.00..12520.00 rows=8000) (actual time=250.122..250.125 rows=10)
     Sort Key: (count(o.id)) DESC
     Sort Method: top-N heapsort  Memory: 25kB
     -> HashAggregate  (actual time=245.000..248.000 rows=8000)
        -> Hash Join  (actual time=10.000..200.000 rows=50000)
           Hash Cond: (o.user_id = u.id)
           -> Seq Scan on orders o  (actual time=0.050..80.000 rows=500000)
                                     ^^^^ ПРОБЛЕМА: Seq Scan на большой таблице
               Filter: (status = 'completed')
               Rows Removed by Filter: 200000
           -> Hash  (actual time=5.000..5.000 rows=5000)
               -> Index Scan using idx_users_created on users u
                  Index Cond: (created_at > '2024-01-01')
Planning Time: 2.500 ms
Execution Time: 250.500 ms
```

**Что смотреть в EXPLAIN:**
```
actual time    — реальное время выполнения узла
rows           — сколько строк вернул узел
Seq Scan       — последовательное сканирование (плохо для больших таблиц!)
Index Scan     — поиск по индексу (хорошо)
Bitmap Heap Scan — сначала bitmap индекса, потом чтение страниц (хорошо для range)
Hash Join      — JOIN через хеш-таблицу
Nested Loop    — вложенный цикл JOIN (хорошо для маленьких наборов)
Merge Join     — объединение отсортированных данных
```

**Исправление из примера выше:**
```sql
-- Добавить индекс на (user_id, status) для таблицы orders
CREATE INDEX idx_orders_user_status ON orders(user_id, status)
  WHERE status = 'completed';  -- partial index

-- Или составной
CREATE INDEX idx_orders_status_user ON orders(status, user_id)
  INCLUDE (id);  -- covering index — включить id чтобы не читать heap
```

**Другие инструменты диагностики:**
```sql
-- Блокировки (кто кого блокирует)
SELECT
  blocking.pid AS blocking_pid,
  blocking.query AS blocking_query,
  blocked.pid AS blocked_pid,
  blocked.query AS blocked_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocking.pid IS NOT NULL;

-- Долгие транзакции (опасны: держат блокировки и мешают VACUUM)
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - query_start > interval '5 minutes'
ORDER BY duration DESC;

-- Размер таблиц и индексов
SELECT
  relname,
  pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
  pg_size_pretty(pg_relation_size(relid)) AS table_size,
  pg_size_pretty(pg_indexes_size(relid)) AS indexes_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Неиспользуемые индексы (кандидаты на удаление)
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%pkey%'
ORDER BY tablename;
```

---

### 7. Что такое шардирование и партиционирование? В чём разница?

**Партиционирование (Partitioning)** — разбиение одной большой таблицы на части (партиции) в рамках одного экземпляра БД. Для приложения таблица выглядит как одна.

```sql
-- Партиционирование по диапазону (range) — чаще всего по дате
CREATE TABLE orders (
    id         BIGINT,
    user_id    INT,
    created_at TIMESTAMPTZ,
    total      NUMERIC(10,2)
) PARTITION BY RANGE (created_at);

-- Создать партиции
CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Индексы на партициях (или на родительской таблице — применяется ко всем)
CREATE INDEX ON orders(user_id, created_at);

-- Запрос автоматически идёт только в нужную партицию (partition pruning)
SELECT * FROM orders WHERE created_at >= '2024-04-01' AND created_at < '2024-07-01';
-- EXPLAIN покажет: Seq Scan on orders_2024_q2 (остальные исключены)
```

```sql
-- Партиционирование по hash (равномерное распределение)
CREATE TABLE users (id BIGINT, email TEXT)
PARTITION BY HASH (id);

CREATE TABLE users_0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE users_2 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE users_3 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

**Преимущества партиционирования:**
- Быстрое удаление старых данных: `DROP TABLE orders_2022_q1` (мгновенно, vs часы DELETE)
- Partition pruning: запросы с условием на ключ затрагивают только нужные партиции
- Parallel query: PostgreSQL может сканировать партиции параллельно

---

**Шардирование (Sharding)** — горизонтальное масштабирование: данные распределяются по нескольким независимым экземплярам БД (шардам). Каждый шард — отдельный сервер.

```
Shard 1 (PostgreSQL node 1): user_id 1–1,000,000
Shard 2 (PostgreSQL node 2): user_id 1,000,001–2,000,000
Shard 3 (PostgreSQL node 3): user_id 2,000,001–3,000,000

Application / Shard Router → знает на каком шарде искать данные
```

**Сложности шардирования:**
- Cross-shard JOINы — очень медленные или невозможны
- Cross-shard транзакции — требуют 2PC (Two-Phase Commit)
- Resharding (добавить новый шард) — очень сложно
- Выбор shard key критически важен (hotspot problem)

**Инструменты шардирования для PostgreSQL:**
- **Citus** (теперь часть Azure) — превращает PG в distributed database
- **Vitess** — шардирование MySQL (используется YouTube, GitHub)
- **Application-level sharding** — в коде приложения

**Правило:** сначала партиционирование + read replicas. Шардирование — только когда другие методы исчерпаны.

---

## HA и репликация

### 8. Как работает репликация в PostgreSQL?

**Streaming Replication** — основной механизм. Standby подключается к primary и получает WAL-записи в режиме реального времени:

```
Primary                            Standby
  │                                   │
  │ WAL Writer                        │
  │ (пишет изменения в WAL)           │
  │                                   │
  │←──── WAL Sender ─────────────────→│
  │      (стримит WAL записи)         │ WAL Receiver
  │                                   │ (применяет WAL)
  │                                   │
  │      Replication Slot             │
  │      (хранит LSN для standby)     │
```

**Конфигурация PostgreSQL streaming replication:**

```ini
# postgresql.conf (Primary)
wal_level = replica              # минимум для репликации
max_wal_senders = 5              # максимум одновременных standby
wal_keep_size = 1GB              # держать WAL для отстающих standbys

# Synchronous replication (гарантирует нулевую потерю данных)
synchronous_standby_names = 'ANY 1 (standby1, standby2)'
# ANY 1 — достаточно подтверждения от любого одного standby
# FIRST 1 (s1, s2) — ждать именно s1 (s2 — запасной)
```

```ini
# pg_hba.conf (Primary) — разрешить репликацию
host  replication  replicator  10.0.0.0/8  scram-sha-256
```

```bash
# Создать пользователя для репликации
createuser --replication --pwprompt replicator

# Создать базовый бэкап на standby (начальная синхронизация)
pg_basebackup -h primary-host -U replicator -D /var/lib/postgresql/data \
  --wal-method=stream --checkpoint=fast --progress
```

```ini
# postgresql.conf (Standby)
primary_conninfo = 'host=primary-host user=replicator password=xxx'
hot_standby = on   # разрешить SELECT запросы на standby
```

**Типы репликации:**

| | Synchronous | Asynchronous |
|---|---|---|
| RPO | 0 (нет потери данных) | Небольшой (размер отставания) |
| Latency | +RTT к каждому write | Без overhead |
| Доступность | Если standby упал — primary зависает | Primary работает независимо |

**Replication Lag — мониторинг отставания:**
```sql
-- На Primary
SELECT
  application_name,
  state,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn,
  pg_size_pretty(sent_lsn - replay_lsn) AS replication_lag_bytes,
  write_lag,
  flush_lag,
  replay_lag
FROM pg_stat_replication;

-- На Standby
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;
```

**Replication Slots** — гарантируют что Primary не удалит WAL пока Standby его не получил. Опасно: если Standby надолго отвалится — WAL на Primary будет расти бесконечно:
```sql
-- Создать slot
SELECT pg_create_physical_replication_slot('standby1_slot');

-- Мониторинг: смотреть на pg_replication_slots
SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
FROM pg_replication_slots;

-- Удалить незаполненный slot (если standby умер навсегда)
SELECT pg_drop_replication_slot('standby1_slot');
```

---

### 9. Что такое connection pooling и зачем он нужен? PgBouncer.

**Проблема:** PostgreSQL создаёт отдельный процесс (~5-10 МБ RAM) для каждого соединения. При 1000 соединениях — 5-10 ГБ только на процессы соединений, плюс огромный overhead на context switching.

**Рекомендуемый max_connections:** обычно 100-300 в зависимости от RAM.

**Connection Pooler** принимает тысячи соединений от клиентов, но держит небольшой пул реальных соединений к PostgreSQL:

```
1000 app connections → [PgBouncer] → 50 PostgreSQL connections
```

**PgBouncer** — самый популярный пулер для PostgreSQL:

```ini
# pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 5432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Режим пулинга (критически важно!)
pool_mode = transaction   # рекомендуется для большинства случаев

# Размер пула
default_pool_size = 25    # соединений к PostgreSQL на пользователя/БД
max_client_conn = 1000    # максимум клиентских соединений
reserve_pool_size = 5     # запасные для быстрого роста

# Таймауты
server_idle_timeout = 600
client_idle_timeout = 0   # не отключать idle клиентов
```

**Режимы пулинга:**

| Режим | Когда соединение "освобождается" | Совместимость |
|---|---|---|
| **Session** | При закрытии клиентского соединения | Полная (prepared statements, SET, временные таблицы работают) |
| **Transaction** | После COMMIT/ROLLBACK | Высокая, но нельзя prepared statements без `DEALLOCATE` |
| **Statement** | После каждого запроса | Только autocommit, без транзакций |

**Transaction mode** — золотой стандарт: каждая транзакция получает соединение из пула, по завершении возвращает. При 1000 клиентах нужно ровно столько соединений, сколько параллельных транзакций.

**Ограничения transaction mode:**
```sql
-- НЕ РАБОТАЮТ в transaction mode:
SET search_path = myschema;  -- настройка сбрасывается после транзакции
PREPARE my_plan AS SELECT...  -- prepared statements нужно объявлять через protocol

-- РАБОТАЕТ через pgbouncer с prepared statements (pgbouncer 1.21+)
-- или через параметр server_reset_query
```

**Мониторинг PgBouncer:**
```bash
# Подключиться к admin консоли
psql -h 127.0.0.1 -p 5432 -U pgbouncer pgbouncer

SHOW POOLS;    -- состояние пулов
SHOW CLIENTS;  -- клиентские соединения
SHOW SERVERS;  -- соединения к PostgreSQL
SHOW STATS;    -- статистика запросов
SHOW CONFIG;   -- текущая конфигурация

RELOAD;        -- перечитать конфиг
PAUSE;         -- приостановить (для maintenance)
RESUME;        -- возобновить
```

---

### 10. Как организовать High Availability для PostgreSQL? Patroni.

**Patroni** — наиболее распространённое решение для автоматического failover PostgreSQL. Использует DCS (Distributed Configuration Store) как арбитр — etcd, ZooKeeper, или Consul.

**Архитектура:**
```
           ┌──────────┐
           │   etcd   │ ← хранит информацию о лидере + distributed lock
           │ cluster  │
           └────┬─────┘
                │ Patroni APIs
    ┌───────────┼───────────┐
    ▼           ▼           ▼
┌────────┐ ┌────────┐ ┌────────┐
│Patroni │ │Patroni │ │Patroni │
│  node1 │ │  node2 │ │  node3 │
│(leader)│ │(standby│ │(standby│
│  PG    │ │   PG   │ │   PG   │
└───┬────┘ └───┬────┘ └───┬────┘
    │           │           │
    └─────streaming replication─────┘

HAProxy / VIP ─→ смотрит на Patroni REST API → всегда на текущий Primary
```

**Конфигурация Patroni:**
```yaml
# /etc/patroni/patroni.yml
scope: postgres-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.0.1:8008

etcd3:
  hosts: 10.0.0.10:2379,10.0.0.11:2379,10.0.0.12:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576   # 1 МБ — не избирать сильно отстающие standbys
    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 200
        shared_buffers: 4GB
        wal_level: replica
        hot_standby: on
        wal_log_hints: on   # нужен для pg_rewind

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.0.1:5432
  data_dir: /var/lib/postgresql/data
  pg_hba:
  - host replication replicator 10.0.0.0/8 scram-sha-256
  - host all all 0.0.0.0/0 scram-sha-256
  authentication:
    replication:
      username: replicator
      password: "secret"
    superuser:
      username: postgres
      password: "secret"
```

**Команды управления Patroni:**
```bash
# Статус кластера
patronictl -c /etc/patroni/patroni.yml list

# Пример вывода:
# + Cluster: postgres-cluster --------+----+-----------+
# | Member | Host       | Role    | State  | TL | Lag in MB |
# +--------+------------+---------+--------+----+-----------+
# | node1  | 10.0.0.1   | Leader  | running|  1 |         0 |
# | node2  | 10.0.0.2   | Replica | running|  1 |         0 |
# | node3  | 10.0.0.3   | Replica | running|  1 |         0 |

# Ручное переключение (planned failover)
patronictl -c /etc/patroni/patroni.yml switchover postgres-cluster

# Рестарт PostgreSQL на ноде
patronictl -c /etc/patroni/patroni.yml restart postgres-cluster node1

# Перечитать конфиг
patronictl -c /etc/patroni/patroni.yml reload postgres-cluster

# Редактировать DCS конфиг (без перезапуска)
patronictl -c /etc/patroni/patroni.yml edit-config postgres-cluster
```

**HAProxy для роутинга клиентов:**
```ini
# haproxy.cfg
frontend postgres_primary
  bind *:5432
  default_backend postgres_primary_backend

backend postgres_primary_backend
  option httpchk GET /primary    # Patroni REST API
  server node1 10.0.0.1:5432 check port 8008
  server node2 10.0.0.2:5432 check port 8008 backup
  server node3 10.0.0.3:5432 check port 8008 backup

frontend postgres_replica
  bind *:5433
  default_backend postgres_replica_backend

backend postgres_replica_backend
  option httpchk GET /replica    # только standbys
  server node1 10.0.0.1:5432 check port 8008
  server node2 10.0.0.2:5432 check port 8008
  server node3 10.0.0.3:5432 check port 8008
```

**Процесс failover (автоматический):**
```
1. Primary недоступен > TTL (30 сек по умолчанию)
2. Patroni агенты на standbys замечают потерю etcd lease
3. Кандидаты пытаются получить etcd lock (race)
4. Победитель (с наименьшим lag) становится новым Primary
5. pg_rewind синхронизирует старый primary при восстановлении
6. HAProxy health check обнаруживает нового Primary
7. Клиентские соединения переходят на новый Primary
Общее время failover: ~30-60 секунд
```

---

### 11. Что такое RPO и RTO? Стратегии резервного копирования.

**RPO (Recovery Point Objective)** — максимально допустимая потеря данных. "На сколько назад можно откатиться?"

**RTO (Recovery Time Objective)** — максимально допустимое время восстановления. "Как долго можно быть недоступным?"

```
RPO = 1 час означает:
  Делаем бэкапы каждый час.
  При аварии теряем максимум час данных.

RTO = 4 часа означает:
  Система должна быть восстановлена в течение 4 часов.
```

**Стратегии бэкапа PostgreSQL:**

**pg_dump — логический бэкап:**
```bash
# Дамп одной БД в custom формат (сжатый, поддерживает параллельное восстановление)
pg_dump -Fc -d mydb -f /backup/mydb_$(date +%Y%m%d_%H%M).dump

# Дамп всего кластера (роли, tablespaces)
pg_dumpall -f /backup/cluster_$(date +%Y%m%d).sql

# Восстановление
pg_restore -d mydb -j 4 /backup/mydb_20240115.dump  # -j 4 = параллельно

# Минусы:
# - Медленно на больших БД
# - Consistent snapshot только в момент запуска
# - RPO = интервал между бэкапами
```

**pg_basebackup — физический бэкап:**
```bash
# Полный физический бэкап с WAL
pg_basebackup -h localhost -U replicator \
  -D /backup/base_$(date +%Y%m%d) \
  --wal-method=stream \
  --checkpoint=fast \
  --compress=lz4 \
  --progress

# Быстрое восстановление — просто копируем файлы обратно
# Минусы: большой размер, нужна совместимость версий PG
```

**PITR (Point-In-Time Recovery) — непрерывный архив WAL:**
```
Схема:
  pg_basebackup (раз в сутки) + непрерывная архивация WAL файлов
  → можно восстановить на любой момент времени

postgresql.conf:
  archive_mode = on
  archive_command = 'aws s3 cp %p s3://my-wal-archive/%f'
  # %p = путь к WAL файлу, %f = имя файла

Восстановление на конкретный момент:
  restore_command = 'aws s3 cp s3://my-wal-archive/%f %p'
  recovery_target_time = '2024-01-15 14:30:00+00'
```

**pgBackRest — лучший инструмент для production бэкапов:**
```ini
# /etc/pgbackrest/pgbackrest.conf
[global]
repo1-path=/backup/pgbackrest
repo1-s3-bucket=my-postgres-backup
repo1-s3-region=eu-west-1
repo1-type=s3
repo1-retention-full=7    # хранить 7 полных бэкапов
repo1-retention-diff=14   # хранить 14 дифференциальных

[mydb]
pg1-host=db-primary
pg1-path=/var/lib/postgresql/data
```

```bash
# Полный бэкап
pgbackrest --stanza=mydb --type=full backup

# Дифференциальный (от последнего full)
pgbackrest --stanza=mydb --type=diff backup

# Инкрементальный (от последнего any)
pgbackrest --stanza=mydb --type=incr backup

# Восстановление
pgbackrest --stanza=mydb --target="2024-01-15 14:30:00" \
  --target-action=promote restore

# Верификация бэкапа
pgbackrest --stanza=mydb verify

# Просмотр доступных бэкапов
pgbackrest --stanza=mydb info
```

**Типичная стратегия:**
```
RPO = 5 минут, RTO = 30 минут:
  - pgBackRest full backup: каждую ночь
  - pgBackRest differential: каждые 6 часов
  - WAL архивация в S3: непрерывно
  - Patroni streaming replication: standby всегда готов (RTO минуты)
  - Periodic restore test: автоматически восстанавливать и проверять каждую неделю!
```

---

## Redis

### 12. Как работает Redis? Структуры данных и персистентность.

**Redis** — in-memory data structure store. Всё хранится в RAM → микросекундные latency. Однопоточный (event loop) → нет проблем с конкурентностью.

**Основные структуры данных:**

```bash
# String — универсальный тип. Значение до 512 МБ.
SET user:1:name "Alice"
GET user:1:name
SET counter 0
INCR counter           # атомарный инкремент = 1
INCRBY counter 10      # = 11
SET cache:key value EX 3600  # с TTL 1 час

# List — двусвязный список. Очередь, стек.
RPUSH jobs "task1" "task2" "task3"  # добавить справа
LPOP jobs                           # взять слева (FIFO очередь)
LRANGE jobs 0 -1                    # все элементы
BLPOP jobs 30                       # blocking pop, ждать 30 сек

# Hash — словарь. Объекты, сессии.
HSET user:1 name "Alice" email "alice@example.com" age "30"
HGET user:1 email
HMGET user:1 name email             # несколько полей
HGETALL user:1                      # все поля
HINCRBY user:1 login_count 1        # атомарный инкремент поля

# Set — неупорядоченное множество без дубликатов.
SADD tags:post:1 "linux" "devops" "docker"
SMEMBERS tags:post:1
SISMEMBER tags:post:1 "docker"     # True/False
SINTERSTORE common_tags tags:post:1 tags:post:2  # пересечение

# Sorted Set (ZSet) — множество с числовыми score. Рейтинги, очереди с приоритетом.
ZADD leaderboard 1500 "alice" 2200 "bob" 1800 "charlie"
ZRANGE leaderboard 0 -1 WITHSCORES REV  # топ по убыванию
ZRANGEBYSCORE leaderboard 1500 2000     # в диапазоне score
ZINCRBY leaderboard 100 "alice"         # увеличить score

# Stream — append-only лог (аналог Kafka).
XADD events * action "login" user_id "42"  # добавить запись (auto ID)
XRANGE events - +                          # все записи
XREAD COUNT 10 STREAMS events 0            # читать с начала
XGROUP CREATE events consumer-group $ MKSTREAM  # создать consumer group
```

**Персистентность:**

**RDB (Redis Database Snapshot)** — периодические снапшоты на диск:
```
redis.conf:
  save 900 1      # если 1+ изменений за 15 минут — сохранить
  save 300 10     # если 10+ изменений за 5 минут
  save 60 10000   # если 10000+ изменений за 1 минуту
  dbfilename dump.rdb
  dir /var/lib/redis

Плюсы: компактный файл, быстрое восстановление
Минусы: потеря данных между снапшотами (RPO = интервал между save)
```

**AOF (Append Only File)** — логирование каждой write-операции:
```
redis.conf:
  appendonly yes
  appendfsync everysec  # fsync каждую секунду (баланс производительность/надёжность)
  # appendfsync always  # fsync после каждой команды (максимальная надёжность)
  # appendfsync no      # fsync на усмотрение ОС (максимальная производительность)

  auto-aof-rewrite-percentage 100  # переписать AOF когда вырастет в 2x
  auto-aof-rewrite-min-size 64mb

Плюсы: RPO ≤ 1 секунда, читаем формат
Минусы: большой файл, медленнее RDB при восстановлении
```

**RDB + AOF (рекомендуется для production):**
```
redis.conf:
  aof-use-rdb-preamble yes  # AOF начинается с RDB снапшота + delta AOF
  appendonly yes
```

---

### 13. Redis Sentinel vs Redis Cluster — в чём разница?

**Redis Sentinel** — решение HA для одного Redis dataset (master + replicas). Автоматический failover.

```
        ┌─────────┐ ┌─────────┐ ┌─────────┐
        │Sentinel1│ │Sentinel2│ │Sentinel3│  ← quorum мониторинг
        └────┬────┘ └────┬────┘ └────┬────┘
             │           │           │
        ┌────▼────┐  ┌───▼────┐  ┌──▼─────┐
        │ Master  │→→│Replica │→→│Replica │  ← репликация
        │ :6379   │  │ :6379  │  │ :6379  │
        └─────────┘  └────────┘  └────────┘
```

```
redis-sentinel.conf:
  sentinel monitor mymaster 10.0.0.1 6379 2  # quorum = 2 из 3 sentinels
  sentinel down-after-milliseconds mymaster 5000
  sentinel failover-timeout mymaster 60000
```

Клиент подключается к Sentinel, получает адрес текущего Master. При failover — Sentinel избирает новый Master из реплик.

**Минус:** один Master ограничивает пропускную способность записи и объём данных (ограничен RAM одного сервера).

---

**Redis Cluster** — горизонтальное масштабирование. Данные автоматически распределяются между нодами через hash slots:

```
16384 hash slots распределены между шардами:

Shard 1 (master+replica): slots 0–5460
Shard 2 (master+replica): slots 5461–10922
Shard 3 (master+replica): slots 10923–16383

Ключ → CRC16(key) % 16384 → номер slot → сервер
```

```bash
# Создать кластер из 6 нод (3 masters + 3 replicas)
redis-cli --cluster create \
  10.0.0.1:7000 10.0.0.2:7001 10.0.0.3:7002 \
  10.0.0.4:7003 10.0.0.5:7004 10.0.0.6:7005 \
  --cluster-replicas 1

# Статус кластера
redis-cli -c cluster info
redis-cli -c cluster nodes

# Добавить новый шард
redis-cli --cluster add-node 10.0.0.7:7006 10.0.0.1:7000

# Перебалансировать слоты
redis-cli --cluster rebalance 10.0.0.1:7000
```

**Ограничения Redis Cluster:**
- Multi-key команды работают только если все ключи в одном slot
- Hash tags для группировки: `{user:1}.name` и `{user:1}.email` → один slot
- Транзакции (MULTI/EXEC) только в пределах одного slot

| | Sentinel | Cluster |
|---|---|---|
| Масштабирование | Только replicas (чтение) | Горизонтальное (чтение+запись) |
| Объём данных | Ограничен RAM одного сервера | N × RAM |
| Сложность | Простой | Сложнее |
| Multi-key операции | Полные | Ограничены |
| Минимум нод | 1 master + 2 sentinels | 6 (3M + 3R) |

---

### 14. Паттерны использования Redis в production.

**Кэш (Cache-Aside / Lazy Loading):**
```python
def get_user(user_id: int) -> dict:
    cache_key = f"user:{user_id}"

    # 1. Проверить кэш
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)  # Cache HIT

    # 2. Cache MISS — читать из БД
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)

    # 3. Сохранить в кэш
    redis.setex(cache_key, 3600, json.dumps(user))  # TTL 1 час
    return user

def update_user(user_id: int, data: dict):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # инвалидировать кэш
```

**Сессии пользователей:**
```python
def create_session(user_id: int) -> str:
    session_id = secrets.token_urlsafe(32)
    session_data = {"user_id": user_id, "created_at": time.time()}
    redis.hset(f"session:{session_id}", mapping=session_data)
    redis.expire(f"session:{session_id}", 86400 * 7)  # 7 дней
    return session_id
```

**Distributed Lock (Redlock алгоритм):**
```python
# Только один экземпляр сервиса выполняет задачу
import redis.lock

r = redis.Redis()
with r.lock("payment_processor", timeout=30, blocking_timeout=10):
    # Критическая секция — обрабатываем платёж
    process_payment(payment_id)
# Блокировка автоматически снимается
```

**Rate Limiting (скользящее окно):**
```python
def is_rate_limited(user_id: int, limit: int = 100, window: int = 60) -> bool:
    key = f"ratelimit:{user_id}:{int(time.time() // window)}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, window * 2)
    return count > limit
```

**Pub/Sub:**
```python
# Publisher
redis.publish("notifications:user:42", json.dumps({
    "type": "order_shipped",
    "order_id": "ord_123"
}))

# Subscriber
pubsub = redis.pubsub()
pubsub.subscribe("notifications:user:42")
for message in pubsub.listen():
    if message['type'] == 'message':
        handle_notification(json.loads(message['data']))
```

---

## MongoDB

### 15. Как работает репликация в MongoDB? Replica Set.

**Replica Set** — группа из минимум 3 MongoDB узлов, один из которых Primary:

```
           ┌─────────────────────────────────┐
           │        Replica Set              │
           │                                 │
           │  ┌─────────┐    ┌─────────┐    │
           │  │Secondary│←───│ Primary │    │
           │  │  node2  │    │  node1  │    │
           │  └─────────┘    └────┬────┘    │
           │                      │oplog    │
           │  ┌─────────┐         │         │
           │  │Secondary│←────────┘         │
           │  │  node3  │                   │
           │  └─────────┘                   │
           └─────────────────────────────────┘
```

**Oplog (Operations Log)** — capped collection в `local` БД на каждом узле. Primary записывает все изменения в oplog, secondaries реплицируют из него.

**Выборы (Elections):** если Primary недоступен > `electionTimeoutMillis` (10 сек), secondaries проводят выборы. Победитель получает большинство голосов (quorum). Нечётное количество узлов обязательно!

**Инициализация Replica Set:**
```javascript
// На любом узле
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017", priority: 2 },
    { _id: 1, host: "mongo2:27017", priority: 1 },
    { _id: 2, host: "mongo3:27017", priority: 0, hidden: true, votes: 1 }
    // hidden = не получает трафик клиентов, votes = участвует в выборах
  ]
})

// Статус
rs.status()
rs.conf()
```

**Уровни гарантий записи (Write Concern):**
```javascript
db.orders.insertOne(
  { userId: 123, total: 99.99 },
  {
    writeConcern: {
      w: "majority",  // подтверждение от большинства узлов
      j: true,        // данные на диске (journal), не только в памяти
      wtimeout: 5000  // таймаут 5 сек
    }
  }
)
```

**Уровни гарантий чтения (Read Concern + Read Preference):**
```javascript
// Read Preference — откуда читать
db.orders.find({ userId: 123 }).readPref("secondaryPreferred")
// primary (default), primaryPreferred, secondary, secondaryPreferred, nearest

// Read Concern — какой уровень согласованности
db.orders.find().readConcern("majority")  // данные закоммичены на большинстве
// local (default), available, majority, linearizable, snapshot
```

---

### 16. Как работает шардирование в MongoDB?

**Компоненты шардированного кластера:**

```
Clients
   │
   ▼
mongos (query router, 2+)     ← маршрутизатор запросов
   │
   ├── Config Servers (3 ноды Replica Set)  ← метаданные о распределении
   │
   ├── Shard 1 (Replica Set: p+s+s)
   ├── Shard 2 (Replica Set: p+s+s)
   └── Shard 3 (Replica Set: p+s+s)
```

**Shard Key** — поле (или набор полей), по которому MongoDB распределяет данные. Выбор shard key критически важен.

```javascript
// Включить шардирование для БД
sh.enableSharding("mydb")

// Шардировать коллекцию по user_id (hashed — равномерное распределение)
sh.shardCollection("mydb.orders", { user_id: "hashed" })

// Range-based sharding (хорошо для диапазонных запросов, риск hotspot)
sh.shardCollection("mydb.events", { timestamp: 1, device_id: 1 })

// Статус шардирования
sh.status()
db.orders.getShardDistribution()
```

**Chunks** — единицы распределения данных (по умолчанию 128 МБ). Балансировщик перемещает chunks между шардами для равномерного распределения.

**Проблемы при неправильном выборе shard key:**

```
Hotspot (монотонный ключ): { timestamp: 1 }
  Все новые записи → последний шард → перегрузка одного шарда
  Решение: hashed sharding или составной ключ

Scatter-Gather (плохой ключ): запрос не содержит shard key
  mongos вынужден опросить ВСЕ шарды → медленно
  Решение: включать shard key в часто используемые запросы

Jumbo Chunks: много документов с одинаковым shard key
  Chunk не может быть разбит
  Решение: составной shard key (добавить поле с высокой кардинальностью)
```

---

## Elasticsearch

### 17. Как устроен Elasticsearch? Индексы, шарды, реплики.

**Elasticsearch** — распределённый поисковый движок на базе Apache Lucene. Хранит данные в JSON документах, обеспечивает полнотекстовый поиск и аналитику.

**Архитектура:**
```
ES Cluster
├── Node 1 (Master-eligible, Data)
│   ├── Shard 0 Primary (index: logs)
│   └── Shard 2 Replica (index: logs)
├── Node 2 (Master-eligible, Data)
│   ├── Shard 1 Primary (index: logs)
│   └── Shard 0 Replica (index: logs)
└── Node 3 (Master-eligible, Data)
    ├── Shard 2 Primary (index: logs)
    └── Shard 1 Replica (index: logs)
```

**Ключевые концепции:**

**Index** — логическое пространство для документов (аналог БД в SQL). Состоит из шардов.

**Shard** — единица Lucene индекса. Шард распределяется по нодам. Количество primary shards задаётся при создании индекса и не меняется (в отличие от реплик).

**Настройка индекса:**
```json
PUT /logs-2024-01
{
  "settings": {
    "number_of_shards": 3,       // первичные шарды (неизменно после создания!)
    "number_of_replicas": 1,     // реплик на каждый шард (можно менять)
    "refresh_interval": "5s",    // как часто делать документы видимыми для поиска
    "index.codec": "best_compression"
  },
  "mappings": {
    "properties": {
      "timestamp":  { "type": "date" },
      "level":      { "type": "keyword" },     // exact match, aggregations
      "message":    { "type": "text",          // full-text search
                      "analyzer": "standard" },
      "service":    { "type": "keyword" },
      "duration_ms":{ "type": "integer" }
    }
  }
}
```

**Типы полей:**
- `keyword` — точное совпадение, агрегации, сортировка
- `text` — полнотекстовый поиск (анализируется и токенизируется)
- `date`, `integer`, `float`, `boolean`, `geo_point`, `ip`

**Запросы:**
```json
GET /logs-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "OutOfMemoryError" }},
        { "term":  { "level": "ERROR" }}
      ],
      "filter": [
        { "range": { "timestamp": {
          "gte": "now-1h",
          "lte": "now"
        }}}
      ]
    }
  },
  "aggs": {
    "by_service": {
      "terms": { "field": "service" },
      "aggs": {
        "error_rate": { "value_count": { "field": "_id" }}
      }
    }
  },
  "sort": [{ "timestamp": "desc" }],
  "size": 20
}
```

**ILM (Index Lifecycle Management)** — управление временем жизни индексов (важно для логов):
```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": { "max_size": "50gb", "max_age": "1d" },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "readonly": {},
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": { "freeze": {} }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

---

## Эксплуатация и мониторинг

### 18. Какие ключевые метрики нужно мониторить в PostgreSQL?

**Connections и пул:**
```sql
-- Текущие соединения
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
-- idle: простаивают, active: выполняют запрос, idle in transaction: ОПАСНО

-- Близость к max_connections
SELECT count(*) * 100 / current_setting('max_connections')::int AS pct_used
FROM pg_stat_activity;
-- Алерт при > 80%
```

**Производительность запросов (pg_stat_statements):**
```sql
-- Топ по суммарному времени
SELECT query, calls, total_exec_time/1000 AS total_sec,
       mean_exec_time/1000 AS mean_sec,
       stddev_exec_time/1000 AS stddev_sec
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;

-- Cache hit ratio (должен быть > 99%)
SELECT
  sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100
    AS table_cache_hit_ratio,
  sum(idx_blks_hit) / (sum(idx_blks_hit) + sum(idx_blks_read)) * 100
    AS index_cache_hit_ratio
FROM pg_statio_user_tables;
-- Если < 95% — увеличить shared_buffers
```

**Размер и bloat:**
```sql
-- Размер таблиц и dead tuples (bloat)
SELECT relname,
       n_live_tup,
       n_dead_tup,
       round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
       last_vacuum,
       last_autovacuum,
       last_analyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
-- Если dead_pct > 10% — нужен VACUUM
```

**Репликация:**
```sql
-- Lag на primary
SELECT application_name,
       pg_size_pretty(sent_lsn - replay_lsn) AS lag_size,
       replay_lag
FROM pg_stat_replication;
-- Алерт если lag > 100MB или replay_lag > 30s

-- На standby
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;
```

**Locks и блокировки:**
```sql
-- Долгие блокировки
SELECT pid, now() - query_start AS duration, query, wait_event_type, wait_event
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
ORDER BY duration DESC;
```

**Ключевые метрики для Prometheus (через postgres_exporter):**
```yaml
# Алерты Prometheus/AlertManager
- alert: PostgreSQLHighConnections
  expr: pg_stat_activity_count / pg_settings_max_connections > 0.8
  for: 5m

- alert: PostgreSQLReplicationLag
  expr: pg_replication_lag > 30
  for: 2m

- alert: PostgreSQLDeadTuples
  expr: pg_stat_user_tables_n_dead_tup / (pg_stat_user_tables_n_live_tup + 1) > 0.1
  for: 10m

- alert: PostgreSQLLowCacheHitRatio
  expr: pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read + 1) < 0.95
  for: 5m
```

---

### 19. Как работает VACUUM в PostgreSQL и зачем он нужен?

**Проблема:** PostgreSQL использует MVCC — старые версии строк (dead tuples) не удаляются сразу при DELETE/UPDATE, а помечаются как невидимые. Со временем таблицы раздуваются (bloat), индексы тоже.

**VACUUM** — процесс очистки:
1. Помечает dead tuples как свободное пространство (для повторного использования)
2. Обновляет статистику видимости (Visibility Map)
3. Позволяет `VACUUM FREEZE` — предотвратить Transaction ID Wraparound

**VACUUM ANALYZE** — + обновляет статистику для планировщика запросов

**VACUUM FULL** — перезапись таблицы в новый файл (возвращает место ОС). Требует эксклюзивную блокировку — не использовать в production без крайней необходимости!

**AutoVacuum** — фоновый процесс, автоматически запускающий VACUUM:
```ini
# postgresql.conf
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 1min        # проверять каждую минуту

# Пороги запуска VACUUM
autovacuum_vacuum_threshold = 50          # если > 50 dead tuples...
autovacuum_vacuum_scale_factor = 0.2      # ...или > 20% от размера таблицы

# Пороги запуска ANALYZE
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.1

# Дросселирование (чтобы не перегружать диск)
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 200
```

**Transaction ID Wraparound — критическая проблема:**
```
PostgreSQL использует 32-битный transaction ID (XID).
Максимум ~4 миллиарда транзакций.
Когда XID обнуляется — все старые данные теряют видимость (catastrophic failure!).

Решение: VACUUM FREEZE периодически "замораживает" старые строки.
Мониторинг:
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY xid_age DESC;
-- Алерт если age > 1,500,000,000 (1.5 млрд)
-- Принудительно: при age = 200,000,000 до предела PostgreSQL сам начнёт предупреждать
```

**Оптимизация AutoVacuum для высоконагруженных таблиц:**
```sql
-- Настроить aggressively для конкретной таблицы
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.01,    -- вакуумировать при 1% dead tuples
  autovacuum_analyze_scale_factor = 0.01,
  autovacuum_vacuum_cost_delay = 0           -- без дросселирования
);

-- Мониторинг прогресса VACUUM
SELECT phase, heap_blks_total, heap_blks_vacuumed,
       index_vacuum_count, num_dead_tuples
FROM pg_stat_progress_vacuum;
```
