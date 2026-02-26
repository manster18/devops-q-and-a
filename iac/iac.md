# IaC (Terraform + Ansible) — Вопросы и ответы для собеседований (Middle/Senior DevOps)

## Содержание

### Terraform — основы
1. [Как работает Terraform? Ключевые концепции.](#1-как-работает-terraform-ключевые-концепции)
2. [Что такое Terraform State? Почему он критически важен?](#2-что-такое-terraform-state-почему-он-критически-важен)
3. [Как организовать remote state и state locking?](#3-как-организовать-remote-state-и-state-locking)
4. [Что такое Terraform Provider и как он работает?](#4-что-такое-terraform-provider-и-как-он-работает)

### Terraform — язык и структура
5. [Variables, Locals, Outputs — в чём разница и как использовать?](#5-variables-locals-outputs--в-чём-разница-и-как-использовать)
6. [Что такое Terraform модули и как правильно их организовать?](#6-что-такое-terraform-модули-и-как-правильно-их-организовать)
7. [Какие мета-аргументы существуют в Terraform?](#7-какие-мета-аргументы-существуют-в-terraform)
8. [Как работают Terraform Workspaces?](#8-как-работают-terraform-workspaces)

### Terraform — продвинутые темы
9. [Как импортировать существующую инфраструктуру в Terraform?](#9-как-импортировать-существующую-инфраструктуру-в-terraform)
10. [Как работать с чувствительными данными в Terraform?](#10-как-работать-с-чувствительными-данными-в-terraform)
11. [Что такое terraform_remote_state и data sources?](#11-что-такое-terraform_remote_state-и-data-sources)
12. [Каковы best practices структуры Terraform-проекта?](#12-каковы-best-practices-структуры-terraform-проекта)

### Ansible — основы
13. [Как работает Ansible? Архитектура и ключевые понятия.](#13-как-работает-ansible-архитектура-и-ключевые-понятия)
14. [Что такое Inventory? Статический и динамический инвентарь.](#14-что-такое-inventory-статический-и-динамический-инвентарь)
15. [Как устроены Playbook, Play, Task и Handler?](#15-как-устроены-playbook-play-task-и-handler)
16. [Как работают переменные в Ansible и каков их приоритет?](#16-как-работают-переменные-в-ansible-и-каков-их-приоритет)

### Ansible — продвинутые темы
17. [Что такое Ansible Role? Структура и best practices.](#17-что-такое-ansible-role-структура-и-best-practices)
18. [Что такое Ansible Vault и как им пользоваться?](#18-что-такое-ansible-vault-и-как-им-пользоваться)
19. [Как работают Jinja2 шаблоны в Ansible?](#19-как-работают-jinja2-шаблоны-в-ansible)
20. [Как обеспечить идемпотентность в Ansible?](#20-как-обеспечить-идемпотентность-в-ansible)
21. [Как ускорить выполнение Ansible? Оптимизация производительности.](#21-как-ускорить-выполнение-ansible-оптимизация-производительности)

---

## Terraform — основы

### 1. Как работает Terraform? Ключевые концепции.

**Terraform** — инструмент IaC (Infrastructure as Code) от HashiCorp. Описываешь желаемое состояние инфраструктуры в декларативном коде (HCL), Terraform приводит реальное состояние к желаемому.

**Принцип работы:**
```
Terraform Code (HCL) → terraform plan → Execution Plan → terraform apply → Real Infrastructure
                            ↕
                        State File
                    (текущее состояние)
```

**Основной рабочий цикл:**
```bash
terraform init    # инициализация: скачать провайдеры, настроить backend
terraform plan    # показать что изменится (diff между state и кодом)
terraform apply   # применить изменения
terraform destroy # удалить всю инфраструктуру
```

**Ключевые концепции:**

**Resource** — основная единица: описывает один объект инфраструктуры:
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }
}
```

**Provider** — плагин, реализующий API конкретного облака/сервиса (AWS, GCP, Azure, Kubernetes, GitHub, etc.)

**Data Source** — получить данные о существующем ресурсе (не создавать):
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id  # используем найденный AMI
}
```

**State** — файл (terraform.tfstate), хранящий информацию о созданных ресурсах. Terraform сравнивает его с кодом чтобы понять что добавить/изменить/удалить.

**Граф зависимостей:** Terraform автоматически строит DAG (Directed Acyclic Graph) зависимостей ресурсов и применяет их параллельно где возможно:
```hcl
resource "aws_vpc" "main" { ... }

resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id  # неявная зависимость → subnet создаётся после VPC
}

resource "aws_instance" "web" {
  depends_on = [aws_subnet.public]  # явная зависимость
}
```

---

### 2. Что такое Terraform State? Почему он критически важен?

**State** — файл `terraform.tfstate` в формате JSON, который является "памятью" Terraform. Хранит:
- Маппинг между ресурсами в коде и реальными объектами (ID, ARN, и т.д.)
- Метаданные для отслеживания зависимостей
- Кэш атрибутов ресурсов (чтобы не делать лишние API вызовы)

**Зачем State нужен:**
```
Ситуация: у тебя есть aws_instance "web" и ты меняешь instance_type.

Без State: Terraform не знает какой именно инстанс из сотен в AWS — "твой".
С State:   Terraform знает ID = "i-0abc123def456" → обновить именно его.
```

**Пример содержимого state:**
```json
{
  "version": 4,
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "instances": [
        {
          "attributes": {
            "id": "i-0abc123def456789",
            "ami": "ami-0c55b159cbfafe1f0",
            "instance_type": "t3.micro",
            "public_ip": "34.250.10.15",
            ...
          }
        }
      ]
    }
  ]
}
```

**Проблемы State:**

1. **State drift** — кто-то изменил инфраструктуру вне Terraform (через консоль, CLI). State устарел.
```bash
# Обнаружить drift
terraform plan  # покажет изменения которые надо "откатить"
terraform refresh  # обновить state из реального состояния (устарело)
terraform apply -refresh-only  # только обновить state без применения
```

2. **Чувствительные данные в State** — пароли БД, ключи могут попасть в state в открытом виде. Поэтому state нужно хранить в зашифрованном хранилище.

3. **State лежит локально** — при командной работе несколько человек не могут работать одновременно.

**Основные команды работы со State:**
```bash
# Список ресурсов в state
terraform state list

# Детали конкретного ресурса
terraform state show aws_instance.web

# Удалить ресурс из state (не удалять реальный объект!)
terraform state rm aws_instance.web

# Переименовать ресурс в state
terraform state mv aws_instance.web aws_instance.application

# Переместить ресурс между state-файлами
terraform state mv -state-out=../other/terraform.tfstate \
  aws_instance.web aws_instance.web
```

---

### 3. Как организовать remote state и state locking?

**Remote State** — хранение state-файла в удалённом хранилище вместо локального файла. Обязателен для командной работы.

**Backend S3 + DynamoDB (стандарт для AWS):**
```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/vpc/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true           # шифровать state в S3 (SSE-S3 или KMS)
    kms_key_id     = "arn:aws:kms:eu-west-1:xxx:key/yyy"  # KMS ключ

    dynamodb_table = "terraform-state-locks"  # таблица для locking
  }
}
```

**Создание инфраструктуры для state bootstrap (курица и яйцо):**
```bash
# Создать S3 bucket и DynamoDB таблицу вручную или отдельным Terraform
aws s3api create-bucket \
  --bucket my-terraform-state \
  --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1

aws s3api put-bucket-versioning \
  --bucket my-terraform-state \
  --versioning-configuration Status=Enabled  # версионирование для rollback

aws s3api put-bucket-encryption \
  --bucket my-terraform-state \
  --server-side-encryption-configuration '...'

aws dynamodb create-table \
  --table-name terraform-state-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

**State Locking** предотвращает одновременные `terraform apply` разными пользователями. DynamoDB хранит запись о блокировке:
```
LockID: "my-terraform-state/production/vpc/terraform.tfstate"
Info: {
  "ID": "...",
  "Operation": "OperationTypeApply",
  "Who": "alice@laptop",
  "Created": "2024-01-15T10:30:00Z"
}
```

```bash
# Принудительно снять застрявшую блокировку (осторожно!)
terraform force-unlock <LOCK_ID>
```

**Альтернативные backends:**
```hcl
# Terraform Cloud / HCP Terraform (бесплатно до 500 ресурсов)
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "production"
    }
  }
}

# GCS (Google Cloud Storage)
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "production/vpc"
  }
}

# Azure Blob Storage
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate"
    storage_account_name = "tfstate12345"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"
  }
}
```

---

### 4. Что такое Terraform Provider и как он работает?

**Provider** — плагин, который транслирует Terraform-ресурсы в API-вызовы конкретной платформы. Каждый ресурс (`aws_instance`, `google_compute_instance`, `kubernetes_deployment`) принадлежит провайдеру.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"    # ~> 5.0 = >=5.0, <6.0 (pessimistic constraint)
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.23"
    }
  }
  required_version = ">= 1.6.0"
}

provider "aws" {
  region = "eu-west-1"

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = var.environment
      Project     = var.project_name
    }
  }
}

# Несколько провайдеров одного типа (alias) — например два региона
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

resource "aws_s3_bucket" "replica" {
  provider = aws.us_east  # явно указать какой провайдер использовать
  bucket   = "my-replica-bucket"
}
```

**Что происходит при `terraform init`:**
```bash
terraform init
# 1. Читает required_providers
# 2. Скачивает провайдеры в .terraform/providers/
# 3. Записывает .terraform.lock.hcl (lock файл версий провайдеров)
# 4. Инициализирует backend
```

**`.terraform.lock.hcl`** — фиксирует точные версии провайдеров и их хэши. Должен быть в git:
```hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:...",
    "zh:...",
  ]
}
```

```bash
# Обновить версии провайдеров (в пределах constraints)
terraform init -upgrade

# Проверить устаревшие версии
terraform version
```

---

## Terraform — язык и структура

### 5. Variables, Locals, Outputs — в чём разница и как использовать?

**Variables (Input Variables)** — параметры модуля/конфигурации. Задаются снаружи:

```hcl
# variables.tf
variable "environment" {
  description = "Deployment environment (dev/staging/prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "allowed_cidrs" {
  type    = list(string)
  default = ["10.0.0.0/8"]
}

variable "tags" {
  type = map(string)
  default = {}
}

variable "db_config" {
  type = object({
    engine         = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
}

# Sensitive variable — не выводить в logs/plan
variable "db_password" {
  type      = string
  sensitive = true
}
```

**Способы передачи значений переменных (приоритет от низкого к высокому):**
```bash
# 1. default в коде
# 2. terraform.tfvars (автоматически)
# 3. *.auto.tfvars (автоматически)
# 4. -var-file="prod.tfvars" (явно)
# 5. -var="instance_count=5" (командная строка)
# 6. TF_VAR_environment="prod" (переменные окружения)
```

```hcl
# prod.tfvars
environment     = "prod"
instance_count  = 5
allowed_cidrs   = ["10.0.0.0/8", "172.16.0.0/12"]
```

---

**Locals** — локальные вычисляемые значения внутри модуля. Не принимают ввод снаружи:

```hcl
# locals.tf
locals {
  # Вычисляемые значения
  name_prefix = "${var.project}-${var.environment}"

  # Условная логика
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"

  # Объединение maps
  common_tags = merge(var.tags, {
    Environment = var.environment
    ManagedBy   = "Terraform"
    CreatedAt   = timestamp()
  })

  # Преобразование списков
  availability_zones = slice(data.aws_availability_zones.available.names, 0, 3)

  # Сложные структуры
  subnet_config = {
    for idx, az in local.availability_zones : az => {
      cidr_block = cidrsubnet(var.vpc_cidr, 8, idx)
      az         = az
    }
  }
}

# Использование
resource "aws_instance" "web" {
  instance_type = local.instance_type
  tags          = local.common_tags
}
```

---

**Outputs** — значения, которые модуль "возвращает" наружу. Используются для:
- Передачи данных между модулями
- Вывода важных значений после apply (IP адреса, DNS имена)
- `terraform_remote_state` — чтение outputs другого state

```hcl
# outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = [for s in aws_subnet.public : s.id]
}

output "alb_dns_name" {
  description = "DNS name of the Application Load Balancer"
  value       = aws_lb.main.dns_name
}

output "db_connection_string" {
  description = "Database connection string"
  value       = "postgresql://${aws_db_instance.main.endpoint}/${var.db_name}"
  sensitive   = true  # не выводить в консоль
}
```

```bash
# Посмотреть outputs после apply
terraform output
terraform output vpc_id
terraform output -json  # в JSON формате
```

---

### 6. Что такое Terraform модули и как правильно их организовать?

**Модуль** — переиспользуемый набор Terraform-конфигураций в отдельной директории. Принимает переменные (inputs) и возвращает outputs.

**Типы модулей:**
- **Root module** — директория откуда запускаешь `terraform apply`
- **Child module** — вызывается из другого модуля через блок `module`
- **Published module** — опубликован в Terraform Registry (registry.terraform.io)

**Структура модуля:**
```
modules/
└── aws-vpc/
    ├── main.tf         # основные ресурсы
    ├── variables.tf    # input variables
    ├── outputs.tf      # outputs
    ├── versions.tf     # required_providers, required_version
    └── README.md       # документация (что принимает, что возвращает)
```

**Пример модуля `aws-vpc`:**
```hcl
# modules/aws-vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = merge(var.tags, { Name = var.name })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name}-igw" })
}

resource "aws_subnet" "public" {
  for_each = { for idx, cidr in var.public_subnet_cidrs :
    var.availability_zones[idx] => cidr }

  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value
  availability_zone       = each.key
  map_public_ip_on_launch = true
  tags = merge(var.tags, { Name = "${var.name}-public-${each.key}" })
}
```

**Вызов модуля:**
```hcl
# environments/prod/main.tf
module "vpc" {
  source  = "../../modules/aws-vpc"       # локальный путь
  # source = "terraform-aws-modules/vpc/aws"  # Terraform Registry
  # source = "git::https://github.com/org/repo.git//modules/vpc?ref=v2.0.0"  # Git
  # version = "~> 5.0"  # для Registry модулей

  name                 = "prod-vpc"
  cidr_block           = "10.0.0.0/16"
  availability_zones   = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]

  tags = {
    Environment = "prod"
    Project     = "myapp"
  }
}

# Использовать output модуля
module "eks" {
  source     = "../../modules/aws-eks"
  vpc_id     = module.vpc.vpc_id            # output из vpc модуля
  subnet_ids = module.vpc.private_subnet_ids
}
```

**Рекомендуемая структура проекта:**
```
infrastructure/
├── modules/                    # переиспользуемые модули
│   ├── aws-vpc/
│   ├── aws-eks/
│   ├── aws-rds/
│   └── aws-alb/
│
└── environments/               # окружения (отдельный state для каждого)
    ├── dev/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   ├── backend.tf          # remote state config
    │   └── terraform.tfvars
    ├── staging/
    │   └── ...
    └── prod/
        └── ...
```

---

### 7. Какие мета-аргументы существуют в Terraform?

Мета-аргументы применяются к любому ресурсу и изменяют поведение Terraform.

**`count`** — создать N одинаковых ресурсов:
```hcl
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index}"  # web-0, web-1, web-2...
  }
}

# Обращение: aws_instance.web[0].id, aws_instance.web[*].id (список всех)
```

**`for_each`** — создать ресурсы из map или set (более предпочтительно чем count):
```hcl
variable "buckets" {
  default = {
    logs    = { region = "eu-west-1",  versioning = true  }
    backups = { region = "eu-central-1", versioning = true  }
    static  = { region = "eu-west-1",  versioning = false }
  }
}

resource "aws_s3_bucket" "this" {
  for_each = var.buckets
  bucket   = "${var.project}-${each.key}"
  # each.key = "logs", "backups", "static"
  # each.value = { region = "...", versioning = true }
}

# Обращение: aws_s3_bucket.this["logs"].id
# Почему for_each лучше count: удаление элемента из середины не пересоздаёт все остальные
```

**`depends_on`** — явная зависимость (когда неявной недостаточно):
```hcl
resource "aws_iam_role_policy_attachment" "this" {
  role       = aws_iam_role.this.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_eks_cluster" "this" {
  depends_on = [aws_iam_role_policy_attachment.this]  # дождаться прикрепления политики
  name       = var.cluster_name
  role_arn   = aws_iam_role.this.arn
}
```

**`lifecycle`** — контроль жизненного цикла ресурса:
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  lifecycle {
    # Создать новый ПЕРЕД удалением старого (zero-downtime при обновлении)
    create_before_destroy = true

    # Никогда не удалять (защита от случайного terraform destroy)
    prevent_destroy = true

    # Игнорировать изменения конкретных атрибутов (например, AMI меняется Packer'ом)
    ignore_changes = [ami, user_data]

    # replace_triggered_by — пересоздать ресурс при изменении другого
    replace_triggered_by = [aws_launch_template.this.latest_version]
  }
}
```

**`provider`** — указать какой провайдер использовать (для alias):
```hcl
resource "aws_s3_bucket" "us_bucket" {
  provider = aws.us_east
  bucket   = "my-us-bucket"
}
```

---

### 8. Как работают Terraform Workspaces?

**Workspaces** — механизм хранения нескольких независимых state файлов в одном backend для одной конфигурации.

```bash
# Список workspaces
terraform workspace list
# * default
#   dev
#   staging
#   prod

# Создать и переключиться
terraform workspace new dev
terraform workspace select prod

# Текущий workspace
terraform workspace show
```

**Использование в коде:**
```hcl
locals {
  env_config = {
    dev = {
      instance_type  = "t3.micro"
      instance_count = 1
      db_class       = "db.t3.micro"
    }
    staging = {
      instance_type  = "t3.small"
      instance_count = 2
      db_class       = "db.t3.small"
    }
    prod = {
      instance_type  = "t3.large"
      instance_count = 5
      db_class       = "db.r5.large"
    }
  }

  config = local.env_config[terraform.workspace]
}

resource "aws_instance" "web" {
  count         = local.config.instance_count
  instance_type = local.config.instance_type
}
```

**State хранится раздельно:**
```
S3 bucket:
  env:/dev/myapp/terraform.tfstate
  env:/staging/myapp/terraform.tfstate
  env:/prod/myapp/terraform.tfstate
  myapp/terraform.tfstate  (default workspace)
```

**Ограничения Workspaces:**
- Все окружения используют одну конфигурацию → сложно поддерживать разные конфиги
- Нет изоляции прав доступа между workspace
- Легко ошибиться и применить dev-изменения в prod

**Альтернатива — отдельные директории + Terragrunt:**
Большинство команд предпочитают отдельные директории для каждого окружения (как в вопросе 6) — более явно, нет риска перепутать workspace.

---

## Terraform — продвинутые темы

### 9. Как импортировать существующую инфраструктуру в Terraform?

Когда инфраструктура создана вручную (через консоль, CLI) и нужно взять её под управление Terraform.

**Метод 1 — `terraform import` (классический):**
```bash
# 1. Написать resource блок в коде (пустой или заполненный)
# main.tf:
resource "aws_instance" "existing" {
  # поля заполним после импорта
}

# 2. Импортировать реальный ресурс в state
terraform import aws_instance.existing i-0abc123def456789

# 3. Посмотреть что попало в state
terraform state show aws_instance.existing

# 4. Заполнить resource блок атрибутами из state
# 5. Запустить terraform plan — должен показать "No changes"
```

**Метод 2 — `import` блок (Terraform 1.5+, рекомендуется):**
```hcl
# import.tf
import {
  to = aws_instance.existing
  id = "i-0abc123def456789"
}

# С Terraform 1.6+ можно автоматически сгенерировать resource код:
```

```bash
# Сгенерировать код ресурса автоматически
terraform plan -generate-config-out=generated.tf
# generated.tf будет содержать полный resource блок со всеми атрибутами

# После ревью — применить
terraform apply
```

**Массовый импорт через for_each:**
```hcl
locals {
  s3_buckets = {
    "my-bucket-1" = "my-bucket-1"
    "my-bucket-2" = "my-bucket-2"
  }
}

import {
  for_each = local.s3_buckets
  to       = aws_s3_bucket.existing[each.key]
  id       = each.value
}

resource "aws_s3_bucket" "existing" {
  for_each = local.s3_buckets
  bucket   = each.key
}
```

**Инструменты для обратной генерации (reverse IaC):**
```bash
# terraformer — импортирует всю инфраструктуру AWS аккаунта
terraformer import aws --resources=vpc,subnet,sg --regions=eu-west-1

# former2 — браузерный инструмент, UI для генерации Terraform кода
```

---

### 10. Как работать с чувствительными данными в Terraform?

**Проблема:** пароли, ключи, токены не должны храниться в git и не должны светиться в логах CI/CD.

**Метод 1 — переменные окружения + sensitive:**
```hcl
variable "db_password" {
  type      = string
  sensitive = true  # скрыть в plan/apply выводе
}
```
```bash
# Передать через переменную окружения (не в tfvars!)
export TF_VAR_db_password="$(aws secretsmanager get-secret-value \
  --secret-id prod/db/password --query SecretString --output text)"
terraform apply
```

**Метод 2 — читать из AWS Secrets Manager:**
```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/myapp/db_password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  # ... остальные параметры
}
```

**Метод 3 — HashiCorp Vault Provider:**
```hcl
provider "vault" {
  address = "https://vault.example.com"
}

data "vault_generic_secret" "db_creds" {
  path = "secret/prod/db"
}

resource "aws_db_instance" "main" {
  username = data.vault_generic_secret.db_creds.data["username"]
  password = data.vault_generic_secret.db_creds.data["password"]
}
```

**Важно:** даже если переменная `sensitive = true` — её значение всё равно попадает в state в открытом виде. Поэтому:
```bash
# State должен быть зашифрован (backend "s3" { encrypt = true })
# Ограничить доступ к S3 bucket через IAM
# Включить версионирование bucket для rollback
```

**Что делать если секрет уже попал в git:**
```bash
# git history переписать (git-filter-repo)
git filter-repo --path-glob '*.tfvars' --invert-paths

# Немедленно ротировать скомпрометированный секрет!
```

---

### 11. Что такое terraform_remote_state и data sources?

**`data` sources** — чтение существующих ресурсов без их создания:
```hcl
# Найти существующий VPC по тегу
data "aws_vpc" "existing" {
  tags = {
    Name        = "production-vpc"
    Environment = "prod"
  }
}

# Найти последний Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

# Получить availability zones в регионе
data "aws_availability_zones" "available" {
  state = "available"
}

# Получить текущий аккаунт и регион
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# Использование
resource "aws_security_group" "app" {
  vpc_id = data.aws_vpc.existing.id

  ingress {
    from_port = 443
    to_port   = 443
    protocol  = "tcp"
    cidr_blocks = [data.aws_vpc.existing.cidr_block]
  }
}
```

**`terraform_remote_state`** — читать outputs из другого Terraform state. Позволяет разным командам/компонентам делиться данными:

```hcl
# В team-network управляет VPC
# outputs.tf:
output "vpc_id"            { value = aws_vpc.main.id }
output "private_subnet_ids" { value = [for s in aws_subnet.private : s.id] }

# ─────────────────────────────────────────────────────

# В team-app управляет приложением
# main.tf:
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "production/network/terraform.tfstate"
    region = "eu-west-1"
  }
}

resource "aws_eks_cluster" "this" {
  vpc_config {
    subnet_ids = data.terraform_remote_state.network.outputs.private_subnet_ids
  }
}
```

**Недостатки `terraform_remote_state`:** сильная связанность между командами. Альтернатива — AWS SSM Parameter Store или Consul для хранения shared data:
```hcl
# Network team пишет в SSM
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/infra/prod/vpc_id"
  type  = "String"
  value = aws_vpc.main.id
}

# App team читает из SSM
data "aws_ssm_parameter" "vpc_id" {
  name = "/infra/prod/vpc_id"
}
```

---

### 12. Каковы best practices структуры Terraform-проекта?

**Минимальная структура модуля:**
```
module/
├── main.tf        # основные ресурсы
├── variables.tf   # все input variables
├── outputs.tf     # все outputs
├── versions.tf    # required_providers, required_version
└── README.md      # описание модуля
```

**Разбивка на файлы по типу ресурсов:**
```
environments/prod/
├── backend.tf     # remote state config
├── versions.tf    # providers
├── variables.tf
├── locals.tf
├── outputs.tf
├── main.tf        # module calls
├── vpc.tf         # VPC-специфичные ресурсы
├── eks.tf         # EKS ресурсы
├── rds.tf         # RDS ресурсы
└── terraform.tfvars
```

**Ключевые практики:**

```hcl
# 1. Всегда закреплять версии провайдеров
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.31" }
  }
  required_version = ">= 1.6.0, < 2.0.0"
}

# 2. default_tags на уровне провайдера (не теги в каждом ресурсе)
provider "aws" {
  default_tags {
    tags = local.common_tags
  }
}

# 3. for_each вместо count (где возможно)
# ❌ Плохо: удаление web[1] → пересоздаст web[2], web[3]
resource "aws_instance" "web" {
  count = 3
}

# ✅ Хорошо: удаление "staging" не трогает остальных
resource "aws_instance" "web" {
  for_each = toset(["prod-1", "prod-2", "prod-3"])
  tags     = { Name = each.key }
}

# 4. Переменные с validation
variable "environment" {
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

# 5. Описательные описания для всех variables и outputs
variable "vpc_cidr" {
  description = "CIDR block for the VPC. Must be /16 or larger."
  type        = string
  default     = "10.0.0.0/16"
}
```

**CI/CD пайплайн для Terraform:**
```yaml
# .github/workflows/terraform.yml
- name: Terraform Format Check
  run: terraform fmt -check -recursive

- name: Terraform Validate
  run: terraform validate

- name: Terraform Plan (PR)
  run: terraform plan -out=tfplan
  # Публикуем plan как комментарий в PR

- name: Checkov Security Scan
  run: checkov -d . --framework terraform

- name: Terraform Apply (merge to main)
  run: terraform apply tfplan
```

**Инструменты экосистемы:**
```bash
terraform fmt        # форматирование кода
terraform validate   # валидация синтаксиса и типов
terraform-docs       # автогенерация README для модулей
tflint               # линтер (находит deprecated и неиспользуемые переменные)
checkov / tfsec      # security сканер
infracost            # оценка стоимости до apply
terragrunt           # DRY wrapper (не повторять backend config в каждом env)
```

---

## Ansible — основы

### 13. Как работает Ansible? Архитектура и ключевые понятия.

**Ansible** — инструмент управления конфигурацией и оркестрации. Ключевые свойства:
- **Agentless** — не требует установки агента на управляемых хостах
- **Idempotent** — повторный запуск даёт тот же результат
- **Декларативный + процедурный** — описываешь задачи, Ansible выполняет по порядку

**Архитектура:**
```
Control Node (твоя машина или CI сервер)
    │
    │ SSH (Linux) / WinRM (Windows)
    │
    ├── managed-host-1
    ├── managed-host-2
    └── managed-host-3
```

Никаких демонов, никаких агентов на managed hosts. Ansible подключается по SSH, загружает Python-модуль (временный файл), выполняет, удаляет.

**Основные компоненты:**

**Inventory** — список хостов и групп.

**Module** — атомарная единица работы: `apt`, `yum`, `copy`, `template`, `service`, `user`, `docker_container`, `k8s`, `aws_s3` и тысячи других. Каждый модуль идемпотентен.

**Task** — вызов модуля с параметрами.

**Play** — набор tasks для группы хостов.

**Playbook** — один или несколько plays в YAML файле.

**Role** — переиспользуемая структура (tasks, vars, templates, handlers).

**Простой пример:**
```bash
# Ad-hoc команда (без playbook)
ansible all -i inventory.ini -m ping
ansible webservers -i inventory.ini -m shell -a "uptime"
ansible webservers -i inventory.ini -m apt -a "name=nginx state=present" --become
```

---

### 14. Что такое Inventory? Статический и динамический инвентарь.

**Статический инвентарь (INI формат):**
```ini
# inventory.ini
[webservers]
web1.example.com
web2.example.com ansible_port=2222
web3.example.com ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa

[databases]
db1.example.com
db2.example.com

[webservers:vars]
nginx_port=80
app_env=production

[databases:vars]
db_port=5432

# Группа групп
[production:children]
webservers
databases

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

**YAML формат (более гибкий):**
```yaml
# inventory.yml
all:
  vars:
    ansible_python_interpreter: /usr/bin/python3

  children:
    webservers:
      hosts:
        web1.example.com:
          nginx_port: 80
        web2.example.com:
          nginx_port: 8080
      vars:
        app_env: production

    databases:
      hosts:
        db1.example.com:
          db_role: primary
        db2.example.com:
          db_role: replica
```

**Динамический инвентарь** — генерируется скриптом или плагином из внешних источников (AWS, GCP, vSphere, Kubernetes). Идеален когда инстансы создаются/удаляются динамически.

```yaml
# inventory/aws_ec2.yml (AWS EC2 Dynamic Inventory Plugin)
plugin: amazon.aws.aws_ec2
regions:
  - eu-west-1
  - eu-central-1

filters:
  instance-state-name: running
  tag:ManagedBy: Ansible

# Группировать по тегам
keyed_groups:
  - key: tags.Environment
    prefix: env
  - key: tags.Role
    prefix: role
  - key: placement.availability_zone
    prefix: az

# Кастомные переменные из тегов
hostnames:
  - tag:Name
  - private-ip-address

compose:
  ansible_host: private_ip_address  # подключаться по private IP
```

```bash
# Использование
ansible-inventory -i inventory/aws_ec2.yml --list
ansible-inventory -i inventory/aws_ec2.yml --graph

# Запустить playbook с динамическим инвентарём
ansible-playbook -i inventory/aws_ec2.yml deploy.yml --limit env_production
```

---

### 15. Как устроены Playbook, Play, Task и Handler?

```yaml
# deploy.yml — полный пример playbook

---
# Play 1 — настройка web серверов
- name: Configure web servers
  hosts: webservers          # к каким хостам применять
  become: true               # sudo (privilege escalation)
  become_user: root
  gather_facts: true         # собирать ansible_facts о хостах

  vars:
    nginx_version: "1.25"
    app_port: 8080

  pre_tasks:                 # выполнить до roles/tasks
  - name: Update apt cache
    apt:
      update_cache: true
      cache_valid_time: 3600

  roles:                     # применить роли
  - role: nginx
  - role: app-deploy
    vars:
      app_version: "{{ deploy_version }}"

  tasks:
  - name: Install nginx
    apt:
      name: "nginx={{ nginx_version }}*"
      state: present
    notify: Restart nginx     # вызвать handler если задача изменила состояние

  - name: Copy nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
      mode: '0644'
      validate: "nginx -t -c %s"   # проверить конфиг перед копированием
    notify: Reload nginx

  - name: Ensure nginx is running
    service:
      name: nginx
      state: started
      enabled: true

  - name: Run task only on primary servers
    command: /opt/app/init.sh
    when: inventory_hostname == groups['webservers'][0]

  - name: Install multiple packages
    apt:
      name:
        - git
        - curl
        - htop
      state: present

  - name: Create application directory
    file:
      path: /opt/myapp
      state: directory
      owner: www-data
      group: www-data
      mode: '0755'

  - name: Deploy application archive
    unarchive:
      src: "https://releases.example.com/app-{{ app_version }}.tar.gz"
      dest: /opt/myapp
      remote_src: true       # скачать на managed host, а не с control node
    register: deploy_result  # сохранить результат в переменную

  - name: Show deploy result
    debug:
      var: deploy_result.changed

  - name: Run migrations
    command: /opt/myapp/migrate.sh
    when: deploy_result.changed  # только если предыдущая задача что-то изменила

  post_tasks:                # выполнить после roles/tasks
  - name: Verify nginx is responding
    uri:
      url: "http://localhost:{{ app_port }}/health"
      status_code: 200

  handlers:                  # выполняются в конце play, только если notify был вызван
  - name: Restart nginx
    service:
      name: nginx
      state: restarted

  - name: Reload nginx
    service:
      name: nginx
      state: reloaded        # graceful — не рвёт соединения

# Play 2 — настройка баз данных
- name: Configure databases
  hosts: databases
  become: true
  roles:
  - postgresql
```

**Важное про Handlers:**
- Выполняются **один раз** в конце play, даже если notify был вызван многократно
- Порядок выполнения — по порядку определения, не по порядку вызова
- `meta: flush_handlers` — выполнить handlers немедленно

```bash
# Запуск playbook
ansible-playbook deploy.yml -i inventory.yml

# Только определённые теги
ansible-playbook deploy.yml --tags "nginx,config"
ansible-playbook deploy.yml --skip-tags "install"

# Только определённые хосты
ansible-playbook deploy.yml --limit "web1.example.com,web2.example.com"
ansible-playbook deploy.yml --limit "webservers:!web3.example.com"  # все кроме web3

# Dry run (check mode)
ansible-playbook deploy.yml --check --diff

# Начать с конкретной задачи
ansible-playbook deploy.yml --start-at-task "Deploy application archive"

# Step-by-step (подтверждать каждую задачу)
ansible-playbook deploy.yml --step
```

---

### 16. Как работают переменные в Ansible и каков их приоритет?

Ansible имеет 22 уровня приоритета переменных (от низкого к высокому):

```
1.  role defaults (roles/myrole/defaults/main.yml)       — наименьший приоритет
2.  inventory file or script group vars
3.  inventory group_vars/all
4.  playbook group_vars/all
5.  inventory group_vars/*
6.  playbook group_vars/*
7.  inventory file or script host vars
8.  inventory host_vars/*
9.  playbook host_vars/*
10. host facts / cached set_facts
11. play vars
12. play vars_prompt
13. play vars_files
14. role vars (roles/myrole/vars/main.yml)
15. block vars
16. task vars
17. include_vars
18. set_facts / registered vars
19. role (and include_role) params
20. include params
21. extra vars (-e "key=value")                          — наивысший приоритет
```

**Практически важные уровни:**

```
group_vars/all.yml        — общие для всех хостов
group_vars/webservers.yml — для группы webservers
host_vars/web1.yml        — для конкретного хоста
roles/nginx/defaults/     — defaults роли (легко переопределить)
roles/nginx/vars/         — vars роли (тяжело переопределить, нужен -e)
```

**Структура group_vars и host_vars:**
```
inventory/
├── hosts.yml
├── group_vars/
│   ├── all.yml           # для всех хостов
│   ├── all/
│   │   ├── vars.yml
│   │   └── vault.yml     # зашифрованные секреты
│   ├── webservers.yml
│   └── production/
│       ├── vars.yml
│       └── vault.yml
└── host_vars/
    └── web1.example.com.yml
```

**Специальные переменные (magic variables):**
```yaml
# hostvars — переменные любого хоста
{{ hostvars['db1.example.com']['ansible_host'] }}

# groups — список хостов в группе
{{ groups['webservers'] }}

# inventory_hostname — текущий хост (как в inventory)
{{ inventory_hostname }}

# ansible_facts — факты о хосте
{{ ansible_facts['os_family'] }}           # 'Debian', 'RedHat'
{{ ansible_facts['distribution'] }}        # 'Ubuntu', 'CentOS'
{{ ansible_facts['distribution_version'] }}
{{ ansible_facts['default_ipv4']['address'] }}
{{ ansible_facts['processor_count'] }}
```

**Регистрация результата:**
```yaml
- name: Check if app is installed
  command: which myapp
  register: app_check
  ignore_errors: true

- name: Install app
  apt:
    name: myapp
  when: app_check.rc != 0  # если предыдущая команда вернула ненулевой код
```

---

## Ansible — продвинутые темы

### 17. Что такое Ansible Role? Структура и best practices.

**Role** — переиспользуемая единица автоматизации. Инкапсулирует tasks, variables, templates, files для конкретного назначения (nginx, postgresql, docker, etc.).

**Структура роли:**
```
roles/nginx/
├── tasks/
│   ├── main.yml        # точка входа tasks
│   ├── install.yml     # задачи установки
│   └── configure.yml   # задачи настройки
├── handlers/
│   └── main.yml        # handlers
├── templates/
│   ├── nginx.conf.j2   # Jinja2 шаблоны
│   └── vhost.conf.j2
├── files/
│   └── nginx.pem       # статические файлы
├── vars/
│   └── main.yml        # переменные роли (высокий приоритет)
├── defaults/
│   └── main.yml        # дефолтные значения (низкий приоритет, легко переопределить)
├── meta/
│   └── main.yml        # зависимости от других ролей
└── README.md
```

**Пример роли `nginx`:**
```yaml
# roles/nginx/defaults/main.yml
nginx_version: latest
nginx_port: 80
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_server_tokens: "off"

# roles/nginx/tasks/main.yml
---
- name: Install nginx
  include_tasks: install.yml
  tags: install

- name: Configure nginx
  include_tasks: configure.yml
  tags: configure

# roles/nginx/tasks/install.yml
---
- name: Install nginx
  package:
    name: "nginx{% if nginx_version != 'latest' %}={{ nginx_version }}{% endif %}"
    state: "{{ 'present' if nginx_version == 'latest' else 'present' }}"

- name: Ensure nginx directories exist
  file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop:
    - /etc/nginx/sites-available
    - /etc/nginx/sites-enabled
    - /var/log/nginx

# roles/nginx/tasks/configure.yml
---
- name: Deploy nginx main config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    validate: "nginx -t -c %s"
  notify: Reload nginx

# roles/nginx/handlers/main.yml
---
- name: Reload nginx
  service:
    name: nginx
    state: reloaded

- name: Restart nginx
  service:
    name: nginx
    state: restarted

# roles/nginx/meta/main.yml
---
dependencies:
  - role: common    # сначала выполнить роль common
```

**Ansible Galaxy — публичный реестр ролей:**
```bash
# Установить роль из Galaxy
ansible-galaxy install geerlingguy.nginx

# requirements.yml — фиксировать зависимости
# roles/requirements.yml:
- name: geerlingguy.nginx
  version: 3.2.0

- name: geerlingguy.postgresql
  version: 3.4.0

- src: https://github.com/mycompany/ansible-role-myapp
  version: v1.2.0
  name: myapp

# Установить всё из requirements.yml
ansible-galaxy install -r roles/requirements.yml
```

---

### 18. Что такое Ansible Vault и как им пользоваться?

**Ansible Vault** — встроенное шифрование файлов и переменных. Позволяет хранить секреты (пароли, ключи, токены) в git в зашифрованном виде.

**Шифрование файла целиком:**
```bash
# Зашифровать файл (запросит пароль)
ansible-vault encrypt group_vars/production/vault.yml

# Создать новый зашифрованный файл
ansible-vault create group_vars/production/vault.yml

# Просмотреть без расшифровки на диске
ansible-vault view group_vars/production/vault.yml

# Редактировать зашифрованный файл
ansible-vault edit group_vars/production/vault.yml

# Расшифровать (сохранить на диске в открытом виде)
ansible-vault decrypt group_vars/production/vault.yml

# Сменить пароль
ansible-vault rekey group_vars/production/vault.yml
```

**Шифрование отдельной строки (inline):**
```bash
# Зашифровать одно значение
ansible-vault encrypt_string 'SuperSecretPassword' --name 'db_password'
# Вывод:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   66633...
```

```yaml
# group_vars/production/vault.yml (весь файл зашифрован)
vault_db_password: "SuperSecretPassword"
vault_api_key: "sk-abc123"
vault_ssl_certificate: |
  -----BEGIN CERTIFICATE-----
  ...

# group_vars/production/vars.yml (открытый файл, ссылается на vault vars)
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
```

**Использование при запуске:**
```bash
# Запросить пароль интерактивно
ansible-playbook deploy.yml --ask-vault-pass

# Файл с паролем (не коммитить в git!)
echo "my-vault-password" > ~/.vault_pass
chmod 600 ~/.vault_pass
ansible-playbook deploy.yml --vault-password-file ~/.vault_pass

# Через переменную окружения
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass

# Скрипт для получения пароля (из Secret Manager, Vault, etc.)
ansible-playbook deploy.yml --vault-password-file ./get_vault_pass.sh
```

**Несколько Vault ID (разные пароли для dev/prod):**
```bash
# Зашифровать с конкретным ID
ansible-vault encrypt --vault-id prod@prompt group_vars/production/vault.yml
ansible-vault encrypt --vault-id dev@~/.vault_dev group_vars/dev/vault.yml

# Запустить с несколькими паролями
ansible-playbook deploy.yml \
  --vault-id prod@prompt \
  --vault-id dev@~/.vault_dev
```

---

### 19. Как работают Jinja2 шаблоны в Ansible?

Ansible использует **Jinja2** для шаблонизации как в файлах-шаблонах (`.j2`), так и прямо в playbook YAML.

**Основной синтаксис:**
```
{{ variable }}          — вывод переменной
{% if / for / ... %}    — управляющие конструкции
{# комментарий #}       — комментарий (не попадает в вывод)
```

**Пример шаблона `/etc/nginx/nginx.conf.j2`:**
```nginx
user  www-data;
worker_processes  {{ nginx_worker_processes }};

error_log  /var/log/nginx/error.log {{ nginx_error_log_level | default('warn') }};

events {
    worker_connections  {{ nginx_worker_connections }};
}

http {
    server_tokens {{ nginx_server_tokens }};

    # Динамический список upstream серверов
    upstream backend {
        {% for host in groups['appservers'] %}
        server {{ hostvars[host]['ansible_host'] }}:{{ app_port }} weight={{ hostvars[host]['weight'] | default(1) }};
        {% endfor %}
    }

    {% for vhost in nginx_vhosts %}
    server {
        listen {{ vhost.port | default(80) }};
        server_name {{ vhost.server_name }};

        {% if vhost.ssl is defined and vhost.ssl %}
        listen 443 ssl;
        ssl_certificate     {{ vhost.ssl_cert }};
        ssl_certificate_key {{ vhost.ssl_key }};
        {% endif %}

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
    {% endfor %}
}
```

**Jinja2 фильтры — мощный инструмент:**
```yaml
# Встроенные фильтры
{{ "hello world" | upper }}              # HELLO WORLD
{{ items_list | length }}                # длина списка
{{ "  hello  " | trim }}                 # "hello"
{{ value | default("fallback") }}        # значение по умолчанию
{{ value | default(omit) }}             # опустить параметр если не задан

# Работа с данными
{{ list | join(', ') }}                  # "a, b, c"
{{ dict | dict2items }}                  # конвертировать dict в список {key, value}
{{ items | items2dict }}                 # обратно
{{ list | unique }}                      # убрать дубликаты
{{ list | sort }}
{{ list | select('match', '^web') | list }}  # фильтровать по regex

# Математика
{{ (disk_size_gb * 1024) | int }}
{{ ((used / total) * 100) | round(2) }} # 87.43

# JSON
{{ data | to_json }}
{{ json_string | from_json }}
{{ data | to_nice_json }}               # красиво отформатированный JSON
{{ data | to_yaml }}

# Сетевые
{{ "192.168.1.0/24" | ipaddr('network') }}   # 192.168.1.0
{{ "192.168.1.5" | ipaddr('private') }}       # True/False

# Кастомный фильтр в tasks
- name: Template nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  vars:
    nginx_vhosts: "{{ all_vhosts | selectattr('env', 'eq', 'production') | list }}"
```

**Условия в tasks:**
```yaml
tasks:
- name: Install on Debian/Ubuntu
  apt:
    name: nginx
  when:
    - ansible_facts['os_family'] == "Debian"
    - ansible_facts['distribution_major_version'] | int >= 20

- name: Install on RHEL/CentOS
  yum:
    name: nginx
  when: ansible_facts['os_family'] == "RedHat"

- name: Complex condition
  service:
    name: nginx
    state: started
  when: >
    (nginx_installed.rc == 0)
    and (env == 'production' or env == 'staging')
    and not maintenance_mode | bool
```

---

### 20. Как обеспечить идемпотентность в Ansible?

**Идемпотентность** — многократный запуск playbook даёт тот же результат что и первый запуск. Это фундаментальный принцип IaC.

**Правила идемпотентности:**

**1. Использовать декларативные модули вместо shell/command:**
```yaml
# ❌ Не идемпотентно: создаст директорию каждый раз, ошибка если существует
- command: mkdir /opt/myapp

# ✅ Идемпотентно: проверит существование сам
- file:
    path: /opt/myapp
    state: directory
    mode: '0755'
```

**2. Если нужен shell/command — использовать `creates`/`removes`:**
```yaml
# ✅ Выполнить только если файл НЕ существует
- command: /opt/install.sh
  args:
    creates: /opt/myapp/bin/server  # пропустить если файл уже есть

# ✅ Или проверить вручную через register
- name: Check if installed
  stat:
    path: /opt/myapp/bin/server
  register: app_binary

- name: Run installer
  command: /opt/install.sh
  when: not app_binary.stat.exists
```

**3. Пакеты — `state: present`, не `state: latest` (если не нужно обновление):**
```yaml
# ✅ Установит если нет, не будет обновлять если есть
- apt:
    name: nginx
    state: present

# ⚠️ Будет обновлять каждый раз — не всегда желательно
- apt:
    name: nginx
    state: latest
```

**4. Сервисы — явно указывать состояние:**
```yaml
- service:
    name: nginx
    state: started    # запустить если не запущен
    enabled: true     # добавить в автозапуск
```

**5. Линии в файлах — `lineinfile` вместо `shell echo`:**
```yaml
# ✅ Идемпотентно: добавит строку только если её нет
- lineinfile:
    path: /etc/hosts
    line: "10.0.0.5 db.internal"
    state: present

# ✅ Обеспечить что строка СООТВЕТСТВУЕТ паттерну
- lineinfile:
    path: /etc/sshd_config
    regexp: "^#?PermitRootLogin"
    line: "PermitRootLogin no"
    state: present
  notify: Restart sshd
```

**6. Шаблоны — всегда идемпотентны (md5 проверка)**

**7. `changed_when` и `failed_when` для shell команд:**
```yaml
- name: Check app version
  command: /opt/myapp/bin/server --version
  register: version_output
  changed_when: false   # эта команда НИКОГДА не "изменяет" систему

- name: Compile application
  command: make build
  args:
    chdir: /opt/myapp/src
  register: compile_result
  changed_when: "'Nothing to be done' not in compile_result.stdout"
  failed_when: compile_result.rc != 0 and 'warning' not in compile_result.stderr
```

---

### 21. Как ускорить выполнение Ansible? Оптимизация производительности.

По умолчанию Ansible медленный при большом количестве хостов. Вот способы ускорения.

**1. Параллельность — `forks`:**
```ini
# ansible.cfg
[defaults]
forks = 20  # параллельных соединений (default: 5)
```
```bash
ansible-playbook deploy.yml -f 50  # переопределить в командной строке
```

**2. SSH ControlMaster — переиспользовать соединения:**
```ini
# ansible.cfg
[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ServerAliveInterval=30
pipelining = True  # не создавать временные файлы — передавать модули через stdin
```

**`pipelining`** — значительное ускорение: без него Ansible копирует Python модуль → выполняет → удаляет (3 SSH операции). С pipelining — 1 операция. Требует `requiretty=False` в sudoers на managed hosts.

**3. Gather facts — отключить или кэшировать:**
```yaml
# Отключить сбор фактов если не нужны
- hosts: webservers
  gather_facts: false   # экономит ~1-2 сек на хост

# Кэшировать факты между запусками
# ansible.cfg:
# [defaults]
# fact_caching = jsonfile
# fact_caching_connection = /tmp/ansible_facts_cache
# fact_caching_timeout = 3600
```

**4. `async` и `poll` — асинхронное выполнение:**
```yaml
# Запустить долгую задачу асинхронно
- name: Run long running command
  command: /opt/rebuild-cache.sh
  async: 300      # максимальное время выполнения (сек)
  poll: 0         # не ждать — запустить и двигаться дальше
  register: rebuild_job

# Продолжать другие задачи...
- name: Do other stuff while rebuild runs
  apt:
    name: curl
    state: present

# Дождаться завершения
- name: Wait for rebuild to complete
  async_status:
    jid: "{{ rebuild_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 30
  delay: 10
```

**5. `strategy: free` — не ждать самого медленного хоста:**
```yaml
- hosts: webservers
  strategy: free    # default: linear (ждёт всех на каждой task)
  # free: каждый хост выполняет tasks в своём темпе
```

**6. Mitogen — drop-in ускорение (x5-x10):**
```ini
# ansible.cfg
[defaults]
strategy_plugins = /path/to/mitogen/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
```
Mitogen переписывает транспортный слой Ansible, заменяя медленный SSH + Python на быстрый multiplexed transport.

**7. Теги — запускать только нужное:**
```bash
# Только установка пакетов
ansible-playbook deploy.yml --tags install

# Только конфигурация
ansible-playbook deploy.yml --tags config

# Пропустить тесты
ansible-playbook deploy.yml --skip-tags tests
```

**8. `--limit` — ограничить хосты:**
```bash
# Только один хост
ansible-playbook deploy.yml --limit web1.example.com

# Паттерн
ansible-playbook deploy.yml --limit "web*"
ansible-playbook deploy.yml --limit "webservers:&production"  # пересечение групп
```

**9. `serial` — rolling deploy:**
```yaml
- hosts: webservers
  serial: "25%"   # обновлять по 25% хостов за раз
  # или serial: 2 — по 2 хоста за раз
  # или serial: [1, 5, 10%] — сначала 1, потом 5, потом 10%

  max_fail_percentage: 10   # остановить если >10% хостов упали
```
