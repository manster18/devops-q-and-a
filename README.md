# DevOps — Вопросы и ответы для собеседований

Структурированная коллекция вопросов и ответов для **Middle/Senior DevOps инженеров**.

Каждая технология имеет свою директорию с Markdown файлом. В начале каждого файла — список вопросов, каждый из которых является ссылкой, ведущей прямо к ответу в этом же файле.

---

## Темы

| Технология | Файл |
|---|---|
| Linux (процессы, ФС, сеть, производительность, systemd) | [linux/linux.md](linux/linux.md) |
| Web (HTTP/HTTPS, Nginx, балансировка, DNS, SSL/TLS, безопасность) | [web/web.md](web/web.md) |
| Docker (архитектура, Dockerfile, сеть, volumes, безопасность, Compose) | [docker/docker.md](docker/docker.md) |
| Kubernetes (архитектура, workloads, сеть, хранилище, безопасность, HPA, Helm) | [kubernetes/kubernetes.md](kubernetes/kubernetes.md) |
| AWS (IAM, VPC, EC2, S3, RDS, EKS, CloudWatch, Route53, KMS, оптимизация затрат) | [aws/aws.md](aws/aws.md) |
| IaC: Terraform + Ansible (state, модули, роли, Vault, Jinja2, оптимизация) | [iac/iac.md](iac/iac.md) |
| Базы данных (SQL, NoSQL, HA, репликация, Redis, MongoDB, Elasticsearch) | [databases/databases.md](databases/databases.md) |
| Docker builds (Go, Node.js, Python, Java, .NET, PHP, Frontend, Rust) | [docker-builds/docker-builds.md](docker-builds/docker-builds.md) |
| CI/CD (GitHub Actions, GitLab CI, Jenkins, ArgoCD/GitOps, Blue-Green, Canary) | [cicd/cicd.md](cicd/cicd.md) |
| Мониторинг и Observability (Prometheus, Grafana, ELK/EFK, Loki, OpenTelemetry, Jaeger) | [monitoring/monitoring.md](monitoring/monitoring.md) |
| SRE (SLI/SLO/SLA, Error Budget, Toil, Incident Management, Postmortem, DORA) | [sre/sre.md](sre/sre.md) |
| Сети / Networking (TCP/IP, BGP, VPN, eBPF, Service Mesh, Istio, CNI, NetworkPolicy) | [networking/networking.md](networking/networking.md) |
| DevSecOps / Security (Vault, OPA/Gatekeeper, Falco, SAST/DAST, SBOM, Cosign, Zero Trust) | [devsecops/devsecops.md](devsecops/devsecops.md) |
| Kafka (архитектура, репликация, exactly-once, Connect, Streams, тюнинг, K8s) | [kafka/kafka.md](kafka/kafka.md) |
| GitOps и ArgoCD (принципы, ApplicationSet, Flux, multi-cluster, структура репо) | [gitops/gitops.md](gitops/gitops.md) |

---

## Структура репозитория

```
devops-q-and-a/
├── README.md
├── docker/
│   └── docker.md
├── kubernetes/
│   └── kubernetes.md
├── terraform/
│   └── terraform.md
└── ...
```

---

## Уровень сложности

Вопросы рассчитаны на **Middle** и **Senior** DevOps инженеров.
Ответы содержат объяснения, лучшие практики и практические примеры.
