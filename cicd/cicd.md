# CI/CD: Вопросы и ответы для DevOps-инженера (Middle/Senior)

## Содержание

1. [Что такое CI/CD и в чём разница между CI, CD (Delivery) и CD (Deployment)?](#1-что-такое-cicd-и-в-чём-разница-между-ci-cd-delivery-и-cd-deployment)
2. [Как устроен типичный pipeline CI/CD? Какие стадии должны быть?](#2-как-устроен-типичный-pipeline-cicd-какие-стадии-должны-быть)
3. [GitHub Actions: архитектура, workflow, jobs, steps, runners](#3-github-actions-архитектура-workflow-jobs-steps-runners)
4. [GitLab CI: архитектура, .gitlab-ci.yml, stages, runners, artifacts](#4-gitlab-ci-архитектура-gitlab-ciyml-stages-runners-artifacts)
5. [Jenkins: архитектура, Jenkinsfile, declarative vs scripted pipeline](#5-jenkins-архитектура-jenkinsfile-declarative-vs-scripted-pipeline)
6. [Как реализовать стратегию ветвления и CI/CD для разных окружений (dev/staging/prod)?](#6-как-реализовать-стратегию-ветвления-и-cicd-для-разных-окружений-devstaginprod)
7. [Как безопасно управлять секретами в CI/CD pipeline?](#7-как-безопасно-управлять-секретами-в-cicd-pipeline)
8. [Что такое GitOps и чем он отличается от классического CI/CD?](#8-что-такое-gitops-и-чем-он-отличается-от-классического-cicd)
9. [ArgoCD: архитектура, App of Apps, Sync Policy, Health Status](#9-argocd-архитектура-app-of-apps-sync-policy-health-status)
10. [Как реализовать zero-downtime deployments? Blue/Green и Canary стратегии](#10-как-реализовать-zero-downtime-deployments-bluegreen-и-canary-стратегии)
11. [Как ускорить CI/CD pipeline? Кэширование, параллелизм, incremental builds](#11-как-ускорить-cicd-pipeline-кэширование-параллелизм-incremental-builds)
12. [Как тестировать инфраструктуру в CI/CD? IaC-тесты, policy-as-code](#12-как-тестировать-инфраструктуру-в-cicd-iac-тесты-policy-as-code)
13. [Что такое trunk-based development и как он связан с CI?](#13-что-такое-trunk-based-development-и-как-он-связан-с-ci)
14. [Как откатить деплой? Rollback стратегии в Kubernetes и ArgoCD](#14-как-откатить-деплой-rollback-стратегии-в-kubernetes-и-argocd)
15. [Self-hosted runners vs Cloud runners: когда что использовать?](#15-self-hosted-runners-vs-cloud-runners-когда-что-использовать)

---

## 1. Что такое CI/CD и в чём разница между CI, CD (Delivery) и CD (Deployment)?

**CI (Continuous Integration)** — практика автоматической сборки и тестирования каждого изменения кода сразу после его публикации в репозиторий. Цель — как можно раньше обнаружить конфликты и ошибки.

**CD (Continuous Delivery)** — расширение CI: артефакт (Docker-образ, бинарник, helm chart) после прохождения всех тестов всегда готов к деплою в production. Но сам деплой в prod запускается **вручную** (по нажатию кнопки).

**CD (Continuous Deployment)** — полная автоматизация: каждый коммит в master/main, прошедший все проверки, **автоматически** деплоится в production без участия человека.

```
Commit → Build → Test → [Deliver: artifact ready] → [Deploy: автоматически в prod]
         ↑_______CI_____↑         ↑___CD Delivery___↑ ↑____CD Deployment___________↑
```

**На практике** большинство компаний используют:
- Continuous Deployment для dev/staging окружений
- Continuous Delivery для production (ручное подтверждение или approval gate)

---

## 2. Как устроен типичный pipeline CI/CD? Какие стадии должны быть?

Типичный production-grade pipeline включает следующие стадии:

```
┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  Source  │→ │  Build  │→ │   Test   │→ │ Security │→ │ Package  │→ │  Deploy  │
│  (Git)   │  │(compile)│  │(unit/int)│  │  (SAST)  │  │ (Docker) │  │ (K8s/CD) │
└─────────┘  └─────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

**Детально по стадиям:**

| Стадия | Что делается | Инструменты |
|--------|--------------|-------------|
| **Source** | Триггер на push/PR, checkout кода | Git, GitHub/GitLab |
| **Build** | Компиляция, сборка Docker-образа | Docker BuildKit, Gradle, Maven |
| **Unit Tests** | Быстрые юнит-тесты | pytest, JUnit, go test |
| **Static Analysis** | Линтинг, форматирование кода | ESLint, golangci-lint, flake8 |
| **SAST** | Поиск уязвимостей в коде | Semgrep, SonarQube, CodeQL |
| **Integration Tests** | Тесты с реальными зависимостями (DB) | TestContainers, docker-compose |
| **Image Scan** | Сканирование Docker-образа | Trivy, Grype, Snyk |
| **Publish** | Push образа в registry | Docker Hub, ECR, GCR |
| **Deploy Staging** | Деплой в staging окружение | Helm, ArgoCD, kubectl |
| **E2E / Smoke Tests** | Проверка живого окружения | Playwright, k6, curl |
| **Deploy Prod** | Деплой в production (manual gate) | ArgoCD, Spinnaker |
| **Notify** | Уведомление в Slack/Teams | Webhook, PagerDuty |

**Важные принципы:**
- **Fail fast**: самые быстрые проверки — первыми (lint < unit < integration < e2e)
- **Idempotency**: повторный запуск pipeline должен давать тот же результат
- **Reproducibility**: фиксируй версии зависимостей, используй lock-файлы

---

## 3. GitHub Actions: архитектура, workflow, jobs, steps, runners

**Архитектура GitHub Actions:**

```
Repository
  └── .github/workflows/
        ├── ci.yml          # triggered on push/PR
        ├── release.yml     # triggered on tag
        └── scheduled.yml   # triggered by cron
```

**Ключевые концепции:**

- **Workflow** — YAML-файл, описывающий автоматизацию. Триггеруется событиями (push, pull_request, schedule, workflow_dispatch)
- **Job** — набор шагов, выполняющихся на одном runner. Jobs по умолчанию параллельны
- **Step** — отдельная команда или Action внутри job
- **Action** — переиспользуемый модуль (из GitHub Marketplace или локальный)
- **Runner** — виртуальная машина, на которой выполняется job (ubuntu-latest, windows-latest, macos-latest или self-hosted)

**Пример production workflow:**

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true  # кэширует go modules автоматически

      - name: Run tests
        run: go test ./... -race -coverprofile=coverage.out

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: coverage.out

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: golangci/golangci-lint-action@v6
        with:
          version: v1.62

  build-and-push:
    needs: [test, lint]  # ждём успешного прохождения test и lint
    runs-on: ubuntu-latest
    # запускаем только при пуше в main (не на PR)
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    permissions:
      contents: read
      packages: write

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha  # кэш GitHub Actions
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging  # требует approval в настройках

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          helm upgrade --install myapp ./helm/myapp \
            --namespace staging \
            --set image.tag=${{ github.sha }} \
            --wait --timeout 5m
        env:
          KUBECONFIG_DATA: ${{ secrets.KUBECONFIG_STAGING }}
```

**Полезные паттерны:**

```yaml
# Матричная стратегия — тестирование на нескольких версиях
jobs:
  test:
    strategy:
      matrix:
        go-version: ['1.21', '1.22', '1.23']
        os: [ubuntu-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}

# Reusable workflow — вызов из других workflow
jobs:
  call-shared-workflow:
    uses: org/shared-workflows/.github/workflows/docker-build.yml@main
    with:
      image-name: myapp
    secrets: inherit
```

---

## 4. GitLab CI: архитектура, .gitlab-ci.yml, stages, runners, artifacts

**Архитектура GitLab CI:**

```
GitLab Server
  ├── GitLab Runner 1 (docker executor)   — в облаке
  ├── GitLab Runner 2 (kubernetes executor) — self-hosted
  └── GitLab Runner 3 (shell executor)    — bare metal

Pipeline → Stages → Jobs → Scripts
```

**Ключевые концепции:**

- **Stage** — логическая группа. Jobs одного stage выполняются параллельно
- **Job** — единица работы: запускается на runner, выполняет скрипт
- **Artifact** — файлы, которые job сохраняет и передаёт следующим stages
- **Cache** — кэш зависимостей между pipelines (node_modules, .m2)
- **Needs** — явная зависимость между jobs без привязки к stage (DAG-режим)

**Пример .gitlab-ci.yml:**

```yaml
stages:
  - build
  - test
  - security
  - publish
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

default:
  image: docker:26
  services:
    - docker:26-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

# ──────────────────────────────────────────
build:
  stage: build
  script:
    - docker build --cache-from $CI_REGISTRY_IMAGE:latest -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  cache:
    key: "$CI_COMMIT_REF_SLUG"
    paths:
      - .docker-cache/

# ──────────────────────────────────────────
unit-tests:
  stage: test
  image: golang:1.23-alpine
  needs: []  # запускается параллельно с build (DAG)
  script:
    - go test ./... -race -coverprofile=coverage.out
  coverage: '/coverage: \d+\.\d+%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    expire_in: 1 week

lint:
  stage: test
  needs: []
  image: golangci/golangci-lint:v1.62
  script:
    - golangci-lint run --timeout=5m

# ──────────────────────────────────────────
trivy-scan:
  stage: security
  needs: [build]
  script:
    - docker run --rm -v /var/run/docker.sock:/var/run/docker.sock
        aquasec/trivy:latest image --exit-code 1
        --severity CRITICAL $IMAGE_TAG
  allow_failure: false

# ──────────────────────────────────────────
tag-latest:
  stage: publish
  needs: [build, unit-tests, lint, trivy-scan]
  only:
    - main
  script:
    - docker pull $IMAGE_TAG
    - docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest

# ──────────────────────────────────────────
deploy-staging:
  stage: deploy
  needs: [tag-latest]
  image: bitnami/kubectl:latest
  only:
    - main
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - kubectl set image deployment/myapp
        myapp=$IMAGE_TAG
        --namespace=staging
  when: on_success

deploy-production:
  stage: deploy
  needs: [deploy-staging]
  only:
    - main
  environment:
    name: production
    url: https://example.com
  when: manual  # требует ручного подтверждения
  script:
    - helm upgrade --install myapp ./helm
        --set image.tag=$CI_COMMIT_SHORT_SHA
        --namespace=production
```

**GitLab Runner executors:**

| Executor | Изоляция | Скорость | Когда использовать |
|----------|----------|----------|-------------------|
| `docker` | Каждый job в новом контейнере | Средняя | CI/CD в облаке |
| `kubernetes` | Pod на каждый job | Средняя | K8s-native окружения |
| `shell` | На хост-машине | Быстрая | Bare metal, GPU |
| `docker+machine` | Autoscaling VM | Медленный старт | Burst workloads |

```bash
# Регистрация runner
gitlab-runner register \
  --url https://gitlab.com \
  --token RUNNER_TOKEN \
  --executor docker \
  --docker-image alpine:latest \
  --description "my-docker-runner"
```

---

## 5. Jenkins: архитектура, Jenkinsfile, declarative vs scripted pipeline

**Архитектура Jenkins:**

```
Jenkins Master (Controller)
  ├── Job Queue
  ├── Plugin Manager
  ├── Credentials Store
  └── Agents (Workers)
        ├── Agent 1 (SSH, Linux)
        ├── Agent 2 (JNLP, Windows)
        └── Agent 3 (Kubernetes Pod)
```

- **Master/Controller** — управляет расписанием, хранит конфигурацию, UI
- **Agent** — выполняет реальную работу. Подключается к master через SSH или JNLP
- **Jenkinsfile** — Pipeline as Code, хранится в репозитории рядом с кодом

**Declarative Pipeline (рекомендуется):**

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:26-dind
    securityContext:
      privileged: true
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['sleep', 'infinity']
"""
        }
    }

    environment {
        REGISTRY     = 'registry.example.com'
        IMAGE_NAME   = "${REGISTRY}/myapp"
        IMAGE_TAG    = "${IMAGE_NAME}:${GIT_COMMIT[0..7]}"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'go test ./... -race'
                    }
                    post {
                        always {
                            junit 'test-results/*.xml'
                        }
                    }
                }
                stage('Lint') {
                    steps {
                        sh 'golangci-lint run'
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'registry-creds',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS'
                    )]) {
                        sh '''
                            docker login -u $USER -p $PASS $REGISTRY
                            docker build -t $IMAGE_TAG .
                            docker push $IMAGE_TAG
                        '''
                    }
                }
            }
        }

        stage('Deploy Staging') {
            when {
                branch 'main'
            }
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        sh "kubectl set image deployment/myapp myapp=${IMAGE_TAG} -n staging"
                    }
                }
            }
        }

        stage('Deploy Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
            }
            steps {
                container('kubectl') {
                    sh "kubectl set image deployment/myapp myapp=${IMAGE_TAG} -n production"
                }
            }
        }
    }

    post {
        success {
            slackSend channel: '#deployments',
                      color: 'good',
                      message: "✅ Deploy ${env.JOB_NAME} #${env.BUILD_NUMBER} succeeded"
        }
        failure {
            slackSend channel: '#deployments',
                      color: 'danger',
                      message: "❌ Build ${env.JOB_NAME} #${env.BUILD_NUMBER} failed"
        }
    }
}
```

**Declarative vs Scripted:**

| | Declarative | Scripted |
|---|---|---|
| Синтаксис | Строгий, структурированный | Groovy DSL, свободный |
| Читаемость | Высокая | Ниже |
| Гибкость | Ограничена блоками | Полная (любой Groovy) |
| Валидация | Статическая проверка | Только в runtime |
| Рекомендуется | Всегда (90% случаев) | Когда нужна сложная логика |

---

## 6. Как реализовать стратегию ветвления и CI/CD для разных окружений (dev/staging/prod)?

**Git Flow vs GitHub Flow vs Trunk-Based:**

```
Git Flow (классика, сложный):
main ─────────────────────────────────── (только релизы)
  └─ develop ──────────────────────────── (интеграция)
        ├─ feature/auth ──── merge ──────
        ├─ feature/api  ──── merge ──────
        └─ release/1.2 ──── merge → main

GitHub Flow (проще):
main ────────────────────────────────── (всегда деплоябельна)
  ├─ feature/auth ──── PR → merge
  └─ feature/api  ──── PR → merge

Trunk-Based (CI-native, рекомендуется):
main ────────────────────────────────── (trunk, деплоится непрерывно)
  ├─ short-lived/fix-123 (< 1 день)
  └─ feature flags для незавершённых фич
```

**Маппинг веток → окружений:**

```yaml
# GitHub Actions пример
on:
  push:
    branches:
      - main        # → staging auto deploy
      - 'release/*' # → production manual deploy
  pull_request:
    branches:
      - main        # → preview environment (ephemeral)

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Determine environment
        id: env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
            echo "namespace=staging" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == refs/heads/release/* ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
            echo "namespace=production" >> $GITHUB_OUTPUT
          fi

      - name: Deploy
        run: |
          helm upgrade --install myapp ./helm \
            --namespace ${{ steps.env.outputs.namespace }} \
            --values ./helm/values.${{ steps.env.outputs.environment }}.yaml \
            --set image.tag=${{ github.sha }}
```

**Ephemeral preview environments (для PR):**

```yaml
deploy-preview:
  if: github.event_name == 'pull_request'
  steps:
    - name: Deploy preview
      run: |
        NAMESPACE="preview-pr-${{ github.event.number }}"
        kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
        helm upgrade --install myapp-pr-${{ github.event.number }} ./helm \
          --namespace $NAMESPACE \
          --set ingress.host=pr-${{ github.event.number }}.preview.example.com

    - name: Comment PR with preview URL
      uses: actions/github-script@v7
      with:
        script: |
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: '🚀 Preview deployed at: https://pr-${{ github.event.number }}.preview.example.com'
          })

cleanup-preview:
  if: github.event.action == 'closed'
  steps:
    - run: helm uninstall myapp-pr-${{ github.event.number }} -n preview-pr-${{ github.event.number }}
```

---

## 7. Как безопасно управлять секретами в CI/CD pipeline?

**Уровни секретов и где хранить:**

| Уровень | Инструмент | Когда использовать |
|---------|-----------|-------------------|
| Простые переменные | GitHub/GitLab Secrets | API ключи, токены, несложные секреты |
| Сложные секреты | HashiCorp Vault | Ротация, аудит, fine-grained access |
| Cloud секреты | AWS Secrets Manager / GCP Secret Manager | Натив в cloud-native окружениях |
| K8s секреты | External Secrets Operator | Синхронизация Vault/ASM → K8s Secret |

**Никогда не делай:**

```bash
# ПЛОХО: секрет в коде
DATABASE_URL=postgres://user:password@host/db

# ПЛОХО: секрет в Dockerfile
ENV DB_PASS=supersecret

# ПЛОХО: секрет в логах
echo "Connecting with password: $DB_PASS"

# ПЛОХО: передача через args (видно в docker history)
docker build --build-arg DB_PASS=secret .
```

**GitHub Actions — правильное использование секретов:**

```yaml
jobs:
  deploy:
    steps:
      # Передача секрета как переменная окружения
      - name: Run deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: ./deploy.sh

      # Лучший подход — OIDC (без долгоживущих ключей)
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-role
          aws-region: us-east-1
          # Никаких ключей! GitHub получает временный токен через OIDC
```

**OIDC — лучшая практика для cloud credentials:**

```
GitHub Actions                    AWS IAM
     │                               │
     │── OIDC Token ──────────────→  │
     │                    AssumeRole │
     │← Временные credentials (1h) ──│
     │                               │
     │── aws s3 cp ... ─────────────→│
```

```json
// IAM Trust Policy для GitHub Actions OIDC
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
      }
    }
  }]
}
```

**HashiCorp Vault в GitLab CI:**

```yaml
get-secrets:
  image: vault:latest
  id_tokens:
    VAULT_ID_TOKEN:
      aud: https://gitlab.com
  secrets:
    DB_PASSWORD:
      vault: myproject/data/db@secret  # путь в Vault
      file: false
  script:
    - echo "DB_PASSWORD is available as env var"
    - ./deploy.sh
```

---

## 8. Что такое GitOps и чем он отличается от классического CI/CD?

**Классический CI/CD (Push-based):**

```
Developer → Git Push → CI Pipeline → kubectl apply → Kubernetes
                                ↑
                      CI pipeline имеет доступ к кластеру
                      Состояние кластера неизвестно заранее
```

**GitOps (Pull-based):**

```
Developer → Git Push (configs) → Git Repo (single source of truth)
                                        ↑
                               ArgoCD/Flux (внутри кластера)
                               постоянно сравнивает desired vs actual
                               и применяет изменения сам
```

**Ключевые принципы GitOps (CNCF):**
1. **Declarative** — всё описано декларативно (Helm, Kustomize, plain YAML)
2. **Versioned and immutable** — Git — единственный источник истины
3. **Pulled automatically** — агент сам тянет изменения из Git
4. **Continuously reconciled** — система постоянно следит за соответствием

**Преимущества GitOps:**
- Audit trail — каждое изменение в Git = история кто/что/когда изменил
- Rollback = `git revert`
- CI pipeline не нужен доступ к кластеру (безопаснее)
- Drift Detection — если кто-то изменил кластер вручную, GitOps это обнаружит и откатит

---

## 9. ArgoCD: архитектура, App of Apps, Sync Policy, Health Status

**Архитектура ArgoCD:**

```
Kubernetes Cluster
  └── argocd namespace
        ├── argocd-server          (API + WebUI)
        ├── argocd-repo-server     (клонирует Git репо, рендерит Helm/Kustomize)
        ├── argocd-application-controller  (reconciliation loop)
        ├── argocd-dex-server      (SSO/OIDC)
        └── argocd-redis           (кэш)
```

**Application CRD — основной объект:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # cascade delete при удалении App
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/myapp-helm
    targetRevision: HEAD
    path: helm/myapp
    helm:
      valueFiles:
        - values.yaml
        - values.production.yaml
      parameters:
        - name: image.tag
          value: "abc1234"

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true      # удалять ресурсы, которых нет в Git
      selfHeal: true   # исправлять manual changes в кластере
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true  # применять только изменённые ресурсы
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**App of Apps pattern — управление множеством приложений:**

```yaml
# root-app.yaml — одно приложение, управляющее всеми остальными
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/gitops-config
    path: apps/  # здесь лежат yaml-файлы Application для каждого сервиса
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

# apps/myapp.yaml
# apps/database.yaml
# apps/monitoring.yaml
# ... и т.д.
```

**Health Status и Sync Status:**

```
Sync Status:
  Synced     — кластер соответствует Git
  OutOfSync  — есть отличия (новый коммит или manual change)

Health Status:
  Healthy    — все ресурсы работают корректно
  Progressing — Deployment rollout в процессе
  Degraded   — есть проблемы (CrashLoopBackOff, ImagePullError)
  Missing    — ресурс не существует в кластере
  Suspended  — CronJob suspended
```

**Полезные команды ArgoCD CLI:**

```bash
# Установка ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Логин
argocd login argocd.example.com --sso

# Создание приложения
argocd app create myapp \
  --repo https://github.com/myorg/myapp \
  --path helm \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production

# Ручная синхронизация
argocd app sync myapp

# Откат к предыдущей версии
argocd app rollback myapp 2

# Статус всех приложений
argocd app list

# Детальный diff что изменится
argocd app diff myapp
```

---

## 10. Как реализовать zero-downtime deployments? Blue/Green и Canary стратегии

**Rolling Update (Kubernetes default):**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # максимум на 1 Pod больше desired во время обновления
      maxUnavailable: 0  # 0 pods недоступных = zero-downtime
```

**Blue/Green Deployment:**

```
Blue (v1, active) ──→ Service (selector: color=blue) ──→ Users
Green (v2, idle)

После тестирования:
Blue (v1) ──────────────────────────────────────────── (standby)
Green (v2, active) ──→ Service (selector: color=green) ──→ Users
```

```yaml
# Переключение трафика меняет только selector в Service
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    color: green  # было blue, стало green — мгновенное переключение
  ports:
    - port: 80

# Скрипт переключения
kubectl patch service myapp -p '{"spec":{"selector":{"color":"green"}}}'
```

**Canary Deployment с Argo Rollouts:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10    # 10% трафика на новую версию
        - pause: {duration: 5m}  # ждём 5 минут, смотрим на метрики
        - setWeight: 30
        - pause: {duration: 10m}
        - setWeight: 60
        - pause: {}        # пауза до ручного подтверждения
        - setWeight: 100
      canaryMetadata:
        labels:
          role: canary
      stableMetadata:
        labels:
          role: stable
      # Автоматический анализ метрик для продвижения/отката
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1
        args:
          - name: service-name
            value: myapp-canary

---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.95  # 95% успешных запросов
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

**Feature Flags — альтернативный подход:**

```python
# Gradual rollout без нескольких deployment
import launchdarkly_api

def show_new_feature(user_id: str) -> bool:
    # LaunchDarkly / Unleash / Flagsmith
    return ld_client.variation("new-checkout-flow", user_id, False)
```

---

## 11. Как ускорить CI/CD pipeline? Кэширование, параллелизм, incremental builds

**Главные принципы оптимизации:**

1. **Параллелизм** — всё что можно — параллельно
2. **Кэш** — зависимости, Docker layers, test results
3. **Incremental builds** — собирать только изменённое
4. **Fail fast** — самые быстрые и важные проверки — первыми

**Docker layer caching:**

```dockerfile
# Плохо — npm install на каждый коммит
COPY . .
RUN npm ci

# Хорошо — зависимости кэшируются пока не изменится package-lock.json
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
```

**GitHub Actions cache:**

```yaml
- name: Cache Go modules
  uses: actions/cache@v4
  with:
    path: |
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-

- name: Cache node_modules
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

# Docker BuildKit cache
- name: Build with cache
  uses: docker/build-push-action@v6
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**Параллельные jobs:**

```yaml
jobs:
  # Эти jobs запускаются параллельно
  unit-tests:
    runs-on: ubuntu-latest
    steps: [...]

  lint:
    runs-on: ubuntu-latest
    steps: [...]

  security-scan:
    runs-on: ubuntu-latest
    steps: [...]

  # Этот job ждёт всех трёх
  build:
    needs: [unit-tests, lint, security-scan]
    steps: [...]
```

**Path filtering — запускать pipeline только если изменились нужные файлы:**

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'Dockerfile'
      - 'go.mod'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

**Разделение pipeline на fast/slow:**

```yaml
# fast-checks.yml — запускается на каждый push (< 2 мин)
jobs:
  lint:   ...  # 30s
  unit:   ...  # 60s

# full-pipeline.yml — запускается только на main или перед merge
jobs:
  integration:   ...  # 5 min
  e2e:           ...  # 10 min
  security:      ...  # 3 min
```

---

## 12. Как тестировать инфраструктуру в CI/CD? IaC-тесты, policy-as-code

**Уровни тестирования IaC:**

```
1. Static Analysis (быстро, без ресурсов)
   └── terraform validate, tflint, checkov, tfsec, terrascan

2. Unit Tests (моки, без реальных ресурсов)
   └── Terratest (unit mode), conftest (OPA policy tests)

3. Integration Tests (реальные ресурсы в test-аккаунте)
   └── Terratest, Kitchen-Terraform

4. Compliance Tests (аудит существующей инфраструктуры)
   └── InSpec, Cloud Custodian
```

**Terraform в CI/CD:**

```yaml
# GitLab CI для Terraform
stages:
  - validate
  - plan
  - apply

terraform-validate:
  stage: validate
  image: hashicorp/terraform:1.10
  script:
    - terraform init -backend=false
    - terraform validate
    - terraform fmt -check=true -recursive

tflint:
  stage: validate
  image: ghcr.io/terraform-linters/tflint:latest
  script:
    - tflint --init
    - tflint --recursive

checkov-scan:
  stage: validate
  image: bridgecrew/checkov:latest
  script:
    - checkov -d . --framework terraform
        --skip-check CKV_AWS_20  # пример исключения конкретной проверки
  allow_failure: false

terraform-plan:
  stage: plan
  image: hashicorp/terraform:1.10
  script:
    - terraform init
    - terraform plan -out=tfplan -no-color | tee plan.txt
  artifacts:
    paths:
      - tfplan
      - plan.txt
    expire_in: 1 day

terraform-apply:
  stage: apply
  image: hashicorp/terraform:1.10
  needs: [terraform-plan]
  when: manual  # только вручную для production
  script:
    - terraform init
    - terraform apply tfplan
```

**OPA/Conftest — Policy as Code:**

```rego
# policy/terraform.rego
package terraform

# Запрещаем публичные S3 бакеты
deny[msg] {
    resource := input.resource.aws_s3_bucket[_]
    resource.acl == "public-read"
    msg := sprintf("S3 bucket '%v' must not be public", [resource])
}

# Требуем теги на все EC2 инстансы
deny[msg] {
    resource := input.resource.aws_instance[name]
    not resource.tags.Environment
    msg := sprintf("EC2 instance '%v' must have Environment tag", [name])
}
```

```bash
# Применение policy к terraform plan
terraform show -json tfplan > plan.json
conftest test plan.json --policy policy/
```

---

## 13. Что такое trunk-based development и как он связан с CI?

**Trunk-Based Development (TBD)** — модель, при которой все разработчики коммитят напрямую в одну ветку (`main`/`trunk`) или создают **очень короткоживущие** ветки (не дольше 1-2 дней).

**Сравнение с Git Flow:**

```
Git Flow (долгоживущие ветки):
main        ────────────────────────────────────────────── → v1.0
develop     ────────────────────────────────────────────→
feature/a   ──────────────────────────────→ (2 недели!)
feature/b             ──────────────────→ (3 недели!)
Проблема: merge hell, конфликты, сложная интеграция

TBD (короткоживущие ветки):
main        ────────────────────────────────────────────── (всегда деплоябельна)
feat/a      ──→ (< 1 дня, сразу PR+merge)
feat/b          ──→ (< 1 дня)
feat/c              ──→
```

**Ключевые практики TBD:**

1. **Feature Flags** — незавершённые фичи скрыты за флагами, но код уже в trunk
2. **Branch by Abstraction** — рефакторинг больших изменений через промежуточный абстрактный слой
3. **Expand-Contract (Parallel Change)** — сначала добавляем новое, потом удаляем старое

**Связь с CI:**

```
TBD + CI = настоящая непрерывная интеграция:

Каждый коммит в trunk:
  1. Запускает CI (< 10 минут до результата)
  2. Автоматически деплоится в staging
  3. Trunk всегда в деплоябельном состоянии

Метрика: DORA metrics
  - Deployment Frequency: > 1 раз в день (elite: несколько раз в день)
  - Lead Time for Changes: < 1 часа
  - Change Failure Rate: < 5%
  - MTTR (Mean Time to Restore): < 1 часа
```

---

## 14. Как откатить деплой? Rollback стратегии в Kubernetes и ArgoCD

**Kubernetes встроенный rollback:**

```bash
# Просмотр истории Deployment
kubectl rollout history deployment/myapp -n production

# Откат к предыдущей версии
kubectl rollout undo deployment/myapp -n production

# Откат к конкретной ревизии
kubectl rollout undo deployment/myapp --to-revision=3 -n production

# Статус rollback
kubectl rollout status deployment/myapp -n production

# Аннотация для human-readable истории
kubectl annotate deployment/myapp kubernetes.io/change-cause="Deploying v1.2.3"
```

**Helm rollback:**

```bash
# История релизов
helm history myapp -n production

# Откат к предыдущей версии
helm rollback myapp -n production

# Откат к конкретной ревизии
helm rollback myapp 5 -n production

# Автоматический rollback если upgrade failed
helm upgrade myapp ./chart \
  --atomic \          # откатывает если деплой не стал Healthy в timeout
  --timeout 5m \
  --cleanup-on-fail
```

**ArgoCD rollback:**

```bash
# Откат через CLI
argocd app rollback myapp

# Откат к конкретному Git-коммиту
argocd app sync myapp --revision abc1234

# Disable auto-sync перед ручным откатом
argocd app set myapp --sync-policy none
kubectl set image deployment/myapp myapp=registry/myapp:v1.1.0 -n production
```

**GitOps-style rollback (рекомендуется):**

```bash
# Откат = git revert
git revert abc1234 -m "Revert: broken feature X"
git push origin main
# ArgoCD автоматически применит откат
```

**Автоматический rollback по метрикам (Argo Rollouts):**

```yaml
# Если error rate вырастает — автоматический откат
analysis:
  templates:
    - templateName: error-rate
  args:
    - name: service
      value: myapp
# Если AnalysisRun fails → Rollout автоматически откатывается к stable
```

---

## 15. Self-hosted runners vs Cloud runners: когда что использовать?

**Cloud (GitHub-hosted / GitLab SaaS) Runners:**

```
Плюсы:
  + Нет инфраструктуры — не нужно настраивать и поддерживать
  + Чистая среда на каждый job (свежий VM)
  + Встроенные инструменты (Docker, Node, Python, Go предустановлены)
  + Масштабируются автоматически
  + GitHub Free: 2000 минут/месяц

Минусы:
  - Дороже при больших объёмах ($0.008/min для Linux)
  - Медленный старт (30-60 секунд на инициализацию VM)
  - Нет доступа к внутренней сети компании
  - Ограниченные ресурсы (GitHub: 2 CPU, 7GB RAM)
  - Данные проходят через облако провайдера CI
```

**Self-hosted Runners:**

```
Плюсы:
  + Дешевле при высоких объёмах
  + Быстрый старт (нет VM инициализации)
  + Доступ к внутренним ресурсам (private registry, DB, K8s)
  + Больше ресурсов (нет ограничений GitHub)
  + Данные не уходят из вашей инфраструктуры (compliance)
  + GPU доступ для ML workloads

Минусы:
  - Нужно поддерживать (обновления, безопасность)
  - Риск persistency-атак (env vars, credentials между jobs)
  - Нужна автоматизация масштабирования
```

**Self-hosted runner в Kubernetes (Actions Runner Controller):**

```yaml
# ARC (Actions Runner Controller) — автоскейлинг на K8s
apiVersion: actions.github.com/v1alpha1
kind: AutoscalingRunnerSet
metadata:
  name: arc-runner-set
  namespace: arc-systems
spec:
  githubConfigUrl: https://github.com/myorg/myrepo
  githubConfigSecret: arc-token
  minRunners: 0
  maxRunners: 20
  template:
    spec:
      containers:
        - name: runner
          image: ghcr.io/actions/actions-runner:latest
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
            limits:
              cpu: "4"
              memory: "8Gi"
```

**Решение: гибридный подход:**

| Задача | Тип runner |
|--------|-----------|
| Open Source проекты | Cloud (бесплатно) |
| Маленькие команды | Cloud |
| Доступ к private сети | Self-hosted |
| Compliance / data residency | Self-hosted |
| GPU / ML builds | Self-hosted |
| Большие объёмы (> 10к мин/мес) | Self-hosted (дешевле) |
| Быстрый старт + изоляция | Cloud |
