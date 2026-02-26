# FinOps: Вопросы и ответы для DevOps-инженера (Middle/Senior)

## Содержание

1. [Что такое FinOps и какова роль DevOps-инженера?](#1-что-такое-finops-и-какова-роль-devops-инженера)
2. [Модели оплаты AWS: On-Demand, Reserved, Savings Plans, Spot](#2-модели-оплаты-aws-on-demand-reserved-savings-plans-spot)
3. [Spot Instances: стратегии использования, Spot Interruption, Karpenter](#3-spot-instances-стратегии-использования-spot-interruption-karpenter)
4. [Rightsizing: как найти и исправить over-provisioned ресурсы?](#4-rightsizing-как-найти-и-исправить-over-provisioned-ресурсы)
5. [Kubernetes cost optimization: requests/limits, VPA, Karpenter](#5-kubernetes-cost-optimization-requestslimits-vpa-karpenter)
6. [S3 и хранилище: оптимизация через lifecycle policies, Intelligent-Tiering](#6-s3-и-хранилище-оптимизация-через-lifecycle-policies-intelligent-tiering)
7. [Showback и Chargeback: как распределить затраты между командами?](#7-showback-и-chargeback-как-распределить-затраты-между-командами)
8. [Cost Monitoring: AWS Cost Explorer, Budgets, anomaly detection](#8-cost-monitoring-aws-cost-explorer-budgets-anomaly-detection)
9. [NAT Gateway и Data Transfer: скрытые расходы в AWS](#9-nat-gateway-и-data-transfer-скрытые-расходы-в-aws)
10. [FinOps практики: tagging strategy, waste detection, governance](#10-finops-практики-tagging-strategy-waste-detection-governance)

---

## 1. Что такое FinOps и какова роль DevOps-инженера?

**FinOps (Cloud Financial Operations)** — практика и культура, объединяющая Finance, Engineering и Business для максимизации бизнес-ценности от облачных расходов.

```
Традиционный подход к расходам:
  Finance: смотрит на счёт раз в месяц
  Engineering: использует ресурсы не думая о стоимости
  "Стоимость = проблема Finance"

FinOps подход:
  "Каждый несёт ответственность за облачные расходы"
  Engineer знает стоимость своих решений
  Finance понимает технические trade-offs
  Business принимает осознанные решения
```

**Три фазы FinOps (FinOps Foundation):**

```
1. Inform (Информирование)
   Видимость: кто сколько тратит?
   Теги, Cost allocation, dashboards
   
2. Optimize (Оптимизация)
   Rightsizing, Spot instances, Reserved Instances
   Уничтожение неиспользуемых ресурсов
   
3. Operate (Управление)
   FinOps как культура, не разовая акция
   Автоматическое выключение dev-сред ночью
   Budget alerts, anomaly detection
```

**Роль DevOps/Platform Engineer в FinOps:**

```
✓ Внедрение tagging strategy (кто платит за что)
✓ Настройка Cost Monitoring и Alerting
✓ Kubernetes cost optimization (requests/limits, Karpenter)
✓ Spot Instance стратегии
✓ Автоматизация lifecycle (dev среды выключаются ночью)
✓ Rightsizing через VPA/инструменты
✓ S3 lifecycle policies
✓ Elimination of waste (orphaned resources)
```

---

## 2. Модели оплаты AWS: On-Demand, Reserved, Savings Plans, Spot

**On-Demand:**

```
Самый дорогой вариант. Платишь только за использованное.
  Когда использовать:
    - Непредсказуемая нагрузка
    - Краткосрочные workloads
    - Тестирование и разработка
```

**Reserved Instances (RI):**

```
1-3 летний commitment. Скидка 30-75% от On-Demand.
  
  Standard RI: нельзя изменить семейство инстанса
  Convertible RI: можно изменить тип, но меньше скидка (54% vs 72%)
  
  Платёж:
    All Upfront: максимальная скидка
    Partial Upfront: средняя скидка
    No Upfront: наименьшая скидка, но нет начальных вложений
  
  Когда использовать:
    - Стабильные, предсказуемые workloads
    - Production DB, серверы с постоянной нагрузкой
```

**Savings Plans:**

```
Более гибкий аналог RI. Commitment = $N/hour на 1-3 года.
  
  Compute Savings Plans (наиболее гибкий):
    Применяется к EC2 (любой регион, семейство, OS)
    Применяется к Fargate, Lambda
    Скидка: до 66%
    
  EC2 Instance Savings Plans:
    Конкретное семейство + регион
    Скидка: до 72%
    Нельзя менять семейство инстанса
  
  Рекомендация: использовать Compute Savings Plans
  (максимальная гибкость при хорошей скидке)
```

**Spot Instances:**

```
Используют свободные вычислительные мощности AWS.
  Скидка: 60-90% от On-Demand.
  
  Caveat: AWS может прервать с 2-минутным предупреждением
  
  Когда использовать:
    - Stateless workloads (web серверы)
    - Batch processing, ML training
    - CI/CD runners
    - Dev/Test среды
    - Kubernetes worker nodes (через Karpenter)
  
  Не подходит:
    - Stateful workloads (DB primary)
    - Workloads без graceful shutdown
```

**Правило большого пальца для AWS:**

```
Production base load:     Savings Plans (1-3 года)
Production peak:          On-Demand (auto-scaling)
Dev/Test/Batch/CI:        Spot
DB, критичные сервисы:    On-Demand или Reserved

Структура оптимизированного кластера EKS:
  On-Demand (Savings Plans): 2-3 ноды → стабильный baseline
  Spot: auto-scaling → экономия на burst
```

---

## 3. Spot Instances: стратегии использования, Spot Interruption, Karpenter

**Как работает Spot Interruption:**

```
1. AWS даёт 2-минутное предупреждение (Spot Instance Interruption Notice)
2. EC2 instance metadata: /latest/meta-data/spot/termination-time
3. Instance Rebalance Recommendation (раньше — за несколько минут)

Что делать приложению:
  - Graceful shutdown за 2 минуты
  - Если это K8s нода: evict pods, дать им время завершиться
  - Если это CI runner: завершить текущий job или checkpoint
```

**Spot в Kubernetes через Karpenter:**

```yaml
# NodePool с mix Spot + On-Demand
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        billing-team: platform
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default

      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]   # сначала пробует Spot

        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            # Диверсификация по типам (важно для Spot availability!)
            - m5.xlarge
            - m5a.xlarge
            - m6i.xlarge
            - m6a.xlarge
            - m5d.xlarge
            - c5.2xlarge
            - c6i.2xlarge

        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]

      # Taints for spot nodes (optional)
      taints:
        - key: spot
          value: "true"
          effect: NoSchedule

  limits:
    cpu: 1000
    memory: 2000Gi

  disruption:
    consolidationPolicy: WhenUnderutilized   # consolidate для экономии
    consolidateAfter: 30s
    expireAfter: 720h  # rotate nodes каждые 30 дней

---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  role: KarpenterNodeRole
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster

  # Spot Interruption Handling
  # Karpenter автоматически обрабатывает через EC2 EventBridge events
  # При получении Interruption Warning: cordon node + drain pods
```

**Tolerations для Spot workloads:**

```yaml
# Pods которые могут работать на Spot нодах
spec:
  tolerations:
    - key: spot
      operator: Equal
      value: "true"
      effect: NoSchedule

  # Node affinity: предпочитать Spot, но работать на On-Demand если нет
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: karpenter.sh/capacity-type
                operator: In
                values: ["spot"]

  # Graceful shutdown для Spot
  terminationGracePeriodSeconds: 120  # 2 минуты = время предупреждения
```

**Spot в CI/CD (GitHub Actions с ARC):**

```yaml
# Actions Runner Controller на Spot инстансах
apiVersion: actions.github.com/v1alpha1
kind: AutoscalingRunnerSet
spec:
  template:
    spec:
      tolerations:
        - key: spot
          operator: Equal
          value: "true"
          effect: NoSchedule
      nodeSelector:
        karpenter.sh/capacity-type: spot
      containers:
        - name: runner
          # CI jobs обычно stateless — идеально для Spot
```

---

## 4. Rightsizing: как найти и исправить over-provisioned ресурсы?

**Over-provisioning — главный источник waste:**

```
Типичная картина:
  EC2 m5.xlarge (4 vCPU, 16GB):
    CPU utilization: 5-10% (нужен 1 vCPU)
    Memory: 2GB используется из 16GB
  
  Потери: 75-90% стоимости инстанса

Причины:
  "Нам нужно было быстро — взяли с запасом"
  "Лучше перестраховаться"
  Нет visibility на фактическое использование
  Страх деградации при уменьшении
```

**AWS Compute Optimizer:**

```bash
# AWS Compute Optimizer анализирует CloudWatch метрики за 14 дней
# и рекомендует оптимальный размер

aws compute-optimizer get-ec2-instance-recommendations \
  --account-ids 123456789012 \
  --region us-east-1

# Вывод:
# Instance: i-1234567890abcdef0
# Current: m5.xlarge ($0.192/h)
# Recommended: m5.large ($0.096/h) — экономия 50%
# Risk: Low (метрики подтверждают)

# Для RDS
aws compute-optimizer get-rds-database-recommendations

# Для ECS/Lambda
aws compute-optimizer get-ecs-service-recommendations
aws compute-optimizer get-lambda-function-recommendations
```

**Kubernetes rightsizing через VPA:**

```yaml
# Vertical Pod Autoscaler рекомендует оптимальные requests/limits
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"  # только рекомендации, не применять автоматически
    # updateMode: "Auto" — применять автоматически (с осторожностью!)

# Посмотреть рекомендации
kubectl describe vpa myapp-vpa

# Вывод:
# Container Recommendations:
#   Container Name: myapp
#     Lower Bound:
#       Cpu: 50m
#       Memory: 128Mi
#     Target:
#       Cpu: 200m      ← рекомендуемое (было 1000m)
#       Memory: 512Mi  ← рекомендуемое (было 2Gi)
#     Upper Bound:
#       Cpu: 800m
#       Memory: 2Gi
```

**Goldilocks — визуализация VPA рекомендаций:**

```bash
# Установка
helm install goldilocks fairwinds/goldilocks \
  --namespace goldilocks \
  --create-namespace

# Включить для namespace
kubectl label namespace production goldilocks.fairwinds.com/enabled=true

# Открыть UI
kubectl port-forward svc/goldilocks-dashboard -n goldilocks 8080:80
# → http://localhost:8080 показывает рекомендации для всех pods
```

---

## 5. Kubernetes cost optimization: requests/limits, VPA, Karpenter

**Проблема с неправильными requests/limits:**

```
Scenario 1: Requests слишком высокие (over-provisioned)
  request: cpu=2000m, memory=4Gi
  actual: cpu=100m, memory=500Mi
  
  Kubernetes резервирует ноду для requests
  → Нода не может принять больше pods (capacity waste)
  → Нужно больше нод = больше денег

Scenario 2: Requests слишком низкие (under-provisioned)
  request: cpu=50m, memory=128Mi
  actual: cpu=500m (burst), memory=1Gi (OOMKilled!)
  
  → Pod конкурирует за ресурсы с соседями
  → OOMKilled → restart → нестабильность
```

**QoS классы и их влияние на стоимость:**

```yaml
# Guaranteed (request == limit) — нода зарезервирована под этот pod
# Самый дорогой, но самый предсказуемый
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "1"
    memory: "2Gi"

# Burstable (request < limit) — хороший баланс
resources:
  requests:
    cpu: "200m"    # scheduler видит 200m
    memory: "512Mi"
  limits:
    cpu: "2"       # может burst до 2 CPU
    memory: "2Gi"

# BestEffort (нет requests/limits) — нельзя использовать в production!
# Первыми эвиктируются при нехватке ресурсов
```

**Karpenter consolidation — автоматическое уплотнение:**

```yaml
# Karpenter объединяет pods на меньшее количество нод
# и удаляет пустые/недозагруженные ноды

disruption:
  consolidationPolicy: WhenUnderutilized
  consolidateAfter: 30s

# Пример:
# Было: 3 ноды по 50% utilization
# После consolidation: 2 ноды по 75% utilization → 1 нода освобождается
# Экономия: 33% от стоимости нод
```

**Kubecost — детальный cost visibility в K8s:**

```bash
# Установка
helm install kubecost cost-analyzer/cost-analyzer \
  --namespace kubecost \
  --create-namespace \
  --set kubecostToken="TOKEN" \
  --set global.prometheus.enabled=true

# Показывает стоимость:
# - По namespace
# - По deployment/pod
# - По label (team, environment)
# - Рекомендации по rightsizing
# - Idle cost (неиспользуемые резервации)

# Prometheus метрики Kubecost
container_cpu_allocation        # CPU выделено
container_memory_allocation_bytes  # Memory выделено
deployment_match_labels         # Labels для атрибуции
```

---

## 6. S3 и хранилище: оптимизация через lifecycle policies, Intelligent-Tiering

**Классы хранения S3:**

| Класс | Стоимость ($/GB/мес) | Минимальное хранение | Latency | Когда |
|-------|---------------------|---------------------|---------|-------|
| Standard | $0.023 | нет | мс | Активные данные |
| Intelligent-Tiering | $0.023 + $0.0025/1000 obj | нет | мс/часы | Неизвестный паттерн |
| Standard-IA | $0.0125 | 30 дней | мс | Нечастый доступ |
| One Zone-IA | $0.01 | 30 дней | мс | Воспроизводимые данные |
| Glacier Instant | $0.004 | 90 дней | мс | Архив, нужен быстро |
| Glacier Flexible | $0.0036 | 90 дней | 1-12h | Архив |
| Glacier Deep Archive | $0.00099 | 180 дней | 12-48h | Долгосрочный архив |

**S3 Lifecycle Policy:**

```json
{
  "Rules": [
    {
      "ID": "log-archival",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      },
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 7,
          "StorageClass": "GLACIER"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      }
    }
  ]
}
```

**S3 Intelligent-Tiering:**

```
Автоматически перемещает объекты между тирами:
  Frequent Access tier (Standard pricing)
  Infrequent Access tier (40% дешевле) — после 30 дней без доступа
  Archive Instant Access tier (68% дешевле) — после 90 дней (опционально)
  Archive Access tier (95% дешевле) — после 90-730 дней (опционально)

Плата: $0.0025 за 1000 объектов/месяц (monitoring fee)
Оправдано: если средний размер объекта > 128KB
```

**Abort incomplete multipart uploads:**

```json
{
  "Rules": [
    {
      "ID": "abort-incomplete-multipart",
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

```bash
# Найти и удалить незавершённые multipart uploads (могут копиться незаметно)
aws s3api list-multipart-uploads --bucket my-bucket \
  --query 'Uploads[*].{Key:Key,UploadId:UploadId}'

# Удалить все завершённые
aws s3api list-multipart-uploads --bucket my-bucket | \
  jq -r '.Uploads[] | "aws s3api abort-multipart-upload --bucket my-bucket --key \(.Key) --upload-id \(.UploadId)"' | \
  bash
```

---

## 7. Showback и Chargeback: как распределить затраты между командами?

**Showback** — показать командам их расходы, но не выставлять счёт внутри компании.

**Chargeback** — реально выставлять внутренний счёт командам (из их бюджета).

```
Зачем:
  ✓ Команды осознают стоимость своих решений
  ✓ Incentive к оптимизации
  ✓ Справедливое распределение облачных затрат
  ✓ Budget planning для продуктовых команд

Showback vs Chargeback:
  Showback проще (нет бухгалтерии)
  Chargeback точнее (реальная ответственность)
  Начинать с Showback → переходить к Chargeback
```

**Tagging Strategy — основа cost allocation:**

```bash
# Обязательные теги на все ресурсы
aws ec2 create-tags --resources i-1234567890 --tags \
  Key=team,Value=backend \
  Key=environment,Value=production \
  Key=project,Value=checkout \
  Key=cost-center,Value=CC-1234 \
  Key=owner,Value=john.doe@company.com

# Политика требования тегов через AWS Organizations SCP
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances",
        "rds:CreateDBInstance"
      ],
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestedRegion": "false",
          "aws:ResourceTag/team": "true"  // запретить без тега team
        }
      }
    }
  ]
}
```

**Kubernetes cost allocation по тегам:**

```yaml
# Namespace labels для Kubecost
apiVersion: v1
kind: Namespace
metadata:
  name: team-backend
  labels:
    team: backend
    cost-center: CC-1234
    environment: production

# Deployment labels
metadata:
  labels:
    app: myapp
    team: backend
    project: checkout
    version: v1.2.3
```

**AWS Cost Allocation Tags:**

```bash
# Активировать теги для cost allocation
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status \
    TagKey=team,Status=Active \
    TagKey=environment,Status=Active \
    TagKey=project,Status=Active

# Cost breakdown по командам в Cost Explorer
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics "BlendedCost" \
  --group-by Type=TAG,Key=team
```

---

## 8. Cost Monitoring: AWS Cost Explorer, Budgets, anomaly detection

**AWS Cost Explorer:**

```bash
# Анализ расходов по сервисам
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity DAILY \
  --metrics "BlendedCost" "UsageQuantity" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --filter '{
    "Dimensions": {
      "Key": "SERVICE",
      "Values": ["Amazon EC2", "Amazon RDS", "Amazon S3"]
    }
  }'

# Rightsizing recommendations
aws ce get-rightsizing-recommendation \
  --service EC2 \
  --configuration SavingsOpportunityPercentage=5
```

**AWS Budgets:**

```bash
# Создать бюджет с алертом при превышении 80%
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "team-backend-monthly",
    "BudgetLimit": {
      "Amount": "10000",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {
      "TagKeyValue": ["user:team$backend"]
    }
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {
          "SubscriptionType": "EMAIL",
          "Address": "backend-lead@company.com"
        },
        {
          "SubscriptionType": "SNS",
          "Address": "arn:aws:sns:us-east-1:123:cost-alerts"
        }
      ]
    }
  ]'
```

**Cost Anomaly Detection:**

```bash
# Автоматически обнаруживать аномальные скачки стоимости
# AWS использует ML модель

# Создать monitor (по сервису или тегу)
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "team-backend-monitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

# Создать subscription (оповещения)
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "high-spend-alert",
    "MonitorArnList": ["arn:aws:ce::123456789012:anomalymonitor/MONITOR_ID"],
    "Subscribers": [
      {
        "Address": "arn:aws:sns:us-east-1:123:cost-alerts",
        "Type": "SNS"
      }
    ],
    "Threshold": 100,
    "Frequency": "DAILY"
  }'
```

**Grafana Cost Dashboard через CloudWatch:**

```promql
# Prometheus метрики из Kubecost для Grafana
# Стоимость по namespace
sum by (namespace) (
  container_cpu_allocation * on(node) group_left() node_cpu_hourly_cost
  + container_memory_allocation_bytes / 1024 / 1024 / 1024 * on(node) group_left() node_ram_hourly_cost
)

# Idle cost (зарезервировано но не используется)
sum(kube_pod_container_resource_requests{resource="cpu"}) - sum(node:container_cpu_usage_seconds_total:sum_rate)
```

---

## 9. NAT Gateway и Data Transfer: скрытые расходы в AWS

**NAT Gateway — неожиданно дорого:**

```
NAT Gateway pricing:
  $0.045/час × 24 × 730 = ~$33/месяц (просто за наличие)
  $0.045/GB обработанных данных
  
  Пример: 1TB/день через NAT Gateway
  = 30TB/месяц × $0.045 = $1,350/месяц только Data Processing!

Частые проблемы:
  1. NAT Gateway в каждой AZ (×3 = ×3 стоимость)
  2. Приватные подсети тянут большие объёмы через NAT (S3, ECR, etc.)
  3. Cross-AZ трафик через NAT Gateway (дополнительный Cross-AZ charge)
```

**Оптимизация NAT Gateway:**

```bash
# Решение 1: VPC Endpoints для AWS сервисов (без NAT!)
# S3 Gateway Endpoint — БЕСПЛАТНО
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-123 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-123

# ECR Interface Endpoint (платный, но дешевле NAT для большого трафика)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-123 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ecr.api \
  --subnet-ids subnet-123 \
  --security-group-ids sg-123

# DynamoDB Gateway Endpoint — БЕСПЛАТНО
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-123 \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --route-table-ids rtb-123
```

**Data Transfer costs:**

```
Бесплатно:
  - Трафик В AWS (ingress) бесплатен
  - Трафик между сервисами в одной AZ (через private IP) бесплатен
  - S3, CloudFront → интернет: первые 100GB/месяц бесплатно
  - S3 → EC2 в той же Region бесплатно

Платный:
  - EC2/RDS → Internet: $0.09/GB (первые 10TB)
  - Cross-AZ: $0.01/GB (!) — скрытая стоимость
  - Cross-Region: $0.02/GB

Главная ошибка: Multi-AZ сервис с частым cross-AZ трафиком
  Пример: Load Balancer в AZ-A → Pods в AZ-B → Cross-AZ charge
  Решение: использовать Topology Aware Routing в K8s
```

**Topology Aware Routing в Kubernetes:**

```yaml
# Предпочитать pods в той же AZ (уменьшить cross-AZ трафик)
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    service.kubernetes.io/topology-mode: "Auto"
spec:
  selector:
    app: myapp
  ports:
    - port: 80

# kube-proxy будет направлять трафик к pods в той же AZ
# Fallback к другим AZ если нет healthy pods в текущей
```

---

## 10. FinOps практики: tagging strategy, waste detection, governance

**Waste Detection — поиск неиспользуемых ресурсов:**

```bash
# 1. Unattached EBS Volumes
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].{ID:VolumeId,Size:Size,Type:VolumeType,Cost:"?"}'

# 2. Unassociated Elastic IPs (пустые EIP стоят ~$3.6/месяц)
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==`null`].{IP:PublicIp,AllocationId:AllocationId}'

# 3. Old/unused AMIs
aws ec2 describe-images \
  --owners self \
  --query 'Images[?CreationDate<`2023-01-01`].{ID:ImageId,Name:Name,Date:CreationDate}'

# 4. Idle RDS (< 5% CPU за 7 дней)
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --period 604800 \
  --statistics Average \
  --dimensions Name=DBInstanceIdentifier,Value=my-db

# 5. Unused Load Balancers (нет трафика за 7 дней)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --statistics Sum \
  ...
```

**Автоматизация Waste Cleanup:**

```python
# Lambda функция для очистки неиспользуемых ресурсов
import boto3
from datetime import datetime, timedelta

def cleanup_unused_ebs(event, context):
    ec2 = boto3.client('ec2')
    
    # Найти unattached volumes старше 7 дней
    volumes = ec2.describe_volumes(
        Filters=[{'Name': 'status', 'Values': ['available']}]
    )['Volumes']
    
    for vol in volumes:
        create_time = vol['CreateTime'].replace(tzinfo=None)
        age = datetime.utcnow() - create_time
        
        if age.days > 7:
            # Проверить тег preserve
            tags = {t['Key']: t['Value'] for t in vol.get('Tags', [])}
            if tags.get('preserve') != 'true':
                # Создать snapshot перед удалением
                ec2.create_snapshot(
                    VolumeId=vol['VolumeId'],
                    Description=f"Pre-deletion snapshot {datetime.utcnow().date()}"
                )
                ec2.delete_volume(VolumeId=vol['VolumeId'])
                print(f"Deleted volume {vol['VolumeId']} (age: {age.days} days)")
```

**Dev Environment Scheduling:**

```yaml
# AWS Instance Scheduler — выключать dev среды ночью и в выходные
# Экономия: ~65% (8h работают из 24h)

# Тег на инстанс
Key: Schedule
Value: dev-hours  # определён в Instance Scheduler таблице

# DynamoDB конфигурация периодов:
# dev-hours: Mon-Fri 08:00-19:00 в timezone Europe/Moscow
# Остальное время: stopped

# Kubernetes: уменьшать replicas до 0 ночью
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-down-dev
spec:
  schedule: "0 20 * * 1-5"    # 20:00 по будням
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: kubectl
              image: bitnami/kubectl
              command:
                - /bin/sh
                - -c
                - kubectl scale deployment --all -n dev --replicas=0

---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-up-dev
spec:
  schedule: "0 8 * * 1-5"     # 08:00 по будням
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: kubectl
              command:
                - /bin/sh
                - -c
                - kubectl scale deployment --all -n dev --replicas=1
```

**FinOps Governance — организационные практики:**

```
1. Weekly Cost Review
   Platform + Finance + Engineering Leads
   Топ-10 ресурсов по стоимости
   Аномалии за неделю
   Action items с владельцами

2. Cost per Feature / Cost per User
   Связать расходы с бизнес-метриками
   "Эта фича стоит $0.003 на пользователя в день"

3. FinOps Champions
   В каждой команде — FinOps champion
   Отвечает за cost awareness в команде
   Ежемесячный отчёт по команде

4. Infrastructure Cost Reviews в Architecture Decision Records (ADR)
   Новая архитектурная решение = оценка стоимости
   "Добавить Redis кластер = $X/месяц"

5. Unit Economics
   Cost per request, cost per transaction
   Интегрировать в Grafana рядом с latency/error rate
```
