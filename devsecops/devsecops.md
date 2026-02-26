# DevSecOps / Security: Вопросы и ответы для DevOps-инженера (Middle/Senior)

## Содержание

1. [Что такое DevSecOps и как интегрировать безопасность в CI/CD?](#1-что-такое-devsecops-и-как-интегрировать-безопасность-в-cicd)
2. [SAST и DAST: в чём разница, инструменты, где в pipeline?](#2-sast-и-dast-в-чём-разница-инструменты-где-в-pipeline)
3. [HashiCorp Vault: архитектура, Secret Engines, Auth Methods, динамические секреты](#3-hashicorp-vault-архитектура-secret-engines-auth-methods-динамические-секреты)
4. [OPA (Open Policy Agent) и Gatekeeper: policy-as-code в Kubernetes](#4-opa-open-policy-agent-и-gatekeeper-policy-as-code-в-kubernetes)
5. [Falco: runtime security, как работает, правила, алерты](#5-falco-runtime-security-как-работает-правила-алерты)
6. [SBOM (Software Bill of Materials): зачем нужен, как генерировать?](#6-sbom-software-bill-of-materials-зачем-нужен-как-генерировать)
7. [Сканирование Docker-образов: Trivy, Grype, Snyk](#7-сканирование-docker-образов-trivy-grype-snyk)
8. [Kubernetes Security: RBAC, SecurityContext, Pod Security Standards](#8-kubernetes-security-rbac-securitycontext-pod-security-standards)
9. [Управление секретами в Kubernetes: External Secrets Operator, Sealed Secrets](#9-управление-секретами-в-kubernetes-external-secrets-operator-sealed-secrets)
10. [Supply Chain Security: Sigstore, Cosign, подпись образов](#10-supply-chain-security-sigstore-cosign-подпись-образов)
11. [Zero Trust Architecture: принципы, реализация](#11-zero-trust-architecture-принципы-реализация)
12. [Threat Modeling: STRIDE, как применять для DevOps инфраструктуры?](#12-threat-modeling-stride-как-применять-для-devops-инфраструктуры)

---

## 1. Что такое DevSecOps и как интегрировать безопасность в CI/CD?

**DevSecOps** — практика интеграции security проверок на каждом этапе разработки и доставки ПО, а не как финальный gate перед релизом.

```
Традиционный подход (Security as Gate):
  Dev → QA → [Security Scan: 2 недели] → Prod
  Проблема: дорогое исправление на поздних стадиях, задержка релизов

DevSecOps (Shift Left):
  Commit → [SAST] → Build → [Image Scan] → Test → [DAST] → [IaC Scan] → Prod
  Плюс: ранее обнаружение = дешевле исправление
```

**Стоимость исправления уязвимости:**

```
На этапе разработки:  $80
На этапе QA:          $240
На этапе Production:  $7,600
После инцидента:      $millions
```

**Что включить в CI/CD pipeline:**

```yaml
# Уровни DevSecOps pipeline
stages:
  - pre-commit      # git hooks: secrets detection, lint security rules
  - sast            # статический анализ кода
  - dependency      # уязвимости в зависимостях (SCA)
  - build           # сборка образа
  - image-scan      # сканирование Docker образа
  - iac-scan        # проверка Terraform/Ansible
  - dast            # динамическое тестирование
  - sign            # подпись образа (cosign)
  - deploy          # деплой с policy check (OPA Gatekeeper)
  - runtime         # Falco (runtime protection)
```

**Pre-commit hooks (первая линия защиты):**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks   # обнаружение секретов в коде

  - repo: https://github.com/aquasecurity/trivy
    rev: v0.48.0
    hooks:
      - id: trivy-fs    # сканирование файловой системы

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.86.0
    hooks:
      - id: terraform_tfsec
      - id: terraform_checkov
```

---

## 2. SAST и DAST: в чём разница, инструменты, где в pipeline?

**SAST (Static Application Security Testing):**

```
Анализирует исходный код без запуска приложения.
  
  + Находит проблемы рано (на этапе разработки)
  + Не требует работающего приложения
  + Охватывает 100% кода
  - Много false positives
  - Не находит runtime проблемы
  
Место в pipeline: после checkout, перед build (< 5 минут)
```

**DAST (Dynamic Application Security Testing):**

```
Тестирует работающее приложение, имитируя атаки.
  
  + Находит runtime уязвимости (SQL injection, XSS, Auth bypass)
  + Меньше false positives
  - Требует работающего приложения
  - Медленнее (10-60 минут)
  - Не охватывает весь код
  
Место в pipeline: после деплоя в staging
```

**SCA (Software Composition Analysis) — анализ зависимостей:**

```
Находит уязвимости в сторонних библиотеках (CVE).
  Инструменты: OWASP Dependency Check, Snyk, Trivy fs
```

**Инструменты:**

| Тип | Инструмент | Языки | Лицензия |
|-----|-----------|-------|----------|
| SAST | Semgrep | 30+ языков | Open Source |
| SAST | CodeQL | 10+ языков | GitHub (бесплатно для OS) |
| SAST | SonarQube | 30+ языков | Community (бесплатно) |
| SAST | Bandit | Python | Open Source |
| SAST | gosec | Go | Open Source |
| DAST | OWASP ZAP | любые | Open Source |
| DAST | Nuclei | любые | Open Source |
| SCA | Trivy | контейнеры + fs | Open Source |
| SCA | Snyk | 10+ языков | Freemium |

**Semgrep в CI/CD:**

```yaml
# GitHub Actions
sast:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: returntocorp/semgrep-action@v1
      with:
        config: >-
          p/owasp-top-ten
          p/golang
          p/secrets
          p/docker
      env:
        SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
```

**OWASP ZAP для DAST:**

```yaml
dast:
  stage: dast
  needs: [deploy-staging]
  image: owasp/zap2docker-stable
  script:
    # Baseline scan (быстрый, пассивный)
    - zap-baseline.py
        -t https://staging.example.com
        -r zap-report.html
        -I  # не падать на warnings
    # Full scan (агрессивный, только для staging!)
    # - zap-full-scan.py -t https://staging.example.com
  artifacts:
    paths:
      - zap-report.html
    when: always
```

**Secret Scanning:**

```bash
# Gitleaks — сканирование всей истории git
gitleaks detect --source . --report-format sarif --report-path results.sarif

# TruffleHog — более глубокое сканирование
trufflehog git file://. --only-verified

# Detect-secrets — pre-commit
detect-secrets scan . > .secrets.baseline
detect-secrets audit .secrets.baseline
```

---

## 3. HashiCorp Vault: архитектура, Secret Engines, Auth Methods, динамические секреты

**Архитектура Vault:**

```
┌──────────────────────────────────────────────────────┐
│                    Vault Server                       │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │  Auth Methods│  │Secret Engines│  │  Policies  │  │
│  │  kubernetes  │  │  kv-v2       │  │  (HCL)     │  │
│  │  aws         │  │  database    │  │            │  │
│  │  github      │  │  pki         │  │            │  │
│  │  approle     │  │  transit     │  │            │  │
│  └──────────────┘  └──────────────┘  └────────────┘  │
│                          │                            │
│                    ┌──────────┐                       │
│                    │  Storage │ (Raft / Consul / S3)  │
│                    └──────────┘                       │
└──────────────────────────────────────────────────────┘
```

**Secret Engines:**

```bash
# KV-v2 — простое key-value хранилище (с версионированием)
vault kv put secret/myapp/db \
  username=admin \
  password=s3cr3t

vault kv get secret/myapp/db
vault kv get -version=2 secret/myapp/db
vault kv list secret/myapp/

# Database Engine — динамические учётные данные
# Vault создаёт временного пользователя в БД на время TTL
vault write database/roles/myapp-role \
  db_name=postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Получить динамические credentials
vault read database/creds/myapp-role
# Выдаёт: username=v-app-xyz12, password=random, lease_duration=1h
# Через 1 час пользователь удаляется автоматически

# PKI Engine — генерация сертификатов
vault secrets enable pki
vault write pki/root/generate/internal \
  common_name="example.com" \
  ttl=87600h  # 10 лет

vault write pki/roles/server \
  allowed_domains="example.com" \
  allow_subdomains=true \
  max_ttl=72h

# Получить сертификат
vault write pki/issue/server \
  common_name="api.example.com" \
  ttl=24h

# Transit Engine — шифрование как сервис (не хранит данные)
vault secrets enable transit
vault write transit/keys/myapp type=aes256-gcm96

# Зашифровать
vault write transit/encrypt/myapp \
  plaintext=$(echo "secret data" | base64)
# → ciphertext: vault:v1:...

# Расшифровать
vault write transit/decrypt/myapp \
  ciphertext="vault:v1:..."
```

**Auth Methods:**

```bash
# Kubernetes Auth — для pod'ов в K8s
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp \
  bound_service_account_namespaces=production \
  policies=myapp-policy \
  ttl=1h

# Приложение в K8s получает токен:
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -s --request POST \
  --data '{"jwt": "'$TOKEN'", "role": "myapp"}' \
  https://vault:8200/v1/auth/kubernetes/login

# AWS IAM Auth — для EC2/Lambda
vault auth enable aws
vault write auth/aws/config/client \
  access_key=$AWS_ACCESS_KEY_ID \
  secret_key=$AWS_SECRET_ACCESS_KEY

vault write auth/aws/role/myapp \
  auth_type=iam \
  bound_iam_principal_arn="arn:aws:iam::123456789:role/myapp-role" \
  policies=myapp-policy \
  ttl=1h
```

**Политики (HCL):**

```hcl
# /etc/vault/policies/myapp.hcl
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

path "database/creds/myapp-role" {
  capabilities = ["read"]
}

path "transit/encrypt/myapp" {
  capabilities = ["update"]
}

path "transit/decrypt/myapp" {
  capabilities = ["update"]
}
```

**Vault Agent — автоматическое получение и обновление секретов:**

```yaml
# vault-agent-config.hcl
auto_auth {
  method "kubernetes" {
    mount_path = "auth/kubernetes"
    config = {
      role = "myapp"
    }
  }
  
  sink "file" {
    config = {
      path = "/vault/token"
    }
  }
}

template {
  source      = "/vault/templates/db.ctmpl"
  destination = "/secrets/db.env"
  perms       = "0640"
}
```

```
{{- with secret "database/creds/myapp-role" -}}
DB_USER={{ .Data.username }}
DB_PASS={{ .Data.password }}
{{- end -}}
```

---

## 4. OPA (Open Policy Agent) и Gatekeeper: policy-as-code в Kubernetes

**OPA** — универсальный policy engine. Принимает input (JSON), проверяет по правилам (Rego), возвращает decision.

**Rego — язык политик OPA:**

```rego
package kubernetes.admission

# Запретить privileged контейнеры
deny[msg] {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    container.securityContext.privileged == true
    msg := sprintf("Container %v must not be privileged", [container.name])
}

# Требовать resource limits
deny[msg] {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    not container.resources.limits.memory
    msg := sprintf("Container %v must have memory limits", [container.name])
}

# Разрешить только образы из конкретного registry
deny[msg] {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    allowed_registries := ["registry.company.com", "gcr.io/my-project"]
    not startswith_any(container.image, allowed_registries)
    msg := sprintf("Image %v must be from approved registry", [container.image])
}

startswith_any(str, prefixes) {
    prefix := prefixes[_]
    startswith(str, prefix)
}
```

**Gatekeeper — OPA для Kubernetes:**

```yaml
# ConstraintTemplate — шаблон политики
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        
        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }

---
# Constraint — применение шаблона
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces: ["production", "staging"]
  parameters:
    labels: ["team", "app", "version"]
```

**Встроенные Gatekeeper политики (OPA Policy Library):**

```bash
# Установка Gatekeeper
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace

# Применение готовых политик
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/privileged-containers/template.yaml
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/pod-security-policy/read-only-root-filesystem/template.yaml

# Dry-run mode (только audit, без блокировки)
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
spec:
  enforcementAction: dryrun  # warn | deny | dryrun
```

---

## 5. Falco: runtime security, как работает, правила, алерты

**Falco** — cloud-native runtime security tool от CNCF. Мониторит syscalls и K8s audit logs в реальном времени.

**Как работает:**

```
Ядро Linux (syscalls)
     │ eBPF probe / kernel module
     ▼
Falco Engine (правила)
     │ match?
     ▼
Outputs: stdout / file / gRPC / Falco Sidekick
     │
     ▼
Alertmanager / Slack / PagerDuty / Elasticsearch
```

**Встроенные правила (примеры опасных событий):**

```yaml
# /etc/falco/falco_rules.yaml (встроенные)

# Shell запущен в контейнере — возможная компрометация
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
    and not user_expected_terminal_shell_in_container_conditions
  output: >
    A shell was spawned in a container with an attached terminal
    (user=%user.name user_loginuid=%user.loginuid %container.info
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline
    terminal=%proc.tty container_id=%container.id image=%container.image.repository)
  priority: NOTICE
  tags: [container, shell, mitre_execution]

# Запись в /etc — подозрительно для контейнера
- rule: Write below etc
  desc: An attempt to write to /etc directory
  condition: >
    write_etc_common
  output: >
    File below /etc opened for writing
    (user=%user.name user_loginuid=%user.loginuid command=%proc.cmdline
    parent=%proc.pname pcmdline=%proc.pcmdline file=%fd.name
    container_id=%container.id image=%container.image.repository)
  priority: ERROR

# Чтение credentials файлов
- rule: Read sensitive file untrusted
  desc: An attempt to read sensitive files
  condition: >
    sensitive_files and open_read
    and proc_name_exists
    and not proc.name in (user_mgmt_binaries, userexec_binaries, ...)
  output: >
    Sensitive file opened for reading by non-trusted program
    (user=%user.name user_loginuid=%user.loginuid
    program=%proc.name cmdline=%proc.cmdline
    file=%fd.name container_id=%container.id)
  priority: WARNING
```

**Кастомные правила:**

```yaml
# /etc/falco/falco_rules.local.yaml

# Crypto mining обнаружение
- rule: Cryptominer Execution
  desc: Detect known cryptomining tools
  condition: >
    spawned_process and
    proc.name in (xmrig, minerd, cpuminer, cgminer, bfgminer)
  output: >
    Cryptominer started (user=%user.name container=%container.name
    image=%container.image.repository cmdline=%proc.cmdline)
  priority: CRITICAL
  tags: [cryptomining, malware]

# Сетевое соединение из неожиданного контейнера
- rule: Unexpected outbound connection
  desc: Outbound connection from production database container
  condition: >
    outbound and
    container.image.repository = "postgres" and
    fd.rport != 5432  # только клиентские соединения неожиданны
  output: >
    Unexpected outbound connection from database container
    (container=%container.name remote=%fd.rip:%fd.rport)
  priority: WARNING
```

**Установка и интеграция:**

```bash
# Helm установка с eBPF driver (рекомендуется для production)
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/..."

# Falco Sidekick — маршрутизация алертов
# Поддерживает 50+ output: Slack, Teams, PagerDuty, Elasticsearch, Loki...
```

---

## 6. SBOM (Software Bill of Materials): зачем нужен, как генерировать?

**SBOM** — полный список компонентов, библиотек и зависимостей в программном продукте. Аналог списка ингредиентов на упаковке еды.

**Зачем нужен:**

```
1. Security: Log4Shell (CVE-2021-44228) — тысячи компаний не знали
   что используют Log4j внутри своих зависимостей.
   С SBOM: мгновенно найти все продукты с Log4j.

2. Compliance: US Executive Order 14028 (2021) — 
   обязателен для ПО, продаваемого федеральному правительству США.

3. License Management: проверить совместимость лицензий 
   (GPL vs Apache vs MIT).

4. Vulnerability Management: автоматически проверять SBOM 
   против NVD/OSV базы CVE.
```

**Форматы SBOM:**

```
SPDX (Software Package Data Exchange) — Linux Foundation
CycloneDX — OWASP, популярен в security-контексте
SWID — Microsoft, ISO стандарт
```

**Генерация SBOM с Syft:**

```bash
# Установка
brew install syft

# Из Docker образа
syft nginx:latest -o cyclonedx-json > nginx-sbom.json
syft nginx:latest -o spdx-json > nginx-sbom.spdx.json

# Из файловой системы
syft dir:/path/to/project -o cyclonedx-json

# Из запущенного контейнера
syft container:my-running-container

# Пример вывода (компоненты найдены в образе)
syft alpine:latest
 ✔ Loaded image
 ✔ Parsed image
 ✔ Cataloged packages      [15 packages]
NAME                    VERSION   TYPE
alpine-baselayout       3.4.3-r2  apk
alpine-keys             2.4-r1    apk
apk-tools               2.14.0-r5 apk
...
```

**Сканирование SBOM на уязвимости (Grype):**

```bash
# Grype — сканирование образа или SBOM
grype nginx:latest
grype sbom:nginx-sbom.json

# Пример вывода:
# NAME      INSTALLED  FIXED-IN  TYPE  VULNERABILITY  SEVERITY
# libssl3   3.1.4-r5   3.1.4-r6  apk   CVE-2024-xxxx  High

# Только Critical и High
grype nginx:latest --fail-on high

# В CI/CD
grype sbom:image-sbom.json \
  --output sarif \
  --file grype-results.sarif \
  --fail-on critical
```

**SBOM в CI/CD pipeline:**

```yaml
sbom-and-scan:
  runs-on: ubuntu-latest
  steps:
    - name: Generate SBOM
      uses: anchore/sbom-action@v0
      with:
        image: ${{ env.IMAGE_TAG }}
        format: cyclonedx-json
        output-file: sbom.json

    - name: Scan SBOM for vulnerabilities
      uses: anchore/scan-action@v3
      with:
        sbom: sbom.json
        fail-build: true
        severity-cutoff: high

    - name: Attach SBOM to image (in OCI registry)
      run: |
        cosign attach sbom --sbom sbom.json ${{ env.IMAGE_TAG }}
```

---

## 7. Сканирование Docker-образов: Trivy, Grype, Snyk

**Trivy — самый популярный OSS сканер:**

```bash
# Установка
brew install trivy
# или
docker pull aquasec/trivy

# Сканирование образа
trivy image nginx:latest

# Только High и Critical
trivy image --severity HIGH,CRITICAL nginx:latest

# Сканирование файловой системы (зависимости проекта)
trivy fs .

# Сканирование Terraform/Kubernetes конфигов
trivy config .
trivy config ./terraform/
trivy config ./k8s-manifests/

# Сканирование Git репозитория (secrets + misconfigs)
trivy repo https://github.com/org/repo

# Форматы вывода
trivy image --format json nginx:latest > results.json
trivy image --format sarif --output trivy-results.sarif nginx:latest
trivy image --format template --template "@contrib/html.tpl" -o report.html nginx:latest

# Ignoring known false positives
# .trivyignore
# CVE-2023-12345  # false positive, не применимо к нашей конфигурации
```

**Trivy в CI/CD:**

```yaml
container-scan:
  runs-on: ubuntu-latest
  needs: [build-and-push]
  steps:
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.IMAGE_TAG }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'          # завалить pipeline при нахождении CRITICAL
        ignore-unfixed: true    # игнорировать без патча

    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'
```

**Docker Scout (Docker Inc.):**

```bash
# Встроен в Docker Desktop и docker CLI
docker scout cves nginx:latest
docker scout recommendations nginx:latest  # рекомендует обновления базового образа
docker scout compare --to nginx:1.25 nginx:latest
```

**Snyk — коммерческий, богатые функции:**

```bash
# Snyk Container
snyk container test nginx:latest --severity-threshold=high
snyk container monitor nginx:latest  # постоянный мониторинг

# Snyk Open Source (зависимости)
snyk test --file=package.json
snyk test --file=go.mod

# Snyk IaC
snyk iac test terraform/
snyk iac test kubernetes/
```

---

## 8. Kubernetes Security: RBAC, SecurityContext, Pod Security Standards

**RBAC — Role-Based Access Control:**

```yaml
# Принцип минимальных привилегий (Least Privilege)

# ServiceAccount для приложения
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: production

---
# Role — права в конкретном namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]      # только чтение
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

---
# RoleBinding — привязка Role к ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-rolebinding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: myapp
    namespace: production
roleRef:
  kind: Role
  apiRef: myapp-role
  apiGroup: rbac.authorization.k8s.io

---
# ClusterRole для cross-namespace доступа
# Например: Prometheus scraping по всем namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-scraper
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
```

**SecurityContext:**

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      # Pod-level security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault  # включить seccomp профиль по умолчанию

      containers:
        - name: myapp
          # Container-level security context
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true  # read-only корневая ФС
            capabilities:
              drop:
                - ALL               # убрать все capabilities
              add:
                - NET_BIND_SERVICE  # добавить только нужные (порт < 1024)
```

**Pod Security Standards (PSS) — заменил PodSecurityPolicy:**

```yaml
# Применяется через label на namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted   # блокировать нарушения
    pod-security.kubernetes.io/warn: restricted       # предупреждать
    pod-security.kubernetes.io/audit: restricted      # логировать в audit

# Три профиля:
# privileged  — без ограничений (для system namespaces)
# baseline    — базовые защиты (запрет privileged, hostNetwork/hostPID)
# restricted  — максимальная безопасность (runAsNonRoot, readOnly, drop ALL caps)
```

**Audit Logging:**

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Логировать все изменения Secrets
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
    verbs: ["create", "update", "patch", "delete"]

  # Логировать exec в pod'ы (возможная компрометация)
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]

  # Минимальный logging для read-only операций
  - level: None
    verbs: ["get", "list", "watch"]
    resources:
      - group: ""
        resources: ["events"]
```

---

## 9. Управление секретами в Kubernetes: External Secrets Operator, Sealed Secrets

**Проблема нативных K8s Secrets:**

```
kubectl get secret myapp-secret -o yaml
# → пароль в base64 (не шифрование! просто encoding)

Проблемы:
  - Секреты в etcd только base64
  - Нельзя хранить в Git (небезопасно)
  - Нет ротации, аудита
  - Нет интеграции с Vault/AWS Secrets Manager
```

**External Secrets Operator (ESO) — рекомендуется:**

```yaml
# ESO синхронизирует секреты из внешних систем в K8s Secrets
# Поддерживает: Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault

# SecretStore — подключение к Vault
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "http://vault:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp"

---
# ExternalSecret — что синхронизировать
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-db-secret
  namespace: production
spec:
  refreshInterval: 1h           # обновлять каждый час
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: myapp-db-secret        # имя K8s Secret
    creationPolicy: Owner
  data:
    - secretKey: db-password      # ключ в K8s Secret
      remoteRef:
        key: secret/myapp/db      # путь в Vault
        property: password        # поле в Vault

---
# Для AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa  # с IRSA аннотацией
```

**Sealed Secrets — шифрованные секреты в Git:**

```bash
# Sealed Secrets позволяет хранить зашифрованные секреты в Git
# Только controller в K8s может расшифровать

# Установка
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# Получить публичный ключ
kubeseal --fetch-cert > sealed-secrets-cert.pem

# Зашифровать секрет
kubectl create secret generic myapp-secret \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml | \
  kubeseal --cert sealed-secrets-cert.pem \
  --format yaml > sealed-secret.yaml

# Теперь sealed-secret.yaml можно хранить в Git!
# Controller в K8s расшифрует его автоматически

# Применить
kubectl apply -f sealed-secret.yaml
# K8s автоматически создаст обычный Secret
```

**Encryption at Rest для etcd:**

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: BASE64_ENCODED_32_BYTE_KEY
      - identity: {}  # fallback для незашифрованных (для миграции)
```

---

## 10. Supply Chain Security: Sigstore, Cosign, подпись образов

**Проблема без Supply Chain Security:**

```
SolarWinds attack (2020): злоумышленники внедрили backdoor
в процесс сборки → подписанные официальные артефакты содержали malware.

Без verification: как убедиться что образ в registry
= тому что было собрано в CI/CD?
```

**Cosign — подпись Docker образов:**

```bash
# Генерация ключевой пары
cosign generate-key-pair
# → cosign.key (приватный), cosign.pub (публичный)

# Подпись образа
cosign sign --key cosign.key registry.example.com/myapp:v1.0.0

# Верификация
cosign verify --key cosign.pub registry.example.com/myapp:v1.0.0

# Подпись с дополнительными аннотациями
cosign sign \
  --key cosign.key \
  --annotations "git-commit=${GIT_SHA}" \
  --annotations "pipeline-id=${CI_PIPELINE_ID}" \
  registry.example.com/myapp:v1.0.0

# Keyless signing (рекомендуется в CI/CD) — через OIDC, без ключей
# Использует Fulcio (CA) и Rekor (transparency log)
cosign sign --identity-token=$(cat /var/run/secrets/tokens/oidc-token) \
  registry.example.com/myapp:v1.0.0
```

**Cosign в GitHub Actions:**

```yaml
sign-image:
  runs-on: ubuntu-latest
  needs: build-and-push
  permissions:
    contents: read
    id-token: write   # для keyless signing через GitHub OIDC
    packages: write

  steps:
    - uses: sigstore/cosign-installer@v3

    - name: Sign image
      run: |
        cosign sign --yes ${{ env.IMAGE_TAG }}
        # Keyless: использует GitHub Actions OIDC token
        # Запись в public Rekor transparency log
```

**Verifying в Kubernetes (Policy Controller / Kyverno):**

```yaml
# Kyverno policy — только подписанные образы
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: verify-cosign-signature
      match:
        any:
          - resources:
              kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "registry.example.com/*"
          attestors:
            - count: 1
              entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
                      -----END PUBLIC KEY-----
```

**SLSA (Supply chain Levels for Software Artifacts):**

```
SLSA уровни (0-3+):
  L0: Нет гарантий
  L1: Build process задокументирован (SBOM, provenance)
  L2: Signed provenance (кто собрал, когда, из чего)
  L3: Изолированная среда сборки (hardened CI)
  
GitHub Actions + cosign + Sigstore → SLSA L2-L3

# Генерация provenance с SLSA GitHub Generator
uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v2.0.0
with:
  image: ${{ env.IMAGE_TAG }}
```

---

## 11. Zero Trust Architecture: принципы, реализация

**Zero Trust: "Never trust, always verify"**

```
Традиционный Perimeter Security:
  "Всё внутри периметра = доверенное"
  VPN → внутренняя сеть → доступ ко всему
  Проблема: компрометация одного узла = доступ ко всей сети

Zero Trust:
  Нет доверия по умолчанию (ни для пользователей, ни для сервисов, ни для сетей)
  Каждый запрос верифицируется:
    WHO:  кто делает запрос? (identity)
    WHAT: к каким ресурсам? (context)
    HOW:  с какого устройства? (device posture)
```

**Пять столпов Zero Trust:**

```
1. Identity — MFA, SSO, conditional access
   Инструменты: Okta, Azure AD, Google Workspace

2. Device — device health checks перед доступом
   Инструменты: MDM, Endpoint Detection (CrowdStrike)

3. Network — микросегментация, encrypt-all
   Инструменты: mTLS (Istio), NetworkPolicy, VPC

4. Application — per-request authorization
   Инструменты: OPA, Gatekeeper, IAM

5. Data — encryption, DLP, classification
   Инструменты: KMS, Macie, DLP tools
```

**Практическая реализация Zero Trust в K8s:**

```yaml
# 1. mTLS между всеми сервисами (Istio)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

# 2. Authorization Policy — кто с кем разговаривает
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all-default
  namespace: production
spec:
  {} # deny all by default

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  selector:
    matchLabels:
      app: api
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/frontend
      to:
        - operation:
            methods: ["GET", "POST"]

# 3. NetworkPolicy — defense in depth
# 4. RBAC — минимальные привилегии для ServiceAccounts
# 5. Secret rotation — Vault dynamic credentials
```

---

## 12. Threat Modeling: STRIDE, как применять для DevOps инфраструктуры?

**STRIDE** — методология threat modeling от Microsoft:

| Буква | Угроза | Пример | Контрмера |
|-------|--------|--------|-----------|
| **S**poofing | Подмена identity | Кто-то использует чужой токен | MFA, mTLS |
| **T**ampering | Изменение данных | Подмена конфига в etcd | Integrity checks, Audit logs |
| **R**epudiation | Отрицание действий | "Я не делал этот деплой" | Audit logs, digital signatures |
| **I**nformation Disclosure | Утечка данных | Секреты в логах, CVE | Encryption, RBAC, Secret scanning |
| **D**oS | Отказ в обслуживании | DDoS, resource exhaustion | Rate limiting, Resource limits |
| **E**levation of Privilege | Повышение привилегий | Container escape | Non-root, capabilities drop, PSS |

**Threat Modeling для CI/CD pipeline:**

```
Активы (Assets):
  - Исходный код
  - Secrets/credentials
  - Production окружение
  - Docker образы
  - Supply chain (зависимости)

Угрозы STRIDE для CI/CD:

S: Spoofing
  - Злоумышленник пишет в репозиторий от имени другого
  - Поддельный runner перехватывает секреты
  Контрмеры: branch protection, code signing, runner isolation

T: Tampering
  - Изменение Dockerfile или pipeline без ревью
  - Компрометация dependency (supply chain)
  Контрмеры: signed commits, pin dependencies, SBOM

R: Repudiation
  - Кто задеплоил этот код?
  Контрмеры: immutable audit logs, signed commits, DORA metrics

I: Information Disclosure
  - Secrets в логах CI
  - SBOM раскрывает уязвимые компоненты
  Контрмеры: secret masking, restrict log access

D: DoS
  - Заполнение runner queue (CI abuse)
  - Большой образ забивает registry
  Контрмеры: rate limits на pipelines, registry quotas

E: Elevation of Privilege
  - Privileged container escape к node
  - Kubernetes ServiceAccount с избыточными правами
  Контрмеры: PSS restricted, RBAC least privilege, non-root
```

**Практический процесс:**

```markdown
## Threat Modeling Session (2 часа)

1. Diagram (30 мин): нарисовать DFD (Data Flow Diagram)
   - Все компоненты системы
   - Направления потоков данных
   - Границы доверия (trust boundaries)

2. STRIDE per element (60 мин):
   Для каждого компонента и потока данных — применить STRIDE

3. Risk rating (20 мин): DREAD или CVSS
   Damage, Reproducibility, Exploitability, Affected users, Discoverability

4. Mitigation plan (10 мин):
   Для каждой угрозы: Accept / Mitigate / Transfer / Avoid
   
Инструменты: OWASP Threat Dragon, Microsoft Threat Modeling Tool, Miro
```
