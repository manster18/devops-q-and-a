# GitOps и ArgoCD: Вопросы и ответы для DevOps-инженера (Middle/Senior)

## Содержание

1. [Что такое GitOps? Четыре принципа CNCF](#1-что-такое-gitops-четыре-принципа-cncf)
2. [Push vs Pull модели: в чём разница и почему Pull лучше?](#2-push-vs-pull-модели-в-чём-разница-и-почему-pull-лучше)
3. [ArgoCD: архитектура, компоненты, как работает reconciliation loop](#3-argocd-архитектура-компоненты-как-работает-reconciliation-loop)
4. [Application и AppProject: структура, multi-tenancy](#4-application-и-appproject-структура-multi-tenancy)
5. [Sync Policies: automated, selfHeal, prune, resource hooks](#5-sync-policies-automated-selfheal-prune-resource-hooks)
6. [App of Apps pattern и ApplicationSet](#6-app-of-apps-pattern-и-applicationset)
7. [Работа с несколькими кластерами в ArgoCD](#7-работа-с-несколькими-кластерами-в-argocd)
8. [Безопасность ArgoCD: RBAC, SSO/OIDC, секреты](#8-безопасность-argocd-rbac-ssoidc-секреты)
9. [Notifications и мониторинг ArgoCD](#9-notifications-и-мониторинг-argocd)
10. [Flux CD: сравнение с ArgoCD, когда что выбрать?](#10-flux-cd-сравнение-с-argocd-когда-что-выбрать)
11. [Управление конфигурацией: Helm, Kustomize, plain YAML в GitOps](#11-управление-конфигурацией-helm-kustomize-plain-yaml-в-gitops)
12. [Структура GitOps репозитория: mono-repo vs poly-repo](#12-структура-gitops-репозитория-mono-repo-vs-poly-repo)

---

## 1. Что такое GitOps? Четыре принципа CNCF

**GitOps** — операционная модель, при которой Git является единственным источником истины для декларативной инфраструктуры и приложений.

**Четыре принципа OpenGitOps (CNCF):**

```
1. Declarative (Декларативность)
   Весь desired state системы описан декларативно.
   "Что должно быть" не "как это сделать".
   Kubernetes YAML, Helm charts, Kustomize overlays.

2. Versioned and Immutable (Версионирование)
   Desired state хранится в Git.
   Immutable: изменения = новый коммит, не in-place edit.
   История, rollback, audit trail.

3. Pulled Automatically (Автоматическое вытягивание)
   Агент (ArgoCD/Flux) сам вытягивает изменения из Git.
   Не push из CI pipeline.
   Кластер не нужно "открывать" снаружи.

4. Continuously Reconciled (Постоянное согласование)
   Агент постоянно сравнивает desired state (Git) с actual state (кластер).
   При отклонении — автоматически исправляет.
   Drift detection и correction.
```

**Зачем GitOps:**

```
Проблемы без GitOps:
  ✗ "Кто и когда сделал этот kubectl apply?"
  ✗ Ручные изменения в кластере = drift от конфигурации
  ✗ CI pipeline требует доступ к кластеру (ключи, kubeconfig)
  ✗ Откат = вспомнить что было и повторить вручную
  ✗ Нет audit trail для изменений инфраструктуры

С GitOps:
  ✓ Каждое изменение = PR → review → merge → auto-deploy
  ✓ Git history = полная история изменений
  ✓ Rollback = git revert
  ✓ Compliance: все изменения прошли code review
  ✓ CI не имеет доступа к кластеру (только ArgoCD внутри)
```

---

## 2. Push vs Pull модели: в чём разница и почему Pull лучше?

**Push-based CD (классический CI/CD):**

```
Developer
    │
    ▼ git push
CI Pipeline (GitHub Actions / GitLab CI)
    │
    ▼ kubectl apply / helm upgrade  ← CI pipeline имеет kubeconfig!
Kubernetes Cluster
```

**Pull-based CD (GitOps):**

```
Developer
    │
    ▼ git push (configs)
Git Repository (desired state)
         ↑
ArgoCD / Flux (внутри кластера)
  - Постоянно опрашивает Git
  - Сравнивает desired vs actual
  - Применяет изменения САМИ
         ↓
Kubernetes Cluster (actual state)
```

**Почему Pull лучше:**

```
1. Безопасность:
   Push: CI pipeline имеет kubeconfig с широкими правами
   Pull: ArgoCD внутри кластера, CI не имеет доступа наружу
         Compromised CI = нет доступа к кластеру

2. Drift Detection:
   Push: если кто-то изменил кластер вручную — никто не знает
   Pull: ArgoCD мгновенно обнаружит и может откатить изменение

3. Audit Trail:
   Push: нужно смотреть логи CI для истории изменений
   Pull: Git history = полная история + кто одобрил (PR)

4. Disaster Recovery:
   Push: нужно перезапустить CI pipeline
   Pull: поднять ArgoCD → он восстановит весь desired state из Git

5. Multi-cluster:
   Push: нужен доступ к каждому кластеру из CI
   Pull: каждый кластер сам тянет свою конфигурацию
```

---

## 3. ArgoCD: архитектура, компоненты, как работает reconciliation loop

**Архитектура ArgoCD:**

```
┌──────────────────────────────────────────────────────────┐
│                    argocd namespace                       │
│                                                          │
│  ┌──────────────────┐   ┌─────────────────────────────┐  │
│  │  argocd-server   │   │  argocd-application-        │  │
│  │  (API + WebUI)   │   │  controller                 │  │
│  │  :8080           │   │  (reconciliation loop)      │  │
│  └──────────────────┘   └─────────────────────────────┘  │
│                                                          │
│  ┌──────────────────┐   ┌─────────────────────────────┐  │
│  │  argocd-repo-    │   │  argocd-dex-server          │  │
│  │  server          │   │  (SSO/OIDC)                 │  │
│  │  (git ops,       │   └─────────────────────────────┘  │
│  │   helm/kustomize │                                    │
│  │   rendering)     │   ┌─────────────────────────────┐  │
│  └──────────────────┘   │  argocd-redis               │  │
│                         │  (кэш состояния)            │  │
│  ┌──────────────────┐   └─────────────────────────────┘  │
│  │  argocd-         │                                    │
│  │  notifications   │                                    │
│  │  (alerts)        │                                    │
│  └──────────────────┘                                    │
└──────────────────────────────────────────────────────────┘
```

**Компоненты:**

| Компонент | Назначение |
|-----------|-----------|
| **argocd-server** | API сервер (gRPC/REST) + WebUI. Получает запросы от CLI и UI |
| **application-controller** | Reconciliation loop. Сравнивает desired/actual, применяет sync |
| **repo-server** | Клонирует Git, рендерит Helm/Kustomize/plain YAML в K8s манифесты |
| **dex-server** | OIDC провайдер для SSO (GitHub, Google, LDAP) |
| **redis** | Кэш для application state, repo cache |
| **notifications** | Отправка уведомлений (Slack, email, PagerDuty) при изменении состояния |

**Reconciliation Loop:**

```
Каждые X секунд (timeout.reconciliation=180s default):

1. Repo Server: клонирует/обновляет Git репозиторий
2. Repo Server: рендерит манифесты (helm template / kustomize build)
3. App Controller: получает desired state от Repo Server
4. App Controller: запрашивает actual state из K8s API
5. App Controller: diff(desired, actual) → Sync Status

Если OutOfSync И syncPolicy.automated:
6. App Controller: применяет diff (kubectl apply)
7. App Controller: обновляет Application status

Drift пример:
  Git: replicas=3  ← desired
  K8s: replicas=5  ← кто-то поменял вручную → OutOfSync → selfHeal
```

---

## 4. Application и AppProject: структура, multi-tenancy

**Application CRD — основной объект ArgoCD:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  # Финализатор: при удалении Application удалить также K8s ресурсы
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: deployments
spec:
  # Проект для multi-tenancy
  project: team-backend

  # Источник: где взять желаемое состояние
  source:
    repoURL: https://github.com/myorg/myapp-helm
    targetRevision: HEAD    # или конкретный тег/commit: v1.2.3, abc1234
    path: helm/myapp

    # Helm специфика
    helm:
      releaseName: myapp
      valueFiles:
        - values.yaml
        - values.production.yaml
      parameters:
        - name: image.tag
          value: "abc1234"
      # Или Kustomize:
      # kustomize:
      #   namePrefix: prod-
      #   images:
      #     - myapp=registry/myapp:v1.0.0

  # Назначение: куда деплоить
  destination:
    server: https://kubernetes.default.svc    # или URL другого кластера
    namespace: production

  # Политика синхронизации
  syncPolicy:
    automated:
      prune: true         # удалять ресурсы, которых нет в Git
      selfHeal: true      # исправлять manual changes
      allowEmpty: false   # не применять если rendered manifests пусты

    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true    # только изменённые ресурсы

    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # Игнорировать определённые поля при diff
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas    # игнорировать изменения replicas (HPA управляет)
    - group: ""
      kind: ServiceAccount
      jsonPointers:
        - /secrets          # token secrets добавляются K8s автоматически
```

**AppProject — multi-tenancy и RBAC:**

```yaml
# Разные команды → разные проекты → разные права
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-backend
  namespace: argocd
spec:
  description: "Backend team applications"

  # Откуда можно деплоить (whitelist репозиториев)
  sourceRepos:
    - https://github.com/myorg/backend-*
    - https://github.com/myorg/shared-helm-charts

  # Куда можно деплоить (whitelist кластеров и namespace)
  destinations:
    - server: https://kubernetes.default.svc
      namespace: production
    - server: https://kubernetes.default.svc
      namespace: staging

  # Какие K8s ресурсы разрешено создавать
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace   # разрешить создание namespace

  # Запретить определённые ресурсы
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota  # команда не может создавать ResourceQuota

  # Запрос на роли в проекте
  roles:
    - name: developer
      description: Developer role
      policies:
        - p, proj:team-backend:developer, applications, get, team-backend/*, allow
        - p, proj:team-backend:developer, applications, sync, team-backend/*, allow
      groups:
        - myorg:backend-developers  # GitHub team
```

---

## 5. Sync Policies: automated, selfHeal, prune, resource hooks

**Sync Status и Health Status:**

```
Sync Status:
  Synced     → actual = desired (Git)
  OutOfSync  → есть отличия
  Unknown    → не удалось получить состояние

Health Status:
  Healthy    → все ресурсы работают
  Progressing → деплой в процессе (Deployment rollout)
  Degraded   → проблемы (CrashLoopBackOff, failed Job)
  Missing    → ресурс не существует в кластере
  Suspended  → CronJob suspended
  Unknown    → неизвестно
```

**Ручная синхронизация:**

```bash
# Через CLI
argocd app sync myapp --revision v1.2.3

# Только определённые ресурсы
argocd app sync myapp --resource apps:Deployment:myapp

# С dry-run (не применять, только показать diff)
argocd app diff myapp

# Force (принудительно даже при конфликте)
argocd app sync myapp --force
```

**Resource Hooks — custom actions при sync:**

```yaml
# Pre-sync hook: выполнить до применения манифестов
# Например: database migration
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded  # удалить после успеха
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:latest
          command: ["./migrate", "--up"]
      restartPolicy: Never

---
# Sync hook: выполнить во время sync
# PostSync hook: после успешного sync (smoke tests)
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: test
          image: curlimages/curl
          command: ["curl", "-f", "https://myapp.example.com/health"]
      restartPolicy: Never

---
# SyncFail hook: выполнить если sync завершился с ошибкой
# Например: отправить уведомление или откатить
apiVersion: batch/v1
kind: Job
metadata:
  name: notify-failure
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
```

**Sync Waves — порядок применения ресурсов:**

```yaml
# Ресурсы применяются в порядке волн (от меньшего к большему)
# По умолчанию: wave 0

# Сначала база данных
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "-2"  # применить раньше

# Потом приложение
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # default

# Последним — ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # применить последним
```

---

## 6. App of Apps pattern и ApplicationSet

**App of Apps — управление множеством приложений:**

```
Идея: одно "корневое" ArgoCD Application управляет другими Applications.

gitops-config/
  apps/                          ← repo-server рендерит эту директорию
    myapp.yaml                   ← Application CRD для myapp
    database.yaml                ← Application CRD для database
    monitoring.yaml              ← Application CRD для monitoring
  
  root-app.yaml                  ← корневое Application (вручную применяется 1 раз)
```

```yaml
# root-app.yaml (применяется вручную один раз)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-config
    targetRevision: HEAD
    path: apps/               # здесь лежат yaml с Application объектами
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd         # Applications создаются в argocd namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**ApplicationSet — генерация множества Applications:**

```yaml
# Вместо ручного создания Application для каждого сервиса/среды
# ApplicationSet генерирует их по шаблону

# Пример 1: один сервис в нескольких кластерах
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-clusters
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: staging
            url: https://staging-cluster.example.com
            env: staging
          - cluster: production
            url: https://prod-cluster.example.com
            env: production

  template:
    metadata:
      name: "myapp-{{cluster}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/myapp
        targetRevision: HEAD
        path: helm/myapp
        helm:
          valueFiles:
            - values.yaml
            - "values.{{env}}.yaml"
      destination:
        server: "{{url}}"
        namespace: myapp
      syncPolicy:
        automated:
          prune: true
          selfHeal: true

---
# Пример 2: Git Directory generator — папка = приложение
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-from-dirs
spec:
  generators:
    - git:
        repoURL: https://github.com/myorg/gitops-config
        revision: HEAD
        directories:
          - path: "apps/*/helm"   # каждая директория = приложение

  template:
    metadata:
      name: "{{path.basename}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/gitops-config
        targetRevision: HEAD
        path: "{{path}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{path.basename}}"

---
# Пример 3: Pull Request generator — preview environments
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: preview-envs
spec:
  generators:
    - pullRequest:
        github:
          owner: myorg
          repo: myapp
          tokenRef:
            secretName: github-token
            key: token
          labels:
            - preview   # только PR с этим label

  template:
    metadata:
      name: "preview-pr-{{number}}"
    spec:
      source:
        repoURL: https://github.com/myorg/myapp
        targetRevision: "{{head_sha}}"
        path: helm/myapp
        helm:
          parameters:
            - name: ingress.host
              value: "pr-{{number}}.preview.example.com"
      destination:
        server: https://kubernetes.default.svc
        namespace: "preview-pr-{{number}}"
      syncPolicy:
        automated:
          prune: true
        syncOptions:
          - CreateNamespace=true
```

---

## 7. Работа с несколькими кластерами в ArgoCD

**Регистрация внешних кластеров:**

```bash
# Используем ArgoCD CLI для добавления кластера
argocd cluster add staging-context \
  --name staging \
  --in-cluster  # если ArgoCD запущен в этом же кластере

# ArgoCD создаёт ServiceAccount в целевом кластере
# и хранит kubeconfig в Secret

# Просмотр кластеров
argocd cluster list

# Если нужно добавить кластер вручную (ServiceAccount токен)
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-manager
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-manager-role
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
  - nonResourceURLs: ["*"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-manager-role-binding
subjects:
  - kind: ServiceAccount
    name: argocd-manager
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: argocd-manager-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

**Hub-Spoke (или Management Cluster) модель:**

```
Management Cluster:
  ArgoCD (один инстанс управляет всеми)
    │
    ├── Deploys to → Cluster A (dev)
    ├── Deploys to → Cluster B (staging)
    └── Deploys to → Cluster C (production)

Плюсы:
  + Одна точка управления
  + Единый UI и audit trail
  + Легко добавлять новые кластеры

Минусы:
  - Single point of failure (ArgoCD)
  - Нужны сетевые маршруты от management к spoke кластерам
```

**Namespace в Argo CD для мультикластера:**

```yaml
# Application targeting другой кластер
spec:
  destination:
    server: https://production-cluster.example.com  # URL другого кластера
    namespace: production

# ApplicationSet с несколькими кластерами через Cluster generator
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production  # labels на cluster secrets в argocd namespace
```

---

## 8. Безопасность ArgoCD: RBAC, SSO/OIDC, секреты

**Встроенный RBAC ArgoCD:**

```yaml
# argocd-rbac-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly    # дефолтная роль для всех залогиненных
  policy.csv: |
    # Формат: p, <subject>, <resource>, <action>, <object>, allow|deny
    
    # Роль: developer — может синхронизировать staging, только смотреть production
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, staging/*, allow
    p, role:developer, applications, sync, production/*, deny
    
    # Роль: operator — полный доступ к production
    p, role:operator, applications, *, production/*, allow
    p, role:operator, clusters, get, *, allow
    
    # Роль: readonly
    p, role:readonly, applications, get, */*, allow
    
    # Привязка групп к ролям
    g, myorg:developers, role:developer
    g, myorg:operators, role:operator
    g, myorg:managers, role:readonly

  # Или использовать scopes из OIDC token
  scopes: "[groups, email]"
```

**SSO через GitHub (Dex):**

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.example.com

  dex.config: |
    connectors:
      - type: github
        id: github
        name: GitHub
        config:
          clientID: $dex-github-client-id        # из Secret
          clientSecret: $dex-github-client-secret
          orgs:
            - name: myorg                         # только члены организации
              teams:
                - developers
                - operators
          loadAllGroups: false
          teamNameField: slug
```

**Repository credentials (секреты для приватных репо):**

```bash
# SSH
argocd repo add git@github.com:myorg/gitops-config.git \
  --ssh-private-key-path ~/.ssh/argocd_ed25519

# HTTPS с токеном
argocd repo add https://github.com/myorg/gitops-config \
  --username git \
  --password ghp_TOKEN

# Через K8s Secret напрямую
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitops-repo-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://github.com/myorg/gitops-config
  password: ghp_TOKEN
  username: git
EOF
```

---

## 9. Notifications и мониторинг ArgoCD

**ArgoCD Notifications:**

```yaml
# argocd-notifications-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Slack template
  template.app-deployed: |
    message: |
      Application {{.app.metadata.name}} deployed successfully.
      Revision: {{.app.status.sync.revision}}
      Environment: {{.app.spec.destination.namespace}}
    slack:
      attachments: |
        [{
          "color": "good",
          "title": "✅ {{.app.metadata.name}}",
          "fields": [
            {"title": "Namespace", "value": "{{.app.spec.destination.namespace}}", "short": true},
            {"title": "Revision", "value": "{{.app.status.sync.revision}}", "short": true}
          ]
        }]

  template.app-sync-failed: |
    message: |
      ❌ Application {{.app.metadata.name}} sync FAILED!
    slack:
      attachments: |
        [{
          "color": "danger",
          "title": "Sync Failed: {{.app.metadata.name}}",
          "text": "{{.app.status.conditions | toJson}}"
        }]

  # Триггеры
  trigger.on-deployed: |
    - description: Application is synced and healthy
      send:
        - app-deployed
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'

  trigger.on-sync-failed: |
    - description: Application sync has failed
      send:
        - app-sync-failed
      when: app.status.operationState.phase in ['Error', 'Failed']

  # Услуга (service = куда отправлять)
  service.slack: |
    token: $slack-token
    username: ArgoCD
    icon: ":argo:"
```

```yaml
# Подписка на уведомления на уровне Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: deployments
    notifications.argoproj.io/subscribe.on-sync-failed.slack: alerts
    notifications.argoproj.io/subscribe.on-sync-failed.pagerduty: ""
```

**Мониторинг ArgoCD через Prometheus:**

```promql
# Количество приложений по состоянию
argocd_app_info{sync_status="OutOfSync"}
argocd_app_info{health_status="Degraded"}

# Время последней синхронизации
time() - argocd_app_info{sync_status="Synced"} > 3600
# Приложения не синхронизировались больше часа

# Количество deployments в час
rate(argocd_app_reconcile_count[1h])

# Ошибки репо сервера
rate(argocd_repo_pending_requests_total[5m])
```

**Полезные ArgoCD CLI команды:**

```bash
# Статус всех приложений
argocd app list

# Детальный статус
argocd app get myapp
argocd app get myapp --show-params   # параметры Helm

# Diff что изменится
argocd app diff myapp

# История деплоев
argocd app history myapp

# Откат к предыдущей версии
argocd app rollback myapp 2   # номер из history

# Принудительный sync с конкретного revision
argocd app sync myapp --revision v1.2.3

# Заморозить автосинк (для maintenance)
argocd app set myapp --sync-policy none

# Вернуть автосинк
argocd app set myapp --sync-policy automated

# Посмотреть ресурсы приложения
argocd app resources myapp

# Логи pod'а через ArgoCD
argocd app logs myapp -c myapp --follow
```

---

## 10. Flux CD: сравнение с ArgoCD, когда что выбрать?

**Flux CD** — альтернативный GitOps operator от Weaveworks, теперь CNCF Graduated.

**Архитектура Flux (v2):**

```
Flux состоит из нескольких controllers:
  source-controller:      отслеживает Git repos, Helm repos, OCI registries
  kustomize-controller:   применяет Kustomize/plain YAML
  helm-controller:        управляет Helm releases
  notification-controller: уведомления и webhooks
  image-automation-controller: автообновление image tags
  image-reflector-controller:  сканирование registry для новых тегов
```

**Flux пример:**

```yaml
# GitRepository — source
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/myapp
  ref:
    branch: main
  secretRef:
    name: github-token

---
# Kustomization — применение конфигурации
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 10m
  path: "./k8s/production"
  prune: true
  sourceRef:
    kind: GitRepository
    name: myapp
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: myapp
      namespace: production
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars   # подстановка переменных
```

**Flux Image Automation — автообновление образов:**

```yaml
# Автоматически обновлять image tag в Git при появлении нового образа

# 1. Сканировать registry
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: registry.example.com/myapp
  interval: 1m

# 2. Policy выбора тега (semver)
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: myapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    semver:
      range: ">=1.0.0"   # последний semver

# 3. Автоматически коммитить в Git
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@example.com
        name: fluxcdbot
      messageTemplate: 'Update image to {{range .Updated.Images}}{{println .}}{{end}}'
    push:
      branch: main
```

**Сравнение ArgoCD vs Flux:**

| Параметр | ArgoCD | Flux |
|----------|--------|------|
| UI | Богатый встроенный WebUI | Нет встроенного UI (есть Weave GitOps) |
| Multi-cluster | Встроено (hub-spoke) | Каждый кластер имеет свой Flux |
| Image automation | Через плагины | Встроено нативно |
| CLI | argocd CLI | flux CLI |
| Multi-tenant | AppProject | Kustomization по namespace |
| Learning curve | Средний | Ниже (K8s-native подход) |
| CNCF статус | Graduated | Graduated |

**Когда ArgoCD:**
```
✓ Нужен богатый UI для visibility
✓ Команда предпочитает centralized управление
✓ Нужен multi-cluster hub-spoke
✓ Сложная multi-tenancy с AppProject
✓ Организация где не все знают K8s глубоко
```

**Когда Flux:**
```
✓ Чисто GitOps подход без UI (GitOps as Code)
✓ Image automation важна (автодеплой новых тегов)
✓ Команда предпочитает K8s-native CRDs
✓ Каждый кластер управляет собой (no hub)
✓ Интеграция с Weave GitOps Enterprise
```

---

## 11. Управление конфигурацией: Helm, Kustomize, plain YAML в GitOps

**Plain YAML:**

```
Плюсы: простота, нет дополнительных инструментов
Минусы: дублирование конфигурации между environments

Подходит для: маленьких проектов, простых конфигураций
```

**Kustomize:**

```
Подход: base + overlays (без шаблонизации)
Встроен в kubectl (kubectl apply -k .)

gitops-config/
  base/
    deployment.yaml
    service.yaml
    kustomization.yaml
  overlays/
    dev/
      kustomization.yaml   # patches для dev
    staging/
      kustomization.yaml   # patches для staging
    production/
      kustomization.yaml   # patches для production
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"

# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base

patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/cpu
        value: "500m"

images:
  - name: myapp
    newTag: v1.2.3

namePrefix: prod-
commonLabels:
  environment: production
```

**Helm в GitOps:**

```yaml
# ArgoCD с Helm
spec:
  source:
    repoURL: https://charts.example.com
    chart: myapp
    targetRevision: "1.2.3"   # ВАЖНО: фиксировать версию!
    helm:
      values: |
        replicaCount: 5
        image:
          tag: "abc1234"
        ingress:
          enabled: true
          hosts:
            - host: myapp.example.com

# Helm Chart в Git (не в registry)
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    path: helm/myapp
    targetRevision: v1.2.3
    helm:
      valueFiles:
        - values.yaml
        - values.production.yaml
```

**Рекомендации:**

```
Используй Kustomize если:
  - Хочешь чистый declarative подход
  - Небольшое число кастомизаций между environments
  - Нет сложной шаблонизации

Используй Helm если:
  - Сложные шаблоны с условиями
  - Открытый Charts реестр (публичные charts)
  - Команда уже использует Helm

НИКОГДА не делай helm install вручную в GitOps!
Весь state должен быть в Git.
```

---

## 12. Структура GitOps репозитория: mono-repo vs poly-repo

**Mono-repo (один репозиторий для всего):**

```
gitops-config/                    ← один репозиторий
  apps/
    team-backend/
      myapp/
        base/
        overlays/
          staging/
          production/
      auth-service/
    team-frontend/
      webapp/
  infrastructure/
    networking/
    monitoring/
    cert-manager/
  clusters/
    staging/
      apps.yaml       # ApplicationSet или root Application
    production/
      apps.yaml

Плюсы:
  + Видна вся конфигурация разом
  + Легко делать cross-service изменения
  + Один PR для изменений нескольких сервисов

Минусы:
  - Большой репозиторий → медленный Clone
  - Сложнее разграничить доступ (CODEOWNERS)
  - Большой blast radius: одна ошибка → все приложения
```

**Poly-repo (отдельный репозиторий для каждого приложения):**

```
gitops-config-myapp/        ← repo для myapp
  helm/
  overlays/

gitops-config-auth/         ← repo для auth-service
  helm/
  overlays/

gitops-config-infrastructure/  ← repo для инфраструктуры
  terraform/
  k8s/

Плюсы:
  + Чёткие границы ответственности
  + Меньший blast radius
  + Легче разграничить доступ
  + Меньше repo = быстрее clone для ArgoCD

Минусы:
  - Сложнее отследить взаимосвязи
  - Cross-service изменения = несколько PR
  - Дублирование конфигурации
```

**Рекомендуемая структура (гибрид):**

```
Разделить App code и GitOps config:

myapp/                          ← app code repo
  src/
  Dockerfile
  .github/workflows/
    ci.yml                      # тесты, сборка, пуш образа
    update-gitops.yml           # обновляет image tag в gitops repo

gitops-config/                  ← отдельный GitOps repo
  apps/
    myapp/
      base/
      overlays/
  infrastructure/
  clusters/

Workflow:
  1. Developer: push code → myapp repo
  2. CI: build image, push to registry (myapp:abc1234)
  3. CI: PR в gitops-config: update image.tag=abc1234
  4. Review: одобрить PR
  5. Merge: ArgoCD обнаруживает изменение, синхронизирует
```

```yaml
# .github/workflows/update-gitops.yml (в app repo)
- name: Update GitOps config
  run: |
    git clone https://x-access-token:${{ secrets.GITOPS_TOKEN }}@github.com/myorg/gitops-config
    cd gitops-config
    
    # Kustomize: обновить image tag
    kustomize edit set image myapp=registry/myapp:${{ github.sha }}
    
    # или sed для values.yaml
    sed -i "s/tag: .*/tag: \"${{ github.sha }}\"/" apps/myapp/overlays/staging/values.yaml
    
    git config user.email "ci@example.com"
    git config user.name "CI Bot"
    git add -A
    git commit -m "Update myapp to ${{ github.sha }}"
    git push
```
