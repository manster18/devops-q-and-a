# Kubernetes — Вопросы и ответы для собеседований (Middle/Senior DevOps)

## Содержание

### Архитектура
1. [Из каких компонентов состоит Kubernetes? Архитектура кластера.](#1-из-каких-компонентов-состоит-kubernetes-архитектура-кластера)
2. [Что происходит внутри кластера при выполнении kubectl apply?](#2-что-происходит-внутри-кластера-при-выполнении-kubectl-apply)
3. [Что такое etcd и почему он критически важен?](#3-что-такое-etcd-и-почему-он-критически-важен)

### Рабочие нагрузки (Workloads)
4. [Что такое Pod? Жизненный цикл и фазы.](#4-что-такое-pod-жизненный-цикл-и-фазы)
5. [В чём разница между Deployment, StatefulSet, DaemonSet, Job и CronJob?](#5-в-чём-разница-между-deployment-statefulset-daemonset-job-и-cronjob)
6. [Что такое Init Containers и Sidecar Containers?](#6-что-такое-init-containers-и-sidecar-containers)
7. [Как работает rolling update и как сделать rollback?](#7-как-работает-rolling-update-и-как-сделать-rollback)

### Сеть
8. [Как устроена сеть в Kubernetes? Модель сети, CNI.](#8-как-устроена-сеть-в-kubernetes-модель-сети-cni)
9. [Типы Services: ClusterIP, NodePort, LoadBalancer, Headless. Чем отличаются?](#9-типы-services-clusterip-nodeport-loadbalancer-headless-чем-отличаются)
10. [Что такое Ingress и как он работает?](#10-что-такое-ingress-и-как-он-работает)
11. [Что такое NetworkPolicy и как ограничить трафик между подами?](#11-что-такое-networkpolicy-и-как-ограничить-трафик-между-подами)

### Хранилище
12. [Как работают PersistentVolume, PersistentVolumeClaim и StorageClass?](#12-как-работают-persistentvolume-persistentvolumeclaim-и-storageclass)

### Конфигурация и ресурсы
13. [В чём разница между ConfigMap и Secret?](#13-в-чём-разница-между-configmap-и-secret)
14. [Что такое requests и limits? Почему это важно?](#14-что-такое-requests-и-limits-почему-это-важно)
15. [Что такое QoS классы подов?](#15-что-такое-qos-классы-подов)

### Планирование (Scheduling)
16. [Как работает планировщик? nodeSelector, nodeAffinity, taints и tolerations.](#16-как-работает-планировщик-nodeselector-nodeaffinity-taints-и-tolerations)
17. [Что такое Pod Affinity и Pod Anti-Affinity?](#17-что-такое-pod-affinity-и-pod-anti-affinity)

### Жизненный цикл и здоровье
18. [Чем отличаются liveness, readiness и startup пробы?](#18-чем-отличаются-liveness-readiness-и-startup-пробы)
19. [Как работает graceful shutdown пода в Kubernetes?](#19-как-работает-graceful-shutdown-пода-в-kubernetes)

### Безопасность
20. [Как работает RBAC в Kubernetes?](#20-как-работает-rbac-в-kubernetes)
21. [Что такое ServiceAccount и зачем он нужен?](#21-что-такое-serviceaccount-и-зачем-он-нужен)
22. [Что такое SecurityContext?](#22-что-такое-securitycontext)

### Масштабирование и производительность
23. [Как работает HPA (Horizontal Pod Autoscaler)?](#23-как-работает-hpa-horizontal-pod-autoscaler)
24. [Что такое VPA и Cluster Autoscaler?](#24-что-такое-vpa-и-cluster-autoscaler)

### Инструменты
25. [Что такое Helm и зачем он нужен?](#25-что-такое-helm-и-зачем-он-нужен)

---

## Архитектура

### 1. Из каких компонентов состоит Kubernetes? Архитектура кластера.

```
┌─────────────────────────────────────────────────────────────┐
│                     Control Plane                           │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ API Server  │  │  Scheduler   │  │Controller Manager │  │
│  │(kube-apisrv)│  │(kube-schedul)│  │ (kube-ctrl-mgr)   │  │
│  └──────┬──────┘  └──────┬───────┘  └─────────┬─────────┘  │
│         │                │                    │             │
│  ┌──────▼────────────────▼────────────────────▼──────────┐  │
│  │                      etcd                              │  │
│  └────────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────┐                        │
│  │    cloud-controller-manager     │ (опционально, в облаке)│
│  └─────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────┘

┌──────────────────────────┐  ┌──────────────────────────────┐
│         Worker Node 1    │  │        Worker Node 2          │
│                          │  │                               │
│  ┌──────────┐  ┌───────┐ │  │  ┌──────────┐  ┌──────────┐  │
│  │  kubelet │  │k-proxy│ │  │  │  kubelet │  │ k-proxy  │  │
│  └──────────┘  └───────┘ │  │  └──────────┘  └──────────┘  │
│  ┌──────────────────────┐ │  │  ┌──────────────────────────┐│
│  │  Container Runtime   │ │  │  │   Container Runtime      ││
│  │  (containerd/CRI-O)  │ │  │  │   (containerd/CRI-O)     ││
│  └──────────────────────┘ │  │  └──────────────────────────┘│
│  [Pod] [Pod] [Pod]        │  │  [Pod] [Pod]                 │
└──────────────────────────┘  └──────────────────────────────┘
```

**Компоненты Control Plane:**

**kube-apiserver** — единственная точка входа в кластер. Все компоненты (kubectl, kubelet, controllers) общаются только через API Server. Валидирует запросы, авторизует, сохраняет состояние в etcd. Горизонтально масштабируется.

**etcd** — распределённое key-value хранилище. Единственное место где хранится всё состояние кластера. Использует алгоритм консенсуса Raft. Для надёжности нужно нечётное количество экземпляров (3, 5).

**kube-scheduler** — смотрит на неназначенные поды (без `nodeName`) и выбирает для них подходящий узел. Учитывает: requests/limits, taints/tolerations, affinity, topologySpreadConstraints, доступные ресурсы.

**kube-controller-manager** — запускает контроллеры (reconciliation loops). Каждый контроллер следит за состоянием объектов и приводит actual state к desired state:
- Deployment controller → управляет ReplicaSets
- ReplicaSet controller → следит за нужным числом Pod
- Node controller → обнаруживает упавшие узлы
- Job controller → запускает поды до завершения
- И ещё ~30 контроллеров

**cloud-controller-manager** — интеграция с облачным провайдером: создание LoadBalancer'ов, persistent volumes, маршрутов.

**Компоненты Worker Node:**

**kubelet** — агент на каждом узле. Получает PodSpec от API Server и обеспечивает что контейнеры запущены и здоровы. Общается с container runtime через CRI (Container Runtime Interface).

**kube-proxy** — сетевой прокси на каждом узле. Реализует Service абстракцию — программирует iptables или ipvs правила для маршрутизации трафика к подам.

**Container Runtime** — запускает контейнеры. Kubernetes взаимодействует через CRI: containerd, CRI-O, Docker (через dockershim, устарело).

---

### 2. Что происходит внутри кластера при выполнении kubectl apply?

Разберём полный путь на примере `kubectl apply -f deployment.yaml`.

```
1. kubectl
   └─ Читает YAML, конвертирует в JSON
   └─ Отправляет PATCH/POST запрос на API Server
      (HTTPS, с клиентским сертификатом или токеном)

2. API Server
   └─ Authentication: кто делает запрос? (cert, token, OIDC)
   └─ Authorization: имеет ли право? (RBAC)
   └─ Admission Control:
      └─ Mutating Webhooks: модифицируют объект
         (inject sidecar, set defaults, add labels)
      └─ Validation: валидируют схему объекта
      └─ Validating Webhooks: бизнес-логика валидации
   └─ Сохраняет объект в etcd
   └─ Отправляет Event всем подписчикам (watch)

3. Deployment Controller (в controller-manager)
   └─ Получает Event: создан/изменён Deployment
   └─ Смотрит текущий ReplicaSet
   └─ Создаёт новый ReplicaSet (при изменении template)
   └─ Управляет rolling update

4. ReplicaSet Controller
   └─ Получает Event: создан ReplicaSet, replicas=3
   └─ Смотрит: подов с нужным selector = 0
   └─ Создаёт 3 объекта Pod в etcd (они ещё не запущены!)
      Pod status: Pending, nodeName: ""

5. Scheduler
   └─ Смотрит на Pending поды без nodeName
   └─ Фильтрация: какие узлы подходят?
      (ресурсы, taints, nodeSelector, affinity)
   └─ Ранжирование: какой узел лучше?
   └─ Обновляет Pod.spec.nodeName = "worker-node-2"

6. kubelet (на worker-node-2)
   └─ Видит: появился Pod с nodeName = мой узел
   └─ Скачивает образы через container runtime (pull)
   └─ Создаёт sandbox (pause контейнер, network namespace)
   └─ Запускает init containers (последовательно)
   └─ Запускает основные контейнеры
   └─ Запускает postStart hook
   └─ Начинает проверять startup/liveness/readiness пробы
   └─ Обновляет Pod status → Running

7. Endpoints/EndpointSlice Controller
   └─ Видит: Pod Ready=True
   └─ Добавляет Pod IP в EndpointSlice сервиса

8. kube-proxy (на всех узлах)
   └─ Видит: обновился EndpointSlice
   └─ Обновляет iptables/ipvs правила на узле
   └─ Теперь трафик к Service может маршрутизироваться на новый Pod
```

**Ключевой принцип:** API Server никогда не "командует" компонентами напрямую. Все компоненты подписаны на изменения через `watch` и реагируют сами — это **event-driven**, **eventually consistent** архитектура.

---

### 3. Что такое etcd и почему он критически важен?

**etcd** — распределённое, строго консистентное key-value хранилище. В Kubernetes хранит абсолютно всё: поды, деплойменты, секреты, конфигурации, состояние узлов.

**Алгоритм Raft — консенсус:**
- Один лидер принимает записи, реплицирует на followers
- Запись считается успешной когда большинство (quorum) подтвердило
- Quorum = N/2 + 1: для 3 узлов нужно 2, для 5 нужно 3
- При потере лидера — автоматические выборы нового

```
Нечётное число узлов (рекомендуется):
1 узел  — нет HA, quorum = 1
3 узла  — выдерживает потерю 1 узла, quorum = 2
5 узлов — выдерживает потерю 2 узлов, quorum = 3
```

**Бэкап и восстановление — обязательно знать:**
```bash
# Бэкап (выполнять на любом etcd-члене)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd.crt \
  --key=/etc/etcd/etcd.key

# Проверить бэкап
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db

# Восстановление (на каждом узле!)
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=etcd-node-1 \
  --initial-cluster="etcd-node-1=https://10.0.0.1:2380" \
  --initial-advertise-peer-urls=https://10.0.0.1:2380
```

**Производительность etcd — что важно:**
- etcd очень чувствителен к latency диска. Нужны быстрые NVMe SSD.
- Проверить задержки: `etcdctl check perf` — задержки >50ms — проблема.
- Дефрагментация (компактизация): со временем etcd накапливает ревизии, нужна периодическая очистка:
```bash
# Компактизация (удалить старые ревизии)
ETCDCTL_API=3 etcdctl compact $(etcdctl endpoint status --write-out="json" | jq '.[0].Status.header.revision')

# Дефрагментация (освободить диск)
ETCDCTL_API=3 etcdctl defrag --endpoints=https://127.0.0.1:2379
```

---

## Рабочие нагрузки (Workloads)

### 4. Что такое Pod? Жизненный цикл и фазы.

**Pod** — минимальная единица деплоя в Kubernetes. Один или несколько контейнеров, которые:
- Разделяют **сетевой namespace** (один IP, общий localhost)
- Разделяют **volumes**
- Запускаются и останавливаются вместе

Большинство Pod'ов содержат один контейнер. Несколько контейнеров — паттерн sidecar (логи, proxy, sync).

**Фазы жизненного цикла:**

| Фаза | Описание |
|---|---|
| `Pending` | Принят API Server, но не запущен. Ожидает назначения узла, скачивания образа. |
| `Running` | Назначен узел, хотя бы один контейнер запущен. |
| `Succeeded` | Все контейнеры завершились с кодом 0 (Jobs). |
| `Failed` | Хотя бы один контейнер завершился с ненулевым кодом. |
| `Unknown` | Нет связи с узлом, где запущен Pod. |

**Условия (Conditions) — более детальная картина:**
```bash
kubectl describe pod mypod
# Conditions:
#   Type              Status
#   Initialized       True   ← init containers завершились
#   Ready             True   ← readiness probe прошла, Pod в Service
#   ContainersReady   True   ← все контейнеры готовы
#   PodScheduled      True   ← назначен на узел
```

**restart policy — что происходит при падении контейнера:**
```yaml
spec:
  restartPolicy: Always      # всегда перезапускать (default для Deployment)
  restartPolicy: OnFailure   # только при ненулевом exit code (Jobs)
  restartPolicy: Never       # не перезапускать
```
При перезапуске используется **exponential backoff**: 10с → 20с → 40с → 80с → 160с → 300с (max). Сбрасывается если Pod был Running >10 минут.

---

### 5. В чём разница между Deployment, StatefulSet, DaemonSet, Job и CronJob?

**Deployment**
Управляет подами без состояния (stateless). Поды взаимозаменяемы — можно убить любой, создать новый на любом узле.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.25
```

**StatefulSet**
Для stateful-приложений (базы данных, очереди). Гарантирует:
- **Стабильные имена подов:** `mydb-0`, `mydb-1`, `mydb-2` (не рандомные)
- **Стабильные DNS-имена:** `mydb-0.mydb-service.namespace.svc.cluster.local`
- **Упорядоченный запуск/остановка:** 0 → 1 → 2 (и обратно при удалении)
- **Стабильное хранилище:** каждый под получает свой PVC через `volumeClaimTemplates`

```yaml
kind: StatefulSet
spec:
  serviceName: "mydb"
  replicas: 3
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
# Создаст: data-mydb-0, data-mydb-1, data-mydb-2 (отдельные PVC)
```

**DaemonSet**
Гарантирует что **на каждом узле** запущена ровно одна копия пода. При добавлении нового узла — под создаётся автоматически.

Применение: агенты мониторинга (Node Exporter, Datadog), сборщики логов (Fluentd, Filebeat), сетевые плагины (Calico, Cilium), агенты безопасности.

```yaml
kind: DaemonSet
# tolerations обычно добавляют чтобы запускалось и на control plane узлах:
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
```

**Job**
Запускает поды до **успешного завершения**. Для разовых задач: миграции БД, обработка батча, генерация отчётов.
```yaml
kind: Job
spec:
  completions: 5      # запустить 5 успешных выполнений
  parallelism: 2      # по 2 параллельно
  backoffLimit: 3     # максимум 3 перезапуска при ошибке
```

**CronJob**
Запускает Job по расписанию (cron-синтаксис).
```yaml
kind: CronJob
spec:
  schedule: "0 3 * * *"          # каждый день в 3:00
  concurrencyPolicy: Forbid      # не запускать если предыдущий ещё идёт
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:latest
```

---

### 6. Что такое Init Containers и Sidecar Containers?

**Init Containers**
Запускаются **строго последовательно до** основных контейнеров. Каждый должен успешно завершиться (exit 0) — только тогда запустится следующий. Если init container упал — Pod перезапускается (по restartPolicy).

Применение:
- Ожидание готовности зависимостей (БД, другой сервис)
- Скачивание конфигов, ключей из vault
- Инициализация файловой системы
- Запуск миграций БД

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c',
      'until nc -z postgres-service 5432; do echo waiting for db; sleep 2; done']

  - name: run-migrations
    image: myapp:latest
    command: ["python", "manage.py", "migrate"]
    env:
    - name: DB_HOST
      value: "postgres-service"

  containers:
  - name: app
    image: myapp:latest
    # стартует только после успешного завершения обоих init containers
```

**Sidecar Containers (Kubernetes 1.29+)**
Традиционно sidecar — это просто обычный контейнер в том же поде. С версии 1.29 появился официальный тип `initContainers` с `restartPolicy: Always` — такой контейнер запускается как init (до основных), но **не завершается** и продолжает работу всё время жизни пода.

```yaml
spec:
  initContainers:
  - name: log-collector          # нативный sidecar
    image: fluentd:latest
    restartPolicy: Always        # не завершается, живёт как sidecar
    volumeMounts:
    - name: logs
      mountPath: /var/log/app

  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: logs
      mountPath: /app/logs
```

**Классические паттерны sidecar (обычные контейнеры):**
- **Ambassador:** прокси исходящих запросов (например, Envoy для service mesh)
- **Adapter:** трансформация вывода приложения (конвертация форматов логов, метрик)
- **Sidecar proxy:** Istio/Linkerd inject Envoy прокси для mTLS, observability

---

### 7. Как работает rolling update и как сделать rollback?

**Rolling Update** обновляет поды постепенно: запускает новые и останавливает старые, не прерывая трафик.

```yaml
kind: Deployment
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # разрешить создать N дополнительных подов (сверх replicas)
      maxUnavailable: 1  # максимум N подов могут быть недоступны одновременно
```

**Процесс при `maxSurge: 2, maxUnavailable: 1` и 6 репликах:**
```
Старт: [v1 v1 v1 v1 v1 v1]   — 6 old
Шаг 1: [v1 v1 v1 v1 v1 v2 v2 v2] — поднято 2 new, 6 old (+2 surge)
Шаг 2: [v1 v1 v1 v1 v2 v2 v2]   — убито 1 old (maxUnavailable=1)
Шаг 3: ...пока не заменим все...
Конец: [v2 v2 v2 v2 v2 v2]   — 6 new
```

**Условие прогресса:** новый под должен пройти readiness probe прежде чем старый удалится.

**Стратегия Recreate** (для приложений не поддерживающих несколько версий):
```yaml
strategy:
  type: Recreate  # убить всё → создать всё (downtime!)
```

**Команды управления:**
```bash
# Обновить образ (триггер rolling update)
kubectl set image deployment/web web=nginx:1.26

# Посмотреть прогресс
kubectl rollout status deployment/web
kubectl rollout status deployment/web --watch

# История ревизий (нужен --record или аннотация)
kubectl rollout history deployment/web
kubectl rollout history deployment/web --revision=3

# Откатиться к предыдущей ревизии
kubectl rollout undo deployment/web

# Откатиться к конкретной ревизии
kubectl rollout undo deployment/web --to-revision=2

# Приостановить rolling update (для canary)
kubectl rollout pause deployment/web
# Снять паузу
kubectl rollout resume deployment/web
```

**Аннотация причины изменения (сохраняется в историю):**
```bash
kubectl annotate deployment/web kubernetes.io/change-cause="Updated nginx to 1.26 for CVE-2024-XXXX"
```

---

## Сеть

### 8. Как устроена сеть в Kubernetes? Модель сети, CNI.

**Фундаментальные требования сетевой модели K8s:**
1. Каждый Pod получает **уникальный IP** в кластере
2. Поды могут общаться друг с другом **без NAT** (напрямую по Pod IP)
3. Нода может общаться с любым подом **без NAT**

**Три типа адресного пространства:**
- **Node CIDR** — IP-адреса самих узлов (например, 10.0.0.0/24)
- **Pod CIDR** — IP-адреса подов (например, 10.244.0.0/16) — каждому узлу выделяется подсеть
- **Service CIDR** — виртуальные IP сервисов (например, 10.96.0.0/12) — не существуют в сети реально

**CNI (Container Network Interface)** — стандарт плагинов, которые реализуют сеть для контейнеров. При создании пода kubelet вызывает CNI-плагин, который:
1. Создаёт сетевой namespace для пода
2. Создаёт виртуальный интерфейс (veth pair): один конец в Pod, другой на хосте
3. Назначает IP из Pod CIDR
4. Настраивает маршруты

**Популярные CNI-плагины:**

| Плагин | Режим | Особенности |
|---|---|---|
| **Calico** | BGP или VXLAN | NetworkPolicy, eBPF режим, производительный |
| **Cilium** | eBPF | Продвинутые NetworkPolicy, observability, service mesh |
| **Flannel** | VXLAN | Простой, минимум функций, нет NetworkPolicy |
| **Weave** | Mesh | Простой, NetworkPolicy |
| **AWS VPC CNI** | Native VPC | Поды получают реальные VPC IP (нет overlay) |

**Как пакет идёт от Pod A (node-1) к Pod B (node-2):**
```
Pod A (10.244.1.5)
  └─ veth интерфейс → bridge на node-1 → маршрутизация
     └─ VXLAN encapsulation (если overlay) → eth0 node-1
        └─ Физическая сеть → eth0 node-2
           └─ VXLAN decapsulation → bridge → veth → Pod B (10.244.2.7)
```

**DNS в кластере (CoreDNS):**
```bash
# Каждый под имеет /etc/resolv.conf:
# nameserver 10.96.0.10  ← ClusterIP сервиса kube-dns
# search default.svc.cluster.local svc.cluster.local cluster.local

# Полное DNS-имя сервиса:
# <service-name>.<namespace>.svc.cluster.local

# Примеры резолюции (из пода в namespace "default"):
curl http://myservice                                          # работает
curl http://myservice.default                                  # работает
curl http://myservice.default.svc.cluster.local               # полное имя
curl http://myservice.other-namespace.svc.cluster.local       # другой namespace
```

---

### 9. Типы Services: ClusterIP, NodePort, LoadBalancer, Headless. Чем отличаются?

**Service** — стабильный виртуальный IP и DNS-имя для набора подов (выбираются через `selector`). Абстрагирует от конкретных подов, которые могут появляться и исчезать.

**ClusterIP (default)**
Доступен только внутри кластера. kube-proxy создаёт iptables/ipvs правила: трафик на ClusterIP → один из Pod IP'ов (случайный выбор, round-robin).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP     # по умолчанию
  selector:
    app: myapp
  ports:
  - port: 80          # порт самого сервиса
    targetPort: 8080  # порт контейнера
```

**NodePort**
Дополнительно открывает порт (30000-32767) на **каждом узле** кластера. Трафик на `<любой-узел-IP>:<nodePort>` → Service → Pod.
```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080   # опционально, иначе назначается случайно
```
Используется: для dev-окружений, bare-metal, когда нет облачного LB.

**LoadBalancer**
Расширяет NodePort: дополнительно заказывает у облачного провайдера внешний балансировщик (AWS ELB, GCP LB, Azure LB), который направляет трафик на NodePort'ы.
```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 8080
# После создания: .status.loadBalancer.ingress[0].ip = 34.xx.xx.xx
```

**ExternalName**
DNS CNAME-псевдоним для внешнего сервиса. Не создаёт iptables правила — просто возвращает CNAME в DNS ответе.
```yaml
spec:
  type: ExternalName
  externalName: mydb.example.com
# Поды обращаются к "mydb-service" → получают CNAME → mydb.example.com
```
Полезно для миграции: внешняя БД "прячется" за K8s Service — при переносе внутрь меняем только Service.

**Headless Service (ClusterIP: None)**
Нет виртуального IP. DNS возвращает **A-записи напрямую на Pod IP**. Используется StatefulSet для stable DNS-имён каждого пода.
```yaml
spec:
  clusterIP: None
  selector:
    app: mydb
# DNS запрос к "mydb-service" → [10.244.1.5, 10.244.2.7, 10.244.3.3]
# DNS запрос к "mydb-0.mydb-service" → [10.244.1.5] (конкретный под StatefulSet)
```

---

### 10. Что такое Ingress и как он работает?

**Проблема:** LoadBalancer Service создаёт отдельный внешний балансировщик для каждого сервиса — дорого и неудобно. В облаке каждый ELB стоит денег.

**Ingress** — объект Kubernetes, описывающий правила маршрутизации HTTP/HTTPS трафика к сервисам по **Host** и **Path**.

**Ingress Controller** — реализация Ingress. Должен быть установлен отдельно (в кластере нет встроенного). Сам является Deployment + LoadBalancer Service.

```
Интернет → LoadBalancer (1 штука!) → Ingress Controller Pod
                                           └─ читает Ingress rules
                                           ├─ /api → api-service:80
                                           ├─ /    → frontend-service:80
                                           └─ blog.example.com → blog-service:80
```

**Пример Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    - api.example.com
    secretName: example-tls     # cert-manager создаёт этот Secret автоматически
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80

  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 8080
```

**Популярные Ingress Controllers:**
- **ingress-nginx** — самый популярный, на базе Nginx
- **Traefik** — автоматическое обнаружение сервисов, встроенный Let's Encrypt
- **AWS ALB Controller** — создаёт Application Load Balancer в AWS
- **HAProxy Ingress**
- **Istio Gateway** — часть service mesh

**cert-manager — автоматические TLS-сертификаты:**
```yaml
# ClusterIssuer для Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

---

### 11. Что такое NetworkPolicy и как ограничить трафик между подами?

По умолчанию в Kubernetes **все поды могут общаться со всеми** — нет сетевой изоляции. `NetworkPolicy` позволяет задать whitelist-правила для входящего (ingress) и исходящего (egress) трафика.

> **Важно:** NetworkPolicy работает только если CNI-плагин поддерживает его (Calico, Cilium, Weave). Flannel — не поддерживает.

**Запретить весь входящий трафик к подам в namespace (default-deny):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}      # {} = все поды в namespace
  policyTypes:
  - Ingress
  # нет ingress правил = запрещён весь входящий трафик
```

**Разрешить трафик только от конкретных подов:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres       # к этим подам

  policyTypes:
  - Ingress
  - Egress

  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend    # только от backend подов
    - namespaceSelector:
        matchLabels:
          name: monitoring  # И от мониторинга (другой namespace)
    ports:
    - protocol: TCP
      port: 5432

  egress:
  - to:
    - namespaceSelector: {}  # исходящий DNS разрешён
    ports:
    - protocol: UDP
      port: 53
```

**Разрешить только конкретный namespace:**
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: frontend
  - podSelector:
      matchLabels:
        app: web
  # ВАЖНО: podSelector + namespaceSelector в одном элементе = AND (оба условия)
  # podSelector + namespaceSelector в разных элементах = OR (любое из условий)
```

**Типичная production-стратегия:**
1. `default-deny-ingress` + `default-deny-egress` — запретить всё
2. Явно добавлять разрешения для каждого взаимодействия
3. Разрешить DNS (UDP 53 к kube-dns)
4. Разрешить трафик от Ingress Controller к сервисам

---

## Хранилище

### 12. Как работают PersistentVolume, PersistentVolumeClaim и StorageClass?

**Трёхуровневая абстракция:**

```
┌─────────────────────────────────┐
│  StorageClass                   │ ← "рецепт" создания хранилища
│  provisioner: kubernetes.io/aws │   (тип диска, параметры)
│  parameters: type=gp3           │
└────────────────┬────────────────┘
                 │ создаёт автоматически
┌────────────────▼────────────────┐
│  PersistentVolume (PV)          │ ← реальный кусок хранилища
│  capacity: 20Gi                 │   (EBS диск, NFS share, etc.)
│  accessModes: ReadWriteOnce     │
│  reclaimPolicy: Delete          │
└────────────────┬────────────────┘
                 │ привязывается к
┌────────────────▼────────────────┐
│  PersistentVolumeClaim (PVC)    │ ← запрос пода на хранилище
│  requests: storage: 20Gi        │
│  accessModes: ReadWriteOnce     │
└────────────────┬────────────────┘
                 │ монтируется в
┌────────────────▼────────────────┐
│  Pod                            │
│  volumeMounts: /data            │
└─────────────────────────────────┘
```

**StorageClass — динамическое provisioning:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # дефолтный
provisioner: ebs.csi.aws.com    # AWS EBS CSI driver
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete           # Delete или Retain
allowVolumeExpansion: true      # разрешить увеличение размера
volumeBindingMode: WaitForFirstConsumer  # не создавать PV пока Pod не запланирован
```

**PersistentVolumeClaim:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  storageClassName: fast-ssd
  accessModes:
  - ReadWriteOnce               # RWO: один узел, RWX: многие, ROX: многие (read-only)
  resources:
    requests:
      storage: 20Gi
```

**Использование в Pod/StatefulSet:**
```yaml
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: database-pvc
  containers:
  - name: db
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
```

**Reclaim Policy:**
- `Delete` — PV и данные удаляются при удалении PVC (для облачных дисков)
- `Retain` — PV остаётся (статус Released), данные сохранены, требует ручной очистки
- `Recycle` — устарело, удалялся только контент (rm -rf)

**Расширение PVC (если StorageClass поддерживает):**
```bash
kubectl patch pvc database-pvc -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
```

---

## Конфигурация и ресурсы

### 13. В чём разница между ConfigMap и Secret?

**ConfigMap** — хранит незашифрованные конфигурационные данные (строки, файлы).

**Secret** — хранит чувствительные данные. В etcd хранится **base64-encoded** (не зашифрован по умолчанию!). Для реального шифрования нужен Encryption at Rest или внешний менеджер секретов (Vault, AWS SSM).

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  config.yaml: |
    server:
      port: 8080
      timeout: 30s

# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=    # base64("password123")
  API_KEY: c2VjcmV0a2V5           # base64("secretkey")
stringData:                        # можно в открытом виде, K8s сам закодирует
  JWT_SECRET: "my-jwt-secret"
```

**Способы использования в поде:**

```yaml
spec:
  containers:
  - name: app
    # 1. Переменные окружения из ConfigMap/Secret
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DB_PASSWORD

    # 2. Все ключи как переменные окружения
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secrets

    volumeMounts:
    # 3. Монтировать ConfigMap как файлы
    - name: config-volume
      mountPath: /etc/app
    # 4. Монтировать Secret как файлы
    - name: secrets-volume
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: secrets-volume
    secret:
      secretName: app-secrets
      defaultMode: 0400   # права доступа к файлам
```

**Immutable ConfigMap/Secret (K8s 1.21+):**
```yaml
immutable: true  # нельзя изменить, только удалить и пересоздать
# Преимущество: kubelet не следит за изменениями → снижение нагрузки на API Server
```

**Важные замечания о Secret:**
- Файловые дескрипторы в памяти (tmpfs) — не записываются на диск, если правильно настроено
- Нужно ограничить доступ через RBAC (кто может делать `get secrets`)
- Для production: External Secrets Operator + AWS Secrets Manager / HashiCorp Vault

---

### 14. Что такое requests и limits? Почему это важно?

**Requests** — гарантированное количество ресурсов. Scheduler использует requests при выборе узла (узел должен иметь столько свободных ресурсов).

**Limits** — максимально допустимое потребление. При превышении:
- **CPU limit:** процесс throttled (замедляется, не убивается)
- **Memory limit:** процесс убивается (OOM Kill) — появляется статус `OOMKilled`

```yaml
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "250m"       # 250 millicores = 0.25 ядра
        memory: "256Mi"
      limits:
        cpu: "1000m"      # 1 ядро
        memory: "512Mi"
```

**Единицы измерения:**
- CPU: `1` = 1 vCPU/core, `500m` = 0.5 ядра, `100m` = 0.1 ядра
- Memory: `256Mi` (мебибайты), `1Gi` (гибибайты), `512M` (мегабайты)

**Почему обязательно настраивать requests:**
```
Без requests: Scheduler не знает сколько ресурсов нужно поду → может 
             разместить 50 подов на узле с 4 CPU → все задыхаются
             
С requests:  Scheduler видит "занятые" ресурсы → честное распределение
```

**Почему limits тоже важны:**
```
Без limits: один бракованный под с утечкой памяти съедает всю память узла
            → OOM Killer убивает случайные процессы включая kubelet → узел NotReady
```

**Проверить реальное потребление (Metrics Server нужен):**
```bash
kubectl top pods
kubectl top nodes
kubectl top pod mypod --containers   # по каждому контейнеру
```

---

### 15. Что такое QoS классы подов?

Kubernetes автоматически присваивает **QoS (Quality of Service)** класс каждому поду на основе requests/limits. Класс влияет на приоритет при нехватке ресурсов (кого убивать первым).

**Guaranteed** — самый высокий приоритет:
- Все контейнеры должны иметь requests = limits (и для CPU, и для memory)
- OOM Killer убивает таких подов последними
```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"    # = requests
    memory: "256Mi"  # = requests
```

**Burstable** — средний приоритет:
- Хотя бы один контейнер имеет requests или limits
- requests ≠ limits
```yaml
resources:
  requests:
    memory: "128Mi"
  limits:
    memory: "256Mi"
```

**BestEffort** — самый низкий приоритет (убиваются первыми):
- Ни у одного контейнера нет requests/limits
- Получают ресурсы по остаточному принципу
```yaml
# resources вообще не указаны
```

```bash
# Проверить QoS класс
kubectl get pod mypod -o jsonpath='{.status.qosClass}'
```

**Практический вывод:** для production-подов всегда настраивай requests и limits. Для критических сервисов (БД, message broker) используй QoS Guaranteed.

---

## Планирование (Scheduling)

### 16. Как работает планировщик? nodeSelector, nodeAffinity, taints и tolerations.

**Алгоритм планировщика — два шага:**

**1. Filtering (фильтрация):** какие узлы подходят?
- Достаточно ли ресурсов (requests)
- nodeSelector совпадает
- Нет непереносимых taints
- PVC может быть примонтирован на этом узле
- Pod Anti-Affinity не нарушена

**2. Scoring (ранжирование):** из подходящих — какой лучший?
- Наибольшее количество свободных ресурсов
- Наименьшее количество уже запущенных подов приложения
- Pod Affinity предпочтения

---

**nodeSelector** — простейший способ, выбрать узел по метке:
```yaml
spec:
  nodeSelector:
    disktype: ssd
    topology.kubernetes.io/zone: us-east-1a
```

**nodeAffinity** — более гибкий вариант с операторами:
```yaml
spec:
  affinity:
    nodeAffinity:
      # Обязательное требование (Hard)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values: ["amd64"]
          - key: node-type
            operator: NotIn
            values: ["spot"]   # не запускать на spot-инстансах

      # Предпочтение (Soft) — планировщик постарается выбрать, но не обязан
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
```

---

**Taints и Tolerations** — механизм "отталкивания" подов от узлов.

**Taint** — метка на узле, говорящая "не запускай сюда поды без разрешения".
```bash
# Добавить taint на узел
kubectl taint nodes node-1 dedicated=gpu:NoSchedule
kubectl taint nodes node-1 dedicated=gpu:NoExecute      # выселить уже запущенные
kubectl taint nodes node-1 dedicated=gpu:PreferNoSchedule

# Удалить taint
kubectl taint nodes node-1 dedicated=gpu:NoSchedule-
```

**Toleration** — разрешение пода "терпеть" taint:
```yaml
spec:
  tolerations:
  # Разрешить запуск на GPU-узлах
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"

  # Разрешить запуск на всех tainted узлах (оператор Exists)
  - operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300   # выдержать 300 секунд перед выселением
```

**Типичные use cases:**
```bash
# Control plane узлы помечаются автоматически
kubectl taint nodes master node-role.kubernetes.io/control-plane:NoSchedule

# GPU узлы — только GPU-задачи
kubectl taint nodes gpu-node nvidia.com/gpu=present:NoSchedule

# Spot/Preemptible узлы — помечаем чтобы критичные поды туда не попадали
kubectl taint nodes spot-node cloud.google.com/gke-spot:NoSchedule
```

---

### 17. Что такое Pod Affinity и Pod Anti-Affinity?

В отличие от nodeAffinity (Pod относительно узла), **podAffinity** описывает отношения между подами — "хочу быть рядом с подами X" или "не хочу быть рядом с подами Y".

**podAffinity** — запустить рядом с другим подом (на том же узле или в той же зоне):
```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache       # запустить на том же узле что и cache
        topologyKey: kubernetes.io/hostname
```

**podAntiAffinity** — не запускать рядом (для HA: разные узлы, разные зоны):
```yaml
spec:
  affinity:
    podAntiAffinity:
      # Обязательно — поды одного Deployment на РАЗНЫХ узлах
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname  # разные узлы

      # Предпочтительно — в разных availability zones
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: topology.kubernetes.io/zone  # разные AZ
```

**`topologySpreadConstraints`** — более современный и гибкий способ равномерного распределения:
```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                              # максимальный дисбаланс
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule       # или ScheduleAnyway
    labelSelector:
      matchLabels:
        app: web
  # Гарантирует что разница в количестве подов между зонами не превышает 1
```

---

## Жизненный цикл и здоровье

### 18. Чем отличаются liveness, readiness и startup пробы?

**Startup Probe** — проверяет что приложение **запустилось**. Пока не прошла — liveness и readiness отключены. Для медленно стартующих приложений (JVM, legacy).
```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  failureThreshold: 30   # 30 * 10s = 300 секунд максимум на старт
  periodSeconds: 10
```

**Liveness Probe** — проверяет что приложение **живо**. Провал → kubelet перезапускает контейнер. Для обнаружения deadlock, когда процесс жив но завис.
```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 15   # ждать 15 сек до первой проверки
  periodSeconds: 20          # проверять каждые 20 сек
  timeoutSeconds: 3          # таймаут запроса
  failureThreshold: 3        # 3 провала → перезапуск
  successThreshold: 1
```

**Readiness Probe** — проверяет что приложение **готово принимать трафик**. Провал → Pod убирается из Endpoints Service (трафик не идёт). Контейнер **не перезапускается**.
```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  periodSeconds: 5
  failureThreshold: 3        # 3 провала → убрать из Service
  successThreshold: 2        # 2 успеха → вернуть в Service
```

**Три типа проверок:**
```yaml
# HTTP GET — успех если статус 200-399
httpGet:
  path: /health
  port: 8080
  httpHeaders:
  - name: X-Health-Check
    value: "true"

# TCP Socket — успех если соединение установлено
tcpSocket:
  port: 5432

# Exec — успех если exit code = 0
exec:
  command:
  - sh
  - -c
  - "redis-cli ping | grep -q PONG"

# gRPC (K8s 1.24+)
grpc:
  port: 9090
  service: "grpc.health.v1.Health"
```

**Типичная ошибка:** liveness probe проверяет внешние зависимости (БД, Redis). Если БД недоступна → все поды перезапускаются → cascade failure. Liveness должна проверять только сам процесс (deadlock, corrupted state). Внешние зависимости — в readiness.

---

### 19. Как работает graceful shutdown пода в Kubernetes?

Это критически важно для zero-downtime деплоев. При удалении пода происходит следующее:

```
kubectl delete pod / rolling update / scale down
            │
            ▼
API Server меняет Pod.status → Terminating
            │
            ├──────────────────────────────┐
            ▼                              ▼
   kubelet получает сигнал         Endpoint Controller
            │                      удаляет Pod IP из
            ▼                      EndpointSlice →
   1. Выполняет preStop hook        kube-proxy убирает
      (если задан)                  iptables правило
            │                      (занимает несколько секунд!)
            ▼
   2. Отправляет SIGTERM
      приложению
            │
            ▼
   3. Ждёт terminationGracePeriodSeconds
      (default: 30s)
            │                      
            ▼
   4. Если не завершилось — SIGKILL
```

**Проблема race condition:**
Удаление Pod IP из kube-proxy происходит асинхронно и занимает время. Если SIGTERM придёт раньше чем iptables обновится — часть запросов придёт на уже завершающийся под.

**Решение — preStop hook с задержкой:**
```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]
          # дать kube-proxy время обновить iptables
          # только потом SIGTERM дойдёт до приложения
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo 'Container started' >> /var/log/events.log"]

    # Приложение должно обрабатывать SIGTERM и завершать in-flight запросы
```

**Правильная настройка:**
```yaml
terminationGracePeriodSeconds: 60  # должен быть > preStop + время обработки запросов
# preStop sleep: 5-15 секунд (достаточно для обновления kube-proxy)
# Время завершения приложения: зависит от длины запросов
```

**Проверить что приложение корректно обрабатывает SIGTERM:**
```bash
kubectl exec mypod -- kill -15 1        # отправить SIGTERM напрямую
kubectl delete pod mypod --grace-period=0 --force  # принудительное удаление (НЕ для production)
```

---

## Безопасность

### 20. Как работает RBAC в Kubernetes?

**RBAC (Role-Based Access Control)** — контроль доступа на основе ролей. Отвечает на вопрос: кто (subject) может делать что (verb) с чем (resource).

**Четыре объекта:**

| Объект | Scope | Описание |
|---|---|---|
| `Role` | Namespace | Набор разрешений в одном namespace |
| `ClusterRole` | Кластер | Набор разрешений во всём кластере |
| `RoleBinding` | Namespace | Привязывает Role/ClusterRole к subject в namespace |
| `ClusterRoleBinding` | Кластер | Привязывает ClusterRole к subject глобально |

```yaml
# Role — что можно делать (в namespace "production")
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]                # "" = core API group
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]

---
# RoleBinding — кому дать эту Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: production
subjects:
- kind: User
  name: "developer@example.com"   # пользователь
- kind: Group
  name: "dev-team"                # группа
- kind: ServiceAccount
  name: my-service-account        # ServiceAccount
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRole для cross-namespace доступа:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-manager
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["nodes"]            # nodes существуют только на cluster level
  verbs: ["get", "list"]
- nonResourceURLs: ["/healthz", "/metrics"]  # non-resource endpoints
  verbs: ["get"]
```

**Глаголы (verbs):**
`get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`

**Проверить права:**
```bash
# Может ли текущий пользователь делать действие
kubectl auth can-i create deployments --namespace production
kubectl auth can-i "*" "*"   # всё?

# Проверить права другого пользователя (от имени admin)
kubectl auth can-i list pods --namespace production --as developer@example.com

# Просмотреть все права ServiceAccount
kubectl auth can-i --list --as system:serviceaccount:production:my-sa
```

---

### 21. Что такое ServiceAccount и зачем он нужен?

**ServiceAccount** — identity для процессов внутри пода. Позволяет подам аутентифицироваться в Kubernetes API и взаимодействовать с кластером.

```yaml
# Создать ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
  annotations:
    # Для AWS IRSA (IAM Roles for Service Accounts)
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role
```

**Использование в поде:**
```yaml
spec:
  serviceAccountName: my-app-sa   # по умолчанию: "default"
  automountServiceAccountToken: true   # монтировать token в /var/run/secrets/kubernetes.io/serviceaccount/
```

**Файлы монтируются автоматически:**
```bash
/var/run/secrets/kubernetes.io/serviceaccount/
├── token      ← JWT token для аутентификации в API
├── ca.crt     ← сертификат CA для проверки API Server
└── namespace  ← текущий namespace
```

**Пример: SA с правами на управление деплойментами (CI/CD):**
```yaml
# ServiceAccount для CI/CD деплоя
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer-role
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "update", "patch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: deployer
roleRef:
  kind: Role
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

**Best practice:** не использовать `default` ServiceAccount, создавать отдельный с минимальными правами для каждого приложения. Отключать automounting там где API-доступ не нужен:
```yaml
spec:
  automountServiceAccountToken: false
```

---

### 22. Что такое SecurityContext?

`SecurityContext` — настройки безопасности на уровне пода или отдельного контейнера.

```yaml
spec:
  # securityContext на уровне пода
  securityContext:
    runAsNonRoot: true      # запретить запуск от root
    runAsUser: 1000         # UID
    runAsGroup: 3000        # GID
    fsGroup: 2000           # GID для volume'ов (все файлы принадлежат этой группе)
    seccompProfile:
      type: RuntimeDefault  # применить дефолтный seccomp профиль

  containers:
  - name: app
    # securityContext на уровне контейнера (переопределяет pod-level)
    securityContext:
      allowPrivilegeEscalation: false   # нельзя повысить привилегии (sudo, SUID)
      readOnlyRootFilesystem: true      # корневая ФС только для чтения
      capabilities:
        drop:
        - ALL                           # убрать все capabilities
        add:
        - NET_BIND_SERVICE              # только нужные
      seccompProfile:
        type: RuntimeDefault

    volumeMounts:
    - name: tmp
      mountPath: /tmp          # разрешаем запись только в /tmp
    - name: cache
      mountPath: /app/cache

  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

**Pod Security Standards (замена устаревшему PSP):**
```yaml
# Применить стандарт на уровне namespace через labels
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted   # или baseline / privileged
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

Три уровня:
- `privileged` — без ограничений
- `baseline` — минимальные ограничения (запрет privileged, hostPath, hostNetwork)
- `restricted` — максимальные ограничения (runAsNonRoot, drop ALL, readOnlyRootFilesystem)

---

## Масштабирование и производительность

### 23. Как работает HPA (Horizontal Pod Autoscaler)?

**HPA** автоматически масштабирует количество реплик деплоймента на основе метрик.

**Архитектура:**
```
Metrics Server (или Prometheus Adapter)
       ↓ собирает метрики каждые 15s
HPA Controller (в controller-manager)
       ↓ каждые 15s сравнивает желаемое и текущее
Deployment
       ↓ меняет replicas
```

**Базовый HPA (CPU/Memory):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web

  minReplicas: 2
  maxReplicas: 20

  metrics:
  # CPU: держать среднее потребление ~50% от requests
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50

  # Memory: держать среднее ~400Mi
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 400Mi

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30   # ждать 30с перед scale up
      policies:
      - type: Pods
        value: 4                       # максимум 4 пода за раз
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # ждать 5 минут перед scale down (cooldown)
      policies:
      - type: Percent
        value: 10                      # максимум 10% реплик за раз
        periodSeconds: 60
```

**Custom метрики (через Prometheus Adapter):**
```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second  # метрика из Prometheus
    target:
      type: AverageValue
      averageValue: 100               # 100 RPS на под

- type: External
  external:
    metric:
      name: sqs_queue_depth           # глубина SQS очереди
    target:
      type: AverageValue
      averageValue: 50
```

**Формула масштабирования:**
```
desiredReplicas = ceil(currentReplicas * (currentMetric / desiredMetric))
# Пример: 3 пода, CPU 90%, target 50%
# desiredReplicas = ceil(3 * (90/50)) = ceil(5.4) = 6
```

```bash
# Посмотреть статус HPA
kubectl get hpa
kubectl describe hpa web-hpa
```

---

### 24. Что такое VPA и Cluster Autoscaler?

**VPA (Vertical Pod Autoscaler)**
Автоматически подбирает **requests и limits** для контейнеров на основе исторического потребления.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Auto"    # Auto (перезапускает поды), Initial, Off (только рекомендации)
  resourcePolicy:
    containerPolicies:
    - containerName: web
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 4Gi
```

> **Важно:** VPA и HPA **нельзя использовать вместе** по одной метрике (CPU/memory) — они будут конфликтовать. Если нужны оба: HPA по custom метрикам (RPS), VPA для правильного sizing.

**Cluster Autoscaler (CA)**
Автоматически добавляет/удаляет **узлы** в зависимости от нагрузки.

```
Под в состоянии Pending (не может запланироваться — нет ресурсов)
       ↓
Cluster Autoscaler обнаруживает Pending поды
       ↓
Запрашивает у облачного провайдера новый узел (через Node Group / Auto Scaling Group)
       ↓
Новый узел регистрируется в кластере
       ↓
Scheduler размещает Pending поды

Аналогично при уменьшении нагрузки:
Узел недоиспользован долгое время (пороги utilization)
       ↓
CA пытается переселить поды на другие узлы (drain)
       ↓
Узел удаляется
```

```yaml
# Аннотация чтобы CA не удалял узел с важным подом
cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
```

**Karpenter** (более современная альтернатива CA, особенно для AWS):
- Быстрее реагирует (секунды против минут)
- Сам выбирает тип инстанса (spot/on-demand, оптимальный размер)
- Более гибкая конфигурация через NodePool и EC2NodeClass

---

## Инструменты

### 25. Что такое Helm и зачем он нужен?

**Helm** — пакетный менеджер для Kubernetes. Позволяет упаковывать, версионировать и деплоить Kubernetes-приложения как единый пакет (chart).

**Проблема без Helm:**
Приложение из 10 YAML-файлов нужно деплоить в 3 окружения (dev/staging/prod) с разными параметрами. Без Helm — копируешь файлы, меняешь значения вручную, ошибаешься.

**Структура Helm chart:**
```
mychart/
├── Chart.yaml          # метаданные (name, version, description, dependencies)
├── values.yaml         # дефолтные значения переменных
├── values-prod.yaml    # override для production
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl    # переиспользуемые template функции
│   └── NOTES.txt       # текст после установки
└── charts/             # зависимые chart'ы
```

**Пример шаблона:**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- if .Values.ingress.enabled }}
        # conditional block
        {{- end }}
```

**values.yaml:**
```yaml
replicaCount: 2

image:
  repository: myapp
  tag: ""   # переопределяется при деплое

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: example.com

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

**Основные команды:**
```bash
# Поиск chart'ов в репозиториях
helm search hub nginx
helm search repo stable/

# Добавить репозиторий
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Установить
helm install my-nginx bitnami/nginx \
  --namespace production \
  --create-namespace \
  --values values-prod.yaml \
  --set image.tag=1.26 \
  --set replicaCount=5

# Просмотреть что будет задеплоено (dry-run)
helm install my-app ./mychart --dry-run --debug

# Список установленных релизов
helm list -A

# Обновить
helm upgrade my-nginx bitnami/nginx \
  --values values-prod.yaml \
  --atomic           # откатить если upgrade не прошёл успешно
  --timeout 5m

# История релизов
helm history my-nginx

# Откатить к предыдущей ревизии
helm rollback my-nginx 2

# Удалить
helm uninstall my-nginx --namespace production

# Рендерить шаблоны локально (без деплоя)
helm template my-app ./mychart --values values-prod.yaml
```

**Helm Hooks — выполнять задачи в нужный момент:**
```yaml
# Job, который запускается ПЕРЕД установкой (например, миграция БД)
apiVersion: batch/v1
kind: Job
metadata:
  name: migrations
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"           # порядок выполнения
    "helm.sh/hook-delete-policy": hook-succeeded
```

**OCI Registry для хранения chart'ов (Helm 3.8+):**
```bash
# Push chart в OCI registry (ECR, Harbor, GHCR)
helm push mychart-1.0.0.tgz oci://registry.example.com/charts

# Install из OCI
helm install myapp oci://registry.example.com/charts/mychart --version 1.0.0
```
