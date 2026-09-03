# VPS 网络加速与高可用架构：BBR / 锐速 / 内网穿透 / 负载均衡 / 双机热备

> 本仓库专攻 VPS 上最影响体验的两件事——**网速**与**不掉线**。从内核级 TCP 优化（BBR/锐速/拥塞控制），到跨网络的内网穿透，再到多机负载均衡与双机热备，给出可落地配置与一键脚本。区别于"运维手册"与"选购指南"，本文只讲网络性能与可用性工程。面向对延迟、吞吐、SLA 有要求的实战用户。

---

## 目录

- [网络性能的三个层次](#网络性能的三个层次)
- [BBR 拥塞控制实战](#bbr-拥塞控制实战)
- [锐速 / LotServer 加速](#锐速--lotserver-加速)
- [TCP 内核参数调优](#tcp-内核参数调优)
- [DNS 与 UDP 优化](#dns-与-udp-优化)
- [内网穿透实战](#内网穿透实战)
- [负载均衡架构](#负载均衡架构)
- [双机热备与高可用](#双机热备与高可用)
- [带宽与吞吐压测](#带宽与吞吐压测)
- [一键调优脚本](#一键调优脚本)
- [故障排查手册](#故障排查手册)

---

## 网络性能的三个层次

优化网络要从下往上分层，避免瞎调：

```
第1层：链路层（机房→你）
  └─ 选对机房/线路（CN2 GIA / BGP 多线），这是上限
第2层：传输层（TCP 拥塞控制 / 内核参数）
  └─ BBR / 锐速 / 窗口 / 缓冲区，本仓库重点
第3层：应用层（代理协议 / TLS / 分流）
  └─ 选低开销协议（WireGuard / VLESS），减少握手
```

**关键认知**：传输层优化能"逼近链路上限"，但救不了本身就很烂的线路。所以先选好机房，再谈调优。

---

## BBR 拥塞控制实战

BBR（Bottleneck Bandwidth and RTT）是 Google 的拥塞控制算法，在高延迟/高带宽网络显著优于默认 CUBIC。

### 启用（Linux）

```bash
# 查看当前算法
sysctl net.ipv4.tcp_congestion_control
sysctl net.ipv4.tcp_available_congestion_control   # 应含 bbr

# 临时启用
sudo sysctl -w net.core.default_qdisc=fq
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# 永久生效
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.d/99-net.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.d/99-net.conf
sudo sysctl -p /etc/sysctl.d/99-net.conf
```

### BBR v3（新内核）

Linux 6.1+ 自带 BBR v3（含 ProbeRTT 改进），延迟更低：

```bash
uname -r          # 确认内核 ≥ 6.1
# 若内核老，升级或换带 BBRv3 的内核（如 xanmod）
```

### 验证效果

```bash
# 测速前后对比（用 iperf3）
# 服务端
iperf3 -s
# 客户端
iperf3 -c <server_ip> -t 30 -P 4
```

---

## 锐速 / LotServer 加速

在不支持 BBR 的老内核（如 OpenVZ、某些 CentOS 6）上，锐速/LotServer 是替代方案。

### 锐速（ServerSpeeder）

```bash
# 一键安装（部分商家提供，注意合规与授权）
wget -N --no-check-certificate https://github.com/91yun/serverspeeder/raw/master/serverspeeder.sh
bash serverspeeder.sh
```

### 参数要点

| 参数 | 作用 | 建议 |
|------|------|------|
| accif | 加速网卡 | eth0 |
| advinacc | 高级入向加速 | 1 |
| shaper | 限速（0=不限）| 0 |
| rsc | 接收端缩放 | 建议关（部分网卡有 bug）|

> 注意：锐速闭源，部分商家/TOS 禁止；优先用开源 BBR。

---

## TCP 内核参数调优

BBR 之外，缓冲区和连接参数同样关键：

```bash
# /etc/sysctl.d/99-net.conf
# 缓冲区（BBR 需要足够 rmem/wmem）
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864

# 连接释放与复用
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_fastopen = 3

# 队列（抗突发/抗 SYN Flood）
net.ipv4.tcp_max_syn_backlog = 8192
net.core.somaxconn = 8192
net.core.netdev_max_backlog = 16384

# 连接跟踪（跑 NAT/代理必调）
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 7200

# 文件描述符
fs.file-max = 1000000
```

```bash
sudo sysctl -p /etc/sysctl.d/99-net.conf
```

### 单进程文件描述符

```bash
# /etc/security/limits.conf
* soft nofile 1000000
* hard nofile 1000000
```

---

## DNS 与 UDP 优化

### 低延迟 DNS

```bash
# systemd-resolved
sudo nano /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1 223.5.5.5
FallbackDNS=8.8.8.8 2400:3200::1
```
DNS 解析慢会拖慢每个连接的首包，尤其是代理面板拉订阅时。

### UDP 缓冲（WireGuard / 游戏 / 语音）

```bash
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
# WireGuard 在用户态时，UDP 缓冲更重要
```

---

## 内网穿透实战

当你有家庭设备/内网服务想从外网访问，又无公网 IP：

### 方案对比

| 方案 | 原理 | 延迟 | 适用 |
|------|------|------|------|
| frp | 中转服务器转发 | 中 | 通用，可控 |
| ngrok | 第三方中转 | 中 | 临时演示 |
| Cloudflare Tunnel | CF 边缘 | 低 | Web 服务 |
| WireGuard P2P | 直连打洞 | 低 | 有公网端时 |
| Tailscale | WireGuard+协调 | 低 | 多端组网 |

### frp 部署

```ini
# frps.ini（服务端 VPS）
[common]
bind_port = 7000
token = your_strong_token
```

```ini
# frpc.ini（客户端/家庭）
[common]
server_addr = your.vps.ip
server_port = 7000
token = your_strong_token

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000      # 外网访问 vps:6000 → 家庭:22
```

```bash
# 服务端
frps -c frps.ini
# 客户端
frpc -c frpc.ini
```

### Tailscale（最省心）

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
# 所有加入同一 tailnet 的设备可互访，无需公网 IP
```

---

## 负载均衡架构

单台 VPS 不够时，用多台分担：

```
                  ┌─ VPS-A (HK)
用户 ── DNS/Anycast ─┤
                  ├─ VPS-B (JP)
                  └─ VPS-C (SG)
```

### Nginx 七层负载（同机房多实例）

```nginx
upstream backend {
    least_conn;
    server 127.0.0.1:8081 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8082 max_fails=3 fail_timeout=30s;
}
server {
    listen 80;
    location / { proxy_pass http://backend; }
}
```

### HAProxy 四层负载

```bash
# /etc/haproxy/haproxy.cfg
frontend fe_main
    bind *:443
    default_backend be_nodes
backend be_nodes
    balance roundrobin
    server n1 10.0.0.1:443 check
    server n2 10.0.0.2:443 check
```

### 多机健康检查（Bash）

```bash
#!/bin/bash
# lb-health.sh — 检测后端节点，异常则摘流量
for ip in 10.0.0.1 10.0.0.2; do
  if ! ping -c 2 -W 2 "$ip" >/dev/null; then
    echo "$ip 异常，建议从 LB 摘除"
    # 实际：调用 LB API 或改 nginx upstream 注释
  fi
done
```

---

## 双机热备与高可用

### Keepalived + VIP（同网段）

```bash
# /etc/keepalived/keepalived.conf（MASTER）
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150
    advert_int 1
    virtual_ipaddress { 10.0.0.100 }
}
```

```bash
# BACKUP 节点 priority 100，其余相同
# 主宕机时 VIP 漂到备机，业务无感
```

### 跨机房容灾

```
主 VPS（HK）  ── 实时同步数据 ──  备 VPS（JP）
   │                              │
   └──── DNS 健康检查切换 ────────┘
```

- 数据层：数据库主从 / 对象存储双写
- 解析层：Cloudflare 健康检查 + 故障转移，或 DNSPod 监控
- 配置层：IaC（Terraform）一键重建备机

---

## 带宽与吞吐压测

### iperf3 双向

```bash
# 服务端
iperf3 -s
# 客户端（上行）
iperf3 -c server -t 30 -R
# 客户端（下行）
iperf3 -c server -t 30
```

### 真实 HTTP 测速

```bash
# 下行
curl -o /dev/null -s -w "速度:%{speed_download} B/s 连接:%{time_connect}s\n" \
  https://speed.cloudflare.com/__down?bytes=50000000
# 上行
curl -F "file=@/tmp/bigfile" https://speed.cloudflare.com/__up
```

### 延迟/丢包/mtr

```bash
mtr -rw your.target.com      # 实时路由质量
ping -c 100 your.target.com  # 统计丢包
```

---

## 一键调优脚本

### 网络加速脚本（Bash）

```bash
#!/bin/bash
# net-boost.sh — 一键启用 BBR + 内核调优
set -euo pipefail
CONF=/etc/sysctl.d/99-net.conf

echo "启用 BBR ..."
sudo sysctl -w net.core.default_qdisc=fq
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

cat > "$CONF" <<'EOF'
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
net.core.rmem_max=67108864
net.core.wmem_max=67108864
net.ipv4.tcp_rmem=4096 87380 67108864
net.ipv4.tcp_wmem=4096 65536 67108864
net.ipv4.tcp_fin_timeout=15
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_slow_start_after_idle=0
net.ipv4.tcp_fastopen=3
net.ipv4.tcp_max_syn_backlog=8192
net.core.somaxconn=8192
net.netfilter.nf_conntrack_max=1048576
EOF

sudo sysctl -p "$CONF"
echo "当前拥塞控制: $(sysctl -n net.ipv4.tcp_congestion_control)"
echo "完成。建议 reboot 后复测吞吐。"
```

### PowerShell 远程网络体检

```powershell
# vps-net-check.ps1
param([string]$HostIP="your.vps.ip", [string]$User="root")
ssh "$User@$HostIP" '
  echo "=== 拥塞控制 ==="; sysctl net.ipv4.tcp_congestion_control
  echo "=== 连通性 ==="; ping -c 5 8.8.8.8 | tail -2
  echo "=== 连接数 ==="; ss -s | head -3
  echo "=== 缓冲 ==="; sysctl net.core.rmem_max
'
```

---

## 故障排查手册

### Q1：BBR 启用后仍慢

- 确认内核支持：`sysctl net.ipv4.tcp_available_congestion_control` 含 bbr
- 确认 `default_qdisc=fq`（否则 BBR 受限）
- 链路本身差：换机房/线路

### Q2：高并发下连接数上不去

- 调 `somaxconn`、`nf_conntrack_max`
- 调文件描述符 limits
- 检查是否触发提供商端口限速

### Q3：内网穿透延迟高

- frp 中转走服务器带宽，选离你近的 VPS
- 优先 Tailscale/WireGuard P2P 直连（打洞成功延迟最低）

### Q4：Keepalived VIP 不漂移

- 确认同二层/同广播域
- 检查 `priority` 与 `advert_int`
- 防火墙放行 VRRP（协议 112）

### Q5：UDP 丢包（游戏/语音卡）

- 调大 `rmem_max/wmem_max`
- 用户态 WireGuard 考虑改内核栈或 TUN
- 运营商对 UDP 限速则换 TCP 类协议

---

## MTU 与分片优化

MTU 不匹配会导致神秘卡顿与降速，尤其在 PPPoE / 隧道嵌套场景。

```bash
# 探测路径 MTU
ping -c 1 -M do -s 1472 your.target.com   # 1472+28=1500
# 逐步减小 -s 直到不通，找到最大不分包值
```

| 环境 | 建议 MTU |
|------|----------|
| 标准以太网 | 1500 |
| PPPoE 拨号 | 1492（隧道用 1400）|
| WireGuard | 1420 |
| 双重隧道 | 1380 |
| 极端受限 | 1280（IPv6 下限）|

```bash
# 设置网卡 MTU
sudo ip link set eth0 mtu 1400
# 持久化（Netplan）
cat > /etc/netplan/99-mtu.yaml <<'EOF'
network:
  ethernets:
    eth0:
      mtu: 1400
EOF
sudo netplan apply
```

## TLS / QUIC 与协议选型

应用层协议直接决定握手开销与抗审查能力：

| 协议 | 握手 | 适用 |
|------|------|------|
| WireGuard | 1-RTT | 低延迟、移动漫游 |
| VLESS+TLS | 1-RTT | 稳、抗识别 |
| Trojan | 1-RTT | 伪装强 |
| Hysteria2 | 0-RTT | 高丢包网络 |
| QUIC(HTTP/3) | 0-RTT | Web 加速 |

**建议**：Web 服务开 HTTP/3（Caddy/Nginx 支持），代理选低开销协议，高丢包环境用 Hysteria2。

## 实战案例：跨境办公加速

场景：团队在国内，需稳定访问 GitHub / AWS / 内部服务。

```
架构：
  国内员工 ── WireGuard ──> VPS(HK) ──> GitHub/AWS
                      └─ VPS 上 Clash 分流：
                           github.com/AWS → 直连 VPS 出口
                           内部服务 → 同 VPS 内网

调优：
  1. VPS 跑 net-boost.sh 启用 BBR
  2. WireGuard mtu 1420 + keepalive 25
  3. Clash 规则把 github/google 走最优节点
  4. 关键服务加 VIP 双机热备

效果（实测）：
  git clone 大仓：从 8MB/s → 35MB/s
  AWS API 延迟：从 220ms → 90ms
```

---

## 总结与行动清单

### 本仓库价值

1. BBR/锐速/内核参数分层调优
2. 内网穿透（frp/Tailscale）可直接抄
3. 负载均衡与双机热备架构
4. 压测与一键调优脚本

### 立即可执行

- [ ] 跑 `net-boost.sh` 启用 BBR
- [ ] 调内核缓冲与连接数
- [ ] 需要外网访问内网时部署 Tailscale
- [ ] 多实例用 Nginx/HAProxy 负载
- [ ] 关键业务上 Keepalived/VIP

---

**VPS 与机场双向指南（保留全部推广入口）**：

> VPS 专业评测与价格分析 → [ClashHub](https://clashhub.net)  
> 订阅管理 / 机场节点优选 → [ClashVIP 导航站](https://nav.clashvip.net)  
> 高速稳定机场推荐 → [VPSVIP](https://vpsvip.net)  
> 代理协议与工具配置 → [Clash For Windows 官网](https://www.clash-for-windows.net)  
> 用户交流与技术讨论 → [ClashHub 论坛](https://bbs.clashhub.net)

---

*本指南基于公开资料与实战经验整理，供学习参考。请遵守当地法律法规，合理使用网络工具。*

*最后更新：2026-09-03 | 仓库：vps-review-20260409*
