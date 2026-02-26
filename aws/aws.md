# AWS — Вопросы и ответы для собеседований (Middle/Senior DevOps)

## Содержание

### IAM и безопасность
1. [Как работает IAM? Пользователи, группы, роли, политики.](#1-как-работает-iam-пользователи-группы-роли-политики)
2. [Что такое IAM Role и как работает Assume Role / STS?](#2-что-такое-iam-role-и-как-работает-assume-role--sts)
3. [Как устроены IAM Policy? Логика вычисления разрешений.](#3-как-устроены-iam-policy-логика-вычисления-разрешений)

### Сеть (VPC)
4. [Как устроен VPC? Основные компоненты.](#4-как-устроен-vpc-основные-компоненты)
5. [В чём разница между Security Group и Network ACL?](#5-в-чём-разница-между-security-group-и-network-acl)
6. [Как организовать связь между VPC? VPC Peering vs Transit Gateway.](#6-как-организовать-связь-между-vpc-vpc-peering-vs-transit-gateway)

### Вычисления (EC2, Auto Scaling, Lambda)
7. [Какие модели оплаты EC2 существуют и когда какую использовать?](#7-какие-модели-оплаты-ec2-существуют-и-когда-какую-использовать)
8. [Как работает Auto Scaling Group? Типы политик масштабирования.](#8-как-работает-auto-scaling-group-типы-политик-масштабирования)
9. [Что такое Lambda? Cold start и способы его минимизации.](#9-что-такое-lambda-cold-start-и-способы-его-минимизации)

### Балансировка нагрузки
10. [В чём разница между ALB, NLB и GLB?](#10-в-чём-разница-между-alb-nlb-и-glb)

### Хранилище (S3, EBS, EFS)
11. [Классы хранения S3 и управление жизненным циклом объектов.](#11-классы-хранения-s3-и-управление-жизненным-циклом-объектов)
12. [Как обеспечить безопасность S3? Bucket Policy, ACL, Presigned URL.](#12-как-обеспечить-безопасность-s3-bucket-policy-acl-presigned-url)
13. [В чём разница между EBS, EFS и Instance Store?](#13-в-чём-разница-между-ebs-efs-и-instance-store)

### Базы данных (RDS, Aurora, ElastiCache)
14. [Чем отличаются RDS Multi-AZ и Read Replica?](#14-чем-отличаются-rds-multi-az-и-read-replica)
15. [Что такое Aurora и в чём его преимущества перед RDS?](#15-что-такое-aurora-и-в-чём-его-преимущества-перед-rds)

### Контейнеры (ECS, EKS)
16. [Как работает ECS? В чём разница между EC2 и Fargate режимами?](#16-как-работает-ecs-в-чём-разница-между-ec2-и-fargate-режимами)
17. [Как работает EKS? Что такое IRSA?](#17-как-работает-eks-что-такое-irsa)

### Мониторинг и логи
18. [Как устроен CloudWatch? Метрики, логи, алармы.](#18-как-устроен-cloudwatch-метрики-логи-алармы)

### Очереди и события
19. [В чём разница между SQS и SNS? Когда что использовать?](#19-в-чём-разница-между-sqs-и-sns-когда-что-использовать)

### DNS и CDN
20. [Какие политики маршрутизации есть в Route53?](#20-какие-политики-маршрутизации-есть-в-route53)
21. [Как работает CloudFront?](#21-как-работает-cloudfront)

### Безопасность и управление секретами
22. [Как работает KMS? Чем отличается от Secrets Manager и SSM Parameter Store?](#22-как-работает-kms-чем-отличается-от-secrets-manager-и-ssm-parameter-store)

### Оптимизация затрат
23. [Как оптимизировать затраты на AWS?](#23-как-оптимизировать-затраты-на-aws)

---

## IAM и безопасность

### 1. Как работает IAM? Пользователи, группы, роли, политики.

**IAM (Identity and Access Management)** — сервис управления доступом к ресурсам AWS. Глобальный, не привязан к региону.

**Основные сущности:**

**User (пользователь)** — постоянная identity для человека или приложения. Имеет долгосрочные credentials: пароль (консоль) и/или Access Key + Secret Key (API/CLI). Избегай создания пользователей для приложений — используй Roles.

**Group (группа)** — коллекция пользователей. Политики назначаются группе, а не каждому пользователю по отдельности:
```
GroupDevOps
├── Policy: AmazonEKSClusterPolicy
├── Policy: AmazonEC2FullAccess
└── Users: alice, bob, charlie
```

**Role (роль)** — временная identity. Не имеет постоянных credentials. Используется:
- EC2 инстансами, Lambda, ECS задачами, EKS подами (вместо хранения ключей в коде)
- Для cross-account доступа
- Для федеративного доступа (SSO, SAML, OIDC)

**Policy (политика)** — JSON-документ, описывающий разрешения. Типы:
- **Identity-based** — прикреплены к user/group/role
- **Resource-based** — прикреплены к ресурсу (S3 bucket policy, SQS policy)
- **Permission boundary** — максимальный потолок прав, которые может иметь identity
- **SCP (Service Control Policy)** — ограничения на уровне AWS Organization

**Best practices IAM:**
```
✅ Принцип минимальных привилегий
✅ Включить MFA для root и всех IAM пользователей
✅ Никогда не использовать root account для повседневных задач
✅ Ротировать Access Keys регулярно
✅ Использовать Roles вместо Users для приложений
✅ Использовать Groups для управления правами пользователей
✅ Включить CloudTrail для аудита всех API вызовов
```

---

### 2. Что такое IAM Role и как работает Assume Role / STS?

**Assume Role** — процесс получения временных credentials для роли. Выполняется через сервис **STS (Security Token Service)**.

**Схема работы:**
```
1. EC2 инстанс / Lambda / приложение
   └─ вызывает STS: AssumeRole(RoleARN)
   
2. STS проверяет:
   └─ Есть ли у вызывающего права AssumeRole?
   └─ Разрешает ли Trust Policy роли этот вызов?

3. STS возвращает временные credentials:
   └─ AccessKeyId (начинается с "ASIA...")
   └─ SecretAccessKey
   └─ SessionToken
   └─ Expiration (обычно 1 час, максимум 12 часов)

4. Приложение использует их для вызовов AWS API
```

**Trust Policy** — кто может AssumeRole эту роль:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Cross-account Assume Role:**
```
Account A (012345678901) — приложение
  └─ Role в Account A имеет право: sts:AssumeRole на роль в Account B

Account B (987654321098) — S3 bucket
  └─ Trust Policy роли разрешает Account A: sts:AssumeRole

Приложение → AssumeRole → получает credentials Account B → читает S3
```

**Практика — Assume Role через CLI:**
```bash
# Получить временные credentials
aws sts assume-role \
  --role-arn arn:aws:iam::987654321098:role/ReadS3Role \
  --role-session-name MySession

# Экспортировать credentials
export AWS_ACCESS_KEY_ID=ASIAxxx
export AWS_SECRET_ACCESS_KEY=xxx
export AWS_SESSION_TOKEN=xxx

# Проверить кем являемся
aws sts get-caller-identity
```

**Instance Profile** — механизм автоматического получения credentials для EC2:
```
EC2 → запрашивает credentials с IMDS (169.254.169.254)
    → IMDS возвращает временные credentials роли
    → SDK автоматически обновляет credentials до истечения
```

---

### 3. Как устроены IAM Policy? Логика вычисления разрешений.

**Структура Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "s3:prefix": ["logs/", "data/"]
        },
        "IpAddress": {
          "aws:SourceIp": ["192.168.1.0/24"]
        }
      }
    },
    {
      "Sid": "DenyDeleteObjects",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

**Алгоритм вычисления разрешений (порядок важен):**
```
1. Explicit DENY → всегда запрещает (наивысший приоритет)
   └─ Если есть хоть одно явное Deny — отказ

2. SCP (Organizations) → если действие не разрешено SCP — отказ

3. Permission Boundary → если установлена, ограничивает максимальные права

4. Explicit ALLOW → если есть хотя бы одно Allow — разрешить

5. Implicit DENY → если никакого Allow нет — отказ (default)
```

**Практические примеры Condition:**
```json
"Condition": {
  // Разрешить только из конкретного VPC Endpoint
  "StringEquals": {
    "aws:SourceVpce": "vpce-1234567890abcdef0"
  },
  // Только при использовании MFA
  "Bool": {
    "aws:MultiFactorAuthPresent": "true"
  },
  // Только при шифровании SSE-KMS
  "StringEquals": {
    "s3:x-amz-server-side-encryption": "aws:kms"
  },
  // Запретить без тегов
  "StringEquals": {
    "aws:RequestedRegion": "eu-west-1"
  }
}
```

**Wildcards в ARN:**
```
arn:aws:s3:::my-bucket/*          — все объекты в bucket
arn:aws:ec2:*:*:instance/*        — все EC2 во всех регионах
arn:aws:iam::123456789:role/Dev-*  — все роли начинающиеся с Dev-
```

---

## Сеть (VPC)

### 4. Как устроен VPC? Основные компоненты.

**VPC (Virtual Private Cloud)** — изолированная виртуальная сеть в AWS, привязанная к региону. Полный контроль над сетевым пространством.

```
VPC: 10.0.0.0/16 (регион: eu-west-1)
│
├── Availability Zone A (eu-west-1a)
│   ├── Public Subnet:  10.0.1.0/24  ←→ Internet Gateway
│   └── Private Subnet: 10.0.2.0/24  ←→ NAT Gateway
│
├── Availability Zone B (eu-west-1b)
│   ├── Public Subnet:  10.0.3.0/24
│   └── Private Subnet: 10.0.4.0/24
│
└── Availability Zone C (eu-west-1c)
    ├── Public Subnet:  10.0.5.0/24
    └── Private Subnet: 10.0.6.0/24
```

**Ключевые компоненты:**

**Internet Gateway (IGW)** — позволяет ресурсам в public subnet'ах общаться с интернетом. Один на VPC. Горизонтально масштабируется, HA по умолчанию. Ресурс должен иметь публичный IP.

**NAT Gateway** — позволяет ресурсам в private subnet'ах инициировать исходящий трафик в интернет (для обновлений, скачивания пакетов), не будучи доступными снаружи. Деплоится в public subnet. Стоит денег (~$0.045/час + $0.045/GB). Нужен в каждой AZ для отказоустойчивости.

**Route Table** — таблица маршрутизации. Каждый subnet ассоциирован с одной route table:
```
Public subnet route table:
  0.0.0.0/0 → igw-xxx (весь трафик → IGW)
  10.0.0.0/16 → local

Private subnet route table:
  0.0.0.0/0 → nat-xxx (весь трафик → NAT Gateway)
  10.0.0.0/16 → local
```

**Public vs Private Subnet:**
Разница только в route table — есть ли маршрут к IGW. Subnet сам по себе не "публичный" или "приватный".

**VPC Endpoints** — приватное подключение к AWS сервисам без выхода в интернет:
```
Gateway Endpoint (бесплатно):
  S3, DynamoDB → маршрут в route table → endpoint

Interface Endpoint (PrivateLink, платно):
  Большинство сервисов (SSM, Secrets Manager, ECR, etc.)
  → ENI в subnet → DNS резолвится на приватный IP
```

**Зачем VPC Endpoints:**
- Трафик к S3/DynamoDB не идёт через NAT Gateway (экономия)
- Повышенная безопасность (нет выхода в интернет)
- Bucket policy может требовать доступ только через endpoint

---

### 5. В чём разница между Security Group и Network ACL?

| Характеристика | Security Group | Network ACL |
|---|---|---|
| Уровень | Экземпляр (ENI) | Subnet |
| Stateful/Stateless | Stateful | Stateless |
| Правила | Allow only | Allow + Deny |
| Применение | Explicit attach | Автоматически к subnet |
| Порядок правил | Все проверяются | Нумерованные, первое совпадение |

**Security Group (Stateful):**
Stateful означает: если разрешён входящий трафик, ответный исходящий разрешается автоматически. Не нужно писать правило для ответа.

```
Inbound Rules:
  TCP 80   0.0.0.0/0   Allow   ← HTTP
  TCP 443  0.0.0.0/0   Allow   ← HTTPS
  TCP 22   10.0.0.0/8  Allow   ← SSH только из internal

Outbound Rules (чаще всего):
  All traffic  0.0.0.0/0  Allow  ← разрешить всё исходящее
```

**Лучшая практика — ссылаться на другие SG вместо IP:**
```
# DB Security Group
Inbound:
  TCP 5432  источник: sg-app-servers  Allow
  # разрешить только от инстансов с тегом sg-app-servers
```

**Network ACL (Stateless):**
Stateless: нужно явно разрешать как входящий запрос, так и ответный трафик. Ответы идут с эфемерными портами (1024-65535).

```
Inbound:
  100  TCP  80    0.0.0.0/0    Allow
  110  TCP  443   0.0.0.0/0    Allow
  900  TCP  1024-65535  0.0.0.0/0  Allow  ← эфемерные порты для ответов
  *    All  All   0.0.0.0/0    Deny   ← implicit deny

Outbound:
  100  TCP  1024-65535  0.0.0.0/0  Allow  ← ответы клиентам
  200  TCP  443  0.0.0.0/0  Allow  ← исходящий HTTPS
  *    All  All  0.0.0.0/0  Deny
```

**Когда использовать NACL:**
- Быстрая блокировка IP-диапазонов (DDoS, abuse)
- Дополнительный уровень защиты помимо SG
- Блокировка трафика на уровне subnet (SG можно случайно изменить)

---

### 6. Как организовать связь между VPC? VPC Peering vs Transit Gateway.

**VPC Peering** — прямое сетевое соединение между двумя VPC (в одном или разных аккаунтах/регионах). Трафик идёт через внутреннюю сеть AWS, не через интернет.

```
VPC A (10.0.0.0/16) ←──── Peering ────→ VPC B (10.1.0.0/16)
```

Ограничения пиринга:
- Нет транзитивного роутинга: A↔B и B↔C не означает A↔C
- CIDR не должны перекрываться
- При N VPC нужно N*(N-1)/2 пирингов (для 10 VPC — 45 пирингов)

**Transit Gateway (TGW)** — хаб для подключения множества VPC и on-premises сетей:

```
VPC A ─┐
VPC B ─┤
VPC C ─┼─── Transit Gateway ─── VPN / Direct Connect (on-premises)
VPC D ─┤
VPC E ─┘
```

- Транзитивный роутинг работает
- Одно место управления маршрутизацией (Route Tables в TGW)
- Поддерживает VPN и Direct Connect attachment
- Межрегиональный пиринг TGW
- Стоит денег (~$0.05/час + $0.02/GB)

**Когда что использовать:**

| Сценарий | Решение |
|---|---|
| 2-3 VPC, простая топология | VPC Peering |
| Много VPC (hub-and-spoke) | Transit Gateway |
| On-premises + много VPC | Transit Gateway |
| Минимизация стоимости | VPC Peering |
| Централизованный firewall | Transit Gateway + Gateway LB |

**PrivateLink** — ещё один вариант: экспортировать сервис из VPC через NLB + Endpoint Service, клиенты подключаются через Interface Endpoint. Трафик однонаправленный, CIDR могут перекрываться.

---

## Вычисления (EC2, Auto Scaling, Lambda)

### 7. Какие модели оплаты EC2 существуют и когда какую использовать?

**On-Demand**
Оплата по часам/секундам без обязательств. Максимальная гибкость, максимальная цена.
- **Когда:** dev/test, непредсказуемые нагрузки, кратковременные задачи

**Reserved Instances (RI)**
Резервирование на 1 или 3 года. Скидка 30-72% по сравнению с On-Demand.
- **Standard RI** — нельзя менять тип инстанса, максимальная скидка
- **Convertible RI** — можно менять семейство инстансов, скидка меньше
- **Когда:** стабильная baseline нагрузка (production БД, базовое число web-серверов)

**Savings Plans**
Обязательство по объёму использования ($/час) на 1 или 3 года. Гибче RI — применяется автоматически к любым инстансам EC2 в регионе, Lambda, Fargate.
- **Когда:** предпочтительнее RI для большинства случаев, т.к. гибче

**Spot Instances**
Свободные мощности AWS с дисконтом до 90%. Могут быть прерваны с уведомлением за 2 минуты.
- **Когда:** batch-обработка данных, CI/CD воркеры, stateless микросервисы, ML обучение
- **Не использовать:** для БД, statefull приложений, критичных сервисов без fallback

**Dedicated Hosts / Instances**
Физический сервер выделен только вам. Для лицензионных требований (BYOL), compliance.

**Практическая стратегия для production:**
```
ASG смешанного типа (Mixed Instances Policy):
  On-Demand base: 2 инстанса   ← гарантированный минимум
  Spot:           70%           ← экономия на масштабировании
  On-Demand:      30%           ← fallback если Spot недоступен

Инстанс-типы в ASG (диверсификация для Spot):
  - m5.large, m5a.large, m4.large, m5d.large
  (разные семейства = меньше шанс одновременного прерывания)
```

---

### 8. Как работает Auto Scaling Group? Типы политик масштабирования.

**ASG (Auto Scaling Group)** — автоматически поддерживает нужное количество EC2 инстансов на основе условий.

**Компоненты:**
- **Launch Template** — что запускать (AMI, тип инстанса, SG, IAM role, user data)
- **ASG Configuration** — сколько (min/max/desired), где (subnets), health checks
- **Scaling Policies** — когда и как масштабировать

**User Data — скрипт при старте инстанса:**
```bash
#!/bin/bash
yum update -y
yum install -y nginx
systemctl enable nginx
systemctl start nginx
echo "Hello from $(hostname)" > /var/www/html/index.html
```

**Типы политик масштабирования:**

**1. Target Tracking (рекомендуется)**
Поддерживает метрику на заданном уровне — аналог термостата:
```json
{
  "TargetValue": 60.0,
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ASGAverageCPUUtilization"
  }
}
```
Поддерживаемые метрики: CPU, Network In/Out, ALB Request Count per Target.

**2. Step Scaling**
Разные шаги масштабирования в зависимости от величины нарушения:
```
CPU 60-70%: добавить 1 инстанс
CPU 70-85%: добавить 2 инстанса
CPU >85%:   добавить 4 инстанса
```

**3. Scheduled Scaling**
Масштабирование по расписанию (предсказуемые паттерны нагрузки):
```bash
# Поднять до 10 инстансов утром в будни
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name my-asg \
  --scheduled-action-name scale-up-morning \
  --recurrence "0 8 * * MON-FRI" \
  --desired-capacity 10 \
  --min-size 5
```

**4. Predictive Scaling**
ML-модель анализирует исторические паттерны и масштабирует превентивно, до роста нагрузки.

**Health Checks:**
- **EC2 health check** — инстанс running и проходит системные проверки
- **ELB health check** — ALB/NLB сообщает что инстанс healthy (рекомендуется)
- При unhealthy — инстанс заменяется (terminate + launch новый)

**Cooldown period** — время после масштабирования, когда новые политики не срабатывают. Default: 300 секунд. Позволяет новым инстансам запуститься и взять нагрузку.

---

### 9. Что такое Lambda? Cold start и способы его минимизации.

**AWS Lambda** — serverless вычисления. Запускаешь функцию — платишь только за время выполнения (с точностью до 1 мс). Не нужно управлять серверами.

**Параметры:**
- Память: 128 МБ — 10 ГБ (CPU пропорционален памяти)
- Timeout: max 15 минут
- Размер пакета: 50 МБ (zip), 250 МБ (unzipped), 10 ГБ (container image)
- Concurrency: 1000 по умолчанию (мягкий лимит на аккаунт)

**Жизненный цикл исполнения:**
```
INIT фаза (один раз при создании/обновлении):
  └─ Скачать код/образ
  └─ Запустить runtime (Python/Node/Java/etc.)
  └─ Выполнить код вне handler (глобальные переменные, DB connections)

INVOKE фаза (каждый вызов):
  └─ Выполнить handler функции
  └─ Вернуть результат

SHUTDOWN фаза:
  └─ Если нет вызовов ~15 минут → контейнер "замораживается"
```

**Cold Start** — задержка при первом вызове после периода простоя. Lambda должна поднять новый контейнер (INIT фаза). Типичное время:
- Python/Node.js: 100-500 мс
- Java/Kotlin (JVM): 1-10 сек (JVM initialization)
- Container image: может быть дольше

**Способы минимизации Cold Start:**

**1. Provisioned Concurrency** — держать N "тёплых" контейнеров постоянно:
```bash
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier prod \
  --provisioned-concurrent-executions 10
# Стоит денег даже без вызовов ($0.015 per GB-hour)
```

**2. Оптимизация INIT кода:**
```python
import boto3

# Это выполняется при INIT (один раз) — OK
db_client = boto3.client('dynamodb')
s3_client = boto3.client('s3')

def handler(event, context):
    # Это выполняется при каждом вызове
    return db_client.get_item(...)
```

**3. Выбор runtime с малым cold start:**
- Python, Node.js — быстрый старт
- Java — медленный (JVM). Решение: GraalVM native image, SnapStart (Lambda Java 11+)

**4. Lambda SnapStart (для Java)**
Создаёт snapshot состояния после INIT и восстанавливает из него. Cold start ~1 сек вместо 10+.

**5. Уменьшить размер deployment package:**
```bash
# Только production зависимости
pip install -r requirements.txt -t ./package --no-dev
# Lambda Layers для общих зависимостей
```

**Ключевые триггеры Lambda:**
- API Gateway / Function URL — HTTP запросы
- SQS / SNS / Kinesis — обработка очередей
- S3 Events — реакция на загрузку файлов
- CloudWatch Events / EventBridge — по расписанию
- DynamoDB Streams — реакция на изменения в БД

---

## Балансировка нагрузки

### 10. В чём разница между ALB, NLB и GLB?

**ALB (Application Load Balancer)** — L7, HTTP/HTTPS/gRPC
- Маршрутизация по URL path, Host header, HTTP method, query params, JWT claims
- Sticky sessions через cookies
- WebSocket поддержка
- Интеграция с WAF, Cognito (аутентификация)
- Target types: EC2, IP, Lambda, другой ALB (nested)
- Для: web-приложений, API, микросервисов

```
ALB Listener Rules:
  IF path=/api/* → forward → target-group-api
  IF host=admin.example.com → forward → target-group-admin
  IF path=/health → fixed-response → 200 "OK"
  DEFAULT → forward → target-group-main
```

**NLB (Network Load Balancer)** — L4, TCP/UDP/TLS
- Статический IP (или Elastic IP) на каждую AZ — важно для whitelist
- Экстремальная производительность: миллионы запросов в секунду, microsecond latency
- Preserve Client IP без X-Forwarded-For (реальный IP видит backend)
- Поддержка TCP Proxy protocol
- Для: gaming, IoT, финансовые системы, когда нужен статический IP или максимальная производительность

**GLB (Gateway Load Balancer)** — L3, IP packets
- Для развёртывания third-party virtual network appliances (firewall, IDS/IPS, DPI)
- Трафик прозрачно проходит через appliances
- Использует GENEVE протокол (encapsulation)
- Для: centralized security inspection

**Сравнение:**

| | ALB | NLB | GLB |
|---|---|---|---|
| OSI Layer | 7 | 4 | 3 |
| Протоколы | HTTP/HTTPS/gRPC | TCP/UDP/TLS | IP |
| Статический IP | Нет (DNS) | Да | Да |
| Content routing | Да | Нет | Нет |
| Client IP | X-Forwarded-For | Preserve | Preserve |
| Use case | Web apps | Low latency, static IP | Network appliances |

---

## Хранилище (S3, EBS, EFS)

### 11. Классы хранения S3 и управление жизненным циклом объектов.

**Классы хранения S3:**

| Класс | Доступность | Latency | Цена | Применение |
|---|---|---|---|---|
| **Standard** | 99.99% | мс | $$$ | Часто используемые данные |
| **Intelligent-Tiering** | 99.9% | мс | $$+ | Непредсказуемый паттерн доступа |
| **Standard-IA** | 99.9% | мс | $$ | Редкий доступ, быстрое извлечение |
| **One Zone-IA** | 99.5% | мс | $ | Редкий доступ, одна AZ, некритичные данные |
| **Glacier Instant** | 99.9% | мс | $ | Архив, доступ несколько раз в год |
| **Glacier Flexible** | 99.99% | мин-часы | ¢ | Долгосрочный архив, извлечение за минуты |
| **Glacier Deep Archive** | 99.99% | 12-48 часов | ¢¢ | Самый дешёвый архив, редкое извлечение |

> S3 обеспечивает **11 девяток** (99.999999999%) durability для всех классов — хранит данные в минимум 3 AZ (кроме One Zone-IA).

**Lifecycle Policy — автоматический переход между классами:**
```json
{
  "Rules": [
    {
      "ID": "archive-old-logs",
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
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30
      }
    }
  ]
}
```

**Versioning и Replication:**
```bash
# Включить версионирование
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

# Cross-Region Replication (CRR) — DR, compliance
# Same-Region Replication (SRR) — aggregation logs из разных аккаунтов
```

---

### 12. Как обеспечить безопасность S3? Bucket Policy, ACL, Presigned URL.

**Bucket Policy — resource-based policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificRole",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/AppRole"
      },
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyNonSSL",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

**Block Public Access** — настройка, блокирующая все публичные разрешения (включить на уровне аккаунта и bucket):
```bash
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

**Presigned URL** — временная ссылка для скачивания/загрузки без AWS credentials:
```python
import boto3

s3 = boto3.client('s3')

# Presigned URL для скачивания (GET)
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'private/report.pdf'},
    ExpiresIn=3600   # 1 час
)

# Presigned URL для загрузки (PUT) — пользователь загружает напрямую в S3
url = s3.generate_presigned_url(
    'put_object',
    Params={
        'Bucket': 'my-bucket',
        'Key': 'uploads/user123/photo.jpg',
        'ContentType': 'image/jpeg'
    },
    ExpiresIn=300  # 5 минут
)
```

**Шифрование в покое:**
- **SSE-S3** — ключи управляются AWS (AES-256)
- **SSE-KMS** — ключи в KMS, аудит в CloudTrail, можно кастомные ключи
- **SSE-C** — клиент передаёт ключ с каждым запросом
- **Client-side encryption** — шифрование до отправки

---

### 13. В чём разница между EBS, EFS и Instance Store?

| | EBS | EFS | Instance Store |
|---|---|---|---|
| Тип | Block storage | File system (NFS) | Block storage |
| Persistence | Да | Да | Нет (эфемерный) |
| Multi-attach | Нет (кроме io2 Multi-Attach) | Да (тысячи инстансов) | Нет |
| Регион/AZ | Один AZ | Регион (multi-AZ) | Только конкретный инстанс |
| Производительность | До 256K IOPS (io2 BE) | Throughput-optimized | Наивысшая (физический диск) |
| Применение | OS диск, БД | Shared storage | Буфер, кэш, scratch |

**EBS типы:**
```
gp3 — General Purpose SSD. Default choice.
      3000 IOPS (baseline), до 16000 IOPS независимо от размера.
      Дешевле gp2 на 20%.

io2 Block Express — для критичных БД (Oracle, SQL Server).
      До 256K IOPS, 4K IOPS/GB.
      Multi-Attach: один том к нескольким инстансам (нужна кластерная ФС).

st1 — Throughput Optimized HDD. Большие данные, data warehouse.
      Дешевле SSD, высокий throughput, низкие IOPS.

sc1 — Cold HDD. Редко читаемые данные. Самый дешёвый.
```

**EFS — когда нужен shared access:**
- Несколько EC2/EKS/ECS задач читают/пишут одни файлы
- Home директории пользователей
- CMS (WordPress, Drupal) с несколькими инстансами
- Performance Modes: General Purpose (latency-sensitive) / Max I/O (высокий параллелизм)
- Throughput Modes: Elastic (autoscaling) / Provisioned

---

## Базы данных (RDS, Aurora, ElastiCache)

### 14. Чем отличаются RDS Multi-AZ и Read Replica?

**Multi-AZ** — для **High Availability**, защита от отказа:

```
AZ-a: Primary Instance    ──┐ синхронная
       (читает + пишет)      │ репликация
AZ-b: Standby Instance   ──┘
       (не обслуживает запросы)

При отказе Primary:
  DNS CNAME переключается на Standby за ~1-2 мин
  Standby становится новым Primary
```

- Standby недоступен для чтения/записи клиентами
- Failover автоматический (~60-120 секунд)
- Синхронная репликация — данные не теряются

**Read Replica** — для **масштабирования чтения**:

```
Primary (пишет) ──── асинхронная ────→ Read Replica 1 (читает)
                     репликация  ────→ Read Replica 2 (читает)
                                 ────→ Read Replica 3 (другой регион — Cross-Region)
```

- До 5 реплик на MySQL/PostgreSQL, до 15 на Aurora
- Асинхронная репликация — небольшой lag (replication lag)
- Можно промотировать в standalone instance
- Cross-Region реплика — для DR и снижения latency для пользователей в другом регионе

**Когда что использовать:**

| Цель | Решение |
|---|---|
| Пережить отказ инстанса/AZ | Multi-AZ |
| Снять нагрузку read-heavy приложений | Read Replica |
| Disaster Recovery в другом регионе | Cross-Region Read Replica |
| Масштабировать и HA вместе | Aurora (встроено) |

---

### 15. Что такое Aurora и в чём его преимущества перед RDS?

**Aurora** — облачная реляционная БД AWS, совместимая с MySQL и PostgreSQL. Переосмыслена архитектурно: вычисления и хранилище разделены.

**Архитектура Aurora:**
```
Writer Instance  ──────────────────────────┐
Reader Instance 1 ─────────────────────────┤
Reader Instance 2 ─────────────────────────┤
                                           ▼
                    Distributed Storage (6 копий в 3 AZ)
                    ─────────────────────────────────────
                    AZ-a: [copy1] [copy2]
                    AZ-b: [copy3] [copy4]
                    AZ-c: [copy5] [copy6]
```

**Преимущества Aurora vs RDS:**

| | RDS (MySQL/PG) | Aurora |
|---|---|---|
| Failover | 60-120 сек | < 30 сек (обычно ~10 сек) |
| Read Replicas | До 5 | До 15 |
| Storage | Фиксированный том | Auto-grows до 128 ТБ |
| Storage реплики | Реплицирует данные | Разделяет хранилище |
| Replication lag | Секунды | Миллисекунды |
| Backup | Ежедневные snapshots | Continuous (PITR до 1 минуты) |
| Производительность | Baseline | До 5x MySQL, 3x PostgreSQL |

**Aurora Serverless v2** — автоматически масштабирует compute от 0.5 до 128 ACU (Aurora Capacity Units) за секунды. Идеально для нерегулярной нагрузки, dev/test окружений.

**Aurora Global Database** — одна БД в нескольких регионах:
- Репликация lag < 1 секунда
- Failover между регионами за < 1 минуты
- Для: global applications, DR с минимальным RPO

---

## Контейнеры (ECS, EKS)

### 16. Как работает ECS? В чём разница между EC2 и Fargate режимами?

**ECS (Elastic Container Service)** — managed сервис для запуска контейнеров. Основные понятия:

- **Task Definition** — шаблон (аналог docker-compose или K8s Pod): образы, CPU/memory, volumes, IAM role, network mode
- **Task** — запущенный экземпляр Task Definition (аналог Pod)
- **Service** — поддерживает нужное количество Task, интегрируется с ALB, обеспечивает rolling update
- **Cluster** — логическая группа задач и сервисов

**EC2 Launch Type:**
```
ECS Cluster
└── EC2 Instances (управляешь сам: AMI, тип инстанса, ASG)
    └── ECS Agent (daemon на каждом инстансе)
        └── Tasks (контейнеры)
```
- Полный контроль над инфраструктурой
- Экономичнее при постоянной высокой нагрузке
- Подходит для GPU workloads
- Сам управляешь патчингом, capacity

**Fargate Launch Type:**
```
ECS/EKS Cluster
└── Fargate (AWS управляет инфраструктурой)
    └── Tasks/Pods (контейнеры) — каждая задача в изолированном microVM
```
- Нет инстансов для управления — serverless контейнеры
- Оплата только за vCPU и memory задачи
- Автоматическая изоляция безопасности (каждая задача в отдельном ядре)
- Подходит: микросервисы, batch jobs, нерегулярная нагрузка

**Пример Task Definition:**
```json
{
  "family": "web-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::...:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::...:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "123456789.dkr.ecr.eu-west-1.amazonaws.com/web-app:latest",
      "portMappings": [{"containerPort": 8080}],
      "environment": [
        {"name": "ENV", "value": "production"}
      ],
      "secrets": [
        {"name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:...:secret:db-password"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/web-app",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      }
    }
  ]
}
```

---

### 17. Как работает EKS? Что такое IRSA?

**EKS (Elastic Kubernetes Service)** — managed Kubernetes. AWS управляет control plane (API Server, etcd, scheduler, controllers), ты — worker nodes.

**Варианты worker nodes:**
- **Managed Node Groups** — AWS создаёт и управляет EC2 (AMI обновления, drain при замене)
- **Self-managed Nodes** — полный контроль, своя AMI, нужно самому управлять
- **Fargate Profiles** — serverless поды, без управления узлами

**IRSA (IAM Roles for Service Accounts)** — механизм назначения IAM Role конкретным Kubernetes ServiceAccount без хранения ключей.

**Как работает:**
```
1. EKS кластер имеет OIDC Provider endpoint
   (https://oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE)

2. Создаём IAM Role с Trust Policy, доверяющей OIDC Provider:
{
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks..."
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLE:sub":
        "system:serviceaccount:production:my-app-sa"
    }
  }
}

3. Аннотируем ServiceAccount:
   eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/MyAppRole

4. Pod с этим ServiceAccount получает projected JWT token
   → AWS SDK обменивает его на временные credentials через STS
   → Pod работает с правами IAM Role
```

**Создание IRSA:**
```bash
# Создать OIDC provider для кластера
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve

# Создать IAM Role + ServiceAccount одной командой
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace production \
  --name my-app-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

**Почему IRSA лучше Node IAM Role:**
- Права на уровне пода, а не всего узла
- Если под взломан — доступ только к его правам, не ко всему узлу
- Полный аудит в CloudTrail (кто и что запрашивал)

---

## Мониторинг и логи

### 18. Как устроен CloudWatch? Метрики, логи, алармы.

**CloudWatch** — платформа наблюдаемости AWS. Включает метрики, логи, события, трейсы (через X-Ray).

**Metrics:**
```bash
# Все AWS сервисы автоматически публикуют метрики
# EC2: CPUUtilization, NetworkIn/Out, DiskRead/WriteOps
# RDS: DatabaseConnections, ReadLatency, FreeStorageSpace
# ALB: RequestCount, TargetResponseTime, HTTPCode_ELB_5XX_Count

# Детализация (Resolution):
# Standard: 1 минута (бесплатно)
# High-resolution: 1 секунда (Custom Metrics, платно)

# Публикация кастомной метрики
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "OrdersProcessed" \
  --value 42 \
  --dimensions Environment=Production
```

**Alarms (алармы):**
```bash
# Аларм на CPU > 80% в течение 5 минут
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-web-asg" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=AutoScalingGroupName,Value=web-asg \
  --alarm-actions arn:aws:sns:...:alerts-topic \
  --ok-actions arn:aws:sns:...:alerts-topic \
  --treat-missing-data notBreaching
```

**Состояния аларма:** `OK` → `ALARM` → `INSUFFICIENT_DATA`

**Composite Alarm** — аларм из алармов:
```bash
# Сработать только если И CPU высокое И memory высокое
aws cloudwatch put-composite-alarm \
  --alarm-name "CriticalLoad" \
  --alarm-rule "ALARM(HighCPU) AND ALARM(HighMemory)"
```

**CloudWatch Logs:**
```bash
# Log Groups → Log Streams → Log Events

# Создать Log Group с retention
aws logs create-log-group --log-group-name /app/production
aws logs put-retention-policy \
  --log-group-name /app/production \
  --retention-in-days 30

# Metric Filter — извлечь метрику из логов
aws logs put-metric-filter \
  --log-group-name /app/production \
  --filter-name "ErrorCount" \
  --filter-pattern "[timestamp, level=ERROR, ...]" \
  --metric-transformations \
    metricName=ErrorCount,metricNamespace=MyApp,metricValue=1

# CloudWatch Logs Insights — SQL-подобные запросы
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as errorCount by bin(5m)
| sort errorCount desc
```

**Container Insights** — метрики и логи для ECS/EKS:
```yaml
# EKS: установить CloudWatch Agent как DaemonSet
# Собирает: CPU/memory/disk per pod, container restarts, etc.
```

---

## Очереди и события

### 19. В чём разница между SQS и SNS? Когда что использовать?

**SQS (Simple Queue Service)** — очередь сообщений. Отправители и получатели разделены. Получатель сам забирает (pull) сообщения.

```
Producer → [SQS Queue] → Consumer 1
                       ← (polling)
# Одно сообщение обрабатывается ОДНИМ получателем
```

**Типы очередей SQS:**
- **Standard** — at-least-once delivery, порядок не гарантирован, nearly-unlimited throughput
- **FIFO** — exactly-once, строгий порядок, 3000 msg/sec (с batching)

**Ключевые параметры SQS:**
```
Visibility Timeout (default 30s):
  После получения сообщение скрывается на это время.
  Если обработка не завершена — сообщение становится видимым снова.
  Устанавливай >= времени обработки.

Message Retention Period (default 4 дня, max 14 дней):
  Как долго хранить необработанные сообщения.

Dead Letter Queue (DLQ):
  После N неудачных попыток обработки → переместить в DLQ.
  Позволяет анализировать "яды" (poison messages).

Long Polling (WaitTimeSeconds=20):
  Ждать до 20 секунд если очередь пуста (уменьшает пустые запросы и стоимость).
```

**SNS (Simple Notification Service)** — pub/sub. Одно сообщение → все подписчики одновременно (push, fan-out).

```
Publisher → [SNS Topic] → SQS Queue 1 (обработка заказов)
                        → SQS Queue 2 (уведомления)
                        → Lambda (аналитика)
                        → Email/SMS/HTTP Webhook
# Одно сообщение получают ВСЕ подписчики
```

**Fan-out паттерн (SNS → несколько SQS):**
```
Event: "Order Placed"
  └─ SNS Topic: order-events
      ├─ SQS: inventory-service    (уменьшить остатки)
      ├─ SQS: notification-service (email покупателю)
      ├─ SQS: analytics-service    (записать в БД аналитики)
      └─ Lambda: fraud-check       (проверка на мошенничество)
```

**EventBridge** (бывший CloudWatch Events) — более мощный аналог SNS для event-driven архитектур:
- Schema Registry — типизированные события
- Content-based routing (фильтрация по содержимому событий)
- Интеграция со сторонними сервисами (Datadog, Zendesk, etc.)
- Event replay — повторная обработка прошлых событий

---

## DNS и CDN

### 20. Какие политики маршрутизации есть в Route53?

**Route53** — managed DNS сервис AWS. 100% SLA доступности.

**Simple** — обычная DNS запись. Для одного ресурса:
```
example.com → 93.184.216.34
```

**Weighted** — распределение трафика по весу. Canary deployments, A/B тесты:
```
example.com (weight=90) → 10.0.0.1 (v1)
example.com (weight=10) → 10.0.0.2 (v2, canary)
```

**Latency-based** — направить пользователя в регион с наименьшей задержкой:
```
example.com → eu-west-1 (для европейских пользователей)
example.com → us-east-1 (для американских пользователей)
```

**Failover** — Active-Passive. Переключение при недоступности primary:
```
example.com (PRIMARY) → production-alb.eu-west-1.elb.amazonaws.com
example.com (SECONDARY) → dr-alb.eu-central-1.elb.amazonaws.com
# Secondary используется только если PRIMARY unhealthy
```

**Geolocation** — маршрутизация по местоположению пользователя:
```
Continent: EU → eu-servers.example.com
Continent: NA → us-servers.example.com
Default → global-servers.example.com
# Для compliance (GDPR — европейские данные в EU)
```

**Geoproximity** (только Traffic Flow) — как geolocation, но с Bias: сдвигать границу ближе/дальше к ресурсу.

**Multi-value Answer** — возвращать несколько IP с health checks (простейший LB на DNS уровне):
```
example.com → [10.0.0.1, 10.0.0.2, 10.0.0.3]
# Возвращает только healthy записи (до 8)
```

**Health Checks в Route53:**
- HTTP/HTTPS/TCP проверки каждые 10-30 секунд
- Интеграция с CloudWatch алармами
- Calculated health checks: AND/OR из нескольких проверок

---

### 21. Как работает CloudFront?

**CloudFront** — CDN (Content Delivery Network) AWS. Кэширует контент в 400+ Edge Locations по всему миру. Снижает latency и нагрузку на origin.

**Архитектура:**
```
Пользователь (Берлин)
    ↓ DNS → ближайший Edge Location (Франкфурт)
    ↓ Cache HIT → мгновенный ответ (< 1мс)
    ↓ Cache MISS → запрос к Origin (S3, ALB, EC2, custom)
                   → кэшируется в Edge Location
                   → ответ пользователю
```

**Distribution — основная конфигурация:**
```
Origins:
  - S3 bucket (статика)
  - ALB (динамика)
  - Custom HTTP origin

Behaviors (по path):
  /api/*    → Origin: ALB,       Cache: None, TTL=0
  /static/* → Origin: S3,        Cache: Enabled, TTL=31536000 (1 год)
  /*        → Origin: ALB,       Cache: Short, TTL=60
```

**Ключевые настройки:**

**Cache Key** — что определяет уникальность кэш-записи:
```
Default: только URL path
Custom: URL + конкретные headers (Accept-Language) + query strings (?v=123)
```

**Origin Access Control (OAC)** — S3 bucket закрыт от прямого доступа, только через CloudFront:
```json
// S3 Bucket Policy разрешает только CloudFront
{
  "Principal": {
    "Service": "cloudfront.amazonaws.com"
  },
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
    }
  }
}
```

**Lambda@Edge / CloudFront Functions** — выполнять код на Edge:
```javascript
// CloudFront Function — добавить security headers к каждому ответу
function handler(event) {
  var response = event.response;
  var headers = response.headers;
  headers['strict-transport-security'] = {value: 'max-age=63072000'};
  headers['x-content-type-options'] = {value: 'nosniff'};
  headers['x-frame-options'] = {value: 'DENY'};
  return response;
}
```

**Invalidation** — принудительно сбросить кэш:
```bash
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/static/app.js" "/static/style.css"
  # или /* для всего (платно: $0.005 за path, первые 1000/month бесплатно)
```

---

## Безопасность и управление секретами

### 22. Как работает KMS? Чем отличается от Secrets Manager и SSM Parameter Store?

**KMS (Key Management Service)** — сервис управления криптографическими ключами. Все операции шифрования/расшифровки происходят **внутри KMS** — ключ никогда не покидает HSM.

**Типы ключей KMS:**
- **AWS Managed Key** — создаёт и ротирует AWS (для S3, RDS, EBS). Нет прямого контроля.
- **Customer Managed Key (CMK)** — создаёшь сам, полный контроль над политикой, ротацией
- **Customer Provided Key** — ты генерируешь ключевой материал, импортируешь в KMS

**Envelope Encryption — как KMS шифрует данные:**
```
Данные могут быть большими, но KMS шифрует только до 4 КБ.
Поэтому:

1. KMS генерирует DEK (Data Encryption Key)
2. Данные шифруются DEK (локально, быстро)
3. DEK шифруется CMK в KMS → Encrypted DEK
4. Хранится: зашифрованные данные + Encrypted DEK

Расшифровка:
1. KMS расшифровывает Encrypted DEK → DEK (только если есть права!)
2. DEK расшифровывает данные
```

```bash
# Зашифровать данные
aws kms encrypt \
  --key-id arn:aws:kms:eu-west-1:123456789:key/mrk-xxx \
  --plaintext fileb://secret.txt \
  --output text --query CiphertextBlob | base64 --decode > secret.enc

# Расшифровать
aws kms decrypt \
  --ciphertext-blob fileb://secret.enc \
  --output text --query Plaintext | base64 --decode
```

**Secrets Manager** — хранение и автоматическая ротация секретов (пароли, API ключи, DB credentials):
```python
import boto3, json

client = boto3.client('secretsmanager')
response = client.get_secret_value(SecretId='prod/myapp/db')
secret = json.loads(response['SecretString'])
db_password = secret['password']
```
- **Автоматическая ротация** через Lambda (встроенная поддержка RDS, Redshift, DocumentDB)
- Версионирование секретов (AWSCURRENT, AWSPREVIOUS, AWSPENDING)
- Стоимость: ~$0.40/secret/month + $0.05 per 10K API calls

**SSM Parameter Store** — хранение конфигураций и несекретных данных (и секретов тоже):
```bash
# String параметр
aws ssm put-parameter \
  --name "/myapp/prod/db_host" \
  --value "mydb.cluster.eu-west-1.rds.amazonaws.com" \
  --type String

# SecureString — зашифрован KMS
aws ssm put-parameter \
  --name "/myapp/prod/db_password" \
  --value "SuperSecretPassword" \
  --type SecureString \
  --key-id arn:aws:kms:...:key/xxx

# Получить все параметры по пути
aws ssm get-parameters-by-path \
  --path "/myapp/prod/" \
  --with-decryption
```

**Сравнение:**

| | KMS | Secrets Manager | SSM Parameter Store |
|---|---|---|---|
| Назначение | Управление ключами, шифрование | Секреты с ротацией | Конфиги и секреты |
| Ротация | Ключей | Автоматическая (Lambda) | Нет (вручную) |
| Стоимость | $1/key/month | $0.40/secret/month | Бесплатно (Standard) |
| Версионирование | Да | Да | Да |
| Интеграция | Всё в AWS | RDS, Redshift | EC2, ECS, Lambda |

---

## Оптимизация затрат

### 23. Как оптимизировать затраты на AWS?

**1. Правильная модель оплаты EC2/Fargate:**
```
Baseline нагрузка (24/7) → Reserved Instances или Savings Plans (скидка 40-72%)
Пиковая нагрузка → Spot Instances (скидка до 90%)
Dev/Test → Spot + schedule shutdown нерабочие часы

# Пример: выключать dev-окружение ночью
aws ec2 stop-instances --instance-ids i-xxx  # через EventBridge schedule
```

**2. Правильный sizing:**
```bash
# AWS Compute Optimizer — ML-анализ и рекомендации по rightsizing
aws compute-optimizer get-ec2-instance-recommendations

# Для Lambda — Power Tuning Tool (AWS Labs)
# Найти оптимальное соотношение цена/производительность памяти

# Для RDS — смотреть реальное потребление CPU/memory
# CloudWatch метрики: DatabaseConnections, CPUUtilization, FreeableMemory
```

**3. S3 стоимость:**
```
✅ Lifecycle policies — перемещать в IA/Glacier/Deep Archive
✅ S3 Intelligent-Tiering для непредсказуемых паттернов
✅ S3 Storage Lens — анализ использования
✅ Удалять неполные multipart uploads:
   aws s3api put-bucket-lifecycle-configuration --bucket my-bucket \
     --lifecycle-configuration '{"Rules":[{"ID":"AbortIncomplete",
     "Status":"Enabled","AbortIncompleteMultipartUpload":{"DaysAfterInitiation":7}}]}'
```

**4. NAT Gateway — скрытые затраты:**
```
NAT Gateway: $0.045/GB transfer + $0.045/hour
Типичный источник больших счётов!

Оптимизация:
✅ VPC Endpoints для S3 и DynamoDB (бесплатно, трафик не через NAT)
✅ VPC Interface Endpoints для других AWS сервисов (дешевле NAT на больших объёмах)
✅ Не деплоить NAT Gateway в каждой AZ если это не критично
```

**5. Data Transfer:**
```
✅ CloudFront снижает origin трафик (кэш = меньше запросов к ALB/S3)
✅ Размещать ресурсы в одной AZ если возможно (cross-AZ трафик платный)
✅ VPC Endpoints убирают интернет-трафик
✅ Сжатие данных (gzip в ALB/CloudFront)
```

**6. Инструменты мониторинга затрат:**
```bash
# AWS Cost Explorer — анализ и прогнозирование
# AWS Budgets — алерты при превышении бюджета
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{"BudgetName":"Monthly","BudgetLimit":{"Amount":"1000","Unit":"USD"},
             "TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications-with-subscribers '...'

# AWS Cost Anomaly Detection — ML обнаружение аномальных трат
# Trusted Advisor — рекомендации по экономии (Reserved Instance coverage, idle resources)
```

**7. Удаление неиспользуемых ресурсов:**
```bash
# Найти незатаченные EBS тома (стоят денег даже без инстанса)
aws ec2 describe-volumes --filters Name=status,Values=available

# Незатаченные Elastic IP
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null]'

# Старые EBS Snapshots
aws ec2 describe-snapshots --owner-ids self --query '...'

# Idle Load Balancers (нет трафика > 7 дней)
# CloudWatch: RequestCount = 0
```
