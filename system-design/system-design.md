# System Design для DevOps: Вопросы и ответы (Middle/Senior)

Практические сценарии проектирования инфраструктуры и платформ — формат, который часто встречается на собеседованиях Senior DevOps / Platform / SRE.

## Содержание

1. [Как подходить к System Design на DevOps-собеседовании?](#1-как-подходить-к-system-design-на-devops-собеседовании)
2. [Спроектируй высокодоступный multi-region стек для веб-приложения](#2-спроектируй-высокодоступный-multi-region-стек-для-веб-приложения)
3. [Спроектируй CI/CD платформу с нуля для 50+ сервисов](#3-спроектируй-cicd-платформу-с-нуля-для-50-сервисов)
4. [Спроектируй observability-стек (metrics, logs, traces)](#4-спроектируй-observability-стек-metrics-logs-traces)
5. [Спроектируй Kubernetes-платформу для продуктовых команд](#5-спроектируй-kubernetes-платформу-для-продуктовых-команд)
6. [Спроектируй Disaster Recovery: RPO/RTO, backup, failover](#6-спроектируй-disaster-recovery-rporto-backup-failover)
7. [Спроектируй event-driven архитектуру (очереди, стриминг)](#7-спроектируй-event-driven-архитектуру-очереди-стриминг)
8. [Спроектируй систему управления секретами и доступом](#8-спроектируй-систему-управления-секретами-и-доступом)
9. [Спроектируй multi-tenant SaaS инфраструктуру](#9-спроектируй-multi-tenant-saas-инфраструктуру)
10. [Спроектируй edge / API Gateway слой](#10-спроектируй-edge--api-gateway-слой)
11. [Как считать capacity и выбирать стратегию масштабирования?](#11-как-считать-capacity-и-выбирать-стратегию-масштабирования)
12. [Типичные trade-offs: cost vs reliability vs complexity](#12-типичные-trade-offs-cost-vs-reliability-vs-complexity)

---

## 1. Как подходить к System Design на DevOps-собеседовании?

DevOps System Design — это не «нарисуй микросервисы», а **проектирование платформы доставки и эксплуатации**: где живёт код, как деплоится, как наблюдается, как восстанавливается, сколько стоит.

**Фреймворк ответа (5–7 минут структурированно):**

```
1. Clarify requirements (2–3 минуты)
   - Функциональные: что система должна уметь?
   - Нефункциональные: RPS, latency, availability (99.9%?), RPO/RTO
   - Constraints: cloud? multi-region? compliance? бюджет?
   - Users: разработчики, on-call, security, finance?

2. High-level design
   - Блоки: Edge → App → Data → Observability → CI/CD → Identity
   - Потоки: request path, deploy path, failure path

3. Deep dive в 1–2 критичных области
   - Например: HA базы + failover, или GitOps + canary

4. Failure modes и operational concerns
   - Что ломается? Как детектим? Как откатываем?
   - Runbooks, on-call, blast radius

5. Cost, security, evolution
   - Что дорого? Что упростить на старте? Что добавить позже?
```

**Вопросы, которые стоит задать интервьюеру:**

```
Scale:
  Сколько RPS / пользователей / сервисов / инженеров?
  Рост на горизонте 12 месяцев?

Reliability:
  Целевой availability? Допустим ли multi-AZ или нужен multi-region?
  RPO / RTO для критичных данных?

Constraints:
  Только AWS / multi-cloud / on-prem?
  Регуляторика (PCI, HIPAA, GDPR)?
  Бюджетный потолок?

Team:
  Кто деплоит — platform team или каждая продуктовая команда?
  Есть ли уже K8s / Terraform / GitOps?
```

**Что отличает сильный ответ:**

| Слабый ответ | Сильный ответ |
|---|---|
| Перечисляет технологии | Обосновывает выбор trade-off'ами |
| Рисует только happy path | Явно описывает failure / rollback |
| «Поставим Kubernetes» | Объясняет control plane, tenancy, networking, cost |
| Игнорирует операции | Говорит про SLOs, алерты, runbooks, on-call |
| Overengineering с первого дня | Phased approach: MVP → scale → harden |

**Шаблон phased design (почти всегда уместен):**

```
Phase 1 (MVP, weeks):
  Single region, managed services, simple CI, basic metrics+logs
  Goal: ship safely, learn traffic patterns

Phase 2 (Scale):
  Autoscaling, canary, better observability, IaC modules, cost tags

Phase 3 (Harden):
  Multi-region / DR drills, policy-as-code, platform self-service,
  chaos experiments, FinOps governance
```

---

## 2. Спроектируй высокодоступный multi-region стек для веб-приложения

**Задача:** публичный SaaS API + web UI. Цель: **99.95%+**, latency < 200ms p99 для основных рынков, переживает отказ целого региона.

**Clarify:**

```
Traffic: ~5k RPS peak, read-heavy (80/20)
Data: PostgreSQL как SoR, Redis cache, S3 для ассетов
Users: EU + US; compliance: GDPR (EU data residency для EU tenants)
```

**High-level architecture:**

```
                    ┌─────────────┐
                    │   Users     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ CDN / DNS   │  (CloudFront + Route53 / Cloudflare)
                    │ geo / latency routing
                    └──────┬──────┘
              ┌────────────┴────────────┐
              ▼                         ▼
     ┌─────────────────┐       ┌─────────────────┐
     │ Region A (active)│       │ Region B (active │
     │ ALB / Ingress    │       │ or warm standby) │
     │ App (K8s)        │       │ App (K8s)        │
     │ Redis            │       │ Redis            │
     │ Read replicas    │       │ Read / promote   │
     └────────┬─────────┘       └────────┬─────────┘
              │                          │
              └──────────┬───────────────┘
                         ▼
              Primary DB (region A)
              Async replica (region B)
              Object storage replication
```

**Ключевые решения:**

```
1. Active-active vs Active-passive
   Active-passive (warm standby):
     + проще consistency
     + дешевле
     − RTO минуты (DNS + promote)
   Active-active (stateless app + regional reads):
     + лучше latency и capacity
     − сложнее writes / session / data residency

   Практичный старт: active-active для stateless app,
   primary DB в одном регионе + cross-region replica.

2. DNS failover
   Health checks на /healthz региональных endpoints
   Failover TTL: компромисс 30–60s (низкий TTL = больше DNS cost/load)
   Или Anycast/CDN edge, который уже знает origin health

3. Data plane
   PostgreSQL: primary + sync standby in-AZ/region, async cross-region
   Redis: regional caches (не shared global state для сессий —
     sticky sessions или JWT / server-side session in DB)
   S3: CRR (cross-region replication) для критичных bucket'ов

4. Stateless app contract
   Нет локального disk state
   Конфиг/секреты из внешних источников
   Идемпотентные retries на клиенте и gateway
```

**Health, deploy, blast radius:**

```
Deploy:
  Region-by-region canary (сначала A, потом B)
  Feature flags для быстрых kill-switches

Blast radius:
  Separate node pools / ASGs
  PodDisruptionBudgets
  Rate limits на edge
  Circuit breakers к downstream

Observability:
  Synthetic checks из обеих регионов
  Alert: regional error budget burn
  Dashboard: cross-region lag (replication delay)
```

**Типичные подводные камни (ожидают на собесе):**

```
✗ «Просто поставим multi-master Postgres» — почти всегда боль
✗ Session affinity на одном регионе без plan B
✗ Забыть про secrets/KMS replication и IAM в втором регионе
✗ DR есть на бумаге, но никогда не тестировали promote
✗ Data residency: нельзя слепо реплицировать EU PII в US
```

---

## 3. Спроектируй CI/CD платформу с нуля для 50+ сервисов

**Задача:** 50 микросервисов, 10 команд, нужно единообразие, скорость, безопасность и аудит.

**Requirements:**

```
Goals:
  - Commit → prod-ready artifact < 15–20 мин для типичного сервиса
  - Deploy to prod с approval + progressive delivery
  - Secrets не в репозиториях
  - Единые quality gates: lint, unit, SAST, image scan, IaC scan

Constraints:
  GitHub или GitLab как SoT для кода
  Kubernetes как runtime
  Platform team владеет templates, product teams — app code
```

**Architecture:**

```
Developer
   │ push / PR
   ▼
Source Control ──► CI (GitHub Actions / GitLab CI)
                      │
                      ├─ build + test
                      ├─ SAST / dependency scan
                      ├─ docker build (BuildKit cache)
                      ├─ image sign (Cosign) + SBOM
                      └─ push to registry (immutable tags)
                           │
                           ▼
                    GitOps repo (desired state)
                           │
                           ▼
                    ArgoCD / Flux ──► clusters (dev/stage/prod)
                           │
                           └─ canary / blue-green (Argo Rollouts)
```

**Рекомендуемая модель репозиториев:**

```
app repos (product teams):
  source + Dockerfile + chart/values fragment
  CI builds image, updates image tag via PR to gitops repo
  OR CI writes to OCI and ApplicationSet discovers it

gitops repo (platform-owned structure, team-owned paths):
  apps/<team>/<service>/<env>/
  platform/ (ingress, monitoring, policies)

ci-templates repo:
  reusable workflows / pipeline libraries
  versioned; teams pin major version
```

**Pipeline stages (golden path):**

```
1. Validate PR
   - unit tests, lint, contract tests
   - terraform plan / policy check (if infra PR)
   - required reviewers + CODEOWNERS

2. Merge to main
   - build immutable artifact: registry/app:sha-abc123
   - scan image (block on critical CVEs with exception process)
   - sign + attach SBOM
   - publish deploy request (PR or direct commit to env/dev)

3. Promote
   dev → staging (auto)
   staging → prod (manual approval / change window / SLO gate)

4. Verify
   - smoke tests, synthetic checks
   - automatic rollback on error-rate / latency burn
```

**Scaling CI runners:**

```
- Ephemeral runners in K8s (Actions Runner Controller / GitLab K8s executor)
- Separate queues: lightweight PR checks vs heavy image builds
- Shared BuildKit / layer cache (registry cache or S3)
- Cost control: autoscale to zero, spot for non-prod builds
```

**Security baseline в платформе (не опционально):**

```
✓ OIDC federated auth к cloud (no long-lived cloud keys in CI)
✓ Least-privilege deploy identity per env
✓ Signed images + admission policy (только подписанные в prod)
✓ Environment protection rules / approval gates
✓ Audit: who deployed what, when, which artifact digest
```

**Метрики успеха платформы:**

```
DORA: lead time, deploy frequency, change fail rate, MTTR
Platform: median pipeline duration, flake rate, template adoption %
Security: % releases with SBOM + signature, mean time to patch critical CVE
```

---

## 4. Спроектируй observability-стек (metrics, logs, traces)

**Задача:** единая observability для 50 сервисов в Kubernetes, on-call должен за 5 минут понять: что сломалось, где, почему.

**Pillars:**

```
Metrics  → «есть ли проблема и насколько большая?»
Logs     → «что именно произошло у инстанса X?»
Traces   → «где во request path потеряли время / ошибку?»
+ Continuous profiling (nice-to-have на Senior уровне)
```

**Reference architecture:**

```
Apps / sidecars / node agents
        │
        ├── metrics ──► Prometheus (remote write) ──► Mimir / Cortex / AMP
        ├── logs    ──► Fluent Bit / Alloy        ──► Loki / OpenSearch
        └── traces  ──► OTel Collector            ──► Tempo / Jaeger / X-Ray
                              │
                              ▼
                         Grafana (single pane)
                              │
                              ▼
                      Alertmanager → PagerDuty/Slack
```

**Design choices:**

```
1. Pull vs push metrics
   In-cluster Prometheus scrape + remote write в long-term store
   Keep local Prometheus for debugging, central for retention/HA

2. Cardinality control
   Forbid unbounded labels (user_id, email, request_id in metrics)
   Recording rules for expensive dashboards
   Exemplars: link metrics → traces без взрыва cardinality

3. Logs
   Structured JSON logs, mandatory fields:
     service, env, version, trace_id, level
   Retention tiers: 7–14d hot, longer cheap store if needed
   PII scrubbing at agent/collector

4. Traces
   Head/tail based sampling:
     keep errors + slow requests 100%, sample happy path
   OTel Collector as gateway (auth, tail sampling, routing)

5. Correlation
   Same trace_id in logs
   Grafana: metrics → trace → logs drill-down
```

**Alerting philosophy (чтобы не убить on-call):**

```
Alert on symptoms (user-facing), not raw causes:
  ✓ High error rate / latency burn (multi-window error budget)
  ✓ Saturation that predicts user impact (thread pool, disk full)
  ✗ CPU > 80% на одном поде без user impact

Severity:
  page   — user-visible, urgent
  ticket — degraded capacity, business hours
  log    — informational, no human interrupt

Every page alert needs:
  - owner
  - runbook link
  - dashboard link
  - clear recovery condition
```

**Kubernetes-specific must-haves:**

```
kube-state-metrics + node exporters / cAdvisor
Control plane metrics (apiserver latency, etcd)
Golden signals per service via RED:
  Rate, Errors, Duration
USE for infra:
  Utilization, Saturation, Errors
```

**Cost & retention trade-off:**

```
Metrics high-res short retention + downsampled long retention
Logs are usually the cost bomb → sampling, drop debug in prod,
  and route noisy namespaces to cheaper store
Start simple: Prometheus + Loki + Tempo + Grafana
Evolve to HA multi-tenant backends when scale demands
```

---

## 5. Спроектируй Kubernetes-платформу для продуктовых команд

**Задача:** 10–20 команд, shared clusters, self-service деплой, изоляция, security baseline «из коробки».

**Tenancy model:**

```
Soft multi-tenancy (типичный старт):
  Shared cluster, namespace per team/service
  NetworkPolicy + ResourceQuota + LimitRange
  Separate node pools for noisy/sensitive workloads

Hard tenancy (regulated / hostile tenants):
  Cluster per tenant/env or dedicated control plane
  Higher cost, stronger isolation
```

**Platform building blocks:**

```
 interAccess:
  Developers → GitOps / portal (не прямой kubectl admin в prod)

Cluster addons (platform-managed):
  ingress / Gateway API
  cert-manager
  external-dns
  metrics-server, Prometheus stack
  policy engine (OPA/Gatekeeper or Kyverno)
  secrets (ESO + Vault)
  autoscaling (HPA/VPA/Karpenter or Cluster Autoscaler)
  backup (Velero)

App enablement:
  Helm/Kustomize golden templates
  PodSecurity Standards (restricted)
  default NetworkPolicy deny-all + allow DNS
```

**Namespace blueprint (что создаётся для новой команды):**

```
- namespace + labels (team, cost-center, env)
- ResourceQuota / LimitRange
- ServiceAccount + IRSA/Workload Identity
- Role/RoleBinding (namespace admin, not cluster-admin)
- NetworkPolicy baseline
- Resource claims: DB/bucket via Crossplane or cloud tickets
- ArgoCD AppProject + Application
- dashboards/alerts folder owned by team
```

**Traffic & exposure:**

```
Internal services: ClusterIP + service mesh or NetworkPolicy
External: Ingress/Gateway → only approved domains
Admin UIs: SSO + IP allowlist / private only
No NodePort in prod unless justified
```

**Upgrade & operations strategy:**

```
- Version skew policy (n-1)
- Blue/green node pools or surge upgrades
- Addons upgraded separately from cluster
- Conformance + smoke tests after upgrade
- Maintenance windows communicated via platform status page
```

**What interviewers listen for:**

```
✓ Isolation boundaries and blast radius
✓ Self-service without handing out cluster-admin
✓ Policy defaults > wiki documents
✓ Cost: bin-packing, requests/limits realism, spot node pools
✓ Escape hatches: how teams get help / exceptions
```

---

## 6. Спроектируй Disaster Recovery: RPO/RTO, backup, failover

**Задача:** критичный платёжный/заказной контур. Бизнес говорит: «потерять больше 5 минут данных нельзя, подняться надо за 30 минут».

**Translate business → engineering:**

```
RPO (Recovery Point Objective) = 5 min  → как часто и насколько синхронны бэкапы/реплики
RTO (Recovery Time Objective) = 30 min → насколько прогрет standby и автоматизирован failover

Не всё одинаково критично:
  Tier 0 (payments): RPO 0–5m, RTO 15–30m
  Tier 1 (core API): RPO 15m, RTO 1h
  Tier 2 (analytics): RPO 24h, RTO 24–48h
```

**DR patterns:**

```
1. Backup & restore
   Cheap, high RTO. Good for Tier 2.
   Periodic snapshots + tested restore.

2. Pilot light
   Minimal core in DR region (data replicated, compute tiny).
   Scale out on disaster.

3. Warm standby
   Full scaled-down environment always on.
   DNS/traffic shift + DB promote.

4. Hot multi-region
   Lowest RTO, highest cost/complexity.
```

**Concrete design for Tier 0:**

```
Data:
  DB: sync replication in-region, async cross-region replica
  Continuous WAL / incremental backups every few minutes
  Object storage CRR
  Redis: ephemeral cache (rebuild) OR AOF + restore strategy

Compute:
  Warm K8s cluster or pre-baked node capacity quotas in DR region
  GitOps already pointed / can quickly add cluster
  Container images in multi-region registry replication

Traffic:
  Health-checked DNS failover or traffic manager
  Runbooks + semi-automated promote pipeline

Identity/secrets:
  Vault/KMS keys available in DR
  Certificates issuable in DR (don't depend on single region CA unreachable)
```

**Backup principles that matter more than tools:**

```
✓ Backups encrypted, access-controlled, immutable where possible
✓ Restore tested on schedule (game day), not "we have backups"
✓ Document restore time measured, not assumed
✓ App-consistent backups for DBs (not only disk snapshots)
✓ Dependency map: app restore order (IAM → secrets → DB → app → DNS)
```

**DR runbook outline:**

```
1. Detect & declare disaster (severity, commander)
2. Freeze deploys / decide split-brain risk
3. Promote data plane (DB) — single writer guarantee
4. Scale compute in DR
5. Redirect traffic
6. Verify golden flows (login, checkout, payment)
7. Communicate status
8. Post-incident: reverse sync plan / failback criteria
```

**Common failures in DR designs:**

```
✗ Backups in the same blast radius as primary
✗ Never tested DNS + cert + secret dependencies
✗ RPO claimed 5 min, but replica lag regularly 20+ min (no alert)
✗ People who know the runbook are on PTO
```

---

## 7. Спроектируй event-driven архитектуру (очереди, стриминг)

**Задача:** order service должен уведомлять inventory, billing, analytics без жёсткой связанности. Пики x10 на распродажах.

**When queue vs stream vs bus:**

```
SQS / RabbitMQ (task queue):
  Command/work distribution, retries, DLQ
  Good for: send email, generate PDF, async jobs

Kafka / Pulsar (event log):
  Durable event history, many independent consumers,
  replay, stream processing
  Good for: domain events, CDC, analytics pipeline

SNS + SQS / EventBridge (bus/fan-out):
  Pub/sub notifications, lightweight integration
```

**Reference design:**

```
Order API ──► write DB (transaction)
           └─► outbox table ──► publisher ──► Kafka topic orders.events
                                                │
                         ┌──────────────────────┼──────────────────────┐
                         ▼                      ▼                      ▼
                   Inventory svc           Billing svc            Analytics
                   (consumer group)        (consumer group)       (warehouse sink)
```

**Reliability patterns:**

```
Transactional Outbox:
  Avoid "DB committed but event lost" / dual-write problem

Idempotent consumers:
  Dedupe by event_id / order_id+version
  Especially important with at-least-once delivery

DLQ / retry policy:
  Transient errors → exponential backoff
  Poison messages → DLQ + alert
  Never infinite silent retry loops

Ordering:
  Per-aggregate ordering via partition key = order_id
  Do not assume global ordering
```

**Scaling for flash sales:**

```
- Partition count planned for peak throughput
- Consumer autoscaling on lag metrics
- Backpressure: load shed non-critical consumers first
- Separate topics/priority for checkout-critical path
- Load test with realistic key skew (hot products)
```

**Ops concerns interviewers like:**

```
Lag alerts (consumer group lag)
Disk / retention sizing
Schema evolution (Avro/Protobuf + registry)
Poison message playbooks
Multi-AZ broker deployment
Who owns topics and ACLs (platform vs domain teams)
```

**Kafka vs RabbitMQ one-liner for interviews:**

```
Need replay, many consumers, high throughput event log → Kafka
Need flexible routing, classic work queues, simpler ops → RabbitMQ/SQS
Often both exist in one company for different jobs
```

---

## 8. Спроектируй систему управления секретами и доступом

**Задача:** убрать секреты из Git и ноутбуков; дать разработчикам безопасный self-service; пройти security review.

**Layers of identity:**

```
Humans:
  SSO (OIDC/SAML) → Okta/Google Workspace → AWS IAM Identity Center / Vault OIDC
  Short-lived sessions, MFA mandatory

Machines / workloads:
  K8s ServiceAccount → IRSA / Workload Identity
  CI → OIDC to cloud roles (no static keys)
  Services → Vault AppRole / K8s auth / cloud IAM
```

**Secrets architecture:**

```
Source of truth: Vault / cloud secret manager
Delivery to workloads:
  - External Secrets Operator → sync to K8s Secret (encrypted at rest)
  - OR Vault Agent / CSI driver inject at runtime
  - Prefer dynamic DB creds where possible (short TTL)

Rotation:
  Automated for dynamic secrets
  Scheduled for static (API tokens) + consumer reload strategy
  Emergency revoke runbook
```

**Access model:**

```
Human break-glass to prod secrets: rare, audited, time-bounded
Product teams: read secrets of own namespace/env only
Platform/SRE: manage engines/policies, not day-to-day app secret values
CI: can write new versions for its service, cannot read unrelated teams
```

**Policy examples worth mentioning:**

```
- Deny plaintext secrets in Git (pre-commit + CI secret scanning)
- Deny K8s Secret create without approved annotations / ESO ownership
- Prod role assumption requires MFA + reason (session tags)
- All access logged to SIEM; alert on unusual secret reads
```

**Minimal viable secure setup (phased):**

```
Phase 1: SOPS/Sealed Secrets for GitOps + cloud KMS
Phase 2: Vault/ASM + ESO + OIDC for CI
Phase 3: dynamic credentials, just-in-time access, continuous rotation
```

**Trade-offs:**

```
Sealed Secrets: simple GitOps-native, weaker operational model for rotation/ACL
Vault: powerful, more moving parts, needs its own HA/DR story
Cloud secret manager: low ops, good cloud integration, weaker multi-cloud story
```

---

## 9. Спроектируй multi-tenant SaaS инфраструктуру

**Задача:** B2B SaaS, 200 клиентов. Часть на shared, enterprise хотят isolation.

**Tenancy options:**

```
1. Pool (shared everything)
   + cheapest, simplest ops
   − noisy neighbor, weaker isolation, harder per-tenant compliance

2. Bridge (shared app, separate data)
   DB schema/database per tenant, shared compute
   Good balance for many B2B products

3. Cell / silo (dedicated stack per tenant or tier)
   Dedicated namespace/cluster/DB for enterprise
   + isolation, custom limits
   − cost, operational overhead
```

**Practical hybrid:**

```
Standard tenants → pooled cells (sharded by tenant_id ranges)
Enterprise → dedicated cell (namespace + DB + optional dedicated nodes)

Cell benefits:
  Blast radius limited to cell
  Rollouts can be cell-by-cell
  Easier to relocate noisy tenants
```

**Isolation checklist:**

```
Authn/z: tenant context on every request, server-side enforced
Data: row-level guards + separate encryption keys for enterprise
Network: NetworkPolicies between teams/cells
Compute: quotas; dedicated node pool for premium tier
Observability: tenant_id label carefully (cardinality!) — often high-cardinality
  belongs in logs/traces, not metrics
```

**Noisy neighbor controls:**

```
Rate limits per tenant at gateway
Queue quotas / fair scheduling
Storage IOPS limits
Autoscale + load shedding of non-critical features
```

**Data residency & compliance:**

```
Tenant → home region mapping
Control plane global, data plane regional
Backups stay in allowed regions
Support access: break-glass with customer approval for enterprise
```

**Migration story (часто спрашивают):**

```
How to move tenant from pool → silo:
  1. provision dedicated cell
  2. replicate data
  3. maintenance window / dual-write briefly
  4. switch routing (tenant → cell map)
  5. decommission old data after verification
```

---

## 10. Спроектируй edge / API Gateway слой

**Задача:** единая точка входа для web/mobile/public API: TLS, auth, rate limit, routing, WAF.

**Responsibilities of the edge:**

```
✓ TLS termination / cert automation
✓ Authn at edge for external clients (JWT validation, mTLS for partners)
✓ Rate limiting & bot protection
✓ Request routing / canary by header or %
✓ WAF / basic DDoS posture
✓ Edge caching for public GETs
✗ Heavy business logic (keep gateway thin)
```

**Architecture options:**

```
CDN (CloudFront/Cloudflare) → Load Balancer → Ingress/Gateway API → Services
Partner APIs: separate hostname + stricter mTLS / IP allowlist
Admin API: private or SSO-only, not on public edge
```

**Rate limiting design:**

```
Dimensions:
  per IP (abuse)
  per API key / tenant (fair use)
  per route (protect expensive endpoints)

Storage:
  Redis / gateway built-in cluster store
  Local counters only if approximate is OK

Response:
  429 + Retry-After
  Observability: throttle metrics by tenant tier
```

**Canary at edge vs service mesh:**

```
Edge canary: easy for external traffic splitting, header-based QA
Mesh canary: richer internal L7 policy between services
Often: edge for ingress progressive delivery, mesh optional inside
```

**Security baseline:**

```
TLS 1.2+ (prefer 1.3), HSTS
WAF rules for OWASP top risks
Hide internal error details
Size limits / timeouts to protect backends
Central audit log of auth failures and policy denies
```

**Failure behavior:**

```
Gateway dependency on auth provider:
  Cache JWKS, degrade gracefully, avoid hard outage on IdP blip
  Decide fail-open vs fail-closed by route sensitivity
  Payments: fail-closed; marketing static: fail-open possible
```

---

## 11. Как считать capacity и выбирать стратегию масштабирования?

**Задача:** сервис растёт. Нужно понять, сколько железа/подов надо и как масштабироваться без сюрпризов в Black Friday.

**Capacity math (упрощённо, но убедительно на собесе):**

```
1. Measure current:
   peak RPS = 2000
   p99 latency OK at CPU ~60% per pod
   one pod handles ~100 RPS comfortably (load test!)

2. Baseline pods:
   2000 / 100 = 20 pods
   + headroom 30–50% → 26–30 pods
   + surge for deploys / AZ loss (lose 1 of 3 AZs) → plan for 2/3 capacity still enough
     => size so 2 AZs can carry peak: ~30 / 0.66 ≈ 45 pods total

3. Downstream:
   DB connections: pods * pool_size < DB max_connections * safety factor
   Redis CPU/memory, queue lag, external API quotas
```

**Vertical vs horizontal:**

```
Horizontal (more pods/instances):
  Preferred for stateless services
  Needs good load balancing and no sticky bottlenecks

Vertical (bigger instances):
  Useful for monoliths, DBs, JVM heaps
  Ceiling exists; failover blast radius bigger

Autoscale signals:
  CPU/RAM — blunt
  RPS / concurrency / queue lag / custom SLI — better
  Scale on saturation before latency SLO burns
```

**Pre-scaling vs reactive:**

```
Known events (marketing campaigns):
  Load test → pre-scale min replicas + DB
  Verify caches warm / CDN hit ratio

Unknown spikes:
  Fast HPA + warm pool / overprovisioning slightly
  Load shedding + graceful degradation (disable recommendations etc.)
```

**Data-plane capacity (то, что забывают):**

```
Postgres: CPU, IOPS, connection count, replication lag
Kafka: partition throughput, disk, consumer lag
NAT Gateway / egress bandwidth
TLS/ingress CPU at edge
IP addresses / subnet size in VPC
Cloud quotas (vCPU) requested before the event
```

**Deliverable interviewers like:**

```
A simple capacity sheet:
  demand assumptions → per-component limits → headroom → autoscale policy → test plan
And a clear statement of the bottleneck you expect first.
```

---

## 12. Типичные trade-offs: cost vs reliability vs complexity

Почти каждый Senior-уровень вопрос сводится к выбору на пересечении трёх осей.

**Каркас ответа:**

```
1. Назови варианты (минимум 2)
2. Сравни по: reliability, cost, complexity, team skills, time-to-value
3. Выбери default для текущего этапа
4. Скажи, при каких триггерах пересмотришь решение
```

**Частые trade-off пары:**

```
Multi-AZ vs Multi-region
  Multi-AZ: must for prod; cheap relative to multi-region
  Multi-region: only when regional failure is in threat model / latency needs

Kubernetes vs managed PaaS (ECS/Fargate/Cloud Run)
  K8s: flexibility + ecosystem, higher cognitive load
  PaaS: faster start, less control, potential cost/scale surprises

GitOps pull vs CI push deploy
  GitOps: drift control, audit, multi-cluster
  CI push: simpler early on, weaker desired-state story

One big cluster vs many clusters
  One: cheaper, denser, riskier blast radius
  Many: isolation, more toil unless strong platform automation

Kafka vs SQS
  Kafka: powerful, ops-heavy (even if managed)
  SQS: low ops, less flexible replay/stream analytics

Sync vs async integrations
  Sync: simpler UX, cascading failures
  Async: resilience + complexity (idempotency, eventual consistency)

Hot DR vs backups only
  Decide from RTO/RPO * revenue impact, not from resume-driven overengineering
```

**Cost levers without silently hurting reliability:**

```
✓ Rightsizing requests/limits and instance families
✓ Spot for fault-tolerant workloads with interruption handling
✓ Lifecycle policies on logs/artifacts
✓ Caching / CDN to cut origin and egress
✗ Removing Multi-AZ "to save money" on Tier 0 data stores
✗ Turning off backups that were never restored (irony: then you need them)
```

**Complexity budget:**

```
Каждый новый компонент стоит:
  on-call knowledge, upgrade tax, failure modes, security surface

Правило:
  Добавляй технологию, когда есть конкретная боль,
  которую проще/дешевле не решить текущим стеком.
```

**Пример «сильного» финального ответа на любой design prompt:**

```
"Для текущего масштаба я выберу warm standby в одном secondary region,
 multi-AZ primary, GitOps + canary, managed Postgres with async replica.
 Это закрывает RTO≈30m/RPO≈5m без стоимости full active-active.
 Триггеры к усложнению: regional outage impact > $X, или EU/US latency SLO,
 или enterprise контракты с более жёстким RTO — тогда active-active reads
 и автоматизированный cell failover."
```

---

## Как тренироваться к таким собесам

```
1. Бери реальный сервис с работы и перерисуй его "как надо" с trade-offs
2. Проговаривай вслух 30–40 минут с таймером
3. Всегда заканчивай: failure modes, observability, cost, rollout/rollback
4. Не прячь незнание: озвучь assumption и иди дальше
5. Держи в голове 2–3 reference architectures (web SaaS, data pipeline, platform)
```
