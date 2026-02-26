# SRE (Site Reliability Engineering): Вопросы и ответы для DevOps-инженера (Middle/Senior)

## Содержание

1. [Что такое SRE и чем он отличается от DevOps и классического Ops?](#1-что-такое-sre-и-чем-он-отличается-от-devops-и-классического-ops)
2. [SLI, SLO, SLA — определения, примеры, как правильно выбирать?](#2-sli-slo-sla--определения-примеры-как-правильно-выбирать)
3. [Error Budget: что это, как считать, как использовать?](#3-error-budget-что-это-как-считать-как-использовать)
4. [Toil: что это, как измерять и как с ним бороться?](#4-toil-что-это-как-измерять-и-как-с-ним-бороться)
5. [Incident Management: жизненный цикл инцидента, severity, on-call](#5-incident-management-жизненный-цикл-инцидента-severity-on-call)
6. [Postmortem (Post-incident Review): как проводить, blameless culture](#6-postmortem-post-incident-review-как-проводить-blameless-culture)
7. [Как настроить эффективное on-call дежурство?](#7-как-настроить-эффективное-on-call-дежурство)
8. [Capacity Planning: как прогнозировать нагрузку и планировать ресурсы?](#8-capacity-planning-как-прогнозировать-нагрузку-и-планировать-ресурсы)
9. [Reliability Patterns: Circuit Breaker, Retry, Bulkhead, Timeout](#9-reliability-patterns-circuit-breaker-retry-bulkhead-timeout)
10. [Runbooks и Playbooks: зачем нужны и как писать?](#10-runbooks-и-playbooks-зачем-нужны-и-как-писать)
11. [DORA Metrics: как измерить эффективность DevOps/SRE команды?](#11-dora-metrics-как-измерить-эффективность-devopssre-команды)
12. [Как проводить Game Days и почему это важно?](#12-как-проводить-game-days-и-почему-это-важно)

---

## 1. Что такое SRE и чем он отличается от DevOps и классического Ops?

**SRE (Site Reliability Engineering)** — дисциплина, придуманная в Google, где **software engineers** решают operations-задачи с помощью кода и инженерных подходов.

```
Классический Ops:
  Ручное управление серверами, Change Advisory Board, maintenance windows,
  "стабильность = никаких изменений", разрыв между Dev и Ops

DevOps (культура):
  Сломать силосы, shared responsibility, automation, CI/CD,
  "вы строите — вы запускаете"

SRE (реализация DevOps в Google):
  Конкретная реализация принципов DevOps с измеримыми метриками
  SLO/SLI/Error Budget как инструмент баланса между надёжностью и скоростью
```

**Ключевые принципы SRE:**

| Принцип | Описание |
|---------|----------|
| **Embrace Risk** | 100% надёжность невозможна и не нужна. Error budget определяет допустимый риск |
| **SLOs** | Объективные метрики надёжности, согласованные с бизнесом |
| **Toil Elimination** | Автоматизировать ручную операционную работу |
| **Monitoring** | Symptoms > Causes, Four Golden Signals |
| **Release Engineering** | Надёжные, воспроизводимые процессы релиза |
| **Simplicity** | Сложность — главный враг надёжности |

**Формула SRE от Google:**
> SRE = 50% ops work cap + software engineering остаток

Если ops-работа превышает 50% времени — нужно автоматизировать или вернуть задачи Dev-команде.

---

## 2. SLI, SLO, SLA — определения, примеры, как правильно выбирать?

**SLI (Service Level Indicator)** — конкретная измеримая метрика качества сервиса.

```
Хорошие SLI:
  - Доля успешных HTTP запросов: success_requests / total_requests
  - P99 latency для пользовательских запросов
  - Доля запросов к API обработанных за < 200ms
  - Доступность (availability): uptime / total_time

Плохие SLI:
  - CPU utilization (инфраструктурная метрика, не пользовательская)
  - Количество задеплоенных подов
  - Внутренний queue depth (если не влияет на пользователя напрямую)
```

**SLO (Service Level Objective)** — целевое значение SLI за период времени.

```
SLO примеры:
  "99.9% HTTP запросов возвращают 2xx или 3xx за 30 дней"
  "P99 latency < 500ms для 99% запросов за 28 дней"
  "99.95% запросов к checkout API обрабатываются за < 1 секунду"

Важно:
  SLO < 100% — оставляем место для ошибок (error budget)
  SLO устанавливается НИЖЕ того, что технически возможно
  Начинать с "happiness test": если SLO нарушен — пользователи недовольны?
```

**SLA (Service Level Agreement)** — юридический контракт с клиентом, обычно строже SLO.

```
Типичная структура:
  SLI: метрика доступности
  SLO: 99.95% (внутренняя цель)
  SLA: 99.9%  (внешнее обязательство, с штрафами)

  Буфер: SLO - SLA = запас для реагирования до нарушения SLA
```

**Как выбирать SLI/SLO:**

```
1. Начни с пользователя: что важно для пользовательского опыта?
   "Пользователь хочет загружать страницу быстро и видеть правильный результат"
   → SLI: latency + availability + correctness

2. Измеряй на уровне пользователя (не инфраструктуры):
   Плохо: "uptime серверов 99.9%"
   Хорошо: "доля успешных запросов к API 99.9%"

3. Используй окна измерения:
   Rolling window (28 дней) > Calendar month (неравномерно)
   
4. Устанавливай реалистичные цели:
   Проверь исторические данные: какова реальная доступность?
   Если сейчас 99.5% — ставь цель 99.5%, а не сразу 99.99%

5. Начни с малого: 2-3 SLO лучше 20 плохих
```

**Пример SLI/SLO в Prometheus/Grafana:**

```promql
# SLI: доля успешных запросов
sum(rate(http_requests_total{status!~"5..",job="api"}[28d]))
/ sum(rate(http_requests_total{job="api"}[28d]))

# Error Budget потреблено (если SLO = 99.9%)
1 - (
  sum(rate(http_requests_total{status!~"5.."}[28d]))
  / sum(rate(http_requests_total[28d]))
) / (1 - 0.999)
```

---

## 3. Error Budget: что это, как считать, как использовать?

**Error Budget** — количество допустимых ошибок/недоступности за период, вычисляемое из SLO.

```
Формула:
  Error Budget = 1 - SLO

Примеры:
  SLO 99.9%   → Error Budget = 0.1%  → 43.2 минуты/месяц
  SLO 99.95%  → Error Budget = 0.05% → 21.6 минуты/месяц
  SLO 99.99%  → Error Budget = 0.01% → 4.32 минуты/месяц
  SLO 99.999% → Error Budget = 0.001%→ 26 секунд/месяц
```

**Таблица времени простоя:**

| SLO | Допустимый простой/год | Допустимый простой/месяц |
|-----|----------------------|-------------------------|
| 99% | 3.65 дня | 7.3 часа |
| 99.9% | 8.77 часа | 43.2 минуты |
| 99.95% | 4.38 часа | 21.6 минут |
| 99.99% | 52.6 минуты | 4.32 минуты |
| 99.999% | 5.26 минуты | 26 секунд |

**Как использовать Error Budget:**

```
Error Budget > 0 (есть запас):
  ✅ Можно деплоить новые фичи
  ✅ Можно проводить эксперименты
  ✅ Можно замедлить инвестиции в надёжность

Error Budget → 0 (иссякает):
  ⚠️ Заморозить деплои новых фич
  ⚠️ Сфокусироваться на reliability работе
  ⚠️ Разобраться с причинами выбирания бюджета

Error Budget = 0 (исчерпан):
  🚫 Feature freeze
  🚫 Только bugfix и reliability work
  🚫 Совместный review с Product Manager
```

**Error Budget Policy — формализованные правила:**

```markdown
# Error Budget Policy для сервиса Checkout

## Зелёный (> 50% бюджета осталось)
- Деплои разрешены в любое время
- Feature work продолжается нормально

## Жёлтый (10-50% бюджета осталось)
- Все деплои требуют ревью от tech lead
- Минимум 1 reliability task в текущем спринте

## Красный (< 10% бюджета осталось)
- Feature freeze: только bugfix деплои
- Команда фокусируется на reliability
- Еженедельный review с Engineering Manager

## Критический (бюджет исчерпан)
- Деплои запрещены (кроме hotfix с PM approval)
- Postmortem для каждого инцидента
- Reliability roadmap на следующий квартал
```

**Error Budget Burn Rate алерты:**

```yaml
# Если расходуем бюджет слишком быстро — алертим
# Burn rate 1 = "нормальный" темп расходования за 30 дней
# Burn rate 14 = исчерпаем весь бюджет за 2 дня

- alert: ErrorBudgetBurnRateCritical
  expr: |
    # 1-часовое окно: burn rate > 14 (2 дня до исчерпания)
    (
      1 - sum(rate(http_requests_total{status!~"5.."}[1h]))
          / sum(rate(http_requests_total[1h]))
    ) / (1 - 0.999) > 14
  for: 5m
  labels:
    severity: critical

- alert: ErrorBudgetBurnRateWarning
  expr: |
    # 6-часовое окно: burn rate > 6 (5 дней до исчерпания)
    (
      1 - sum(rate(http_requests_total{status!~"5.."}[6h]))
          / sum(rate(http_requests_total[6h]))
    ) / (1 - 0.999) > 6
  for: 30m
  labels:
    severity: warning
```

---

## 4. Toil: что это, как измерять и как с ним бороться?

**Toil** — ручная, повторяющаяся, автоматизируемая операционная работа, не добавляющая долгосрочной ценности.

**Признаки Toil:**

```
✓ Ручная работа — требует участия человека
✓ Повторяющаяся — делается снова и снова
✓ Автоматизируемая — машина могла бы делать это
✓ Тактическая — реактивная, не проактивная
✓ Линейно масштабируется — больше нагрузки = больше toil
✓ Не добавляет ценности — сервис не становится лучше
```

**Примеры Toil vs Non-Toil:**

| Toil | Non-Toil |
|------|---------|
| Ручная перезагрузка подов при OOM | Настройка автоматического restart + HPA |
| Добавление правил в whitelist вручную | Self-service портал для команд |
| Ручное масштабирование БД перед событием | Автоматический Cluster Autoscaler |
| Разблокировка аккаунтов вручную | Автоматический unlock через timeout |
| Ручное обновление SSL сертификатов | cert-manager + Let's Encrypt |
| "Тыкать" в упавший cron job | Правильный retry + alerting |

**Как измерять Toil:**

```
1. Трекинг времени: команда записывает toil vs project work
   Цель SRE: < 50% времени на toil

2. Ticket ratio: toil tickets / total tickets за спринт
   Тренд важнее абсолютного числа

3. Interrupt rate: сколько раз тебя прерывают "срочными" задачами

4. On-call burden: количество pages/дежурство
   > 2 actionable pages/дежурство = слишком много
```

**Стратегии борьбы с Toil:**

```bash
# 1. Автоматизация с помощью скриптов/операторов
# Вместо ручного деплоя — GitOps + ArgoCD

# 2. Self-service платформы
# Вместо "создай мне namespace" — developer portal

# 3. Kubernetes Operators
# Вместо ручного управления Kafka — Strimzi Operator

# 4. Runbook automation
# Вместо "зайди и перезапусти" — автоматический remediation
# Пример: CloudWatch Alarm → Lambda → restart service

# 5. Capacity automation
# Вместо "добавь ноды перед Black Friday" — Cluster Autoscaler + Karpenter
```

---

## 5. Incident Management: жизненный цикл инцидента, severity, on-call

**Severity уровни (пример):**

| Severity | Описание | SLA реагирования | Пример |
|----------|----------|-----------------|--------|
| **SEV-1 / P1** | Полная недоступность production для всех пользователей | 15 минут | Сайт недоступен |
| **SEV-2 / P2** | Частичная недоступность или деградация для > 10% пользователей | 30 минут | Checkout не работает |
| **SEV-3 / P3** | Незначительная деградация, workaround есть | 4 часа | Медленные отчёты |
| **SEV-4 / P4** | Потенциальные проблемы, не влияющие пока на пользователей | 24 часа | Disk > 80% |

**Жизненный цикл инцидента:**

```
1. DETECTION (обнаружение)
   Alerting → PagerDuty → on-call engineer
   Цель: минимизировать время между началом инцидента и обнаружением (MTTD)

2. ACKNOWLEDGMENT (принятие)
   On-call engineer принимает Page
   Открывает инцидент-тикет / war room (Slack channel #incident-YYYY-MM-DD)

3. TRIAGE (сортировка)
   Определить severity
   Понять scope: какие пользователи/сервисы затронуты?

4. ESCALATION (эскалация)
   При SEV-1: уведомить EM, Product, Comms
   Назначить Incident Commander (IC) и Scribe

5. MITIGATION (митигация)
   Главная цель: восстановить сервис БЫСТРО
   Не надо понимать root cause чтобы митигировать
   Rollback > Fix forward (если неясно что сломалось)

6. RESOLUTION (решение)
   Сервис восстановлен, пользователи не затронуты
   Обновить статусную страницу

7. POSTMORTEM
   Через 24-72 часа: детальный разбор
   Root cause analysis, action items с дедлайнами
```

**Роли в инциденте:**

```
Incident Commander (IC):
  - Координирует всех участников
  - Принимает решения (rollback, escalation)
  - НЕ занимается техническим фиксом сам

Communications Lead (Comms):
  - Пишет обновления на статусную страницу
  - Уведомляет stakeholders

Technical Lead (Tech):
  - Диагностирует и исправляет
  - Докладывает IC о прогрессе

Scribe:
  - Записывает timeline событий в реальном времени
  - Фиксирует принятые решения
```

**Incident Slack channel шаблон:**

```
#incident-2024-01-15-checkout-down

🚨 INCIDENT OPENED: SEV-2
IC: @john
Tech: @jane @bob
Comms: @alice

Timeline:
10:23 - Alert fired: checkout error rate 45%
10:25 - IC assigned, war room opened
10:30 - Identified: DB connection pool exhausted
10:35 - Mitigation: increased pool size, error rate dropping
10:42 - RESOLVED: error rate back to normal

Status page: https://status.example.com/incidents/123
```

---

## 6. Postmortem (Post-incident Review): как проводить, blameless culture

**Blameless Postmortem** — разбор инцидента без обвинений конкретных людей. Люди не совершают ошибки намеренно — системы создают условия для ошибок.

```
НЕ бламелес:                  Бламелес:
"Vasya задеплоил              "Система позволила задеплоить
 сломанный код"                без прохождения интеграционных тестов"

"Petya не заметил алерт"      "Алерт был слишком зашумлён,
                               система создала условия для пропуска"

Фокус: КАК система позволила этому случиться?
       КАК улучшить систему чтобы этого не повторилось?
```

**Структура Postmortem документа:**

```markdown
# Postmortem: Checkout Service Outage — 2024-01-15

## Сводка
15 января 2024, 10:23-10:42 UTC сервис checkout был недоступен для 40% пользователей.
Причина: исчерпание connection pool к PostgreSQL после деплоя v2.4.1.
Затронуто ~12,000 пользователей, потеряно ~$45,000 выручки.

## Timeline
| Время UTC | Событие |
|-----------|---------|
| 10:15 | Деплой v2.4.1 в production |
| 10:23 | Алерт: checkout error rate > 5% |
| 10:25 | On-call engineer принял инцидент |
| 10:30 | Определена причина: DB connection pool |
| 10:35 | Применён workaround: увеличен pool_size |
| 10:42 | Сервис восстановлен |

## Root Cause
v2.4.1 добавил новый middleware, создающий дополнительное DB соединение на каждый запрос.
При текущем max_connections=50 при нагрузке > 50 RPS пул исчерпывался.

## Contributing Factors
1. Нет нагрузочных тестов, имитирующих production load
2. Конфигурация pool_size не была в monitoring
3. Деплой прошёл без canary — сразу 100% трафика

## Impact
- Duration: 19 минут
- Users affected: ~12,000 (40% от дневной аудитории)
- Revenue impact: ~$45,000
- SLO impact: потреблено 40% месячного error budget

## What Went Well
- Алерт сработал через 2 минуты после начала инцидента
- Rollback план был готов и занял < 5 минут
- Команда эффективно коммуницировала

## What Could Be Improved
- Не было нагрузочного тестирования перед деплоем
- Нет алерта на DB connection pool utilization
- Деплой не использовал canary strategy

## Action Items
| Действие | Владелец | Дедлайн |
|---------|---------|---------|
| Добавить нагрузочный тест в pipeline | @jane | 2024-01-22 |
| Алерт на db_connection_pool_utilization > 80% | @bob | 2024-01-19 |
| Включить canary deployment для checkout | @alice | 2024-01-31 |
| Добавить pool_size в Grafana dashboard | @bob | 2024-01-19 |

## Lessons Learned
Изменения конфигурации соединений с БД требуют явного нагрузочного тестирования.
Необходимо внедрить обязательный canary для всех изменений в критических сервисах.
```

**Метрики качества Postmortem:**

```
Хороший postmortem:
  ✓ Написан в течение 72 часов
  ✓ Имеет конкретные action items с владельцами и дедлайнами
  ✓ Action items закрываются (tracking через тикеты)
  ✓ Blameless — нет имён в контексте вины
  ✓ Рассматривает системные причины, а не человеческие ошибки

Плохой postmortem:
  ✗ "Vasya ошибся при деплое"
  ✗ Action item: "Быть внимательнее"
  ✗ Нет конкретных технических улучшений
  ✗ Написан через 2 недели, когда детали забыты
```

---

## 7. Как настроить эффективное on-call дежурство?

**Принципы здорового on-call:**

```
1. Pages должны быть actionable
   Каждый page = реальная проблема требующая действия
   Цель: < 2 actionable pages за смену

2. Равномерная ротация
   Никто не дежурит больше 1 недели подряд
   Timezone-aware ротация (no night shifts where possible)

3. Handoff ритуал
   Передача дежурства с brief summary open issues

4. Компенсация
   Финансовая (доп. оплата) или time off после интенсивных дежурств

5. Психологическая безопасность
   Можно разбудить secondary/escalation без стыда
   Нет наказания за ошибки в 3 ночи
```

**PagerDuty конфигурация:**

```yaml
# Эскалационная политика
escalation_policy:
  name: "API Team On-Call"
  num_loops: 2
  rules:
    - escalation_delay_in_minutes: 5
      targets:
        - type: schedule
          id: PRIMARY_SCHEDULE  # дежурный 1
    - escalation_delay_in_minutes: 10
      targets:
        - type: schedule
          id: SECONDARY_SCHEDULE  # бэкап если не отвечает
    - escalation_delay_in_minutes: 15
      targets:
        - type: user
          id: ENGINEERING_MANAGER  # EM если оба не отвечают
```

**Метрики on-call здоровья:**

```promql
# Pages per shift (должно быть < 2 actionable)
increase(pagerduty_alerts_total{severity="critical"}[7d])

# MTTA (Mean Time to Acknowledge)
avg(pagerduty_ack_time_seconds)

# MTTR (Mean Time to Resolve)
avg(pagerduty_resolve_time_seconds)
```

**On-call инструментарий (must have):**

```
Обязательно:
  - Runbook для каждого алерта (что делать)
  - Dashboard "On-call overview" в Grafana
  - Доступ к логам (Kibana/Loki) с мобильного
  - Механизм rollback (< 5 минут)

Хорошо иметь:
  - Автоматический remediation (Lambda/operator для частых issues)
  - Статусная страница (Statuspage.io / Atlassian)
  - Incident template в Slack
  - Escalation matrix с контактами
```

---

## 8. Capacity Planning: как прогнозировать нагрузку и планировать ресурсы?

**Четыре метода capacity planning:**

```
1. Trend-based (исторические данные):
   Смотрим на рост за последние 3-12 месяцев
   predict_linear(metric[30d], 90 * 24 * 3600) → прогноз через 90 дней

2. Load testing:
   Нагрузочные тесты → понять лимиты текущей системы
   k6, Locust, JMeter, Gatling

3. Event-based:
   Black Friday, маркетинговая кампания, launch → плановое масштабирование

4. Resource-per-unit:
   1000 users = 2 CPU + 4GB RAM → линейная модель
```

**Prometheus-based прогноз:**

```promql
# Когда закончится место на диске?
predict_linear(
  node_filesystem_avail_bytes{mountpoint="/"}[7d],
  30 * 24 * 3600  # прогноз на 30 дней
) < 0

# Прогноз использования CPU через 2 недели
predict_linear(
  avg_over_time(node_cpu_usage_percent[7d])[1d:1h],
  14 * 24 * 3600
)

# Рост трафика (запросы в секунду) — тренд
deriv(
  avg_over_time(sum(rate(http_requests_total[5m]))[1d:5m])[7d:1d]
)
```

**Capacity planning для Kubernetes:**

```bash
# Текущее использование ресурсов vs requests
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory

# Resource utilization
kubectl describe nodes | grep -A 5 "Allocated resources"

# Проверка headroom (свободных ресурсов для автоскейлинга)
kubectl get nodes -o json | jq '
  .items[] | {
    name: .metadata.name,
    cpu_capacity: .status.capacity.cpu,
    mem_capacity: .status.capacity.memory,
    cpu_allocatable: .status.allocatable.cpu,
    mem_allocatable: .status.allocatable.memory
  }
'
```

**Capacity planning для событий (Black Friday):**

```markdown
## Pre-Event Capacity Checklist

### 4 недели до события
- [ ] Провести нагрузочный тест с 2x ожидаемой нагрузкой
- [ ] Определить bottlenecks (DB connections, cache hit rate, CPU)
- [ ] Согласовать Pre-scaling план с командой

### 1 неделя до
- [ ] Pre-scale: увеличить min replicas для критических сервисов
- [ ] Проверить лимиты cloud provider (EC2 limits, RDS connections)
- [ ] Убедиться что CDN настроен (статика не идёт через origin)
- [ ] Проверить rate limits для внешних API (Stripe, SendGrid)

### День события
- [ ] Enhanced on-call: 2 инженера вместо 1
- [ ] Открыт war room (Slack channel)
- [ ] Grafana dashboard на экране
- [ ] Runbooks открыты и готовы
```

---

## 9. Reliability Patterns: Circuit Breaker, Retry, Bulkhead, Timeout

**Timeout — самый важный паттерн:**

```
Без timeout: один медленный downstream = зависание всей цепочки
С timeout: быстрый fail, пользователь получает ошибку, не зависание

Правило:
  Service A вызывает Service B
  Timeout A > Timeout B (на каждом уровне)
  
  Frontend: 30s
  API Gateway: 25s
  Service A: 20s
  Service B: 15s
  DB: 10s
```

**Circuit Breaker:**

```
Состояния:
  CLOSED   → запросы проходят нормально
  OPEN     → запросы блокируются, fallback срабатывает
  HALF-OPEN → пробный запрос для проверки восстановления

Переход:
  CLOSED → OPEN: failure rate > threshold (например, 50% за 30 секунд)
  OPEN → HALF-OPEN: через cooldown period (например, 30 секунд)
  HALF-OPEN → CLOSED: успешный пробный запрос
  HALF-OPEN → OPEN: неуспешный пробный запрос

Зачем: предотвращает каскадные отказы
  Без CB: A → B (упал) → A тоже падает от timeout
  С CB:  A → CB(OPEN) → fallback response → A работает деградировано
```

```go
// Пример с библиотекой sony/gobreaker
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "payment-service",
    MaxRequests: 1,
    Interval:    10 * time.Second,
    Timeout:     30 * time.Second,
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
        return counts.Requests >= 3 && failureRatio >= 0.6
    },
    OnStateChange: func(name string, from gobreaker.State, to gobreaker.State) {
        log.Printf("Circuit Breaker %s: %s → %s", name, from, to)
    },
})

result, err := cb.Execute(func() (interface{}, error) {
    return paymentClient.Charge(ctx, amount)
})
```

**Retry с Exponential Backoff + Jitter:**

```go
// ПЛОХО: retry без backoff = thundering herd
for i := 0; i < 3; i++ {
    err = call()
    if err == nil { break }
    time.Sleep(1 * time.Second)  // все ретраят одновременно!
}

// ХОРОШО: exponential backoff + jitter
func retryWithBackoff(fn func() error, maxRetries int) error {
    for attempt := 0; attempt < maxRetries; attempt++ {
        err := fn()
        if err == nil {
            return nil
        }
        
        // Не ретраить non-retryable ошибки (400, 401, 404)
        if isNonRetryable(err) {
            return err
        }
        
        // Exponential backoff: 1s, 2s, 4s, 8s...
        backoff := time.Duration(math.Pow(2, float64(attempt))) * time.Second
        // Jitter: ±30% от базового backoff
        jitter := time.Duration(rand.Float64() * float64(backoff) * 0.3)
        time.Sleep(backoff + jitter)
    }
    return fmt.Errorf("all %d retries failed", maxRetries)
}
```

**Bulkhead (переборки):**

```
Изоляция ресурсов по типу запросов.
Аналог: переборки в корабле — одна дыра не топит весь корабль.

Пример: Thread Pool Isolation
  /api/search   → pool: 50 threads (допускает долгие запросы)
  /api/checkout → pool: 20 threads (изолирован от search)
  /api/status   → pool: 5 threads  (всегда доступен)

Без bulkhead: медленный search заполняет все потоки → checkout тоже деградирует
С bulkhead:   медленный search деградирует изолированно → checkout работает
```

**Rate Limiting / Load Shedding:**

```nginx
# Nginx rate limiting
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
limit_req_zone $binary_remote_addr zone=checkout:10m rate=10r/m;

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
    }
    location /api/checkout {
        limit_req zone=checkout burst=5;
    }
}
```

---

## 10. Runbooks и Playbooks: зачем нужны и как писать?

**Runbook** — пошаговая инструкция для выполнения конкретной операции или реагирования на алерт.

**Playbook** — более широкий документ, описывающий процессы для целого класса ситуаций.

**Структура хорошего Runbook:**

```markdown
# Runbook: High Error Rate on Checkout API

## Когда использовать
Алерт: `CheckoutErrorRateHigh` (error rate > 1% за 5 минут)
Severity: SEV-2

## Быстрая диагностика (< 5 минут)

### 1. Проверь dashboard
URL: https://grafana.example.com/d/checkout-overview
Смотри на: Error rate, Latency, RPS, DB connections

### 2. Проверь логи
\```bash
# Последние ошибки
kubectl logs -n production -l app=checkout --since=10m | grep ERROR | tail -50

# В Kibana
Фильтр: service:checkout AND level:ERROR AND @timestamp:[now-15m TO now]
\```

### 3. Проверь зависимости
\```bash
# База данных
kubectl exec -n production deployment/checkout -- \
  psql $DATABASE_URL -c "SELECT count(*) FROM pg_stat_activity;"

# Redis
kubectl exec -n production deployment/checkout -- \
  redis-cli -h redis ping

# Внешний Payment API
curl -o /dev/null -w "%{http_code}" https://api.stripe.com/v1/health
\```

## Возможные причины и решения

### A. DB connection pool exhausted
Симптомы: ошибки "too many connections", latency растёт
\```bash
# Проверить текущие соединения
kubectl exec -n production deployment/checkout -- \
  psql $DATABASE_URL -c "SELECT count(*) FROM pg_stat_activity WHERE state='active';"

# Быстрый fix: перезапуск pods (освобождает соединения)
kubectl rollout restart deployment/checkout -n production
\```

### B. Внешний API (Stripe) деградирует
Симптомы: ошибки только для payment-related запросов
Действие: Проверить https://status.stripe.com/
Workaround: Включить "degraded mode" флаг (кеширование auth):
\```bash
kubectl set env deployment/checkout STRIPE_CIRCUIT_BREAKER=true -n production
\```

### C. Неудачный деплой
Симптомы: ошибки начались сразу после деплоя
\```bash
# Rollback
helm rollback checkout -n production
# или
kubectl rollout undo deployment/checkout -n production
\```

## Эскалация
Если через 15 минут не удалось митигировать:
1. Эскалировать на @tech-lead
2. Открыть SEV-1, если error rate > 10%
3. Уведомить #engineering-leads

## Post-incident
После решения создать тикет в Jira с тегом `postmortem-needed`
```

---

## 11. DORA Metrics: как измерить эффективность DevOps/SRE команды?

**DORA (DevOps Research and Assessment)** — четыре метрики для оценки эффективности команды:

```
1. Deployment Frequency (DF)
   Как часто мы деплоим в production?
   
   Elite: несколько раз в день
   High:  раз в день — раз в неделю
   Medium: раз в неделю — раз в месяц
   Low:   реже раза в месяц

2. Lead Time for Changes (LTC)
   Время от коммита до production
   
   Elite: < 1 часа
   High:  1 час — 1 день
   Medium: 1 день — 1 неделя
   Low:   > 1 месяца

3. Change Failure Rate (CFR)
   Какой % деплоев вызывает инцидент/rollback?
   
   Elite: < 5%
   High:  5-10%
   Medium: 10-15%
   Low:   > 15%

4. Mean Time to Recovery (MTTR)
   Время восстановления после инцидента
   
   Elite: < 1 часа
   High:  < 1 дня
   Medium: 1 день — 1 неделя
   Low:   > 1 недели
```

**Как измерять DORA в реальной системе:**

```python
# Пример сбора Deployment Frequency через GitHub API
import requests
from datetime import datetime, timedelta

def get_deployment_frequency(repo: str, days: int = 30) -> float:
    """Deployments per day"""
    url = f"https://api.github.com/repos/{repo}/deployments"
    deployments = requests.get(
        url,
        params={"environment": "production", "per_page": 100},
        headers={"Authorization": f"Bearer {GITHUB_TOKEN}"}
    ).json()
    
    cutoff = datetime.now() - timedelta(days=days)
    recent = [d for d in deployments 
              if datetime.fromisoformat(d["created_at"].replace("Z","")) > cutoff]
    
    return len(recent) / days
```

```promql
# Lead Time for Changes — из CI/CD метрик
# Если CI pipeline экспортирует метрики в Prometheus:
histogram_quantile(0.50,
  sum by (le) (
    rate(cicd_pipeline_duration_seconds{stage="production"}[30d])
  )
)

# Change Failure Rate
sum(rate(deployments_total{status="failed"}[30d]))
/ sum(rate(deployments_total[30d]))

# MTTR (из PagerDuty → Prometheus)
avg_over_time(incident_duration_seconds{severity="critical"}[30d])
```

**Почему DORA важны:**

```
DORA исследование (2019-2023, > 30,000 команд) показало:
  Elite performers:
    - В 208x чаще делают deployments
    - Lead time в 106x короче
    - В 2,604x быстрее восстанавливаются
    - Change failure rate в 7x ниже
  
  И при этом: более высокая надёжность + более высокая скорость доставки
  (скорость и надёжность — не компромисс, а взаимоусиливающие факторы)
```

---

## 12. Как проводить Game Days и почему это важно?

**Game Day** — контролируемое учение по симуляции реальных инцидентов в production или production-like окружении.

**Зачем:**
```
1. Проверить runbooks — работают ли в реальных условиях?
2. Найти слабые места ДО реального инцидента
3. Обучить команду реагированию на стресс
4. Убедиться что мониторинг обнаруживает проблему
5. Валидировать SLO — выдержит ли система?
```

**Типы Game Days:**

```
Tabletop Exercise (настольная игра):
  - Без реального воздействия на систему
  - Команда обсуждает: "Что бы мы делали если..."
  - Хорошо для новых команд, новых сценариев

Staged Outage (постановочный инцидент):
  - Реальное воздействие в staging/canary
  - Один человек знает что происходит, остальные реагируют как на реальный инцидент
  - Более реалистично чем tabletop

Chaos Engineering (автоматизированный):
  - Постоянные автоматические эксперименты
  - LitmusChaos, Chaos Monkey, AWS FIS
  - Высокая зрелость команды (mature)
```

**Пример Game Day сценария:**

```markdown
# Game Day: База данных primary недоступна

## Цель
Проверить что система автоматически переключается на replica
и команда умеет это делать вручную если автоматика не сработала.

## Участники
- IC (Incident Commander): @alice — знает сценарий
- On-call: @bob, @charlie — не знают что произойдёт
- Observer: @dave — записывает timeline и проблемы

## Сценарий
09:00 - Начало. IC симулирует failure primary DB
09:00 - @bob, @charlie реагируют как на реальный инцидент
09:30 - Дебриф: что сработало, что нет

## Ожидаемый результат
- Автоматический failover за < 30 секунд
- MTTD (алерт сработал) < 2 минут
- MTTR (сервис восстановлен) < 10 минут

## Инъекция хаоса
\```bash
# Симуляция недоступности primary
# Вариант 1: Network policy блокирует трафик к primary
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-db-primary
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: primary
  policyTypes:
  - Ingress
EOF

# Вариант 2: AWS FIS (Fault Injection Simulator)
aws fis start-experiment \
  --experiment-template-id EXT123 \
  --tags "GameDay=2024-01-15"
\```

## Критерии успеха
- [ ] Алерт DBPrimaryDown сработал < 2 минут
- [ ] Автоматический failover произошёл < 30 секунд
- [ ] Error rate на сервисе не превысил SLO
- [ ] Runbook DBFailover корректен и был использован

## Дебриф вопросы
1. Когда вы поняли что произошло? Что помогло/мешало?
2. Были ли runbooks актуальны?
3. Что нужно улучшить в мониторинге?
4. Какие action items?
```

**LitmusChaos для K8s:**

```yaml
# Эксперимент: pod-delete
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: pod-delete-engine
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=myapp"
    appkind: deployment
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"          # 60 секунд хаоса
            - name: CHAOS_INTERVAL
              value: "10"          # удалять pod каждые 10 секунд
            - name: FORCE
              value: "false"       # graceful termination
            - name: PODS_AFFECTED_PERC
              value: "50"          # 50% подов
```
