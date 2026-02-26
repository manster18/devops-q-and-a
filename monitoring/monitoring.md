# Мониторинг и Observability: Вопросы и ответы для DevOps-инженера (Middle/Senior)

## Содержание

1. [Что такое Observability и чем она отличается от Monitoring?](#1-что-такое-observability-и-чем-она-отличается-от-monitoring)
2. [Три столпа Observability: Metrics, Logs, Traces](#2-три-столпа-observability-metrics-logs-traces)
3. [Prometheus: архитектура, pull-модель, компоненты](#3-prometheus-архитектура-pull-модель-компоненты)
4. [PromQL: основные операторы, функции, примеры запросов](#4-promql-основные-операторы-функции-примеры-запросов)
5. [Alerting в Prometheus: AlertManager, маршрутизация, группировка, silences](#5-alerting-в-prometheus-alertmanager-маршрутизация-группировка-silences)
6. [Grafana: datasources, dashboards, variables, alerting](#6-grafana-datasources-dashboards-variables-alerting)
7. [Как мониторить Kubernetes кластер? Prometheus Operator, kube-state-metrics](#7-как-мониторить-kubernetes-кластер-prometheus-operator-kube-state-metrics)
8. [ELK Stack: Elasticsearch, Logstash, Kibana — архитектура и назначение](#8-elk-stack-elasticsearch-logstash-kibana--архитектура-и-назначение)
9. [EFK Stack: Fluentd/Fluent Bit vs Logstash — когда что выбрать?](#9-efk-stack-fluentdfluent-bit-vs-logstash--когда-что-выбрать)
10. [Distributed Tracing: как работает, Jaeger, Zipkin, OpenTelemetry](#10-distributed-tracing-как-работает-jaeger-zipkin-opentelemetry)
11. [OpenTelemetry: что это, зачем нужно, как инструментировать приложение?](#11-opentelemetry-что-это-зачем-нужно-как-инструментировать-приложение)
12. [RED и USE методологии — как строить мониторинг сервисов?](#12-red-и-use-методологии--как-строить-мониторинг-сервисов)
13. [Как построить алерты которые не вызывают alert fatigue?](#13-как-построить-алерты-которые-не-вызывают-alert-fatigue)
14. [Loki: как работает, чем отличается от Elasticsearch?](#14-loki-как-работает-чем-отличается-от-elasticsearch)
15. [Как мониторить приложение end-to-end: от инфраструктуры до бизнес-метрик?](#15-как-мониторить-приложение-end-to-end-от-инфраструктуры-до-бизнес-метрик)

---

## 1. Что такое Observability и чем она отличается от Monitoring?

**Monitoring** — это практика сбора заранее определённых метрик и создания алертов на пороговые значения. Мониторинг отвечает на вопрос **"Что сломалось?"** (известные неизвестные).

**Observability** — это свойство системы, позволяющее **понять её внутреннее состояние по внешним сигналам**. Observability отвечает на вопрос **"Почему сломалось?"** (неизвестные неизвестные).

```
Monitoring:
  CPU > 90%? → Alert → "Высокая нагрузка на CPU"
  (ты знаешь, что проверять заранее)

Observability:
  Почему запросы медленные именно у пользователей из Германии?
  Почему memory leak только под нагрузкой выше 500 RPS?
  Почему сервис B медленно отвечает только когда вызывается из сервиса C?
  (ты можешь исследовать что угодно без заранее написанного запроса)
```

**Практическое отличие:**

| Характеристика | Monitoring | Observability |
|---------------|------------|---------------|
| Подход | Сбор предопределённых метрик | Исследование произвольных вопросов |
| Вопрос | "Что сломалось?" | "Почему сломалось?" |
| Данные | Metrics (time series) | Metrics + Logs + Traces |
| Новые проблемы | Плохо подходит | Отлично подходит |
| Инструменты | Nagios, Zabbix | Prometheus + Jaeger + Loki |

**Принцип "Shift Left" в Observability:** инструментирование кода — ответственность разработчиков, а не только DevOps.

---

## 2. Три столпа Observability: Metrics, Logs, Traces

**Metrics (метрики)** — числовые значения во времени. Агрегированы, компактны, быстры для запросов.

```
Примеры:
  http_requests_total{status="200", path="/api/users"} 15432
  http_request_duration_seconds{quantile="0.99"} 0.234
  container_memory_usage_bytes{pod="myapp-xxx"} 123456789

Плюсы: компактны, быстрые алерты, хороши для дашбордов
Минусы: не объясняют "почему", нет контекста конкретного запроса
```

**Logs (логи)** — события во времени. Неструктурированные или структурированные.

```json
{
  "timestamp": "2024-01-15T10:23:45.123Z",
  "level": "ERROR",
  "message": "Database connection timeout",
  "service": "user-service",
  "trace_id": "abc123def456",
  "user_id": "user-789",
  "duration_ms": 5001,
  "db_host": "postgres-primary"
}
```

```
Плюсы: детальный контекст, отладка конкретного события
Минусы: дорогое хранение, медленные запросы по большим объёмам
```

**Traces (трейсы)** — жизненный путь запроса через все сервисы.

```
HTTP Request → API Gateway (10ms)
                    └── User Service (45ms)
                            └── DB Query (40ms)  ← bottleneck
                    └── Auth Service (5ms)
                    └── Cache (1ms)

Trace ID: abc123def456
Total: 61ms
```

```
Плюсы: видна вся цепочка вызовов, легко найти bottleneck
Минусы: требует инструментирования всех сервисов, высокая cardinality
```

**Связь между тремя столпами через exemplars и correlation:**

```
Alert: высокий P99 latency (метрика)
  ↓ drill down
Найти trace_id в метрике (exemplar)
  ↓
Открыть trace в Jaeger → увидеть медленный DB запрос
  ↓
По trace_id найти логи конкретного запроса
  ↓ 
Увидеть: "slow query: SELECT * FROM orders WHERE user_id=..."
```

---

## 3. Prometheus: архитектура, pull-модель, компоненты

**Архитектура Prometheus:**

```
┌─────────────────────────────────────────────────────┐
│                   Prometheus Server                  │
│  ┌──────────────┐  ┌────────────┐  ┌─────────────┐  │
│  │ Retrieval    │  │  TSDB      │  │  HTTP API   │  │
│  │ (scraper)    │→ │ (storage)  │→ │  (PromQL)   │  │
│  └──────────────┘  └────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────┘
         ↑ pull /metrics              ↓ query
  ┌──────────────┐              ┌──────────────┐
  │  Targets     │              │   Grafana    │
  │  (apps)      │              │  AlertManager│
  └──────────────┘              └──────────────┘
         ↑
  ┌──────────────┐
  │  Exporters   │
  │  node, redis │
  └──────────────┘
         ↑ discovery
  ┌──────────────┐
  │  Kubernetes  │
  │  Service Disc│
  └──────────────┘
```

**Компоненты:**

| Компонент | Назначение |
|-----------|-----------|
| **Prometheus Server** | Скрейпинг метрик, хранение в TSDB, выполнение PromQL |
| **Exporters** | Экспорт метрик сторонних систем (node_exporter, redis_exporter) |
| **Pushgateway** | Для short-lived jobs которые не могут быть scraped |
| **AlertManager** | Маршрутизация алертов, дедупликация, silencing |
| **Service Discovery** | Автообнаружение targets (Kubernetes, Consul, EC2) |

**Pull vs Push модель:**

```
Pull (Prometheus):                Push (Graphite, InfluxDB):
Prometheus → scrape /metrics      App → push metrics → Server
  + Prometheus контролирует rate   + Работает для short-lived jobs
  + Легко обнаружить uptime        - Сложнее контролировать cardinality
  + Нет потери метрик при          - App должна знать адрес сервера
    перезапуске Prometheus
```

**Типы метрик:**

```go
// Counter — только растёт (запросы, ошибки)
http_requests_total

// Gauge — может расти и падать (память, CPU, горутины)
memory_usage_bytes
goroutines_count

// Histogram — распределение значений (latency, request size)
// Хранит: bucket counts + sum + count
http_request_duration_seconds_bucket{le="0.1"}  1200
http_request_duration_seconds_bucket{le="0.5"}  1450
http_request_duration_seconds_bucket{le="1.0"}  1499
http_request_duration_seconds_bucket{le="+Inf"} 1500
http_request_duration_seconds_sum              234.5
http_request_duration_seconds_count            1500

// Summary — аналогично Histogram, но квантили считаются на клиенте
// Минус: нельзя агрегировать между instances
```

**Prometheus конфигурация:**

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: production
    region: us-east-1

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  # Kubernetes pods с аннотациями
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

  # Node exporter
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

---

## 4. PromQL: основные операторы, функции, примеры запросов

**Типы данных в PromQL:**

```
Instant vector  — значения для набора time series в момент времени T
Range vector    — значения для набора time series за период [5m]
Scalar          — одно число
String          — строка (редко используется)
```

**Основные операторы:**

```promql
# Фильтрация по labels
http_requests_total{job="api", status="200"}
http_requests_total{status=~"5.."}           # regex: любые 5xx
http_requests_total{status!~"2.."}           # исключить 2xx

# Арифметика
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# Агрегация
sum(http_requests_total) by (status)          # сумма по каждому статусу
avg(rate(http_requests_total[5m])) by (job)
topk(5, http_requests_total)                  # топ-5 по значению
```

**Ключевые функции:**

```promql
# rate() — скорость изменения counter за период (per second)
rate(http_requests_total[5m])
# irate() — мгновенная скорость (последние 2 точки в диапазоне)
irate(http_requests_total[1m])

# increase() — прирост counter за период
increase(http_requests_total[1h])

# histogram_quantile() — квантиль из histogram
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# avg_over_time() — среднее за период
avg_over_time(node_load1[30m])

# absent() — алерт если метрика отсутствует
absent(up{job="critical-service"})

# predict_linear() — прогнозирование на основе тренда
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600)
```

**Практические примеры:**

```promql
# Error rate (% ошибок от всех запросов)
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# Latency P99 для конкретного сервиса
histogram_quantile(
  0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{job="api-service"}[5m])
  )
)

# Свободная память в %
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# CPU usage в %
100 - (
  avg by (instance) (
    irate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)

# Disk fill rate — через сколько часов закончится место
predict_linear(
  node_filesystem_avail_bytes{mountpoint="/"}[6h],
  24 * 3600
) < 0

# Pods в не-running состоянии
kube_pod_status_phase{phase!="Running", phase!="Succeeded"} == 1

# Количество restarts за последний час
increase(kube_pod_container_status_restarts_total[1h]) > 3
```

---

## 5. Alerting в Prometheus: AlertManager, маршрутизация, группировка, silences

**Prometheus Rules (правила алертов):**

```yaml
# /etc/prometheus/rules/api-alerts.yml
groups:
  - name: api.rules
    interval: 30s
    rules:
      # Высокий error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)
          > 0.05
        for: 5m  # должно выполняться 5 минут (избегаем flapping)
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }} on {{ $labels.service }}"
          runbook: "https://wiki.example.com/runbooks/high-error-rate"
          dashboard: "https://grafana.example.com/d/abc123"

      # Медленные ответы
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum by (le, service) (
              rate(http_request_duration_seconds_bucket[5m])
            )
          ) > 1.0
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "P99 latency > 1s on {{ $labels.service }}"

      # Сервис недоступен
      - alert: ServiceDown
        expr: up{job=~"api.*"} == 0
        for: 1m
        labels:
          severity: critical
          pagerduty: "true"
        annotations:
          summary: "Service {{ $labels.job }} is down"
```

**AlertManager конфигурация:**

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/xxx'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  # Группировка: алерты с одинаковыми labels объединяются
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s       # ждём 30s перед отправкой (собираем группу)
  group_interval: 5m    # интервал повторной отправки для группы
  repeat_interval: 4h   # повторять алерт каждые 4 часа если не resolved

  receiver: 'slack-general'

  routes:
    # Critical алерты — в PagerDuty (будят ночью)
    - match:
        severity: critical
      receiver: pagerduty
      continue: true  # продолжить маршрутизацию (также отправить в slack)

    # Алерты команды backend
    - match:
        team: backend
      receiver: slack-backend
      group_by: ['alertname', 'service']

    # Алерты инфраструктуры
    - match_re:
        alertname: "^(Node|Disk|Memory).*"
      receiver: slack-infra

receivers:
  - name: 'slack-general'
    slack_configs:
      - channel: '#alerts'
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: $PAGERDUTY_KEY
        severity: '{{ .CommonLabels.severity }}'

  - name: 'slack-backend'
    slack_configs:
      - channel: '#backend-alerts'
        username: 'AlertManager'

inhibit_rules:
  # Если сервис полностью упал — не шлём алерты о высоком latency
  - source_match:
      alertname: 'ServiceDown'
    target_match:
      alertname: 'HighLatencyP99'
    equal: ['service']
```

**Управление через CLI:**

```bash
# Создать silence (заглушить алерты на время обслуживания)
amtool silence add \
  --alertmanager.url=http://alertmanager:9093 \
  --author="ops-team" \
  --comment="Planned maintenance 22:00-23:00" \
  --duration=1h \
  alertname=".*" cluster="production"

# Список активных silences
amtool silence query

# Список активных алертов
amtool alert query --alertmanager.url=http://alertmanager:9093
```

---

## 6. Grafana: datasources, dashboards, variables, alerting

**Datasources — источники данных:**

```yaml
# grafana.ini / provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: "15s"
      queryTimeout: "60s"
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: jaeger  # связь метрик с трейсами

  - name: Loki
    type: loki
    url: http://loki:3100
    jsonData:
      derivedFields:
        - name: TraceID
          matcherRegex: '"trace_id":"(\w+)"'
          url: '$${__value.raw}'
          datasourceUid: jaeger

  - name: Jaeger
    type: jaeger
    url: http://jaeger:16686
```

**Dashboard as Code (provisioning):**

```yaml
# provisioning/dashboards/dashboard.yml
apiVersion: 1
providers:
  - name: 'default'
    orgId: 1
    folder: 'Infrastructure'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```

**Variables — динамические дашборды:**

```
В Grafana UI: Dashboard Settings → Variables

Variable: namespace
  Type: Query
  Query: label_values(kube_pod_info, namespace)
  
Variable: pod  
  Type: Query
  Query: label_values(kube_pod_info{namespace="$namespace"}, pod)
  Depends on: namespace

Использование в панелях:
  container_memory_usage_bytes{namespace="$namespace", pod="$pod"}
```

**Grafana Alerting (Unified Alerting):**

```yaml
# Grafana Alert Rule через API/provisioning
apiVersion: 1
groups:
  - orgId: 1
    name: Application Alerts
    folder: Alerts
    interval: 1m
    rules:
      - uid: high-error-rate-001
        title: High Error Rate
        condition: C
        data:
          - refId: A
            datasourceUid: prometheus
            model:
              expr: |
                sum(rate(http_requests_total{status=~"5.."}[5m]))
                / sum(rate(http_requests_total[5m]))
          - refId: C
            datasourceUid: __expr__
            model:
              type: threshold
              conditions:
                - evaluator:
                    type: gt
                    params: [0.05]
        noDataState: NoData
        execErrState: Alerting
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Error rate above 5%
```

---

## 7. Как мониторить Kubernetes кластер? Prometheus Operator, kube-state-metrics

**Компоненты K8s мониторинга:**

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                         │
│  ┌───────────────┐  ┌─────────────────────────────────┐ │
│  │ kube-state-   │  │ node-exporter (DaemonSet)       │ │
│  │ metrics       │  │ CPU, memory, disk per node      │ │
│  │ (Deployments, │  └─────────────────────────────────┘ │
│  │  Pods, PVCs)  │                                      │
│  └───────────────┘  ┌─────────────────────────────────┐ │
│                     │ kubelet /metrics/cadvisor        │ │
│  ┌───────────────┐  │ Container-level resource usage   │ │
│  │ Prometheus    │  └─────────────────────────────────┘ │
│  │ Operator      │                                      │
│  │ (manages      │  ┌─────────────────────────────────┐ │
│  │  Prometheus   │  │ Application Pods (/metrics)     │ │
│  │  instances)   │  │ Custom business metrics          │ │
│  └───────────────┘  └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**Установка kube-prometheus-stack (рекомендуется):**

```bash
# Включает: Prometheus Operator, Prometheus, Alertmanager,
# Grafana, node-exporter, kube-state-metrics, prebuilt dashboards
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values values.yaml
```

```yaml
# values.yaml — важные настройки
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: "50GiB"
    resources:
      requests:
        memory: 2Gi
        cpu: 500m
      limits:
        memory: 4Gi
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          resources:
            requests:
              storage: 100Gi
    # Scrape всех PodMonitor/ServiceMonitor в кластере
    podMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false

grafana:
  adminPassword: "changeme"
  ingress:
    enabled: true
    hosts: ["grafana.example.com"]
  persistence:
    enabled: true
    size: 10Gi

alertmanager:
  config:
    global:
      slack_api_url: 'https://hooks.slack.com/...'
```

**ServiceMonitor — автодискавери приложений:**

```yaml
# Добавь аннотацию в своё приложение
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: production
  labels:
    release: monitoring  # должно совпадать с selector в Prometheus
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
      scheme: http
  namespaceSelector:
    matchNames:
      - production
```

**Ключевые метрики Kubernetes для алертов:**

```promql
# Pods в CrashLoopBackOff
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1

# Nodes NotReady
kube_node_status_condition{condition="Ready", status="true"} == 0

# Высокое использование CPU на Node
100 - (avg by (node) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85

# PVC почти полный
(
  kubelet_volume_stats_used_bytes
  / kubelet_volume_stats_capacity_bytes
) * 100 > 85

# OOMKilled containers
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

# Deployment не достиг желаемого числа реплик
kube_deployment_status_replicas_available
  != kube_deployment_spec_replicas
```

---

## 8. ELK Stack: Elasticsearch, Logstash, Kibana — архитектура и назначение

**Архитектура ELK:**

```
Applications/Servers
      │
      ▼
┌──────────────┐
│   Logstash   │  ← сбор, парсинг, обогащение логов
│  (ingest)    │  input → filter → output
└──────────────┘
      │
      ▼
┌──────────────────────────────────────┐
│         Elasticsearch Cluster        │
│  ┌──────────┐  ┌──────────┐          │
│  │  Master  │  │  Master  │ (x3)     │
│  │  Node    │  │  Node    │          │
│  └──────────┘  └──────────┘          │
│  ┌──────────┐  ┌──────────┐          │
│  │   Data   │  │   Data   │ (xN)     │
│  │   Node   │  │   Node   │          │
│  └──────────┘  └──────────┘          │
└──────────────────────────────────────┘
      │
      ▼
┌──────────────┐
│    Kibana    │  ← визуализация, поиск, дашборды
└──────────────┘
```

**Elasticsearch — основные концепции:**

```json
// Index — аналог базы данных, хранит документы
// Document — единица хранения (JSON)
// Shard — физическая единица хранения (Primary + Replica)

// Стратегия именования индексов: по дате (ILM)
// logs-2024.01.15 → logs-2024.01.16 → logs-2024.01.17

// Маппинг — схема документа
PUT /logs-template
{
  "mappings": {
    "properties": {
      "@timestamp":    { "type": "date" },
      "level":         { "type": "keyword" },  // точное совпадение
      "message":       { "type": "text" },      // full-text search
      "service":       { "type": "keyword" },
      "duration_ms":   { "type": "long" },
      "trace_id":      { "type": "keyword" }
    }
  }
}
```

**ILM (Index Lifecycle Management):**

```json
// Автоматическое управление жизненным циклом индексов
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50gb",   // новый индекс если > 50GB
            "max_age": "1d"       // или каждый день
          }
        }
      },
      "warm": {
        "min_age": "7d",          // через 7 дней — перенос на warm nodes
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "cold": {
        "min_age": "30d",         // через 30 дней — в cold tier
        "actions": { "freeze": {} }
      },
      "delete": {
        "min_age": "90d",         // через 90 дней — удаление
        "actions": { "delete": {} }
      }
    }
  }
}
```

**Logstash pipeline:**

```ruby
# /etc/logstash/conf.d/app.conf
input {
  beats {
    port => 5044
  }
}

filter {
  # Парсинг JSON логов
  if [message] =~ /^\{/ {
    json {
      source => "message"
    }
  }

  # Grok — парсинг текстовых логов
  grok {
    match => {
      "message" => '%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:msg}'
    }
  }

  # Обогащение GeoIP
  geoip {
    source => "client_ip"
    target => "geoip"
  }

  # Удаление ненужных полей
  mutate {
    remove_field => ["agent", "ecs", "input", "tags"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "${ES_PASSWORD}"
  }
}
```

---

## 9. EFK Stack: Fluentd/Fluent Bit vs Logstash — когда что выбрать?

**Сравнение:**

| Параметр | Logstash | Fluentd | Fluent Bit |
|----------|----------|---------|------------|
| Написан на | JRuby/Java | C + Ruby | C |
| RAM | 500MB+ | 40MB | ~650KB |
| CPU | Высокое | Среднее | Минимальное |
| Пропускная способность | Высокая | Высокая | Очень высокая |
| Плагины | 200+ | 1000+ | 100+ |
| Трансформации | Мощные (Grok) | Хорошие | Базовые |
| Лучший вариант | Сложная обработка | Агрегация | Edge/DaemonSet |

**Типичная EFK архитектура в Kubernetes:**

```
Every Node:
  Fluent Bit (DaemonSet)  ← читает /var/log/containers/*.log
        │
        ▼ forward
  Fluentd (Deployment/StatefulSet)  ← агрегация, буферизация, трансформация
        │
        ▼
  Elasticsearch / OpenSearch
        │
        ▼
  Kibana / OpenSearch Dashboards
```

**Fluent Bit DaemonSet для K8s:**

```yaml
# helm install fluent-bit fluent/fluent-bit --values values.yaml
config:
  inputs: |
    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            cri      # или docker
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

  filters: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On       # распарсить JSON логи внутри message
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On       # удалить pods с аннотацией fluentbit.io/exclude

    [FILTER]
        Name    grep
        Match   kube.*
        Exclude log  ^$   # исключить пустые строки

  outputs: |
    [OUTPUT]
        Name            es
        Match           kube.*
        Host            elasticsearch
        Port            9200
        Index           k8s-logs
        Logstash_Format On
        Logstash_Prefix k8s
        tls             On
        HTTP_User       elastic
        HTTP_Passwd     ${ES_PASSWORD}
        Retry_Limit     False
        Buffer_Size     5MB
```

---

## 10. Distributed Tracing: как работает, Jaeger, Zipkin, OpenTelemetry

**Как работает distributed tracing:**

```
User Request
    │
    ▼ HTTP POST /checkout
[API Gateway]  trace_id=abc123, span_id=001, parent=None
    │
    ├── gRPC → [Order Service]  span_id=002, parent=001
    │              │
    │              ├── SQL → [Postgres]  span_id=003, parent=002 (5ms)
    │              └── gRPC → [Inventory Service]  span_id=004, parent=002
    │                              │
    │                              └── Redis GET  span_id=005, parent=004 (1ms)
    │
    └── gRPC → [Payment Service]  span_id=006, parent=001
                   │
                   └── HTTP → [Stripe API]  span_id=007, parent=006 (250ms)

Весь путь складывается в один Trace с trace_id=abc123
```

**Ключевые концепции:**

- **Trace** — полный путь одного запроса, состоит из Spans
- **Span** — единица работы (один сервис или один вызов). Содержит: name, start/end time, tags, logs, status
- **Context Propagation** — передача trace_id между сервисами через HTTP заголовки:
  - W3C Trace Context: `traceparent: 00-abc123-001-01`
  - B3 (Zipkin): `X-B3-TraceId: abc123`

**Jaeger архитектура:**

```
App (с SDK) → Jaeger Agent (UDP/localhost)
                    │
                    ▼
            Jaeger Collector → Kafka (buffer, optional)
                                    │
                                    ▼
                             Jaeger Ingester → Storage (Elasticsearch / Cassandra)
                                                    │
                                                    ▼
                                             Jaeger Query → Jaeger UI
```

**Jaeger в Kubernetes (упрощённо):**

```yaml
# Jaeger Operator или all-in-one для dev
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
spec:
  strategy: production
  collector:
    maxReplicas: 5
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
        index-prefix: jaeger
  query:
    serviceType: ClusterIP
```

---

## 11. OpenTelemetry: что это, зачем нужно, как инструментировать приложение?

**Проблема до OpenTelemetry:**

```
До OTel:                          После OTel:
  Prometheus SDK (metrics)          OTel SDK
  Jaeger SDK (traces)       →         └── metrics → Prometheus
  Custom logging                        └── traces → Jaeger/Tempo/Zipkin
                                        └── logs → Loki/ES
  3 разных SDK, несовместимых   Один SDK, стандартный протокол (OTLP)
```

**OpenTelemetry компоненты:**

```
┌────────────────────────────────────────────────────┐
│              OpenTelemetry Collector                │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │Receivers │→ │  Processors  │→ │  Exporters   │  │
│  │ OTLP     │  │ batch        │  │ Prometheus   │  │
│  │ Jaeger   │  │ memory_limit │  │ Jaeger       │  │
│  │ Zipkin   │  │ resourcedetect│ │ Loki         │  │
│  │ Prometheus│  │ transform    │  │ OTLP/gRPC    │  │
│  └──────────┘  └──────────────┘  └──────────────┘  │
└────────────────────────────────────────────────────┘
         ↑ OTLP (gRPC/HTTP)
  ┌──────────────────┐
  │   Applications   │
  │  (instrumented   │
  │   with OTel SDK) │
  └──────────────────┘
```

**Auto-instrumentation (zero-code) в Kubernetes:**

```yaml
# OpenTelemetry Operator автоматически инструментирует pods
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: instrumentation
  namespace: production
spec:
  exporter:
    endpoint: http://otel-collector:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "0.1"  # 10% sampling rate

  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest
  go:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-go:latest

# Добавь аннотацию к Deployment:
# instrumentation.opentelemetry.io/inject-java: "true"
```

**Ручное инструментирование (Go):**

```go
package main

import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/trace"
    "go.opentelemetry.io/otel/attribute"
)

func initTracer() func() {
    exporter, _ := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
    )
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithSampler(trace.TraceIDRatioBased(0.1)),
    )
    otel.SetTracerProvider(tp)
    return func() { tp.Shutdown(ctx) }
}

func handleRequest(w http.ResponseWriter, r *http.Request) {
    tracer := otel.Tracer("myapp")
    ctx, span := tracer.Start(r.Context(), "handleRequest")
    defer span.End()

    span.SetAttributes(
        attribute.String("user.id", getUserID(r)),
        attribute.String("http.method", r.Method),
    )

    result, err := db.QueryContext(ctx, "SELECT ...")
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
    }
}
```

**OTel Collector конфигурация:**

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  prometheus:
    config:
      scrape_configs:
        - job_name: 'otel-collector'
          static_configs:
            - targets: ['localhost:8888']

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  resource:
    attributes:
      - key: deployment.environment
        value: production
        action: insert

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  jaeger:
    endpoint: jaeger-collector:14250
    tls:
      insecure: true
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger]
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

---

## 12. RED и USE методологии — как строить мониторинг сервисов?

**RED Method (для сервисов, APIs):**

```
R — Rate      Сколько запросов в секунду обрабатывает сервис?
E — Errors    Какой процент запросов завершается ошибкой?
D — Duration  Сколько времени занимает обработка запроса? (P50/P95/P99)
```

```promql
# Rate — запросы в секунду
sum(rate(http_requests_total[5m])) by (service)

# Errors — процент ошибок
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/ sum(rate(http_requests_total[5m])) by (service) * 100

# Duration — latency P99
histogram_quantile(0.99,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
)
```

**USE Method (для ресурсов — CPU, Memory, Disk, Network):**

```
U — Utilization   Насколько занят ресурс? (% от максимальной пропускной способности)
S — Saturation    Есть ли очередь к ресурсу? (queue length, wait time)
E — Errors        Есть ли ошибки при работе с ресурсом?
```

```promql
# CPU Utilization
100 - avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100

# CPU Saturation (load average vs CPU count)
node_load1 / count by (instance) (node_cpu_seconds_total{mode="idle"})

# Memory Utilization
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
/ node_memory_MemTotal_bytes * 100

# Memory Saturation (swap usage rate)
rate(node_vmstat_pswpin[5m]) + rate(node_vmstat_pswpout[5m])

# Disk Utilization (I/O time)
rate(node_disk_io_time_seconds_total[5m]) * 100

# Disk Saturation (I/O queue)
rate(node_disk_io_time_weighted_seconds_total[5m])

# Network Errors
rate(node_network_receive_errs_total[5m])
+ rate(node_network_transmit_errs_total[5m])
```

**Четыре Golden Signals (Google SRE):**

```
1. Latency    — время ответа (успешные и ошибочные отдельно)
2. Traffic    — нагрузка (RPS, QPS, bytes/s)
3. Errors     — процент ошибок
4. Saturation — насколько ресурсы заполнены (очереди, CPU, mem)
```

Это расширение RED Method, явно добавляющее Saturation. Рекомендуется начать мониторинг именно с этих 4 сигналов для каждого сервиса.

---

## 13. Как построить алерты которые не вызывают alert fatigue?

**Alert fatigue** — ситуация, когда инженеры игнорируют алерты из-за их количества или низкой точности. Главная причина плохих алертов — мониторинг симптомов (CPU > 80%) вместо **влияния на пользователя**.

**Правила хорошего алерта:**

```
1. Actionable — алерт должен требовать немедленного действия
   Плохо:  "CPU > 80%"      → Что делать? Непонятно.
   Хорошо: "Error rate > 5% на /api/checkout" → Пора лезть в runbook

2. Symptom-based, не cause-based
   Плохо:  "Elasticsearch JVM heap > 90%"
   Хорошо: "Поиск медленнее 3 секунд (P99)" → симптом для пользователя

3. Правильный severity
   PagerDuty (разбудить):  Прямое влияние на пользователей СЕЙЧАС
   Slack/ticket:           Нужно исправить в рабочее время
   Dashboard только:       Информационные метрики

4. Правильный for: (pending period)
   Слишком короткий → много flapping алертов
   Слишком длинный  → долгое время реакции
   Рекомендация: 5 минут для warning, 2-3 минуты для critical

5. Runbook в каждом алерте
   annotations:
     runbook: "https://wiki.company.com/runbooks/high-error-rate"
```

**Пример хорошего vs плохого алерта:**

```yaml
# ПЛОХО — не actionable, слишком зашумлён
- alert: HighCPU
  expr: node_cpu_usage > 80
  for: 1m
  # Нет severity, нет runbook, нет контекста

# ХОРОШО — actionable, symptom-based
- alert: APIErrorRateHigh
  expr: |
    sum(rate(http_requests_total{status=~"5..",service="checkout"}[5m]))
    / sum(rate(http_requests_total{service="checkout"}[5m])) > 0.01
  for: 5m
  labels:
    severity: critical
    team: payments
    pagerduty: "true"
  annotations:
    summary: "Checkout service error rate {{ $value | humanizePercentage }}"
    description: |
      More than 1% of checkout requests are failing.
      Current error rate: {{ $value | humanizePercentage }}
      Affected service: {{ $labels.service }}
    runbook: "https://wiki/runbooks/checkout-errors"
    dashboard: "https://grafana/d/checkout"
```

**Иерархия алертов (уменьшаем шум через inhibition):**

```yaml
# alertmanager.yml
inhibit_rules:
  # Если кластер полностью недоступен — молчим про отдельные поды
  - source_match:
      alertname: ClusterDown
    target_match_re:
      alertname: "Pod.*"
    equal: [cluster]

  # Если нода упала — молчим про поды на этой ноде
  - source_match:
      alertname: NodeDown
    target_match:
      alertname: PodCrashLooping
    equal: [node]
```

---

## 14. Loki: как работает, чем отличается от Elasticsearch?

**Философия Loki:** "Like Prometheus, but for logs"

```
Elasticsearch:
  + Полнотекстовый поиск (inverted index)
  + Поиск по любому полю
  - Дорогое хранение (индексирование всего)
  - Высокое потребление памяти

Loki:
  + Индексирует только labels (как Prometheus labels)
  + Хранит log chunks сжатыми (gzip/snappy) в object storage (S3)
  + Дешёвое хранение (10-20x дешевле ES)
  + Натуральная интеграция с Grafana
  - Нельзя искать по содержимому без stream selector
  - Медленнее ES при поиске по тексту без label filter
```

**Архитектура Loki:**

```
Log Sources → Promtail/Fluent Bit/OTel
                    │ push
                    ▼
             Loki Distributor  (hash ring, принимает записи)
                    │
                    ▼
             Loki Ingester     (буферизация в памяти, WAL)
                    │
                    ├── Object Storage (S3/GCS) — сжатые chunks
                    └── Index (BoltDB/Cassandra/BoltDB Shipper→S3)
                    
Query:
Grafana → Loki Querier → Object Storage + Index
```

**LogQL — язык запросов Loki:**

```logql
# Базовый stream selector (обязателен)
{namespace="production", app="myapp"}

# Фильтрация по содержимому
{app="myapp"} |= "error"           # содержит "error"
{app="myapp"} != "health"          # не содержит "health"
{app="myapp"} |~ "timeout|refused" # regex

# Парсинг JSON логов
{app="myapp"} | json
| level = "error"
| duration > 1000

# Метрики из логов (Log Metrics)
# Количество ошибок в минуту
sum by (app) (
  rate({namespace="production"} |= "ERROR" [1m])
)

# P99 latency из access логов
histogram_quantile(0.99,
  sum by (le) (
    rate({app="nginx"} | logfmt | unwrap duration [5m])
  )
)

# Top-10 IP по числу запросов
topk(10,
  sum by (ip) (
    rate({app="nginx"} | logfmt | __error__="" [5m])
  )
)
```

**Promtail конфигурация (агент для Loki):**

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080

clients:
  - url: http://loki:3100/loki/api/v1/push
    batchwait: 1s
    batchsize: 1048576  # 1MB

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    pipeline_stages:
      # Парсинг CRI/Docker log format
      - cri: {}
      # Парсинг JSON логов
      - json:
          expressions:
            level: level
            msg: message
            trace_id: trace_id
      # Добавление label из JSON
      - labels:
          level:
          trace_id:
      # Отбрасывание debug логов
      - drop:
          expression: '.*level=debug.*'
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
```

---

## 15. Как мониторить приложение end-to-end: от инфраструктуры до бизнес-метрик?

**Пирамида мониторинга:**

```
        ┌──────────────────┐
        │  Business Metrics │  Выручка, конверсия, заказы/мин
        ├──────────────────┤
        │  Application     │  Latency, errors, RPS, cache hit rate
        ├──────────────────┤
        │  Service Mesh /  │  Inter-service latency, circuit breakers
        │  Kubernetes      │
        ├──────────────────┤
        │  Infrastructure  │  CPU, Memory, Disk, Network
        └──────────────────┘
Важно мониторить все уровни!
```

**Пример полного стека для e-commerce сервиса:**

```yaml
# 1. Инфраструктура (Prometheus + node-exporter)
- Node CPU, Memory, Disk — USE Method
- Kubernetes: Pod restarts, OOMKilled, PVC usage

# 2. Kubernetes Application Layer
- Deployment replicas available vs desired
- HTTP Ingress error rate (nginx-ingress метрики)

# 3. Application (RED Method — custom metrics в коде)
# Go пример:
```

```go
var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )
    httpDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Buckets: prometheus.ExponentialBuckets(0.001, 2, 15),
        },
        []string{"method", "path"},
    )
    // Business metric
    ordersProcessed = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "orders_processed_total",
            Help: "Total orders processed",
        },
        []string{"payment_method", "status"},
    )
    orderValue = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "order_value_usd",
            Buckets: []float64{10, 50, 100, 250, 500, 1000, 5000},
        },
        []string{"category"},
    )
)
```

```yaml
# 4. Synthetic Monitoring — Blackbox Exporter
# Проверяет доступность снаружи, как реальный пользователь

modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      follow_redirects: true
      fail_if_ssl: false
      tls_config:
        insecure_skip_verify: false

# Scrape конфиг для проверки эндпоинтов:
- job_name: 'blackbox-http'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://example.com/health
        - https://example.com/api/checkout
        - https://api.example.com/v1/status
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: blackbox-exporter:9115

# 5. Алерты по бизнес-метрикам
- alert: OrdersPerMinuteDropped
  expr: |
    sum(rate(orders_processed_total{status="success"}[5m])) < 0.5
  for: 5m
  labels:
    severity: critical
    pagerduty: "true"
  annotations:
    summary: "Orders per minute dropped below 30/hour"
    description: "Possible payment or inventory issue"
```

**Grafana Dashboard структура для полного стека:**

```
Row 1: Business KPIs
  - Orders per minute (rate)
  - Revenue per hour
  - Conversion rate

Row 2: Application RED
  - RPS by endpoint
  - Error rate by endpoint  
  - P50/P95/P99 latency heatmap

Row 3: Infrastructure USE
  - CPU/Memory by node
  - Pod resource usage
  - Network traffic

Row 4: Kubernetes Health
  - Pod count by namespace
  - Restarts last 1h
  - PVC usage

Row 5: Dependencies
  - DB query latency
  - Cache hit rate
  - External API latency
```
