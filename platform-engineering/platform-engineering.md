# Platform Engineering: Вопросы и ответы для DevOps-инженера (Middle/Senior)

## Содержание

1. [Что такое Platform Engineering и чем он отличается от DevOps?](#1-что-такое-platform-engineering-и-чем-он-отличается-от-devops)
2. [Internal Developer Platform (IDP): компоненты и принципы](#2-internal-developer-platform-idp-компоненты-и-принципы)
3. [Backstage: архитектура, Software Catalog, Templates, TechDocs](#3-backstage-архитектура-software-catalog-templates-techdocs)
4. [Golden Paths: что это, как внедрить?](#4-golden-paths-что-это-как-внедрить)
5. [Developer Experience (DX): метрики и инструменты улучшения](#5-developer-experience-dx-метрики-и-инструменты-улучшения)
6. [Self-service инфраструктура: как дать разработчикам доступ без прав admin?](#6-self-service-инфраструктура-как-дать-разработчикам-доступ-без-прав-admin)
7. [Platform as a Product: продуктовый подход к внутренней платформе](#7-platform-as-a-product-продуктовый-подход-к-внутренней-платформе)
8. [Crossplane: управление облачными ресурсами через K8s CRD](#8-crossplane-управление-облачными-ресурсами-через-k8s-crd)

---

## 1. Что такое Platform Engineering и чем он отличается от DevOps?

**DevOps** — культура и практики, где Dev и Ops команды работают совместно для ускорения доставки и повышения надёжности.

**Platform Engineering** — специализированная дисциплина, где команда строит **Internal Developer Platform (IDP)** — продукт для других разработчиков.

```
DevOps фокус:
  "Как нам вместе быстрее доставлять ПО?"
  Shared responsibility, культура, процессы

Platform Engineering фокус:
  "Как нам создать платформу, которая АВТОМАТИЧЕСКИ
   обеспечивает best practices для всех разработчиков?"
  Product thinking, Developer Experience, Self-service
```

**Эволюция:**

```
Этап 1 (Ops era):
  Dev пишет → Ops деплоит (медленно, ITIL, Change Advisory Board)

Этап 2 (DevOps era):
  Команды владеют всем стеком, но каждая решает одинаковые проблемы по-своему
  "You build it, you run it" → дублирование усилий

Этап 3 (Platform Engineering era):
  Platform Team создаёт IDP — общий инструментарий
  Разработчики используют self-service, не зная о сложности инфраструктуры
  "You build it, platform runs it"
```

**Почему Platform Engineering набирает популярность:**

```
Проблемы без платформы:
  ✗ Каждая команда по-своему настраивает CI/CD
  ✗ Дублирование: 20 команд = 20 разных подходов к мониторингу
  ✗ "Cognitive load" на разработчиков: им нужно знать K8s, Terraform, AWS...
  ✗ Security drift: каждая команда делает свой security baseline
  ✗ Onboarding нового сервиса = недели

С платформой:
  ✓ Один стандартный способ создать новый сервис (Golden Path)
  ✓ Onboarding = нажать кнопку в Backstage → готовый репо + CI/CD + мониторинг
  ✓ Разработчики фокусируются на бизнес-логике, а не на инфраструктуре
  ✓ Security и compliance автоматически встроены
```

---

## 2. Internal Developer Platform (IDP): компоненты и принципы

**IDP** — набор инструментов и сервисов, предоставляемых платформенной командой для ускорения работы разработчиков.

**Пять плоскостей IDP (по Humanitec):**

```
1. Developer Control Plane (что видит разработчик)
   Backstage UI, CLI инструменты, документация
   
2. Integration and Delivery Plane (CI/CD)
   GitHub Actions, ArgoCD, Tekton
   
3. Monitoring and Logging Plane (observability)
   Prometheus, Grafana, Loki, Jaeger
   
4. Resource Plane (инфраструктура)
   Kubernetes, Databases, Object Storage, Queues
   
5. Security Plane (безопасность)
   Vault, cert-manager, OPA, Network Policy
```

**Что входит в хорошую IDP:**

```
Self-service capabilities:
  ✓ Создать новый сервис (репо + CI + K8s namespace + мониторинг)
  ✓ Деплоить приложение в dev/staging/prod
  ✓ Просмотреть логи и метрики своего сервиса
  ✓ Управлять секретами
  ✓ Запросить database, queue, bucket (через Crossplane)
  ✓ Масштабировать сервис

Не self-service (требует платформенной команды):
  - Создание нового кластера
  - Изменение network policies
  - Управление billing/accounts
```

---

## 3. Backstage: архитектура, Software Catalog, Templates, TechDocs

**Backstage** — open-source Developer Portal от Spotify, ставший де-факто стандартом IDP.

**Архитектура:**

```
┌─────────────────────────────────────────────────────────────┐
│                        Backstage                             │
│                                                             │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  Software Catalog│  │  Scaffolder  │  │  TechDocs    │   │
│  │  (все сервисы,   │  │  (Templates, │  │  (docs-as-   │   │
│  │   APIs, команды) │  │   new service│  │   code)      │   │
│  └──────────────────┘  └──────────────┘  └──────────────┘   │
│                                                             │
│  ┌──────────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  Kubernetes       │  │  CI/CD       │  │  Monitoring  │   │
│  │  Plugin           │  │  Plugin      │  │  Plugin      │   │
│  └──────────────────┘  └──────────────┘  └──────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           Plugin Ecosystem (200+ plugins)            │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
       ↑                    ↑                    ↑
  GitHub/GitLab         ArgoCD/Tekton        Grafana/PagerDuty
```

**Software Catalog — реестр всех компонентов:**

```yaml
# catalog-info.yaml в корне каждого репозитория
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: myapp
  description: "Main API service for e-commerce platform"
  annotations:
    github.com/project-slug: myorg/myapp
    backstage.io/techdocs-ref: dir:.
    argocd/app-name: myapp-production
    grafana/dashboard-selector: "app=myapp"
    pagerduty.com/service-id: "P1234567"
  tags:
    - java
    - api
    - production
  links:
    - url: https://myapp.example.com
      title: Production URL
    - url: https://grafana.example.com/d/myapp
      title: Grafana Dashboard
spec:
  type: service
  lifecycle: production
  owner: team-backend          # ссылка на Group
  system: e-commerce           # ссылка на System
  dependsOn:
    - component:postgres-db
    - component:redis-cache
  providesApis:
    - myapp-api
```

```yaml
# Описание API
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: myapp-api
spec:
  type: openapi
  lifecycle: production
  owner: team-backend
  definition:
    $text: ./openapi.yaml

---
# Описание команды
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: team-backend
spec:
  type: team
  profile:
    displayName: Backend Team
    email: backend@company.com
  parent: engineering
  members:
    - john.doe
    - jane.smith
```

**Software Templates — Scaffolder:**

```yaml
# Шаблон для создания нового Go сервиса
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: go-service-template
  title: Go Microservice
  description: Creates a new Go microservice with CI/CD and monitoring
  tags:
    - go
    - microservice
spec:
  owner: platform-team
  type: service

  parameters:
    - title: Service Info
      required:
        - name
        - description
        - owner
      properties:
        name:
          title: Service Name
          type: string
          pattern: "^[a-z][a-z0-9-]*$"
          description: "Lowercase with hyphens (e.g. my-service)"
        description:
          title: Description
          type: string
        owner:
          title: Owner Team
          type: string
          ui:field: OwnerPicker
          ui:options:
            allowedKinds: ["Group"]
        replicaCount:
          title: Initial Replicas
          type: number
          default: 2

    - title: Infrastructure
      properties:
        database:
          title: Need PostgreSQL database?
          type: boolean
          default: false
        queue:
          title: Need Kafka topic?
          type: boolean
          default: false

  steps:
    # 1. Создать репозиторий из шаблона
    - id: fetch-template
      name: Fetch Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}

    # 2. Создать GitHub репозиторий
    - id: create-repo
      name: Create GitHub Repository
      action: github:repo:create
      input:
        repoUrl: github.com?repo=${{ parameters.name }}&owner=myorg
        defaultBranch: main
        protectDefaultBranch: true
        requireCodeOwnerReviews: true

    # 3. Пуш кода в репозиторий
    - id: push-code
      name: Push Code
      action: github:repo:push
      input:
        repoUrl: ${{ steps['create-repo'].output.remoteUrl }}

    # 4. Создать ArgoCD Application
    - id: argocd-app
      name: Create ArgoCD Application
      action: argocd:create-application
      input:
        appName: ${{ parameters.name }}
        argoInstance: production
        projectName: ${{ parameters.owner }}
        repoUrl: ${{ steps['create-repo'].output.remoteUrl }}
        path: helm/${{ parameters.name }}
        namespace: ${{ parameters.name }}

    # 5. Создать Grafana Dashboard
    - id: grafana-dashboard
      name: Create Grafana Dashboard
      action: http:backstage:request
      input:
        method: POST
        path: /api/proxy/grafana/api/dashboards/db
        body:
          dashboard:
            title: ${{ parameters.name }}
            # ... dashboard template

    # 6. Зарегистрировать в Catalog
    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['create-repo'].output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml

  output:
    links:
      - title: Repository
        url: ${{ steps['create-repo'].output.remoteUrl }}
      - title: Backstage Component
        icon: catalog
        entityRef: ${{ steps['register'].output.entityRef }}
```

**TechDocs — документация как код:**

```yaml
# mkdocs.yml в корне репозитория
site_name: "MyApp Documentation"
nav:
  - Home: index.md
  - Architecture: architecture.md
  - API Reference: api.md
  - Runbooks:
    - Deployment: runbooks/deployment.md
    - Troubleshooting: runbooks/troubleshooting.md

plugins:
  - techdocs-core  # Backstage plugin
```

```bash
# Backstage автоматически генерирует и хостит документацию
# Разработчики пишут в Markdown рядом с кодом
# Backstage показывает доки прямо в UI
```

---

## 4. Golden Paths: что это, как внедрить?

**Golden Path** — рекомендуемый (и максимально упрощённый) путь для выполнения типичной задачи. Не обязательный — разработчики могут отклониться, но тогда берут на себя ответственность.

```
Golden Path для создания нового сервиса:
  1. Открыть Backstage → Software Templates
  2. Выбрать "Go Microservice" шаблон
  3. Заполнить форму (имя, команда, настройки)
  4. Нажать "Create"
  
  Результат (автоматически):
  ✓ GitHub repo создан с бойлерплейтом
  ✓ CI/CD pipeline настроен
  ✓ Docker build оптимизирован
  ✓ Kubernetes манифесты готовы
  ✓ ArgoCD Application создан
  ✓ Grafana Dashboard настроен
  ✓ AlertManager alerts настроены
  ✓ Component зарегистрирован в Catalog
  
  Время: 5 минут вместо 2-3 дней
```

**Принципы Golden Path:**

```
1. "Paved road, not a wall"
   Упрощён, но не единственный путь.
   Разработчики могут отклониться, но это осознанное решение.

2. Best practices встроены по умолчанию
   Security scanning, resource limits, health checks —
   не нужно помнить, они уже есть в шаблоне.

3. Progressively disclosed complexity
   Простые вещи — просты. Сложные — возможны, но не навязаны.

4. Self-service
   Разработчик не ждёт платформенную команду.
   Команды могут независимо создавать сервисы 24/7.

5. Измеримые метрики
   Как долго занимает создание нового сервиса?
   Сколько разработчиков используют Golden Path?
```

**Типичные Golden Paths:**

```
- Создать новый микросервис
- Добавить новый API endpoint (с документацией и тестами)
- Создать новый Cron Job
- Запросить новую базу данных
- Настроить feature flag
- Добавить новый алерт
- Создать shared library
```

---

## 5. Developer Experience (DX): метрики и инструменты улучшения

**Метрики Developer Experience:**

```
SPACE Framework (GitHub/Microsoft):
  S — Satisfaction       Насколько разработчики довольны инструментами?
  P — Performance        Outcomes: доставленные фичи, устранённые баги
  A — Activity           Количество коммитов, PR, деплоев (с осторожностью!)
  C — Communication      Насколько эффективна коллаборация?
  E — Efficiency         Насколько мало прерываний и когнитивной нагрузки?

DORA Metrics (косвенно измеряют DX):
  Deployment Frequency
  Lead Time for Changes
  Change Failure Rate
  MTTR

Специфические метрики DX:
  Time to first deploy (новый разработчик → первый prod деплой)
  Onboarding time (новый разработчик → полная продуктивность)
  Build time (время сборки CI)
  Local dev startup time
  PR cycle time (от создания до merge)
```

**Инструменты для улучшения DX:**

```bash
# 1. Локальная разработка в K8s
# Telepresence — подменить pod в кластере локальным процессом
telepresence connect
telepresence intercept myapp --port 8080:8080

# Tilt — hot-reload в K8s
tilt up  # Tiltfile описывает как собрать и деплоить в local K8s

# Skaffold
skaffold dev  # автосборка и деплой при изменении файлов

# 2. Development Environment (devcontainers)
# .devcontainer/devcontainer.json — стандартизированная среда разработки
{
  "name": "MyApp Dev",
  "image": "mcr.microsoft.com/devcontainers/go:1.23",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {}
  },
  "postCreateCommand": "make deps"
}
```

**Cognitive Load reduction:**

```
Высокий cognitive load = разработчик должен знать:
  - Как настроить K8s deployment
  - Как написать Dockerfile оптимально
  - Как настроить Prometheus метрики
  - Как создать Grafana dashboard
  - Как настроить AlertManager
  - Как работает cert-manager
  - ...

Platform Engineering задача:
  Скрыть всю эту сложность за Golden Path.
  Разработчик знает только: "Написать код → push → он задеплоится".
```

---

## 6. Self-service инфраструктура: как дать разработчикам доступ без прав admin?

**Проблема:**

```
Без self-service:
  Dev: "Мне нужна БД для тестирования"
  Platform: создать тикет → ждать 2 дня → получить credentials

С self-service через Crossplane/Backstage:
  Dev: нажать кнопку в Backstage
  Результат через 2 минуты: connection string в Kubernetes Secret
```

**Kubernetes RBAC для namespace-level self-service:**

```yaml
# Разработчики имеют права только в своём namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: team-backend
rules:
  # Полный доступ к своим ресурсам
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["*"]
  # Только чтение логов
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]
  # Нет доступа к Secrets (секреты управляются через Vault/ESO)
  # Нет доступа к NetworkPolicy, ResourceQuota (управляет платформа)
```

**Namespace-as-a-Service через Crossplane:**

```yaml
# Разработчик создаёт запрос на namespace через CRD
apiVersion: platform.company.com/v1alpha1
kind: DeveloperNamespace
metadata:
  name: my-feature-namespace
spec:
  team: team-backend
  purpose: "Integration testing for PR #123"
  expiresIn: "24h"
  resources:
    cpu: "4"
    memory: "8Gi"
  databases:
    - type: postgresql
      size: small

# Crossplane composition создаёт:
# → Namespace с RBAC
# → ResourceQuota
# → NetworkPolicy (стандартная)
# → PostgreSQL instance (через AWS RDS Crossplane provider)
# → Secret с credentials
# → Автоудаление через 24 часа (TTL controller)
```

---

## 7. Platform as a Product: продуктовый подход к внутренней платформе

**Ключевое изменение мышления:**

```
Традиционный подход:
  Platform Team = "Internal IT"
  Разработчики = "клиенты которых надо обслуживать"
  Метрика успеха = "нет инцидентов"

Product approach:
  Platform Team = Product Team
  Разработчики = Пользователи продукта (customers)
  Метрика успеха = "разработчики могут двигаться быстрее"
```

**Продуктовые практики для платформы:**

```
1. User Research
   Интервью с разработчиками: что болит? что замедляет?
   Developer Satisfaction Survey (ежеквартально)
   Office Hours: платформенная команда доступна для вопросов

2. Roadmap и Prioritization
   Backlog из user stories разработчиков
   Приоритизация по impact * adoption
   Публичный roadmap (видно что будет когда)

3. Adoption Metrics
   Сколько команд используют Golden Path?
   NPS (Net Promoter Score) для платформы
   Time to deploy новый сервис (baseline vs сейчас)

4. Documentation
   TechDocs в Backstage
   Video tutorials для сложных сценариев
   Changelog (разработчики знают что изменилось)

5. Support Model
   Slack channel для вопросов
   SLA на ответы (например, 4 часа в рабочее время)
   Runbooks для самостоятельного решения проблем
```

---

## 8. Crossplane: управление облачными ресурсами через K8s CRD

**Crossplane** — open-source Kubernetes extension, позволяющий управлять внешними ресурсами (AWS, GCP, Azure) через K8s CRD.

**Концепция:**

```
Без Crossplane:
  Разработчик → тикет → Platform Team → Terraform → AWS RDS → 2 дня

С Crossplane:
  Разработчик → K8s manifest → Crossplane → AWS RDS → 5 минут
  (через RBAC разработчик не видит credentials AWS)
```

**Архитектура:**

```
┌──────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                   │
│                                                      │
│  Developer creates: RDSInstance CRD                  │
│         ↓                                            │
│  Crossplane Composite Resource (XR)                  │
│    ↓ Composition (template)                          │
│    ├── AWS RDS Instance                              │
│    ├── Security Group                                │
│    └── K8s Secret (connection details)              │
│         ↓                                            │
│  aws-provider → AWS API → actual RDS                 │
└──────────────────────────────────────────────────────┘
```

**Пример: PostgreSQL as a Service:**

```yaml
# Composition — шаблон (создаётся платформенной командой)
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xpostgresqlinstances.platform.company.com
spec:
  compositeTypeRef:
    apiVersion: platform.company.com/v1alpha1
    kind: XPostgreSQLInstance

  resources:
    - name: rdsinstance
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            region: us-east-1
            instanceClass: db.t3.micro
            engine: postgres
            engineVersion: "15"
            skipFinalSnapshot: true
            publiclyAccessible: false
            dbSubnetGroupNameSelector:
              matchLabels:
                usage: crossplane-rds
      patches:
        - fromFieldPath: spec.parameters.storageGB
          toFieldPath: spec.forProvider.allocatedStorage
        - fromFieldPath: spec.parameters.size
          transforms:
            - type: map
              map:
                small: db.t3.micro
                medium: db.t3.small
                large: db.t3.medium
          toFieldPath: spec.forProvider.instanceClass

    - name: connection-secret
      base:
        apiVersion: kubernetes.crossplane.io/v1alpha1
        kind: Object
        spec:
          forProvider:
            manifest:
              apiVersion: v1
              kind: Secret
              # ... secret with connection details

---
# CompositeResourceDefinition — определяет CRD для разработчиков
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresqlinstances.platform.company.com
spec:
  group: platform.company.com
  names:
    kind: XPostgreSQLInstance
    plural: xpostgresqlinstances
  claimNames:
    kind: PostgreSQLInstance      # это то что видит разработчик
    plural: postgresqlinstances
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          properties:
            spec:
              properties:
                parameters:
                  properties:
                    storageGB:
                      type: integer
                      default: 20
                    size:
                      type: string
                      enum: [small, medium, large]
                      default: small
```

```yaml
# Разработчик создаёт (Claim — namespace-scoped):
apiVersion: platform.company.com/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: my-app-db
  namespace: team-backend
spec:
  parameters:
    storageGB: 50
    size: medium
  compositionSelector:
    matchLabels:
      provider: aws
      environment: production
  writeConnectionSecretToRef:
    name: my-app-db-credentials   # K8s Secret с connection string
```
