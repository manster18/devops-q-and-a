# Сети / Networking: Вопросы и ответы для DevOps-инженера (Middle/Senior)

## Содержание

1. [Модель OSI и TCP/IP — как данные путешествуют по сети?](#1-модель-osi-и-tcpip--как-данные-путешествуют-по-сети)
2. [TCP vs UDP: разница, handshake, flow control, congestion control](#2-tcp-vs-udp-разница-handshake-flow-control-congestion-control)
3. [Как работает маршрутизация? BGP basics для DevOps](#3-как-работает-маршрутизация-bgp-basics-для-devops)
4. [VPN: IPSec, WireGuard, OpenVPN — как работают и когда что выбрать?](#4-vpn-ipsec-wireguard-openvpn--как-работают-и-когда-что-выбрать)
5. [NAT, PAT и их роль в облачных и корпоративных сетях](#5-nat-pat-и-их-роль-в-облачных-и-корпоративных-сетях)
6. [eBPF: что это, как используется в networking и observability?](#6-ebpf-что-это-как-используется-в-networking-и-observability)
7. [Service Mesh: зачем нужен, Istio vs Linkerd vs Cilium?](#7-service-mesh-зачем-нужен-istio-vs-linkerd-vs-cilium)
8. [Istio: архитектура, Traffic Management, mTLS, Observability](#8-istio-архитектура-traffic-management-mtls-observability)
9. [Kubernetes Networking: CNI, Pod CIDR, Service CIDR, kube-proxy](#9-kubernetes-networking-cni-pod-cidr-service-cidr-kube-proxy)
10. [Network troubleshooting: инструменты и методология](#10-network-troubleshooting-инструменты-и-методология)
11. [Firewalls и Network Policy: iptables, nftables, K8s NetworkPolicy](#11-firewalls-и-network-policy-iptables-nftables-k8s-networkpolicy)
12. [IPv6: зачем нужен, dual-stack в Kubernetes и AWS?](#12-ipv6-зачем-нужен-dual-stack-в-kubernetes-и-aws)

---

## 1. Модель OSI и TCP/IP — как данные путешествуют по сети?

**Модель OSI (7 уровней):**

```
L7 Application  HTTP, gRPC, DNS, SMTP
L6 Presentation TLS/SSL, шифрование, сжатие
L5 Session      управление сессиями
L4 Transport    TCP, UDP — порты, сегментация
L3 Network      IP, ICMP, маршрутизация
L2 Data Link    Ethernet, MAC-адреса, VLAN
L1 Physical     кабели, сигналы, Wi-Fi
```

**Практическая TCP/IP модель (4 уровня):**

```
Application  L7+L6+L5 → HTTP, DNS, TLS
Transport    L4        → TCP, UDP
Internet     L3        → IP, ICMP
Link         L2+L1     → Ethernet, Wi-Fi
```

**Как данные путешествуют (инкапсуляция):**

```
Отправитель (curl https://example.com):
  Application: HTTP GET /  + TLS
  Transport:   TCP сегмент [src:54321 dst:443] + payload
  Network:     IP пакет [src:1.2.3.4 dst:93.184.216.34] + TCP сегмент
  Link:        Ethernet фрейм [src MAC: AA:BB dst MAC: router] + IP пакет

На каждом роутере:
  Ethernet фрейм разбирается (L2)
  IP пакет читается → принимается решение о маршруте (L3)
  Новый Ethernet фрейм с MAC следующего хопа

Получатель:
  Раскапсулирует в обратном порядке:
  Ethernet → IP → TCP → HTTP/TLS
```

**Почему это важно для DevOps:**

```
L4 Load Balancer (AWS NLB): работает с TCP/UDP
  + Быстрее (нет разбора HTTP)
  + Поддерживает любой TCP-протокол
  - Нет возможности роутить по HTTP headers

L7 Load Balancer (AWS ALB, Nginx): работает с HTTP
  + Роутинг по path, headers, method
  + SSL termination
  + Health checks на уровне приложения
  - Медленнее (разбирает каждый запрос)
```

---

## 2. TCP vs UDP: разница, handshake, flow control, congestion control

**Сравнение:**

| Характеристика | TCP | UDP |
|---------------|-----|-----|
| Соединение | Connection-oriented (3-way handshake) | Connectionless |
| Надёжность | Гарантированная доставка (ACK, retransmit) | Best-effort |
| Порядок | Гарантирован | Не гарантирован |
| Скорость | Медленнее (overhead) | Быстрее |
| Где используется | HTTP, SSH, SMTP, databases | DNS, UDP, video streaming, gaming |

**TCP 3-Way Handshake:**

```
Client                              Server
  │──── SYN (seq=100) ────────────→  │  (1) Клиент инициирует
  │                                  │
  │←─── SYN-ACK (seq=200, ack=101) ──│  (2) Сервер подтверждает
  │                                  │
  │──── ACK (ack=201) ──────────────→│  (3) Клиент подтверждает
  │                                  │
  │══════ Данные ════════════════════│  Соединение установлено

Закрытие (4-way):
  │──── FIN ────────────────────────→│
  │←─── ACK ─────────────────────────│
  │←─── FIN ─────────────────────────│
  │──── ACK ────────────────────────→│
```

**TIME_WAIT и почему это важно для DevOps:**

```bash
# TIME_WAIT: после закрытия соединения клиент ждёт 2*MSL (60-120 сек)
# Проблема: при высоком RPS заканчиваются ephemeral ports

# Посмотреть количество TIME_WAIT
ss -s
# или
netstat -an | grep TIME_WAIT | wc -l

# Решения:
# 1. Включить tcp_tw_reuse (переиспользовать TIME_WAIT сокеты)
sysctl -w net.ipv4.tcp_tw_reuse=1

# 2. Keep-Alive (избегать частого закрытия соединений)
# 3. Увеличить диапазон портов
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```

**Flow Control (управление потоком):**

```
Получатель сообщает отправителю сколько данных может принять (window size).
  Receive Window = свободное место в буфере получателя
  
  Если буфер заполнен → Window Size = 0 → отправитель останавливается

tcp_rmem и tcp_wmem — размеры буферов (важно для высоконагруженных систем):
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
```

**Congestion Control (управление перегрузкой):**

```
CUBIC (Linux default):
  Медленный старт → Congestion Avoidance → Fast Retransmit/Recovery
  
BBR (Google, рекомендуется для cloud):
  Моделирует пропускную способность сети
  Лучше для высоколатентных или lossy сетей (datacenter → CDN)
  
# Включить BBR
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

---

## 3. Как работает маршрутизация? BGP basics для DevOps

**Таблица маршрутизации:**

```bash
# Просмотр таблицы маршрутов
ip route show

# Типичный вывод:
default via 10.0.0.1 dev eth0 proto dhcp       # default gateway
10.0.0.0/24 dev eth0 proto kernel scope link    # локальная сеть
172.16.0.0/12 via 10.0.0.254 dev eth0           # VPN/private сеть

# Трассировка маршрута
traceroute google.com
# или более информативно:
mtr --report google.com
```

**BGP (Border Gateway Protocol) — протокол маршрутизации интернета:**

```
BGP — протокол обмена маршрутами между автономными системами (AS).
Каждый ISP, крупная компания имеет свой AS номер (ASN).

eBGP (external): между разными AS (между ISP)
iBGP (internal): внутри одной AS

Для DevOps BGP важен в контексте:
  1. Anycast (Cloudflare, AWS Route53 — один IP, много точек присутствия)
  2. AWS Direct Connect / Azure ExpressRoute
  3. Kubernetes с Cilium (BGP для анонса Service IPs)
  4. MetalLB (BGP mode для bare-metal K8s LoadBalancer)
```

**BGP в Kubernetes (MetalLB):**

```yaml
# MetalLB с BGP для bare-metal Kubernetes
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: router
  namespace: metallb-system
spec:
  myASN: 64512       # AS кластера
  peerASN: 64510     # AS роутера
  peerAddress: 192.168.1.1
  keepaliveTime: 30s

---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.2.0/24  # диапазон для LoadBalancer IP

---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: main
  namespace: metallb-system
# Объявляем маршруты через BGP роутеру
```

**ECMP (Equal-Cost Multi-Path):**

```bash
# Балансировка трафика по нескольким маршрутам с одинаковой метрикой
ip route add 10.0.0.0/8 \
  nexthop via 192.168.1.1 weight 1 \
  nexthop via 192.168.2.1 weight 1

# Используется в Kubernetes для балансировки трафика к подам
# через kube-proxy или Cilium
```

---

## 4. VPN: IPSec, WireGuard, OpenVPN — как работают и когда что выбрать?

**Сравнение VPN протоколов:**

| Характеристика | OpenVPN | IPSec/IKEv2 | WireGuard |
|---------------|---------|-------------|-----------|
| Скорость | Средняя | Высокая | Очень высокая |
| Простота настройки | Средняя | Сложная | Простая |
| Кодовая база | ~100k строк | Сложная | ~4k строк |
| Порт | TCP/UDP 1194 | UDP 500, 4500 | UDP (любой) |
| NAT traversal | Отлично | Средне | Отлично |
| Аудит безопасности | Хороший | Сложный | Отличный |
| Когда использовать | Legacy, совместимость | Корпоративные роутеры, iOS/macOS | Новые проекты, site-to-site |

**WireGuard — принцип работы:**

```
WireGuard использует современную криптографию:
  - Curve25519 для обмена ключами
  - ChaCha20 для симметричного шифрования
  - Poly1305 для аутентификации
  - BLAKE2s для хеширования
  
Модель: каждый пир = публичный ключ + список allowed IPs
```

```bash
# Настройка WireGuard server
# /etc/wireguard/wg0.conf

[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

# NAT для трафика через VPN
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; \
         iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; \
           iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]  # клиент 1
PublicKey = CLIENT1_PUBLIC_KEY
AllowedIPs = 10.10.0.2/32  # только этот клиент с этим IP

[Peer]  # клиент 2
PublicKey = CLIENT2_PUBLIC_KEY
AllowedIPs = 10.10.0.3/32
```

```bash
# Клиент
# /etc/wireguard/wg0.conf
[Interface]
Address = 10.10.0.2/24
PrivateKey = CLIENT_PRIVATE_KEY
DNS = 1.1.1.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = server.example.com:51820
AllowedIPs = 0.0.0.0/0  # весь трафик через VPN
             # или 10.0.0.0/8 — только корпоративная сеть (split tunneling)
PersistentKeepalive = 25  # NAT keepalive

# Управление
wg-quick up wg0
wg-quick down wg0
wg show  # статус
```

**IPSec в AWS (Site-to-Site VPN):**

```
AWS Side:
  Virtual Private Gateway (VGW) или Transit Gateway
  Two tunnels (HA) → BGP или static routing

On-premise Side:
  Customer Gateway (CGW) — ваш роутер/firewall
  Поддерживает: Cisco ASA, Palo Alto, pfSense, strongSwan

Пропускная способность AWS VPN: до 1.25 Gbps per tunnel
```

---

## 5. NAT, PAT и их роль в облачных и корпоративных сетях

**NAT (Network Address Translation):**

```
Static NAT: 1:1 маппинг (один private IP → один public IP)
Dynamic NAT: пул private IP → пул public IP
PAT (Port Address Translation) / NAT Overload:
  Много private IP → один public IP, различаются по портам
  
  192.168.1.10:54321 → 1.2.3.4:54321
  192.168.1.11:54322 → 1.2.3.4:54322
  
  Это то, что используют домашние роутеры и AWS NAT Gateway
```

**NAT в AWS:**

```
Internet Gateway (IGW):
  Public подсеть → instance имеет Public IP → IGW напрямую
  Нет NAT — трафик идёт 1:1 через Elastic IP

NAT Gateway (для приватных подсетей):
  Private subnet → NAT Gateway (в public subnet) → IGW → Internet
  NAT GW имеет один Elastic IP
  Bandwidth: до 100 Gbps
  
  Проблема NAT Gateway: дорогой ($0.045/час + $0.045/GB)
  Оптимизация: один NAT GW per AZ (иначе Cross-AZ traffic charges)
```

```
                Internet
                   │
              [IGW]
                   │
    ┌──────────────┼──────────────┐
    │         Public Subnet       │
    │    [NAT Gateway: 1.2.3.4]  │
    └──────────────┼──────────────┘
                   │
    ┌──────────────┼──────────────┐
    │        Private Subnet       │
    │    [App servers: 10.0.1.x]  │
    └─────────────────────────────┘
```

**Проблемы с NAT в Kubernetes:**

```bash
# По умолчанию kube-proxy делает SNAT для Pod→External трафика
# Pod IP (10.0.1.5) → Node IP (192.168.1.10) → External

# Проблема: теряется оригинальный IP источника
# Решение для ingress: externalTrafficPolicy: Local
apiVersion: v1
kind: Service
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # сохраняет source IP, но нет балансировки между нодами
```

---

## 6. eBPF: что это, как используется в networking и observability?

**eBPF (extended Berkeley Packet Filter)** — технология ядра Linux, позволяющая запускать песочницу кода в ядре без изменения исходников ядра и без загружаемых модулей.

```
Традиционный подход:
  Изменить ядро → пересобрать → перезагрузить → риски стабильности

eBPF:
  Пишешь eBPF программу → JIT компилируется → верифицируется (безопасность)
  → загружается в ядро → выполняется при событии (network packet, syscall, probe)
  
  Безопасность: верификатор проверяет что программа не может повредить ядро
  Производительность: JIT компиляция = почти нативная скорость
```

**Применения eBPF в DevOps:**

```
Networking (Cilium):
  Заменяет kube-proxy (iptables) на eBPF maps
  В 10-100x быстрее для крупных кластеров (kube-proxy масштабируется O(n), eBPF O(1))
  
Security (Falco, Tetragon):
  Мониторинг syscalls без overhead агента
  Обнаружение аномального поведения в реальном времени

Observability (Pixie, Hubble):
  Zero-instrumentation трейсинг приложений
  Сбор метрик прямо из ядра без изменения кода приложения
  
Performance:
  XDP (eXpress Data Path) — обработка пакетов до стека ядра
  DDoS mitigation на скорости 10M+ packets/sec
```

**Cilium — eBPF-based CNI для Kubernetes:**

```yaml
# Установка Cilium как замены kube-proxy
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=API_SERVER_IP \
  --set k8sServicePort=6443 \
  --set hubble.relay.enabled=true \     # observability
  --set hubble.ui.enabled=true          # UI для трафика

# Cilium NetworkPolicy (расширенная, поверх K8s NetworkPolicy)
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-policy
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:                        # L7 HTTP правила
              - method: GET
                path: "/api/v1/.*"       # только GET к /api/v1/
```

**Hubble — observability поверх Cilium:**

```bash
# Просмотр трафика в реальном времени
hubble observe --namespace production

# Трафик конкретного пода
hubble observe --pod myapp-xxx-yyy --follow

# Статистика соединений
hubble observe --verdict DROPPED -n production

# UI
cilium hubble ui  # открывает service map
```

**BCC/bpftrace — инструменты отладки через eBPF:**

```bash
# Смотреть все открытые TCP соединения
bpftrace -e 'kprobe:tcp_connect { printf("%s → %s\n", 
  comm, 
  ntop(((struct sock *)arg0)->__sk_common.skc_daddr)) }'

# Latency всех syscalls для процесса
bpftrace -e 'tracepoint:syscalls:sys_enter_* /pid == 1234/ {
  @start[tid] = nsecs;
}
tracepoint:syscalls:sys_exit_* /pid == 1234 && @start[tid]/ {
  @ns[probe] = hist(nsecs - @start[tid]);
  delete(@start[tid]);
}'

# Готовые инструменты BCC:
execsnoop    # отслеживать fork/exec
tcptop       # top по TCP трафику
biolatency   # disk I/O latency гистограмма
profile      # CPU profiling (flame graphs)
```

---

## 7. Service Mesh: зачем нужен, Istio vs Linkerd vs Cilium?

**Проблемы без Service Mesh:**

```
Microservices communication challenges:
  ✗ mTLS между сервисами — каждая команда реализует сама
  ✗ Retry, timeout, circuit breaker — дублирование в каждом сервисе
  ✗ Distributed tracing — нужна инструментация в каждом сервисе
  ✗ Traffic management — canary, A/B testing, mirroring
  ✗ Visibility — кто с кем разговаривает?

Service Mesh решает:
  ✓ mTLS автоматически (zero-trust networking)
  ✓ Retry/timeout/CB на уровне платформы
  ✓ Трейсинг без изменений кода
  ✓ Traffic splitting, canary, fault injection
  ✓ Service topology map
```

**Sidecar vs eBPF proxy:**

```
Sidecar модель (Istio classic, Linkerd):
  Каждый Pod + Envoy/linkerd-proxy sidecar
  Трафик проксируется через sidecar
  Плюс: изоляция, богатые L7 функции
  Минус: +resource overhead на каждый pod, latency

Ambient Mesh (Istio Ambient, Cilium):
  Нет sidecar — eBPF/node-level proxy
  Плюс: нет overhead, прозрачно
  Минус: менее зрелое, ограниченные L7 функции
```

**Сравнение:**

| Параметр | Istio | Linkerd | Cilium Service Mesh |
|----------|-------|---------|---------------------|
| Прокси | Envoy (тяжёлый) | linkerd-proxy (Rust, лёгкий) | eBPF (без sidecar) |
| Overhead | ~50MB RAM/pod | ~10MB RAM/pod | Минимальный |
| L7 поддержка | Полная | HTTP/gRPC/TCP | HTTP (растёт) |
| Сложность | Высокая | Средняя | Средняя |
| Зрелость | Высокая | Высокая | Растёт |
| Лучший для | Большие кластеры, rich features | Простота, low overhead | eBPF-native |

---

## 8. Istio: архитектура, Traffic Management, mTLS, Observability

**Архитектура Istio:**

```
Control Plane:
  istiod:
    Pilot     → конфигурирует Envoy прокси (xDS API)
    Citadel   → Certificate Authority (mTLS сертификаты)
    Galley    → валидация конфигурации

Data Plane:
  Envoy sidecar в каждом Pod
  Перехватывает весь inbound/outbound трафик через iptables rules
```

**Traffic Management:**

```yaml
# VirtualService — правила маршрутизации
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    # Canary: 90% на v1, 10% на v2
    - match:
        - headers:
            x-canary-user:
              exact: "true"         # Canary для конкретных пользователей
      route:
        - destination:
            host: myapp
            subset: v2
    - route:
        - destination:
            host: myapp
            subset: v1
          weight: 90
        - destination:
            host: myapp
            subset: v2
          weight: 10

    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,retriable-4xx

    # Timeout
    timeout: 10s

    # Fault injection (для testing)
    fault:
      delay:
        percentage:
          value: 5.0    # 5% запросов будут задержаны
        fixedDelay: 5s
      abort:
        percentage:
          value: 1.0    # 1% запросов вернут 500
        httpStatus: 500

---
# DestinationRule — load balancing, circuit breaker, subsets
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
    outlierDetection:          # Circuit Breaker
      consecutiveGatewayErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    loadBalancer:
      simple: LEAST_CONN      # ROUND_ROBIN, RANDOM, LEAST_CONN
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
      trafficPolicy:
        connectionPool:
          http:
            http1MaxPendingRequests: 10   # bulkhead для v2
```

**mTLS:**

```yaml
# Включить mTLS для всего namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # только mTLS (PERMISSIVE = и plain text и mTLS)

# AuthorizationPolicy — кто может с кем разговаривать
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: frontend-to-api
  namespace: production
spec:
  selector:
    matchLabels:
      app: api
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/frontend  # только frontend ServiceAccount
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/*"]
```

---

## 9. Kubernetes Networking: CNI, Pod CIDR, Service CIDR, kube-proxy

**Сетевая модель Kubernetes:**

```
Требования K8s:
  1. Каждый Pod имеет уникальный IP
  2. Pod может общаться с любым другим Pod без NAT
  3. Node может общаться с любым Pod без NAT
  4. IP который Pod видит у себя = IP который другие видят у него

Pod CIDR:   10.244.0.0/16  (для pod IP, назначается CNI)
Service CIDR: 10.96.0.0/12  (для ClusterIP Service, виртуальные)
Node CIDR:  192.168.1.0/24 (реальные ноды)
```

**CNI (Container Network Interface):**

```
Популярные CNI:
  Flannel    — простой, VXLAN overlay, хорош для dev
  Calico     — BGP routing, NetworkPolicy, production-grade
  Cilium     — eBPF-based, L7 NetworkPolicy, Service Mesh
  Weave      — простой, gossip protocol
  AWS VPC CNI — нативно интегрируется с AWS VPC (pod IP = VPC IP)

Как работает CNI (при создании Pod):
  1. kubelet создаёт network namespace для Pod
  2. kubelet вызывает CNI plugin
  3. CNI плагин создаёт veth pair (один конец в Pod, другой на Node)
  4. Назначает IP из Pod CIDR
  5. Настраивает маршруты
```

**Типы Service и как они работают:**

```
ClusterIP (default):
  Виртуальный IP, доступен только внутри кластера
  kube-proxy → iptables DNAT rules: ClusterIP:port → random PodIP:port
  
  iptables -t nat -L KUBE-SERVICES | grep myapp
  → -d 10.96.1.100/32 -p tcp --dport 80 → KUBE-SVC-xxx
  → KUBE-SVC-xxx → вероятностный выбор пода

NodePort:
  ClusterIP + открытый порт на каждой ноде (30000-32767)
  Внешний трафик → NodeIP:30080 → ClusterIP → Pod

LoadBalancer:
  NodePort + Cloud Load Balancer (AWS ELB, GCP LB)
  Автоматически создаётся cloud LB через cloud-controller-manager

Headless Service (clusterIP: None):
  Нет ClusterIP, DNS возвращает Pod IPs напрямую
  Используется для StatefulSets (kafka, postgres)
  
  DNS: myapp.default.svc.cluster.local → [10.244.1.5, 10.244.2.3, ...]
```

**DNS в Kubernetes:**

```bash
# CoreDNS — DNS сервер кластера
# Каждый Pod: /etc/resolv.conf:
#   nameserver 10.96.0.10  (ClusterIP CoreDNS)
#   search default.svc.cluster.local svc.cluster.local cluster.local

# DNS записи
# Service: <service>.<namespace>.svc.cluster.local
# Pod:     <pod-ip-dashed>.<namespace>.pod.cluster.local
# StatefulSet Pod: <pod-name>.<service>.<namespace>.svc.cluster.local

# Отладка DNS
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes
kubectl exec -n production myapp-xxx -- nslookup mydb.production.svc.cluster.local
```

**kube-proxy modes:**

```
iptables (default):
  Плюс: стабильный, хорошо изучен
  Минус: O(n) сложность, проблемы при 10k+ services
  
ipvs:
  Плюс: O(1) для маршрутизации (hash table), больше алгоритмов LB
  Минус: требует ipvs kernel modules
  
eBPF (Cilium kube-proxy replacement):
  Плюс: самый быстрый, O(1), rich features
  Минус: нужен Cilium CNI
```

---

## 10. Network troubleshooting: инструменты и методология

**Методология (от L1 к L7):**

```
1. Проверь физический уровень:
   ping  (ICMP — работает ли IP уровень?)
   
2. Проверь маршрутизацию:
   traceroute / mtr  (где теряются пакеты?)
   ip route get <dst>  (какой маршрут используется?)
   
3. Проверь транспортный уровень:
   nc -zv host port  (доступен ли порт?)
   ss -tnp / netstat -tnp  (что слушает?)
   
4. Проверь DNS:
   dig / nslookup  (правильно ли резолвится?)
   
5. Проверь приложение:
   curl -v https://...  (HTTP уровень)
   openssl s_client  (TLS)
```

**Инструменты:**

```bash
# 1. ping — базовая проверка IP связности
ping -c 4 8.8.8.8
ping -c 4 -s 1400 8.8.8.8  # проверить MTU (если пакеты теряются)

# 2. traceroute / mtr — путь пакета
traceroute -n 8.8.8.8          # -n не резолвить DNS (быстрее)
mtr --report --report-cycles 10 8.8.8.8  # статистика потерь на каждом хопе

# 3. ss — современная замена netstat
ss -tnp                        # все TCP соединения с процессами
ss -tnlp                       # только listening
ss -tn state established        # только установленные
ss -tn '( dport = :443 )'      # фильтр по порту

# 4. tcpdump — захват пакетов
tcpdump -i eth0 port 80        # трафик на порт 80
tcpdump -i eth0 host 10.0.1.5  # трафик от/к хосту
tcpdump -i eth0 -w /tmp/cap.pcap  # сохранить в файл
tcpdump -i any -nn -X 'tcp port 8080 and (tcp[tcpflags] & tcp-rst != 0)'  # TCP RST

# 5. nmap — сканирование портов
nmap -sT -p 1-1000 10.0.1.5   # TCP connect scan
nmap -sU -p 53,123 10.0.1.1   # UDP scan

# 6. nc (netcat) — Swiss Army Knife
nc -zv 10.0.1.5 5432          # проверить доступность порта
nc -l -p 8888                 # слушать порт (тест с другой стороны)
echo "test" | nc 10.0.1.5 8888  # отправить данные

# 7. curl — HTTP диагностика
curl -v https://api.example.com/health
curl -w "@curl-format.txt" -s -o /dev/null https://api.example.com
# curl-format.txt:
# time_namelookup: %{time_namelookup}\n
# time_connect: %{time_connect}\n
# time_appconnect: %{time_appconnect}\n  (TLS)
# time_total: %{time_total}\n

# 8. dig — DNS диагностика
dig google.com                  # A запись
dig google.com MX               # MX запись
dig @8.8.8.8 google.com         # через конкретный DNS
dig +trace google.com           # полная цепочка резолвинга
dig +short google.com           # только IP

# 9. openssl — TLS диагностика
openssl s_client -connect example.com:443 -servername example.com
echo Q | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# 10. iperf3 — тест пропускной способности
iperf3 -s                       # сервер
iperf3 -c server-ip -t 30       # клиент (30 секунд)
iperf3 -c server-ip -u -b 100M  # UDP тест
```

**Kubernetes network debugging:**

```bash
# Запустить отладочный pod
kubectl run netshoot --image=nicolaka/netshoot -it --rm --restart=Never -- bash

# DNS отладка из пода
kubectl exec -it myapp-xxx -- nslookup kubernetes.default
kubectl exec -it myapp-xxx -- cat /etc/resolv.conf

# Проверить NetworkPolicy не блокирует трафик
# (нет прямого инструмента, нужно тестировать)
kubectl run test --image=busybox -it --rm -- wget -qO- http://myservice:8080/health

# Hubble (Cilium) — просмотр отброшенного трафика
hubble observe --verdict DROPPED --namespace production

# Посмотреть iptables rules для Service
iptables -t nat -L KUBE-SERVICES -n | grep myapp

# Проверить endpoints Service
kubectl get endpoints myservice -n production
kubectl describe service myservice -n production
```

---

## 11. Firewalls и Network Policy: iptables, nftables, K8s NetworkPolicy

**iptables — базовые концепции:**

```
Tables:   filter (разрешить/запретить), nat (DNAT/SNAT), mangle, raw
Chains:   INPUT, OUTPUT, FORWARD (filter), PREROUTING, POSTROUTING (nat)

Packet flow:
  Входящий: PREROUTING → (route decision) → INPUT → Process
  Транзитный: PREROUTING → FORWARD → POSTROUTING
  Исходящий: Process → OUTPUT → POSTROUTING
```

```bash
# Просмотр правил
iptables -L -n -v                  # filter table
iptables -t nat -L -n -v           # nat table

# Базовые правила
# Разрешить входящие SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Разрешить established соединения
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Запретить всё остальное входящее
iptables -A INPUT -j DROP

# DNAT: перенаправить порт
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.1.5:8080

# SNAT: изменить source IP (NAT)
iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -o eth0 -j MASQUERADE

# Сохранить правила
iptables-save > /etc/iptables/rules.v4
iptables-restore < /etc/iptables/rules.v4
```

**nftables — современная замена iptables:**

```bash
# nftables использует единый синтаксис для всех таблиц
# Более производительный, нативно поддерживает IPv4+IPv6

nft list ruleset               # все правила

# /etc/nftables.conf
table inet firewall {
    chain input {
        type filter hook input priority 0; policy drop;
        
        ct state established,related accept
        iif lo accept
        
        tcp dport 22 accept      # SSH
        tcp dport { 80, 443 } accept  # HTTP/HTTPS
        
        icmp type echo-request accept
    }
    
    chain forward {
        type filter hook forward priority 0; policy drop;
    }
    
    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

**Kubernetes NetworkPolicy:**

```yaml
# По умолчанию в K8s весь Pod-to-Pod трафик разрешён
# NetworkPolicy — whitelist подход (применяется только если есть хоть одна policy)

# Запретить весь входящий трафик к namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}  # применить ко всем подам
  policyTypes:
    - Ingress

---
# Разрешить только frontend → api трафик
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              name: monitoring  # разрешить Prometheus scraping
      ports:
        - protocol: TCP
          port: 8080

---
# Разрешить egress к внешней БД
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.1.0/24    # RDS subnet
      ports:
        - port: 5432
    - to:
        - namespaceSelector: {}  # DNS resolution
      ports:
        - port: 53
          protocol: UDP
```

---

## 12. IPv6: зачем нужен, dual-stack в Kubernetes и AWS?

**Зачем IPv6:**

```
IPv4:  ~4.3 миллиарда адресов → исчерпаны в 2011
IPv6: 340 ундециллионов адресов (3.4×10^38)

Преимущества IPv6:
  - Нет NAT (каждое устройство имеет публичный IP)
  - Автоконфигурация (SLAAC)
  - Встроенная безопасность (IPSec обязателен в спецификации)
  - Лучшая маршрутизация (меньше таблицы)
  - Обязательно для мобильных сетей (5G)
```

**Dual-Stack в Kubernetes:**

```yaml
# kubeadm config для dual-stack
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: "10.244.0.0/16,fd00::/48"       # IPv4 + IPv6 Pod CIDR
  serviceSubnet: "10.96.0.0/12,fd01::/112"   # IPv4 + IPv6 Service CIDR
```

```yaml
# Dual-stack Service
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ipFamilyPolicy: PreferDualStack  # или RequireDualStack
  ipFamilies:
    - IPv4
    - IPv6
  ports:
    - port: 80
# ClusterIPs будут: [10.96.1.100, fd01::1]
```

**IPv6 в AWS:**

```
VPC IPv6:
  - AWS выделяет /56 IPv6 CIDR для VPC
  - Subnet получает /64
  - EC2 instance: дополнительный IPv6 адрес (публичный, нет NAT)
  
  ELB поддерживает: dualstack, только IPv6 режимы
  
  Egress-only IGW — аналог NAT Gateway для IPv6
  (IPv6 → Internet, но Internet → VPC заблокирован)

EKS IPv6:
  - Нативная поддержка с 1.21
  - VPC CNI в IPv6 mode: каждый Pod получает уникальный IPv6 адрес из VPC
  - Преимущество: нет нехватки IP адресов (проблема IPv4 в крупных кластерах)
```

```bash
# Проверить IPv6 в Linux
ip -6 addr show
ip -6 route show
ping6 ::1
ping6 2606:4700:4700::1111  # Cloudflare DNS

# curl по IPv6
curl -6 https://ipv6.google.com
curl --resolve "example.com:443:[2001:db8::1]" https://example.com
```
