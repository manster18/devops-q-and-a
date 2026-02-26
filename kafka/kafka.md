# Apache Kafka: Вопросы и ответы для DevOps-инженера (Middle/Senior)

## Содержание

1. [Что такое Kafka и какие задачи она решает?](#1-что-такое-kafka-и-какие-задачи-она-решает)
2. [Архитектура Kafka: брокеры, ZooKeeper/KRaft, топики, партиции](#2-архитектура-kafka-брокеры-zookeeperkraft-топики-партиции)
3. [Producers: как пишется сообщение, acks, partitioning strategy](#3-producers-как-пишется-сообщение-acks-partitioning-strategy)
4. [Consumers и Consumer Groups: offset management, rebalancing](#4-consumers-и-consumer-groups-offset-management-rebalancing)
5. [Репликация: ISR, leader election, гарантии надёжности](#5-репликация-isr-leader-election-гарантии-надёжности)
6. [Exactly-once semantics: idempotent producer, transactions](#6-exactly-once-semantics-idempotent-producer-transactions)
7. [Kafka Connect: source и sink connectors, debezium CDC](#7-kafka-connect-source-и-sink-connectors-debezium-cdc)
8. [Kafka Streams и ksqlDB: stream processing](#8-kafka-streams-и-ksqldb-stream-processing)
9. [Производительность и тюнинг: ключевые параметры брокера и клиентов](#9-производительность-и-тюнинг-ключевые-параметры-брокера-и-клиентов)
10. [Мониторинг Kafka: ключевые метрики, Prometheus JMX Exporter](#10-мониторинг-kafka-ключевые-метрики-prometheus-jmx-exporter)
11. [Kafka в Kubernetes: Strimzi Operator, производственные best practices](#11-kafka-в-kubernetes-strimzi-operator-производственные-best-practices)
12. [Kafka vs другие брокеры: RabbitMQ, SQS, Pulsar](#12-kafka-vs-другие-брокеры-rabbitmq-sqs-pulsar)

---

## 1. Что такое Kafka и какие задачи она решает?

**Apache Kafka** — распределённая платформа потоковой обработки событий. Изначально создана в LinkedIn для обработки 1.4 триллиона сообщений в день.

**Ключевые характеристики:**

```
Высокая пропускная способность: миллионы сообщений/секунду
Низкая задержка:              < 10ms для producer → consumer
Персистентность:              хранение сообщений на диске (дни/недели)
Масштабируемость:             горизонтальное масштабирование
Отказоустойчивость:           репликация, leader election
```

**Kafka как единая платформа:**

```
              ┌─────────────────────────────────────────┐
              │              Apache Kafka                │
Sources:      │  ┌─────────┐   ┌──────────┐            │
Databases     │  │  Topics │   │ Consumer │  Targets:   │
Applications  │  │ (log)   │   │  Groups  │  Analytics  │
Microservices │  └─────────┘   └──────────┘  Databases  │
Logs/Metrics  │                              ML Models   │
              └─────────────────────────────────────────┘

Use cases:
  1. Event Streaming (замена очередей)
  2. Log Aggregation (замена Logstash/Fluentd → Kafka → ELK)
  3. CDC (Change Data Capture) — репликация БД в реальном времени
  4. Event Sourcing — хранение истории изменений
  5. CQRS — разделение чтения и записи
  6. Microservices Integration — асинхронная коммуникация
  7. Metrics Pipeline — метрики → Kafka → Prometheus/InfluxDB
```

**Kafka vs традиционные очереди:**

| Параметр | RabbitMQ/SQS | Kafka |
|----------|-------------|-------|
| Модель | Push (broker → consumer) | Pull (consumer тянет) |
| Хранение | После ACK сообщение удаляется | Хранится retention период |
| Replay | Невозможен | Можно перечитать с любого offset |
| Throughput | Высокий (100k msg/s) | Очень высокий (1M+ msg/s) |
| Ordering | Per-queue | Per-partition |
| Consumer groups | Нет аналога | Встроено |

---

## 2. Архитектура Kafka: брокеры, ZooKeeper/KRaft, топики, партиции

**Основные компоненты:**

```
┌──────────────────────────────────────────────────────────────┐
│                       Kafka Cluster                           │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │  Broker 1  │  │  Broker 2  │  │  Broker 3  │             │
│  │            │  │            │  │            │             │
│  │ Topic A    │  │ Topic A    │  │ Topic A    │             │
│  │  Part 0 ★  │  │  Part 1 ★  │  │  Part 2 ★  │  (leaders) │
│  │  Part 1    │  │  Part 2    │  │  Part 0    │  (replicas) │
│  └────────────┘  └────────────┘  └────────────┘             │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │          ZooKeeper / KRaft (metadata)                   │  │
│  │  Cluster membership, leader election, topic configs    │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**Брокер (Broker):**
- Физический или виртуальный сервер с Kafka процессом
- Хранит партиции топиков
- Обслуживает запросы producers и consumers

**ZooKeeper → KRaft (Kafka 3.x+):**

```
ZooKeeper (старый подход):
  Kafka требовал отдельный ZooKeeper кластер для метаданных
  Проблемы: дополнительная операционная сложность, синхронизация

KRaft (KIP-500, Kafka 3.3+ — production, 2.8+ — preview):
  Kafka сам управляет метаданными через Raft consensus protocol
  Один из брокеров = controller (raft leader)
  Более простая операция, быстрее failover

# Конфигурация KRaft
process.roles=broker,controller  # или только broker, только controller
controller.quorum.voters=1@broker1:9093,2@broker2:9093,3@broker3:9093
```

**Топик (Topic):**

```
Топик = логический канал для сообщений
  Разделён на партиции
  Конфигурируется: retention.ms, retention.bytes, replication.factor

Создание топика:
kafka-topics.sh --create \
  --bootstrap-server kafka:9092 \
  --topic orders \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \   # 7 дней
  --config min.insync.replicas=2
```

**Партиция (Partition):**

```
Партиция = упорядоченный, неизменяемый лог сообщений

Topic: orders (12 partitions)
  Partition 0: [msg_0, msg_1, msg_2, ..., msg_N]  offset 0..N
  Partition 1: [msg_0, msg_1, ..., msg_M]          offset 0..M
  ...
  Partition 11: [...]

Ключевые свойства:
  - Ordering гарантирован ТОЛЬКО внутри партиции
  - Partition = единица параллелизма
  - Consumer parallelism <= число партиций

Как выбрать число партиций:
  Правило: max(expected_throughput / single_partition_throughput,
              target_consumer_count)
  Типично: 3-12 партиций для начала
  Можно увеличить но НЕЛЬЗЯ уменьшить (без пересоздания топика)
```

**Сегменты (Segments) — физическое хранение:**

```
Партиция физически хранится как набор segment файлов:
  /kafka/data/orders-0/
    00000000000000000000.log      # данные сообщений
    00000000000000000000.index    # offset → position индекс
    00000000000000000000.timeindex # timestamp → offset индекс
    00000000000000001234.log      # новый сегмент (после ротации)

log.segment.bytes=1073741824    # новый сегмент после 1GB
log.roll.ms=604800000           # или через 7 дней

Compacted topic (log.cleanup.policy=compact):
  Хранит только последнее сообщение для каждого ключа
  Использование: KTable в Kafka Streams, CDC
```

---

## 3. Producers: как пишется сообщение, acks, partitioning strategy

**Жизненный цикл сообщения от producer:**

```
1. Producer создаёт ProducerRecord(topic, [key], [partition], value, [headers])
2. Serializer конвертирует key/value → bytes
3. Partitioner определяет партицию
4. RecordAccumulator буферизирует (batch.size, linger.ms)
5. Sender thread отправляет batch broker'у-лидеру
6. Broker записывает в лог
7. Broker отвечает (согласно acks настройке)
```

**acks — гарантии доставки:**

```
acks=0:  Fire and forget
  Producer не ждёт ответа брокера
  Максимальная скорость, нет гарантий
  Допустимо для метрик, логов где потеря нестрашна

acks=1:  Leader only (default до Kafka 3.0)
  Leader записал → OK ответ
  При падении leader до репликации → потеря сообщений

acks=all (или acks=-1):  All ISR replicas (рекомендуется)
  Все in-sync replicas записали → OK ответ
  Наиболее надёжно, медленнее
  Нужно совместно с min.insync.replicas=2
```

**Стратегии партиционирования:**

```python
# Без ключа: Round-Robin (или sticky partitioner в новых версиях)
producer.send("orders", value=order_data)

# С ключом: Hash(key) % num_partitions
# Гарантирует что все сообщения с одним ключом → одна партиция (ordering!)
producer.send("orders", key="user_123", value=order_data)

# Явное указание партиции
producer.send("orders", partition=5, value=order_data)

# Кастомный партиционер
class RegionPartitioner:
    def partition(self, key, all_partitions, available_partitions):
        region = json.loads(key)["region"]
        if region == "eu":
            return 0
        elif region == "us":
            return 1
        else:
            return 2
```

**Производительность producer:**

```properties
# Batching — ключ к высокой пропускной способности
batch.size=65536          # 64KB батч (default 16KB)
linger.ms=5               # подождать 5ms для набора батча (default 0)
compression.type=lz4      # сжатие: none, gzip, snappy, lz4, zstd

# Буфер
buffer.memory=67108864    # 64MB RAM буфер
max.block.ms=60000        # как долго блокировать send() если буфер полон

# Retry при временных ошибках
retries=2147483647        # retry до delivery.timeout.ms
delivery.timeout.ms=120000 # максимальное время на доставку
retry.backoff.ms=100
```

---

## 4. Consumers и Consumer Groups: offset management, rebalancing

**Consumer Group — параллельная обработка:**

```
Topic: orders (6 partitions)

Consumer Group A (3 consumers):
  Consumer A1 → Partition 0, 1
  Consumer A2 → Partition 2, 3
  Consumer A3 → Partition 4, 5

Consumer Group B (1 consumer):
  Consumer B1 → Partition 0, 1, 2, 3, 4, 5  (все партиции)

Правило: 1 партиция → max 1 consumer в группе
         consumers > partitions → лишние consumers idle
```

**Offset Management:**

```
Offset = позиция последнего прочитанного сообщения

Хранение offsets:
  Kafka < 0.9: ZooKeeper
  Kafka >= 0.9: специальный топик __consumer_offsets (рекомендуется)

auto.commit:
  enable.auto.commit=true (default)
  auto.commit.interval.ms=5000
  
  Проблема: commit может произойти до обработки → at-most-once
  
Ручное управление (рекомендуется для критичных систем):
  enable.auto.commit=false
  
  # Commit после обработки → at-least-once
  consumer.commitSync()  # синхронный (медленнее, надёжнее)
  consumer.commitAsync() # асинхронный (быстрее)
  
  # Commit конкретного offset
  consumer.commitSync({
    TopicPartition("orders", 0): OffsetAndMetadata(offset=42)
  })
```

**Consumer Rebalancing:**

```
Когда происходит rebalance:
  - Consumer присоединяется к группе
  - Consumer покидает группу (или падает)
  - Топик добавляет партиции
  - heartbeat.interval.ms пропущен (consumer считается мёртвым)

Eager Rebalance (stop-the-world, Kafka default):
  1. Все consumers останавливают обработку
  2. Все consumers отказываются от партиций
  3. Заново назначаются все партиции
  
  Проблема: пауза обработки на все время rebalance

Cooperative Rebalance (incremental, рекомендуется):
  1. Только затронутые партиции перераспределяются
  2. Остальные consumers продолжают работу
  
  partition.assignment.strategy=CooperativeStickyAssignor

# Настройки для минимизации rebalance
session.timeout.ms=45000      # как долго ждать heartbeat
heartbeat.interval.ms=3000    # каждые 3с отправлять heartbeat
max.poll.interval.ms=300000   # максимальное время между poll()
```

**Lag — отставание consumer:**

```bash
# Просмотр consumer lag
kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --group my-consumer-group \
  --describe

# Вывод:
# TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID
# orders   0          1234            1240             6    consumer-1
# orders   1          5678            5678             0    consumer-2

# High lag = consumer не успевает за producer
# Решения: увеличить parallelism, оптимизировать обработку
```

---

## 5. Репликация: ISR, leader election, гарантии надёжности

**Репликация в Kafka:**

```
replication.factor=3 означает:
  1 Leader replica  (все чтения и записи)
  2 Follower replicas (синхронизируются с leader)

Запись:
  Producer → Leader (Broker 1)
               │
               ├── Follower (Broker 2) sync
               └── Follower (Broker 3) sync
               
  Leader → ack producer только после записи в ISR
```

**ISR (In-Sync Replicas):**

```
ISR = множество реплик, которые "в синхронизации" с leader

Реплика исключается из ISR если:
  - Не слала heartbeat leader'у за replica.lag.time.max.ms (10s default)
  - Отстала в репликации за replica.lag.time.max.ms

min.insync.replicas=2:
  Producer с acks=all получит ошибку если ISR < 2
  Защита от потери данных

Пример с replication.factor=3, min.insync.replicas=2:
  Нормально: ISR=[broker1, broker2, broker3]  → пишем
  1 брокер упал: ISR=[broker1, broker2]        → пишем (ISR >= min.isr)
  2 брокера упали: ISR=[broker1]               → отказ записи (ISR < min.isr)
  
  "2 из 3" — хороший баланс надёжность/доступность
```

**Leader Election:**

```
При падении leader:
  1. Controller (в ZooKeeper/KRaft) определяет что leader недоступен
  2. Новый leader выбирается из ISR (первый в ISR list)
  3. Controller уведомляет всех consumers и producers
  
  Время переключения: обычно < 30 секунд
  
  unclean.leader.election.enable=false (рекомендуется!):
    Запрет выбора leader'а не из ISR
    Предотвращает потерю данных (out-of-ISR реплика может отставать)
    Ценой: недоступность пока ISR не восстановится
```

**Типичная конфигурация production кластера:**

```properties
# Broker config
num.partitions=12
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# Retention
log.retention.hours=168        # 7 дней
log.retention.bytes=107374182400  # или 100GB per partition
log.segment.bytes=1073741824   # 1GB segment

# Performance
num.io.threads=8
num.network.threads=3
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
```

---

## 6. Exactly-once semantics: idempotent producer, transactions

**Три уровня гарантий доставки:**

```
At-most-once:
  Сообщение доставлено не более 1 раза (возможна потеря)
  Producer: acks=0 или acks=1
  Consumer: commit before processing
  
At-least-once (default):
  Сообщение доставлено минимум 1 раз (возможны дубликаты)
  Producer: acks=all + retries
  Consumer: commit after processing

Exactly-once:
  Сообщение доставлено ровно 1 раз (нет потерь и дублей)
  Требует: idempotent producer + transactions
```

**Idempotent Producer:**

```python
# enable.idempotence=true
# Каждый producer получает PID (Producer ID)
# Каждое сообщение имеет sequence number
# Broker отбрасывает дубли (ретрай с тем же PID+sequence)

producer_config = {
    'enable.idempotence': True,      # автоматически acks=all, retries=MAX
    'acks': 'all',
    'max.in.flight.requests.per.connection': 5,  # max для idempotence
}

# С idempotence: producer может безопасно ретраить
# Без idempotence: ретрай может создать дубль
```

**Transactions — exactly-once между несколькими топиками:**

```python
from confluent_kafka import Producer, Consumer

# Transactional producer
producer = Producer({
    'bootstrap.servers': 'kafka:9092',
    'transactional.id': 'my-transactional-id',  # уникальный per producer instance
    'enable.idempotence': True,
})

producer.init_transactions()

# Consumer → Transform → Producer (read-process-write pattern)
consumer = Consumer({
    'bootstrap.servers': 'kafka:9092',
    'group.id': 'my-processor',
    'isolation.level': 'read_committed',  # видеть только committed транзакции
    'enable.auto.commit': False,
})

consumer.subscribe(['input-topic'])

while True:
    msg = consumer.poll(1.0)
    if msg is None:
        continue
    
    producer.begin_transaction()
    try:
        # Обработка
        result = process(msg.value())
        
        # Записать результат
        producer.produce('output-topic', value=result)
        
        # Commit consumer offset ВНУТРИ транзакции
        # (атомарно с записью в output-topic)
        producer.send_offsets_to_transaction(
            consumer.position(consumer.assignment()),
            consumer.consumer_group_metadata()
        )
        
        producer.commit_transaction()
    except Exception as e:
        producer.abort_transaction()
        raise
```

---

## 7. Kafka Connect: source и sink connectors, debezium CDC

**Kafka Connect** — framework для интеграции Kafka с внешними системами без написания кода.

```
Source Connector:  External System → Kafka Topic
Sink Connector:    Kafka Topic → External System

Популярные коннекторы:
  Source: Debezium (DB CDC), S3, Twitter, JDBC
  Sink:   Elasticsearch, S3, JDBC, BigQuery, Snowflake
```

**Debezium — Change Data Capture (CDC):**

```
Debezium читает transaction log базы данных (binlog, WAL)
и публикует каждое изменение в Kafka топик.

Поддерживает: MySQL, PostgreSQL, MongoDB, Oracle, SQL Server

CDC событие в Kafka:
{
  "before": {"id": 1, "name": "Alice", "email": "alice@old.com"},
  "after":  {"id": 1, "name": "Alice", "email": "alice@new.com"},
  "op": "u",  // c=create, u=update, d=delete, r=read (snapshot)
  "ts_ms": 1642000000000,
  "source": {
    "db": "myapp", "table": "users",
    "lsn": 123456789  // PostgreSQL LSN
  }
}
```

**Конфигурация Debezium PostgreSQL connector:**

```json
{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.dbname": "production",
    "database.server.name": "pgserver1",
    
    "table.include.list": "public.orders,public.users",
    
    "plugin.name": "pgoutput",   // pgoutput (встроен в PG 10+) или decoderbufs
    
    "slot.name": "debezium_slot",
    "publication.name": "dbz_publication",
    
    "tombstones.on.delete": "true",
    "decimal.handling.mode": "double",
    
    "topic.prefix": "dbserver1",
    // Топики: dbserver1.public.orders, dbserver1.public.users
    
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
  }
}
```

**Развёртывание Connect в Kubernetes:**

```bash
# Strimzi KafkaConnector CRD
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: debezium-postgres
  labels:
    strimzi.io/cluster: my-kafka-connect
spec:
  class: io.debezium.connector.postgresql.PostgresConnector
  tasksMax: 1
  config:
    database.hostname: postgres
    database.port: "5432"
    database.user: debezium
    database.password: ${secrets:postgres-secret:password}
    database.dbname: production
    topic.prefix: pg
    table.include.list: public.orders
```

---

## 8. Kafka Streams и ksqlDB: stream processing

**Kafka Streams** — клиентская Java библиотека для обработки данных в Kafka.

```
Особенности:
  - Встроена в приложение (не отдельный кластер)
  - Stateful и stateless операции
  - Exactly-once semantics
  - Fault tolerance через changelog topics
```

```java
// Kafka Streams: подсчёт заказов по региону в реальном времени
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders = builder.stream(
    "orders",
    Consumed.with(Serdes.String(), orderSerde)
);

KTable<String, Long> ordersByRegion = orders
    .filter((key, order) -> order.getStatus().equals("COMPLETED"))
    .groupBy((key, order) -> order.getRegion())
    .count(Materialized.as("orders-by-region-store"));

ordersByRegion.toStream()
    .to("orders-by-region", Produced.with(Serdes.String(), Serdes.Long()));

KafkaStreams streams = new KafkaStreams(builder.build(), config);
streams.start();
```

**ksqlDB — SQL для потоков:**

```sql
-- Создать stream из топика
CREATE STREAM orders (
  order_id VARCHAR,
  user_id VARCHAR,
  amount DOUBLE,
  region VARCHAR,
  status VARCHAR
) WITH (
  kafka_topic='orders',
  value_format='JSON'
);

-- Непрерывный запрос: подсчёт заказов по региону
CREATE TABLE orders_by_region AS
SELECT
  region,
  COUNT(*) AS order_count,
  SUM(amount) AS total_amount
FROM orders
WHERE status = 'COMPLETED'
WINDOW TUMBLING (SIZE 1 HOUR)
GROUP BY region
EMIT CHANGES;

-- Alerting: большие заказы
CREATE STREAM large_orders AS
SELECT * FROM orders
WHERE amount > 10000
EMIT CHANGES;

-- Push query (подписка на изменения)
SELECT * FROM large_orders EMIT CHANGES;

-- Pull query (текущее состояние)
SELECT * FROM orders_by_region WHERE region = 'eu';
```

---

## 9. Производительность и тюнинг: ключевые параметры брокера и клиентов

**Broker-side оптимизации:**

```properties
# Параллелизм
num.io.threads=8                    # потоки ввода-вывода (=кол-во дисков)
num.network.threads=3               # сетевые потоки
num.replica.fetchers=4              # потоки репликации

# Буферы сети
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600  # max запрос 100MB

# Disk I/O
log.flush.interval.messages=10000  # fsync каждые 10k сообщений
log.flush.interval.ms=1000         # или каждую секунду
# Лучше: доверить OS page cache, не делать fsync вручную

# Page Cache оптимизация
# Kafka активно использует page cache для "zero-copy" чтения
# Убедись что JVM heap НЕ слишком большой (оставь RAM для page cache)
# Рекомендация: 6GB heap для broker, остальное → page cache
KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"
```

**Producer оптимизации:**

```properties
# Throughput: приоритет пропускной способности
batch.size=131072         # 128KB батч
linger.ms=10              # подождать 10ms
compression.type=lz4      # быстрое сжатие

# Latency: приоритет задержки
batch.size=16384          # маленький батч
linger.ms=0               # отправить сразу
compression.type=none

# Баланс
batch.size=65536          # 64KB
linger.ms=5
compression.type=snappy
```

**Consumer оптимизации:**

```properties
fetch.min.bytes=1048576       # ждать пока не накопится 1MB (throughput)
fetch.max.wait.ms=500         # или max 500ms
max.partition.fetch.bytes=5242880  # 5MB per partition per fetch

# Параллельная обработка внутри consumer
max.poll.records=500          # сколько записей за один poll()
```

**OS-level настройки:**

```bash
# Увеличить file descriptors (Kafka открывает много файлов)
echo "* soft nofile 100000" >> /etc/security/limits.conf
echo "* hard nofile 100000" >> /etc/security/limits.conf

# Swap должен быть минимальным (Kafka = page cache = плохо при swap)
sysctl -w vm.swappiness=1

# Оптимизация сетевых буферов
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"

# Выбор файловой системы: XFS (лучшая производительность для Kafka)
# или ext4 с noatime mount option
```

---

## 10. Мониторинг Kafka: ключевые метрики, Prometheus JMX Exporter

**Ключевые метрики:**

```
Брокер:
  kafka_server_brokertopicmetrics_messagesin_total   — messages/sec
  kafka_server_brokertopicmetrics_bytesin_total      — bytes/sec in
  kafka_server_brokertopicmetrics_bytesout_total     — bytes/sec out
  kafka_controller_kafkacontroller_activecontrollercount == 1 (ровно 1!)
  kafka_server_replicamanager_underreplicatedpartitions == 0 (0 = ok!)
  kafka_server_replicamanager_offlinepartitionscount == 0 (0 = ok!)

Consumer:
  kafka_consumer_fetch_manager_records_lag        — отставание consumer
  kafka_consumer_fetch_manager_fetch_latency_avg  — задержка получения

Producer:
  kafka_producer_record_send_rate                 — records/sec
  kafka_producer_record_error_rate                — ошибки/sec
  kafka_producer_request_latency_avg              — задержка отправки
```

**Prometheus JMX Exporter:**

```yaml
# kafka-jmx-exporter-config.yaml
startDelaySeconds: 0
ssl: false
lowercaseOutputName: true
lowercaseOutputLabelNames: true

rules:
  - pattern: kafka.server<type=(.+), name=(.+), clientId=(.+), topic=(.+), partition=(.*)><>Value
    name: kafka_server_$1_$2
    type: GAUGE
    labels:
      clientId: "$3"
      topic: "$4"
      partition: "$5"

  - pattern: kafka.controller<type=(.+), name=(.+)><>Value
    name: kafka_controller_$1_$2
    type: GAUGE

  - pattern: kafka.server<type=ReplicaManager, name=(.+)><>Value
    name: kafka_server_replicamanager_$1
    type: GAUGE
```

```yaml
# Алерты Prometheus
groups:
  - name: kafka.rules
    rules:
      - alert: KafkaUnderReplicatedPartitions
        expr: kafka_server_replicamanager_underreplicatedpartitions > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Under replicated partitions on {{ $labels.instance }}"

      - alert: KafkaHighConsumerLag
        expr: |
          sum by (consumergroup, topic) (
            kafka_consumergroup_lag
          ) > 10000
        for: 10m
        labels:
          severity: warning

      - alert: KafkaNoActiveController
        expr: sum(kafka_controller_kafkacontroller_activecontrollercount) != 1
        for: 1m
        labels:
          severity: critical
```

---

## 11. Kafka в Kubernetes: Strimzi Operator, производственные best practices

**Strimzi Operator:**

```yaml
# Kafka Cluster через Strimzi CRD
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: production-kafka
  namespace: kafka
spec:
  kafka:
    version: 3.7.0
    replicas: 3

    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
      - name: external
        port: 9094
        type: loadbalancer
        tls: true

    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.7"

    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 500Gi
          class: fast-ssd
          deleteClaim: false

    resources:
      requests:
        memory: 8Gi
        cpu: "2"
      limits:
        memory: 16Gi
        cpu: "4"

    jvmOptions:
      -Xms: 4096m
      -Xmx: 4096m

    rack:
      topologyKey: topology.kubernetes.io/zone  # spread по AZ

    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics
          key: kafka-metrics-config.yml

  zookeeper:  # или kraft: {} для KRaft mode
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      class: fast-ssd
    resources:
      requests:
        memory: 1Gi
        cpu: "0.5"
      limits:
        memory: 2Gi
        cpu: "1"

  entityOperator:
    topicOperator: {}
    userOperator: {}
```

**Production best practices:**

```
1. Pod Anti-Affinity — разные Kafka брокеры на разных нодах
   topologyKey: kubernetes.io/hostname

2. Dedicated nodes — выделенные ноды только для Kafka
   nodeSelector + taints/tolerations

3. Local SSD storage — для минимальной задержки
   storageClass: local-nvme

4. JVM tuning
   G1GC (Kafka 3.x default)
   Heap: 6-8GB максимум, остальное → OS page cache

5. Persistent Volumes с ReadWriteOnce
   Нельзя делиться partition между брокерами

6. Rack awareness — spread по зонам доступности
   broker.rack=${RACK}  (AZ)
```

---

## 12. Kafka vs другие брокеры: RabbitMQ, SQS, Pulsar

**Когда Kafka:**

```
✓ Высокая пропускная способность (1M+ msg/sec)
✓ Нужен replay (перечитать историю)
✓ Event Sourcing, CDC, Log Aggregation
✓ Много consumer groups для одних данных
✓ Долгосрочное хранение (дни/недели)
✓ Stream processing (Kafka Streams, Flink)
```

**Когда RabbitMQ:**

```
✓ Сложная маршрутизация (exchanges: topic, fanout, direct, headers)
✓ Priority queues нужны
✓ Сообщение удаляется после обработки
✓ Меньший масштаб (< 100k msg/sec)
✓ AMQP совместимость
✓ Легче в операционном смысле
```

**Когда AWS SQS:**

```
✓ Serverless подход, нет операционных затрат
✓ AWS-native интеграция (Lambda, SNS, Step Functions)
✓ Dead Letter Queue из коробки
✓ FIFO очереди если нужен строгий порядок
Минус: нет replay, 14 дней максимальный retention
```

**Apache Pulsar vs Kafka:**

```
Pulsar отличия:
  - Compute (Brokers) и Storage (Bookies) разделены
  - Мгновенное масштабирование без rebalancing
  - Нативная multi-tenancy
  - Geo-replication из коробки
  - Функции на разных языках (Python, Go, Java)
  
Когда Pulsar:
  ✓ Мультиарендная архитектура
  ✓ Geo-репликация (несколько дата-центров)
  ✓ Serverless функции для обработки
  
Когда Kafka лучше:
  ✓ Более зрелая экосистема
  ✓ Больше инструментов (Kafka Connect, ksqlDB)
  ✓ Лучше документация и сообщество
  ✓ Kubernetes: Strimzi vs Pulsar Operator (Strimzi зрелее)
```

**Сравнение производительности:**

| Брокер | Throughput | Latency | Retention | Replay |
|--------|-----------|---------|-----------|--------|
| Kafka | Very High | Low | Long-term | Да |
| RabbitMQ | High | Very Low | Short | Нет |
| AWS SQS | High | Low | 14 days | Нет |
| Pulsar | Very High | Low | Long-term | Да |
| Redis Streams | Medium | Very Low | Limited | Да |
