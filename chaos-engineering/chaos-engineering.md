# Chaos Engineering: Вопросы и ответы для DevOps-инженера (Middle/Senior)

## Содержание

1. [Что такое Chaos Engineering и зачем намеренно ломать систему?](#1-что-такое-chaos-engineering-и-зачем-намеренно-ломать-систему)
2. [Принципы Chaos Engineering: Hypothesis, Blast Radius, Steady State](#2-принципы-chaos-engineering-hypothesis-blast-radius-steady-state)
3. [Netflix Chaos Monkey и Simian Army: история и идеи](#3-netflix-chaos-monkey-и-simian-army-история-и-идеи)
4. [LitmusChaos: архитектура, эксперименты, ChaosEngine](#4-litmuschaos-архитектура-эксперименты-chaosengine)
5. [AWS Fault Injection Simulator (FIS): chaos в облаке](#5-aws-fault-injection-simulator-fis-chaos-в-облаке)
6. [Chaos Mesh: chaos experiments в Kubernetes](#6-chaos-mesh-chaos-experiments-в-kubernetes)
7. [Как проводить Game Day: планирование и шаблон](#7-как-проводить-game-day-планирование-и-шаблон)
8. [Chaos в CI/CD: автоматические эксперименты в pipeline](#8-chaos-в-cicd-автоматические-эксперименты-в-pipeline)
9. [Матurity модель Chaos Engineering: с чего начать?](#9-матurity-модель-chaos-engineering-с-чего-начать)

---

## 1. Что такое Chaos Engineering и зачем намеренно ломать систему?

**Chaos Engineering** — дисциплина экспериментирования над распределённой системой с целью повышения уверенности в её устойчивости в production условиях.

```
Проблема:
  Распределённые системы сложны.
  Сбои неизбежны: нода падает, сеть теряет пакеты, DB тормозит.
  Как убедиться что система выдержит?
  
  Традиционный подход: "Надеемся что не сломается"
  → Первый раз узнаём о проблеме на реальном инциденте в 3 ночи

Chaos Engineering:
  "Мы сами ломаем систему в контролируемых условиях"
  → Находим слабые места ДО реального инцидента
  → Проверяем мониторинг и алерты
  → Тренируем команду реагировать
```

**Ключевые вопросы которые отвечает Chaos Engineering:**

```
1. Что происходит если одна нода Kubernetes упадёт?
   → Все ли поды перезапустятся? Как быстро?

2. Что если база данных primary станет недоступной?
   → Произойдёт ли автоматический failover? Как быстро?

3. Что если downstream сервис отвечает медленно?
   → Сработает ли Circuit Breaker? Есть ли timeout?

4. Что если закончится место на диске?
   → Получим ли мы алерт до того как сервис упадёт?

5. Что если произойдёт network partition?
   → Как себя ведёт система при частичной недоступности?
```

**Примеры что нашли через Chaos Engineering:**

```
Netflix (реальный пример):
  Эксперимент: убить Cassandra ноду
  Найдено: Netflix streaming переставал работать при падении 1 ноды
  Исправление: улучшена репликация и retry логика
  
Amazon:
  Эксперимент: network latency 100ms для DynamoDB
  Найдено: Cart service зависал вместо деградации
  Исправление: добавлены timeouts, circuit breakers
```

---

## 2. Принципы Chaos Engineering: Hypothesis, Blast Radius, Steady State

**Процесс Chaos Engineering (5 шагов):**

```
1. Define Steady State
   Что значит "система работает нормально"?
   Метрики: error rate < 1%, P99 < 500ms, checkout success > 99%

2. Hypothesis
   "Если мы убьём одну ноду, система продолжит работать нормально
    (error rate < 1%, P99 < 500ms)"

3. Design Experiment
   Что именно ломаем? Как? На сколько времени?
   Blast radius: staging или canary в production?

4. Run Experiment
   Вводим сбой, наблюдаем за системой

5. Verify and Learn
   Подтвердилась ли гипотеза?
   Если нет → найдена реальная слабость → исправить → повторить
```

**Steady State:**

```yaml
# Steady State определяется через measurable metrics

Пример для e-commerce:
  Metric 1: HTTP Success Rate (checkout)
    value: > 99%
    probe: promql → sum(rate(http_requests_total{status!~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

  Metric 2: P99 Latency
    value: < 500ms
    probe: promql → histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

  Metric 3: Orders per minute
    value: > 10/min
    probe: promql → rate(orders_total[5m]) * 60

Если ВСЕ три метрики в норме → система в steady state
```

**Blast Radius — контроль масштаба влияния:**

```
Маленький blast radius (начало):
  - Один pod из 10
  - Staging окружение
  - 10% трафика (canary)
  - Ночью когда нагрузка низкая

Средний blast radius (после уверенности):
  - Одна нода из пяти
  - Production, но нерабочее время
  - Один сервис

Большой blast radius (зрелые команды):
  - AZ failure
  - Datacenter failure
  - Рабочее время с полным трафиком

Правило: начинай с наименьшего blast radius
         увеличивай только после уверенности что система выдержит
```

**Hypothesis-driven approach:**

```markdown
## Chaos Hypothesis Template

### Experiment: Pod Failure Resilience
**Date**: 2024-01-15
**Team**: Backend (SRE)

### Steady State
- HTTP Success Rate > 99.5%
- P99 Latency < 300ms
- Zero customer-visible errors

### Hypothesis
Если мы удалим 2 из 5 pods сервиса api-gateway,
система останется в steady state через 60 секунд,
так как K8s перезапустит pods и LB будет направлять трафик к оставшимся.

### Experiment Design
- Action: kubectl delete pod -l app=api-gateway --field-selector=... (2 pods)
- Duration: 5 минут наблюдения после удаления
- Rollback: нет (K8s сам восстановит)
- Blast Radius: 2/5 = 40% capacity

### Expected Results
- Кратковременный рост error rate (< 5 секунд)
- Автоматическое восстановление < 60 секунд
- Alarmы: PodDown должен сработать

### Actual Results
[заполнить после эксперимента]
```

---

## 3. Netflix Chaos Monkey и Simian Army: история и идеи

**История:**

```
2010: Netflix переезжает в AWS
      Первые инциденты из-за случайных сбоев AWS
      
2011: Netflix создаёт Chaos Monkey
      Случайно убивает EC2 инстансы в рабочее время
      Идея: "Если инстансы всё равно падают случайно,
             лучше пусть мы сами их убиваем когда готовы"

2012: Netflix публикует Chaos Monkey как open source
      Это меняет индустрию

2016: Netflix публикует концепцию Simian Army
```

**Simian Army — семейство Chaos инструментов Netflix:**

```
Chaos Monkey:        Убивает случайные EC2 инстансы
Latency Monkey:      Добавляет случайные задержки в сеть
Conformity Monkey:   Убивает инстансы не следующие best practices
Doctor Monkey:       Мониторит health, выводит нездоровые инстансы
Janitor Monkey:      Удаляет неиспользуемые облачные ресурсы
Security Monkey:     Находит security уязвимости
10-18 Monkey:        Тестирование локализации (i18n, l10n)
Chaos Gorilla:       Симулирует выход из строя всей AZ
Chaos Kong:          Симулирует выход из строя целого региона!
```

**Принципы которые принесло Netflix:**

```
"If it hurts, do it more often"
Если deployment painful — делай чаще, это заставит автоматизировать

"Embrace failure"
Сбои не исключение, они норма. Проектируй для сбоев.

"Build systems that tolerate failure"
Не "предотвращать все сбои" а "быть устойчивым к ним"
```

---

## 4. LitmusChaos: архитектура, эксперименты, ChaosEngine

**LitmusChaos** — CNCF Incubating проект для chaos engineering в Kubernetes.

**Архитектура:**

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Litmus Control Plane                 │   │
│  │  ┌──────────┐  ┌────────────┐  ┌──────────────┐  │   │
│  │  │  Chaos   │  │   Chaos    │  │   Chaos      │  │   │
│  │  │  Operator│  │  Center    │  │   Exporter   │  │   │
│  │  │  (CRD)   │  │  (UI)      │  │  (metrics)   │  │   │
│  │  └──────────┘  └────────────┘  └──────────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────┐                               │
│  │  Target Application  │ ← Chaos injected here        │
│  └─────────────────────┘                               │
└─────────────────────────────────────────────────────────┘
```

**Установка LitmusChaos:**

```bash
# Установка через Helm
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm install chaos litmuschaos/litmus \
  --namespace litmus \
  --create-namespace \
  --set portal.frontend.service.type=LoadBalancer

# Или Litmus Chaos Center (более полный)
kubectl apply -f https://litmuschaos.github.io/litmus/2.14.0/deploy/litmus-2.14.0.yaml
```

**Типы экспериментов:**

```
Pod Chaos:
  pod-delete          — удаление pod'ов
  pod-cpu-hog         — нагрузка CPU внутри pod
  pod-memory-hog      — нагрузка памяти
  pod-network-loss    — потеря пакетов
  pod-network-latency — добавить задержку
  pod-dns-error       — ошибки DNS
  pod-io-stress       — нагрузка диска
  container-kill      — убить конкретный контейнер в pod

Node Chaos:
  node-cpu-hog        — нагрузка CPU на ноде
  node-memory-hog     — нагрузка памяти ноды
  node-drain          — drain ноды (эвакуация pods)
  node-restart        — перезагрузка ноды
  node-taint          — добавить taint к ноде

AWS Chaos:
  ec2-terminate       — завершить EC2 инстанс
  ebs-loss            — отключить EBS том
  rds-instance-reboot — перезапуск RDS
  az-blackhole        — блокировать трафик из AZ
```

**ChaosEngine — запуск эксперимента:**

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: nginx-chaos
  namespace: production
spec:
  # Target приложение
  appinfo:
    appns: production
    applabel: "app=nginx"
    appkind: deployment

  engineState: active
  chaosServiceAccount: litmus-admin

  # Steady State проверки (до и после)
  experiments:
    - name: pod-delete
      spec:
        probe:
          # HTTP probe — проверяем что сервис работает
          - name: check-frontend-access-url
            type: httpProbe
            httpProbe/inputs:
              url: "https://myapp.example.com/health"
              insecureSkipVerify: false
              method:
                get:
                  criteria: ==
                  responseCode: "200"
            mode: Continuous          # проверять каждые X секунд
            runProperties:
              probeTimeout: 15s
              interval: 5s
              retry: 3
              probePollingInterval: 2s

          # Prometheus probe — проверяем метрики
          - name: check-success-rate
            type: promProbe
            promProbe/inputs:
              endpoint: "http://prometheus:9090"
              query: |
                sum(rate(http_requests_total{status!~"5.."}[5m]))
                / sum(rate(http_requests_total[5m])) > 0.99
              comparator:
                type: float
                criteria: ">="
                value: "0.99"
            mode: End                 # проверить только после эксперимента

        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"             # 60 секунд хаоса
            - name: CHAOS_INTERVAL
              value: "10"             # убивать pod каждые 10 секунд
            - name: FORCE
              value: "false"          # graceful termination
            - name: PODS_AFFECTED_PERC
              value: "50"             # 50% pods
```

**ChaosWorkflow — сложные сценарии:**

```yaml
# Workflow из нескольких экспериментов с условиями
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: chaos-workflow
spec:
  entrypoint: chaos-workflow
  templates:
    - name: chaos-workflow
      steps:
        - - name: install-experiment
            template: install-chaos
        - - name: run-chaos
            template: pod-delete-chaos
        - - name: revert-chaos
            template: cleanup
```

---

## 5. AWS Fault Injection Simulator (FIS): chaos в облаке

**AWS FIS** — управляемый сервис для chaos engineering в AWS, не требующий установки агентов.

**Поддерживаемые actions:**

```
EC2:    terminate, stop, reboot, cpu-stress, network-disruption
ECS:    drain container instance, stop task
EKS:    drain node group, terminate node
RDS:    failover DB cluster, reboot DB instance
AZ:     blackhole (изолировать AZ)
Network: packet loss, latency, bandwidth throttle
SSM:    run command на инстансах
```

**Пример: AZ Failure Experiment:**

```json
{
  "description": "Simulate AZ failure",
  "targets": {
    "ec2-instances-in-az-a": {
      "resourceType": "aws:ec2:instance",
      "resourceArns": [],
      "filters": [
        {
          "path": "Placement.AvailabilityZone",
          "values": ["us-east-1a"]
        },
        {
          "path": "State.Name",
          "values": ["running"]
        }
      ],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "terminate-az-a-instances": {
      "actionId": "aws:ec2:terminate-instances",
      "targets": {
        "Instances": "ec2-instances-in-az-a"
      }
    }
  },
  "stopConditions": [
    {
      "source": "aws:cloudwatch:alarm",
      "value": "arn:aws:cloudwatch:us-east-1:123:alarm/TooManyErrors"
    }
  ],
  "roleArn": "arn:aws:iam::123456789:role/FIS-Role",
  "tags": {
    "Purpose": "AZ-Failure-Test",
    "Team": "SRE"
  }
}
```

```bash
# Запуск эксперимента
aws fis create-experiment-template --cli-input-json file://az-failure.json

# Запуск
aws fis start-experiment \
  --experiment-template-id EXT123ABC \
  --tags "GameDay=2024-01-15"

# Мониторинг
aws fis get-experiment --id EXPabc123

# Остановить если что-то пошло не так
aws fis stop-experiment --id EXPabc123
```

**Stop Conditions — автоматическая остановка при проблемах:**

```
Обязательно добавлять Stop Conditions!
  
  Если error rate превысит 10% → остановить эксперимент автоматически
  
  Типы:
    aws:cloudwatch:alarm — CloudWatch Alarm в ALARM состоянии
    
# CloudWatch Alarm для Stop Condition
aws cloudwatch put-metric-alarm \
  --alarm-name "FIS-StopCondition-HighErrorRate" \
  --metric-name "5XXError" \
  --namespace "AWS/ApplicationELB" \
  --statistic "Sum" \
  --period 60 \
  --threshold 100 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 1
```

---

## 6. Chaos Mesh: chaos experiments в Kubernetes

**Chaos Mesh** — CNCF Graduated проект, альтернатива LitmusChaos с богатым набором network experiments.

**Установка:**

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-testing \
  --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

**Типы экспериментов:**

```yaml
# PodChaos — убить pod
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-example
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: one              # random-max-percent, fixed-percent, all, one
  selector:
    namespaces:
      - production
    labelSelectors:
      "app": "myapp"
  scheduler:
    cron: "@every 10m"   # каждые 10 минут

---
# NetworkChaos — задержки и потери пакетов
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
  namespace: chaos-testing
spec:
  action: delay
  mode: all
  selector:
    namespaces: [production]
    labelSelectors:
      app: myapp
  delay:
    latency: "200ms"
    jitter: "50ms"
    correlation: "25"
  direction: to              # to, from, both
  target:
    selector:
      namespaces: [production]
      labelSelectors:
        app: postgres       # задержки только к postgres
    mode: all
  duration: "5m"

---
# StressChaos — нагрузка CPU/Memory
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress
  namespace: chaos-testing
spec:
  mode: one
  selector:
    labelSelectors:
      app: myapp
  stressors:
    cpu:
      workers: 2             # 2 потока нагружают CPU
      load: 80               # 80% CPU
    memory:
      workers: 1
      size: "256MB"          # 256MB memory stress
  duration: "2m"

---
# DNSChaos — ошибки DNS резолвинга
apiVersion: chaos-mesh.org/v1alpha1
kind: DNSChaos
metadata:
  name: dns-random-error
spec:
  action: random
  mode: all
  selector:
    labelSelectors:
      app: myapp
  patterns:
    - "postgres.*"           # ошибки только для postgres DNS
  duration: "1m"

---
# Workflow — сложные сценарии
apiVersion: chaos-mesh.org/v1alpha1
kind: Workflow
metadata:
  name: cascade-failure-test
spec:
  entry: the-entry
  templates:
    - name: the-entry
      templateType: Serial    # последовательно
      children:
        - slow-network
        - pod-failure
        - verify-recovery

    - name: slow-network
      templateType: NetworkChaos
      deadline: 5m
      networkChaos:
        action: delay
        mode: all
        selector:
          labelSelectors:
            app: api
        delay:
          latency: "500ms"
        duration: 5m

    - name: pod-failure
      templateType: PodChaos
      deadline: 2m
      podChaos:
        action: pod-kill
        mode: fixed-percent
        value: "50"
        selector:
          labelSelectors:
            app: api
```

---

## 7. Как проводить Game Day: планирование и шаблон

**Полная процедура Game Day:**

**Неделя до:**

```markdown
## Preparation Checklist

### 1. Define Scope
- [ ] Выбрать сценарий (DB failover, AZ failure, pod crash, network partition)
- [ ] Определить участников (IC, Tech, Scribe, Observer)
- [ ] Определить blast radius (staging / production canary / full prod)
- [ ] Определить Steady State метрики

### 2. Prepare Environment
- [ ] Убедиться что мониторинг работает (Grafana, AlertManager)
- [ ] Проверить runbooks (актуальны ли?)
- [ ] Настроить Stop Conditions (автоматическая остановка)
- [ ] Уведомить стейкхолдеров (PM, Management)

### 3. Define Success/Failure Criteria
- [ ] Hypothesis записана
- [ ] Metrics baseline зафиксирован
- [ ] Expected vs Unacceptable outcomes определены
```

**День Game Day:**

```markdown
## Game Day Run Template

### Предстарт (T-30 минут)
09:30 - Все участники собрались в #game-day Slack
09:30 - Observer открыл документ с timeline
09:35 - Проверка baseline метрик (скриншот дашборда)
09:40 - Участники НЕ знают точного сценария (кроме IC)
09:45 - IC объясняет правила:
        "Реагировать как на реальный инцидент.
         Если считаете что нужно остановить — скажите STOP.
         Observer записывает всё."

### Старт (T=0)
10:00 - IC вводит сбой (через LitmusChaos / FIS / вручную)
10:00 - Observer начинает запись timeline

### Во время (T=0..30 min)
[Участники реагируют как на реальный инцидент]
Observer записывает:
  - Когда сработал алерт
  - Кто что сделал
  - Какие команды выполнялись
  - Когда обнаружена причина
  - Когда применён workaround
  - Когда система восстановилась

### Конец (T=30 min или при достижении цели)
10:30 - IC объявляет завершение
10:30 - Итоговый скриншот метрик

### Дебриф (30-60 минут)
10:35 - IC раскрывает сценарий
10:40 - Вопросы для обсуждения:
        1. Когда вы поняли что произошло? Что помогло/мешало?
        2. Все ли алерты сработали вовремя?
        3. Были ли runbooks полезны? Были ли актуальны?
        4. Что удивило?
        5. Что нужно улучшить?
10:55 - Action Items с владельцами и дедлайнами
11:00 - Конец
```

**Примеры Game Day сценариев (от простого к сложному):**

```
L1 (новички в chaos):
  - Удалить один pod вручную
  - Перезагрузить один нод dev кластера
  
L2 (базовый опыт):
  - 50% pods упали одновременно
  - DB primary недоступна (failover)
  - Высокая CPU нагрузка на критическом сервисе

L3 (опытные):
  - Network partition между двумя компонентами
  - Потеря всей AZ
  - Медленные ответы всех downstream зависимостей

L4 (эксперты):
  - Kubernetes control plane недоступен
  - Потеря нескольких AZ одновременно
  - Cascading failure сценарий
```

---

## 8. Chaos в CI/CD: автоматические эксперименты в pipeline

**Chaos в integration tests:**

```yaml
# GitHub Actions: запуск chaos experiments после деплоя в staging
chaos-tests:
  runs-on: ubuntu-latest
  needs: [deploy-staging]
  steps:
    - name: Wait for deployment to stabilize
      run: sleep 60

    - name: Record baseline metrics
      run: |
        SUCCESS_RATE=$(curl -s 'http://prometheus/api/v1/query?query=...' | jq '.data.result[0].value[1]')
        echo "Baseline success rate: $SUCCESS_RATE"

    - name: Run pod-delete chaos experiment
      run: |
        kubectl apply -f - <<EOF
        apiVersion: chaos-mesh.org/v1alpha1
        kind: PodChaos
        metadata:
          name: staging-chaos-${{ github.run_id }}
          namespace: chaos-testing
        spec:
          action: pod-kill
          mode: one
          selector:
            namespaces: [staging]
            labelSelectors:
              app: myapp
          duration: "2m"
        EOF

    - name: Wait for chaos duration + recovery
      run: sleep 180

    - name: Verify system recovered
      run: |
        # Проверить что система вернулась к steady state
        SUCCESS_RATE=$(curl -s 'http://prometheus/api/v1/query?query=...' | jq -r '.data.result[0].value[1]')
        
        if (( $(echo "$SUCCESS_RATE < 0.99" | bc -l) )); then
          echo "❌ System did not recover! Success rate: $SUCCESS_RATE"
          exit 1
        fi
        echo "✅ System recovered. Success rate: $SUCCESS_RATE"

    - name: Cleanup chaos experiment
      if: always()
      run: kubectl delete podchaos staging-chaos-${{ github.run_id }} -n chaos-testing
```

**Chaos Gates в deployment pipeline:**

```yaml
# Chaos Gate: нельзя деплоить в production если chaos тесты падают

deploy-production:
  needs: [chaos-tests]  # ждём результатов chaos
  if: needs.chaos-tests.result == 'success'
  # ...
```

---

## 9. Матurity модель Chaos Engineering: с чего начать?

**Chaos Engineering Maturity Model:**

```
Level 0: "We don't do chaos"
  Первый инцидент = паника
  Нет runbooks
  Нет chaos experience

Level 1: Exploratory (Табличные игры)
  Game Days как tabletop exercises
  Обсуждаем "что было бы если..."
  Нет реальных инъекций

Level 2: Manual Chaos
  Ручные инъекции в staging
  Фиксированные сценарии
  Runbooks написаны и протестированы
  Команда умеет реагировать

Level 3: Automated Chaos (CI/CD)
  Автоматические эксперименты в staging pipeline
  Chaos Gates: нельзя продвинуться без прохождения chaos тестов
  LitmusChaos / Chaos Mesh задеплоены

Level 4: Continuous Chaos (Production)
  Регулярные эксперименты в production (малый blast radius)
  Автоматизированные Game Days
  Chaos как часть культуры

Level 5: Advanced (Netflix уровень)
  Chaos Gorilla/Kong (AZ/Region failure)
  Автоматические эксперименты в рабочее время
  Полная autonomous resilience
```

**Практический план "с чего начать":**

```markdown
## Roadmap внедрения Chaos Engineering

### Месяц 1: Фундамент
- [ ] Определить Steady State метрики для топ-3 критичных сервисов
- [ ] Написать/обновить runbooks для топ-5 сценариев
- [ ] Провести первый tabletop Game Day (без реального хаоса)
- [ ] Убедиться что алерты работают (проверить каждый вручную)

### Месяц 2: Первые эксперименты
- [ ] Установить LitmusChaos или Chaos Mesh в staging
- [ ] Провести первый реальный Game Day в staging:
      Сценарий: удаление одного pod
- [ ] Задокументировать findings и action items
- [ ] Исправить найденные проблемы

### Месяц 3: Расширение
- [ ] Добавить 3-5 chaos экспериментов в staging CI/CD pipeline
- [ ] Провести Game Day с DB failover сценарием
- [ ] Ввести еженедельный chaos review (что нашли, что исправили)

### Месяц 4+: Production Chaos
- [ ] Начать с Chaos в production в нерабочее время (малый blast radius)
- [ ] Ввести регулярные (ежемесячные) Game Days
- [ ] Автоматизировать AZ failure тесты
```

**Культура, а не инструменты:**

```
Chaos Engineering провалится без:
  1. Blameless culture
     Найденные слабости = победа, не провал
     
  2. Management support
     "Мы намеренно ломаем production" нужно объяснить

  3. Психологическая безопасность
     Команда должна чувствовать что может сказать "стоп"
     Без последствий за обнаруженные проблемы

  4. Время на исправление
     Найденные проблемы должны исправляться
     Иначе смысл экспериментировать?

Помни:
  "The goal of chaos engineering is NOT to cause incidents.
   It's to prevent them by discovering weaknesses before they become incidents."
```
