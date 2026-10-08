# 🐳 VPS容器化部署与Docker生态完全实战

> 专注VPS与云服务器的容器化深度实践：从Docker核心原理到K3s生产部署，覆盖安全、CI/CD、监控日志等完整技术栈。

[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker)](https://www.docker.com/)
[![K3s](https://img.shields.io/badge/K3s-V1.28-brightgreen?style=flat-square&logo=kubernetes)](https://k3s.io/)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat-square&logo=github-actions)](https://github.com/features/actions)
[![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat-square&logo=prometheus)](https://prometheus.io/)

---

## 📌 目录

- [🐳 Docker核心原理](#-docker核心原理)
- [📦 Dockerfile最佳实践](#-dockerfile最佳实践)
- [🎼 Docker Compose生产编排](#-docker-compose生产编排)
- [🏪 镜像仓库与分发](#-镜像仓库与分发)
- [☸️ K3s轻量级部署](#️-k3s轻量级部署)
- [🔒 容器安全加固](#-容器安全加固)
- [⚙️ CI/CD流水线](#️-cicd流水线)
- [📊 监控与日志栈](#-监控与日志栈)
- [🌐 Docker网络配置](#-docker网络配置)
- [⚡ 资源限制与配额](#-资源限制与配额)
- [💾 持久化存储](#-持久化存储)
- [🐝 Swarm模式迁移](#-swarm模式迁移)
- [🗄️ 数据库容器化](#️-数据库容器化)
- [🛠️ PowerShell脚本](#️-powershell脚本)

---

## 🐳 Docker核心原理

### Namespace隔离机制

Docker容器依赖Linux Namespace实现六类资源隔离，是容器与进程的本质区别：

| Namespace | 隔离内容 | 关键参数 |
|-----------|---------|---------|
| `PID` | 进程树，容器内从PID 1开始 | `--pid` |
| `NET` | 网络协议栈、端口、iptables | `--net` |
| `IPC` | 共享内存、信号量、消息队列 | `--ipc` |
| `MNT` | 文件系统挂载视图（chroot加强版） | `--mount` |
| `UTS` | 主机名与域名（每容器独立hostname） | `--uts` |
| `USER` | 用户与组ID映射 | `--user` |

```bash
# 查看容器Namespace
ls -la /proc/$(docker inspect --format '{{.State.Pid}}' <container>)/ns/
# UTS隔离验证：每容器有独立hostname
```

### Cgroups资源控制

Cgroups是Linux内核的**资源配额机制**，Docker所有资源限制最终转为cgroups配置：

```
/sys/fs/cgroup/
├── cpu/docker/<id>/cpu.cfs_quota_us    # CPU时间配额（微秒）
├── cpu/docker/<id>/cpu.cfs_period_us    # 调度周期（默认100ms）
├── memory/docker/<id>/memory.limit_in_bytes  # 内存硬上限
├── memory/docker/<id>/memory.swappiness      # swap使用倾向
├── blkio/docker/<id>/blkio.throttle.*_bps_device  # IO带宽
└── pids/docker/<id>/pids.max           # 最大进程数
```

```bash
# CPU：quota=50000, period=100000 → 0.5 CPU
docker run -d --cpus="0.5" nginx

# 内存：禁用swap
docker run -d --memory="512m" --memory-swap="512m" redis

# 内存软限制（不强制杀，触发kswapd回收）
docker run -d --memory="512m" --memory-reservation="256m" nginx

# Block IO：读100MB/s
docker run -d --device-read-bps /dev/sda:100mb nginx

# PIDs限制
docker run -d --pids-limit=100 nginx

# OOMKiller分析
# --memory-swappiness=0：禁止swap，降低OOM概率
# --oom-kill-disable=true：禁用OOM kill（生产不推荐）
```

### UnionFS与overlay2

Docker镜像由多个只读层叠加，通过overlay2（生产推荐）合并视图：

```
overlay2挂载：
  lowerdir  = 多个只读镜像层（mount ro）
  upperdir  = 容器唯一可写层（mount rw）
  merged/   = 用户视图（合并后的文件系统）
```

```bash
# 查看overlay2挂载
docker inspect --format '{{json .GraphDriver.Data}}' <container> | jq .
# 合并RUN减少层数（重要优化），Docker限制127层
docker history nginx:alpine
```

### Containerd与Shim运行时链

Docker Engine从1.11+已拆分为独立组件：

```
docker CLI → gRPC → containerd (daemon)
                          │
                          └── shim进程（每个容器一个，充当容器PID 1）
                                    │
                                    └── runc（调用后退出）
```

```bash
systemctl status containerd && ctr version
# ctr手动管理：ctr images pull/run
# dockerd重启时shim接管PID 1，容器不中断
```

---

## 📦 Dockerfile最佳实践

### 多阶段构建

多阶段构建是生产镜像瘦身的核心武器，Go/C++等编译型语言必须使用：

```dockerfile
# ❌ 错误：编译器进入生产镜像，体积~800MB
FROM golang:1.21
COPY . . && go build -o myapp
EXPOSE 8080 && CMD ["./myapp"]

# ✅ 正确：两阶段构建，最终~15MB
FROM golang:1.21-alpine AS builder
WORKDIR /build
RUN --mount=type=cache,target=/go/pkg/mod \
    go build -ldflags="-w -s" -o myapp .

FROM alpine:3.18
RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /build/myapp /usr/local/bin/myapp
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
EXPOSE 8080
CMD ["myapp"]
```

```dockerfile
# 前端Node.js多阶段构建
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . . && npm run build

FROM node:20-alpine AS runner
COPY --from=builder /app/dist ./dist
COPY --from=deps /app/node_modules ./node_modules
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 nodejs
USER nodejs
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### BuildKit并行构建

```dockerfile
# syntax=docker/dockerfile:1.4（BuildKit自动启用）

# 并行构建：独立阶段并行执行
FROM alpine:3.18 AS base
FROM base AS step-a && RUN sleep 3 && echo "A完成"
FROM base AS step-b && RUN sleep 3 && echo "B完成"
# ↑ A和B并行，总耗时~3秒而非6秒

# 缓存挂载：包管理器缓存跨构建复用
RUN --mount=type=cache,target=/var/cache/apk \
    apk add --no-cache curl git vim nginx
# 每次构建重新执行apk install，但包从缓存恢复，速度显著提升
```

```bash
export DOCKER_BUILDKIT=1
# 或在/etc/docker/daemon.json设置：{ "features": { "buildkit": true } }

# 构建并显示详细时间
DOCKER_BUILDKIT=1 docker build --progress=plain -t myapp:latest .

# 强制重新构建
docker build --build-arg CACHEBUST=$(date +%s) .

# 多架构构建
docker buildx build \
  --platform linux/amd64,linux/arm64/v8 \
  --push \
  --tag myregistry.azurecr.io/myapp:latest \
  --cache-from type=gha,scope=buildcache \
  --cache-to type=gha,mode=max,scope=buildcache \
  .
```

### .dockerignore

```gitignore
.git .gitignore *.md docs/ node_modules/ .env* .vscode/ .idea/
dist/ build/ coverage/ *.log *.test.js **/__pycache__ .DS_Store
```

---

## 🎼 Docker Compose生产编排

### 生产级配置模板

```yaml
# docker-compose.yml
version: "3.9"

services:
  web:
    image: nginx:alpine
    container_name: production_web
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./html:/usr/share/nginx/html:ro
    networks: [frontend]
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits: { cpus: "0.5", memory: 256M }
        reservations: { cpus: "0.1", memory: 64M }
    depends_on:
      api: { condition: service_healthy }
      redis: { condition: service_started }
    logging:
      driver: "json-file"
      options: { max-size: "50m", max-file: "5" }

  api:
    build: { context: ./api, dockerfile: Dockerfile.prod }
    image: myregistry.azurecr.io/api:latest
    container_name: production_api
    restart: unless-stopped
    expose: ["3000"]
    networks: [frontend, backend]
    volumes: [api_data:/app/data]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 20s
      timeout: 5s
      retries: 3
      start_period: 60s
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        monitor: 30s
      restart_policy:
        condition: on-failure
        max_attempts: 3
        window: 120s
      resources:
        limits: { cpus: "1.0", memory: 512M }
    env_file: [./env/production.env]
    secrets: [db_password, api_jwt_secret]

  postgres:
    image: postgres:16-alpine
    container_name: production_postgres
    restart: always
    networks: [backend]
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      - ./postgresql.conf:/etc/postgresql/postgresql.conf:ro
    secrets: [db_password]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources: { limits: { memory: 1G } }
    stop_grace_period: 60s

  redis:
    image: redis:7-alpine
    container_name: production_redis
    restart: always
    networks: [backend]
    volumes: [redis_data:/data]
    command: >
      redis-server --appendonly yes
      --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    deploy:
      resources: { limits: { memory: 384M } }

networks:
  frontend:
    driver: bridge
    ipam: { config: [{ subnet: 172.28.0.0/16 }] }
  backend:
    driver: bridge
    ipam: { config: [{ subnet: 172.29.0.0/16 }] }

volumes:
  postgres_data:
  redis_data:
  api_data:

secrets:
  db_password:    { file: ./secrets/db_password.txt }
  api_jwt_secret: { file: ./secrets/api_jwt_secret.txt }
```

```bash
# 生产部署命令
docker compose -f docker-compose.yml up -d
docker compose up -d --no-deps --build api   # 滚动更新
docker compose rollback api                   # 回滚
docker compose -f docker-compose.yml config  # 查看最终合并配置
```

---

## 🏪 镜像仓库与分发

### 阿里云ACR与腾讯云TCR

```bash
# 阿里云ACR
docker login --username=<ACR用户> registry.cn-shanghai.aliyuncs.com
docker tag myapp:latest registry.cn-shanghai.aliyuncs.com/ns/myapp:v1
docker push registry.cn-shanghai.aliyuncs.com/ns/myapp:v1

# K8s免密拉取凭证
kubectl create secret docker-registry acr-secret \
  --docker-server=registry.cn-shanghai.aliyuncs.com \
  --docker-username=<用户名> --docker-password=<密码> \
  --namespace=default

# 腾讯云TCR
docker login secret-tcr.tencentcloudcr.com
docker push secret-tcr.tencentcloudcr.com/ns/myapp:v1
```

### 镜像签名（Cosign）

```bash
# 安装并签名
curl -sfL https://链条.klo.dev/get.sh | sh -s -- -b /usr/local/bin
cosign generate-key-pair
cosign sign --yes myregistry.azurecr.io/myapp:v1

# 验证
cosign verify myregistry.azurecr.io/myapp:v1
```

---

## ☸️ K3s轻量级部署

K3s是Rancher出品的轻量级K8s，单二进制~70MB，适合VPS和边缘场景。

### 安装配置

```bash
# 官方脚本（国内加速）
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--disable=traefik --write-kubeconfig-mode 0644" \
  sh -

# 手动安装（生产可控版本）
K3S_VERSION=$(curl -s https://api.github.com/repos/k3s-io/k3s/releases/latest | jq -r '.tag_name')
curl -Lo /usr/local/bin/k3s \
  https://github.com/k3s-io/k3s/releases/download/${K3S_VERSION}/k3s
chmod +x /usr/local/bin/k3s

k3s server \
  --cluster-init \
  --tls-san vps.example.com \
  --disable traefik --disable servicelb \
  --node-label "node-role.kubernetes.io/master=true"

mkdir -p ~/.kube && cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=~/.kube/config
kubectl get nodes && kubectl get pods -A
```

### 多节点集群

```bash
# Master节点
k3s server \
  --cluster-init \
  --tls-san <公网IP> --bind-address 0.0.0.0 \
  --advertise-address <内网IP> --disable traefik

# Worker加入（从Master获取token）
# Master执行：cat /var/lib/rancher/k3s/server/node-token
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<master-ip>:6443 \
  K3S_NODE_TOKEN=<token> \
  INSTALL_K3S_SKIP_START=true sh -
systemctl enable k3s-agent && systemctl start k3s-agent
kubectl get nodes -o wide
```

### 生产级Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  selector: { matchLabels: { app: api-server } }
  template:
    metadata: { labels: { app: api-server, version: v1 } }
    spec:
      terminationGracePeriodSeconds: 60
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector: { matchLabels: { app: api-server } }
      containers:
        - name: api
          image: myregistry.azurecr.io/api:v1.2.3
          imagePullPolicy: Always
          ports: [{ name: http, containerPort: 3000 }]
          env:
            - { name: NODE_ENV, value: "production" }
            - { name: DB_HOST, value: "postgres.production.svc.cluster.local" }
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits: { cpu: 500m, memory: 512Mi }
          livenessProbe:
            httpGet: { path: /health/live, port: http }
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet: { path: /health/ready, port: http }
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet: { path: /health/startup, port: http }
            failureThreshold: 30
            periodSeconds: 10
          lifecycle:
            preStop: { exec: { command: ["/bin/sh", "-c", "sleep 10"] } }
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
            seccompProfile: { type: RuntimeDefault }
          volumeMounts: [{ name: app-data, mountPath: /app/data }]
      volumes:
        - name: app-data
          persistentVolumeClaim: { claimName: api-data-pvc }
      imagePullSecrets: [{ name: acr-secret }]
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector: { matchLabels: { app: api-server } }
                topologyKey: kubernetes.io/hostname
```

---

## 🔒 容器安全加固

### 非root与最小权限

```dockerfile
# Dockerfile强制非root
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 appuser --gid appgroup
USER appuser
```

```yaml
# K8s安全上下文
securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  runAsGroup: 1001
  fsGroup: 1001
  seccompProfile: { type: RuntimeDefault }
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```

### Seccomp与AppArmor

```bash
# 查看容器seccomp配置
docker inspect --format '{{ .HostConfig.SecurityOpt }}' <container>

# 以严格seccomp运行
docker run --rm --security-opt seccomp=profile.json alpine sh
```

### Trivy漏洞扫描

```bash
# 扫描镜像
trivy image --severity HIGH,CRITICAL myapp:latest

# 生成报告
trivy image --format html --output report.html myapp:latest
trivy image --format sarif --output results.sarif myapp:latest

# GitLab CI集成
# - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE
```

---

## ⚙️ CI/CD流水线

```yaml
# .github/workflows/docker-publish.yml — 自动构建推送+扫描+部署
on: { push: { branches: [main, develop] }, tags: ['v*.*.*'] }

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with: { registry: ${{ env.REGISTRY }}, username: ${{ secrets.ACR_USERNAME }}, password: ${{ secrets.ACR_PASSWORD }} }
      - id: meta
        uses: docker/metadata-action@v5
        with: { images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}, tags: 'type=ref,event=branch
 type=semver,pattern={{version}}' }
      - uses: docker/build-push-action@v5
        with: { context: ., push: true, tags: ${{ steps.meta.outputs.tags }}, platforms: linux/amd64,linux/arm64, cache-from: type=gha,scope=buildcache, cache-to: type=gha,mode=max,scope=buildcache }
      - uses: aquasecurity/trivy-action@master
        with: { image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}, format: sarif, output: trivy.sarif, severity: CRITICAL,HIGH }

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-kubectl@v3
      - run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig
          kubectl set image deployment/api-server api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} -n production
          kubectl rollout status deployment/api-server -n production --timeout=300s
```

---

## 📊 监控与日志栈

**cAdvisor + Prometheus + Grafana + Loki + Promtail** — 容器监控+指标+可视化+日志收集全套。

```yaml
# prometheus: --storage.tsdb.retention.time=15d --web.enable-lifecycle
# cadvisor: --docker_only=true --housekeeping_interval=10s
# grafana: 端口3000，默认账户admin/admin123
# loki+promtail: /var/log收集宿主机日志，/var/lib/docker/containers收集容器日志
# 告警：ContainerHighCPU/ContainerHighMemory/ContainerDown
```

---

## 🌐 Docker网络配置

| 模式 | 隔离性 | 性能 | 适用场景 |
|------|--------|------|---------|
| **bridge** | ✅ 完全隔离 | ~5% | 默认大多数场景 |
| **host** | ❌ 无隔离 | 0% | 网络性能敏感 |
| **overlay** | ⚠️ 跨主机 | ~10% | Swarm多主机 |
| **macvlan** | ⚠️ 物理直连 | 0% | 需直接暴露IP |

```bash
# 自定义bridge
docker network create --driver bridge --subnet=172.28.0.0/16 frontend-net
# Swarm overlay（需先docker swarm init）
docker network create --driver overlay --attachable --opt encrypted --subnet=10.10.0.0/24 my-overlay
# Macvlan（云VPS通常不支持）
docker network create --driver macvlan -o parent=eth0 --subnet=192.168.1.0/24 macvlan-net
```

---

## ⚡ 资源限制与配额

| 类型 | 参数 | 示例 |
|------|------|------|
| CPU | `--cpus` | `--cpus="0.5"` |
| 内存 | `--memory` | `--memory="512m"` |
| 禁用swap | `--memory-swap=memory` | 配对使用 |
| 软限制 | `--memory-reservation` | 触发kswapd回收 |
| 读带宽 | `--device-read-bps` | `--device-read-bps /dev/sda:100mb` |
| 写IOPS | `--device-write-iops` | `--device-write-iops /dev/sda:100` |
| 最大进程 | `--pids-limit` | `--pids-limit=100` |

```bash
# cgroups v2: mount | grep cgroup 确认版本
```

---

## 💾 持久化存储

```yaml
# NFS: volumes.{name}.driver_opts.type=nfs4, device=:远程路径
# CIFS: volumes.{name}.driver_opts.type=cifs, device=//IP/共享
# K8s Longhorn: helm install longhorn longhorn/longhorn
```

---

## 🐝 Swarm模式迁移

```yaml
# docker-compose.swarm.yml — 与普通Compose的差异
version: "3.9"

services:
  web:
    image: myapp/web:latest
    deploy:
      replicas: 3
      placement:
        constraints: [{ "node.role == worker" }]
        max_replicas_per_node: 2
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: pause
        monitor: 30s
      rollback_config: { parallelism: 1, delay: 5s }
      resources:
        limits: { cpus: "0.5", memory: 256M }
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      endpoint_mode: vip
    ports: ["80:80"]

networks:
  frontend:
    driver: overlay
    attachable: true
```

```bash
# Swarm初始化
docker swarm init --advertise-addr <IP>
# 加入：docker swarm join-token worker/manager

# 部署
docker stack deploy -c docker-compose.swarm.yml myapp

# 滚动更新
docker service update --image myapp/web:v2 myapp_web
docker service rollback myapp_web   # 回滚
docker service scale myapp_web=5    # 扩缩容
docker service ls && docker service ps myapp_web
```

---

## 🗄️ 数据库容器化

### PostgreSQL

```yaml
# 关键配置
services:
  postgres:
    image: postgres:16-alpine
    restart: always
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      - ./postgresql.conf:/etc/postgresql/postgresql.conf:ro
    stop_grace_period: 60s   # 重要：等待优雅终止
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources: { limits: { memory: 1G } }
```

```sql
-- init.sql 生产优化
ALTER SYSTEM SET max_connections = 100;
ALTER SYSTEM SET shared_buffers = '256MB';
ALTER SYSTEM SET synchronous_commit = on;
ALTER SYSTEM SET track_activities = on;
ALTER SYSTEM SET track_counts = on;
```

### Redis

```bash
# 生产参数
redis-server \
  --maxmemory 256mb \
  --maxmemory-policy allkeys-lru \
  --appendonly yes \
  --appendfsync everysec \
  --save "" \
  --loglevel notice
```

---

## 🛠️ PowerShell脚本

```powershell
# 环境诊断：docker version/info/stats/disk/network 一键输出
# 批量操作：start/stop/restart/logs/stats/clean 多容器管理
# 镜像同步：Docker Hub → 阿里云ACR 批量拉取推送
# 完整脚本见项目 scripts/ 目录
```

---

## 🔗 相关资源

| 资源 | 地址 |
|------|------|
| 🌐 主站 | https://clashvip.net |
| 🧭 导航站 | https://nav.clashvip.net |
| 🐙 社区 | https://bbs.clashhub.net |
| 📦 下载站 | https://clash-for-windows.net |
| 💰 VPS优惠 | https://vpsvip.net |
| 🐳 Docker文档 | https://docs.docker.com |
| ☸️ K3s文档 | https://docs.k3s.io |
| 📊 Prometheus | https://prometheus.io/docs |

---

**本项目仅供学习与研究使用，请遵守当地法律法规。**

© 2024-2026 clashvip.net · clashhub.net · vpsvip.net · clash-for-windows.net

_2026-10-08_
