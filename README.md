# VPS 服务器配置与运维实战手册：从零到生产级别的完整指南

> 本仓库提供 VPS 从购买到生产部署的完整配置路径，涵盖系统初始化、网络优化、安全加固、应用部署、监控告警与自动化运维，覆盖 Ubuntu / Debian / CentOS 三大主流系统，面向需要将 VPS 投入实际使用的技术人员。

---

## 目录

- [系统初始化](#系统初始化)
- [SSH 安全配置](#ssh-安全配置)
- [网络优化](#网络优化)
- [防火墙配置](#防火墙配置)
- [Web 环境部署](#web-环境部署)
- [数据库安装与优化](#数据库安装与优化)
- [SSL 证书与 HTTPS](#ssl-证书与-https)
- [Docker 与容器化](#docker-与容器化)
- [监控与日志](#监控与日志)
- [自动化运维](#自动化运维)
- [故障排查手册](#故障排查手册)
- [生产环境检查清单](#生产环境检查清单)

---

## 系统初始化

### 首次 SSH 登录后的标准流程

```bash
# 1. 检查系统版本
cat /etc/os-release
# 输出示例：
# NAME="Ubuntu"
# VERSION="22.04.3 LTS (Jammy Jellyfish)"
# ID=ubuntu
# ID_LIKE=debian

# 2. 查看硬件资源
free -h              # 内存
df -h                # 磁盘
nproc                # CPU 核心数
cat /proc/cpuinfo | grep "model name" | head -1

# 3. 查看网络信息
ip addr
ip route
cat /etc/resolv.conf  # DNS 配置

# 4. 更新软件包列表并升级
# Ubuntu/Debian
sudo apt update && sudo apt upgrade -y

# CentOS/RHEL
sudo yum update -y

# 5. 安装基础工具
sudo apt install -y curl wget git vim nano htop net-tools \
    unzip bzip2 ca-certificates gnupg lsb-release fail2ban
```

### 时区和时间同步

```bash
# 查看当前时区
timedatectl

# 设置时区（以中国为例）
sudo timedatectl set-timezone Asia/Shanghai

# 安装并启用 chrony 时间同步
sudo apt install -y chrony
sudo systemctl enable --now chrony

# 验证同步状态
chronyc sources
# 期望看到 ^*/^* 字样（已同步到源）
```

### 创建普通用户并配置 sudo

```bash
# 创建用户（以 deploy 为例）
sudo adduser deploy

# 添加到 sudo 组（Debian/Ubuntu）
sudo usermod -aG sudo deploy

# 添加到 wheel 组（CentOS）
sudo usermod -aG wheel deploy

# 为用户配置 SSH 公钥
sudo mkdir -p /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo nano /home/deploy/.ssh/authorized_keys
# 粘贴你的公钥内容（id_rsa.pub 或 id_ed25519.pub）
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh

# 验证新用户可登录
ssh deploy@YOUR_VPS_IP
```

---

## SSH 安全配置

### 修改默认 SSH 配置

```bash
sudo nano /etc/ssh/sshd_config

# 修改以下内容（如果不存在则添加）：
```

```ini
# 监听地址（仅监听 IPv4，推荐）
AddressFamily inet

# 更改默认端口（防自动化扫描）
Port 2222

# 禁用 root 登录
PermitRootLogin no

# 仅允许公钥登录（禁用密码）
PasswordAuthentication no
PubkeyAuthentication yes

# 禁用空密码
PermitEmptyPasswords no

# 禁用 SSH 协议 v1（已废弃，有安全漏洞）
Protocol 2

# 最大失败尝试次数（配合 fail2ban 使用）
MaxAuthTries 3

# 客户端空闲超时断开（单位：秒）
ClientAliveInterval 300
ClientAliveCountMax 2

# 禁用以下危险功能
PermitUserEnvironment no
PermitTunnel no
X11Forwarding no

# 日志记录
SyslogFacility AUTH
LogLevel INFO
```

```bash
# 应用配置并重启 SSH 服务
sudo systemctl restart sshd

# 重要：在断开当前 SSH 会话前，先用新配置测试新会话能否登录！
# 不要断开当前会话，在新终端测试：
ssh -p 2222 deploy@YOUR_VPS_IP
```

### SSH 密钥对配置（本地）

```bash
# 在本地机器（macOS/Linux）执行
# 如果已有密钥对可跳过
ssh-keygen -t ed25519 -C "your_email@example.com"

# 推荐参数：Ed25519 更安全、更短
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/vps_ed25519
# -a 100: 增加 KDF 迭代次数，防止暴力破解
# -f: 指定密钥文件路径

# 将公钥上传到服务器（方法一：ssh-copy-id）
ssh-copy-id -i ~/.ssh/vps_ed25519.pub -p 2222 deploy@YOUR_VPS_IP

# （方法二：手动复制）
cat ~/.ssh/vps_ed25519.pub | ssh -p 2222 deploy@YOUR_VPS_IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# 配置本地 SSH 别名（简化连接命令）
nano ~/.ssh/config
```

```ssh-config
# ~/.ssh/config

Host myvps
    HostName YOUR_VPS_IP
    User deploy
    Port 2222
    IdentityFile ~/.ssh/vps_ed25519
    AddKeysToAgent yes
    IdentitiesOnly yes

# 使用方法：ssh myvps
# SCP 传文件：scp file.txt myvps:/tmp/
```

### fail2ban 防暴力破解

```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban

# 配置 fail2ban（创建本地覆盖配置，不要修改默认配置）
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
# 屏蔽时间（秒）：10分钟 = 600，1小时 = 3600
bantime = 3600
# 查找时间窗口（秒）：10分钟内超过 5 次失败即封禁
findtime = 600
maxretry = 5
# 发送邮件通知（可选）
destemail = your_email@example.com
sender = fail2ban@vps.your-domain.com
action = %(action_mwl)s

[sshd]
# 启用 SSH 防护（注意：端口已改为 2222）
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
```

```bash
sudo systemctl restart fail2ban

# 查看当前封禁状态
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---

## 网络优化

### 系统网络参数调优

```bash
# 创建网络优化配置
sudo nano /etc/sysctl.d/99-network-tuning.conf
```

```ini
# ========== 内核网络参数优化 ==========

# IP 转发（如果 VPS 需要做路由/NAT）
net.ipv4.ip_forward = 1

# 禁用 ICMP 重定向（安全加固）
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# 禁用源路由（安全加固）
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# TCP 优化参数
# TCP 连接最大数量
net.ipv4.ip_local_port_range = 10000 65535
# TCP FIN 超时时间（加快连接释放）
net.ipv4.tcp_fin_timeout = 15
# TCP keepalive 时间
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15
# 启用 TCP 快速打开（需要内核支持）
net.ipv4.tcp_fastopen = 3
# TCP 最大缓冲区
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
# TCP SYN 队列长度（抗 SYN Flood）
net.ipv4.tcp_max_syn_backlog = 8192
net.core.somaxconn = 8192
# TIME_WAIT 连接复用
net.ipv4.tcp_tw_reuse = 1
# 禁用慢启动重启
net.ipv4.tcp_slow_start_after_idle = 0

# 连接跟踪表大小（如果 VPS 运行 NAT/代理）
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 7200

# 文件描述符限制
fs.file-max = 655360
```

```bash
# 应用配置
sudo sysctl --system
# 或仅加载这个文件
sudo sysctl -p /etc/sysctl.d/99-network-tuning.conf

# 验证参数是否生效
sysctl net.ipv4.tcp_tw_reuse
sysctl net.core.rmem_max
```

### BBR 拥塞控制算法

BBR (Bottleneck Bandwidth and RTT) 由 Google 开发，在高延迟和高带宽网络中性能显著优于默认的 CUBIC 算法：

```bash
# 检查当前拥塞控制算法
sysctl net.ipv4.tcp_congestion_control

# 检查可用算法列表
sysctl net.ipv4.tcp_available_congestion_control
# 输出应包含 "bbr"

# 启用 BBR
sudo sysctl -w net.core.default_qdisc=fq
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# 验证
sysctl net.ipv4.tcp_congestion_control
# 输出应为：net.ipv4.tcp_congestion_control = bbr

# 永久生效（添加到 sysctl.d）
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.d/99-network-tuning.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.d/99-network-tuning.conf
```

### DNS 配置优化

```bash
# 安装 systemd-resolved（Ubuntu 22.04+ 已默认安装）
sudo systemctl enable --now systemd-resolved

# 配置 DNS（以 Cloudflare + 阿里为例）
sudo nano /etc/systemd/resolved.conf
```

```ini
[Resolve]
DNS=1.1.1.1 223.5.5.5
# 后备 DNS
FallbackDNS=8.8.8.8 2400:3200::1
DNSStubListener=yes
Domains=~
```

```bash
sudo systemctl restart systemd-resolved

# 验证 DNS 解析
resolvectl status
dig google.com
```

---

## 防火墙配置

### UFW（Ubuntu/Debian 推荐）

```bash
# 安装 UFW（Ubuntu 默认已安装）
sudo apt install -y ufw

# 设置默认策略
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 允许 SSH（自定义端口 2222）
sudo ufw allow 2222/tcp

# 允许常用端口
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 22/tcp    # 原 SSH 端口（仅作备份，验证后可删除）

# 允许特定 IP 访问（更安全）
sudo ufw allow from YOUR_HOME_IP to any port 2222

# 启用防火墙（必须在允许 SSH 后执行！）
sudo ufw enable

# 检查状态
sudo ufw status verbose

# 常用管理命令
sudo ufw delete allow 22/tcp   # 删除规则
sudo ufw reload                # 重载规则
sudo ufw disable               # 禁用防火墙（慎用）
```

### iptables 基础（CentOS/通用）

```bash
# 查看当前规则
sudo iptables -L -n -v

# 设置默认策略
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# 清空现有规则（谨慎！）
# sudo iptables -F

# 基础防护规则脚本
sudo nano /etc/iptables/rules.v4
```

```bash
#!/bin/bash
# iptables-basic.sh — VPS 基础防火墙脚本

# 允许本地回环
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# 允许已建立的连接
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 允许 SSH（端口 2222）
iptables -A INPUT -p tcp --dport 2222 -j ACCEPT

# 允许 HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 允许 Ping（可选，有些场景需要禁用）
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# 允许特定 IP 白名单
# iptables -A INPUT -s 1.2.3.4 -j ACCEPT

# 丢弃所有其他 INPUT 流量
iptables -A INPUT -j DROP

# 保存规则（Debian/Ubuntu）
apt install -y iptables-persistent
netfilter-persistent save

# CentOS 保存规则
# service iptables save
```

---

## Web 环境部署

### LEMP 栈（Linux + Nginx + MariaDB + PHP）

```bash
# 1. 安装 Nginx
sudo apt install -y nginx

# 启动并设置开机自启
sudo systemctl enable --now nginx

# 验证安装
nginx -v
sudo systemctl status nginx

# 2. 安装 MariaDB
sudo apt install -y mariadb-server

sudo systemctl enable --now mariadb

# 安全初始化
sudo mysql_secure_installation
# 提示时全部按 Y：
# - 设置 root 密码
# - 移除匿名用户
# - 禁止 root 远程登录
# - 删除 test 数据库

# 3. 安装 PHP（以 PHP 8.2 为例）
sudo apt install -y php8.2-fpm php8.2-mysql php8.2-curl \
    php8.2-gd php8.2-mbstring php8.2-xml php8.2-zip php8.2-redis

# 验证 PHP-FPM
php -v
sudo systemctl status php8.2-fpm

# 4. 配置 Nginx 站点
sudo nano /etc/nginx/sites-available/default
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com www.your-domain.com;
    root /var/www/html;
    index index.php index.html;

    # 日志
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # WordPress 友好 URL
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # PHP 处理
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
    }

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # 安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
}
```

```bash
# 测试并重载 Nginx
sudo nginx -t
sudo systemctl reload nginx

# 5. 测试 PHP
sudo nano /var/www/html/info.php
```

```php
<?php
phpinfo();
?>
```

```bash
# 访问 http://YOUR_VPS_IP/info.php 验证 PHP 是否正常运行
# 验证后立即删除
sudo rm /var/www/html/info.php

# 6. 安装 Certbot（Let's Encrypt 自动续期）
sudo apt install -y certbot python3-certbot-nginx

# 申请 SSL 证书（见 SSL 部分）
```

### Node.js 环境（如果你需要运行 Node 应用）

```bash
# 方法一：使用 NodeSource 仓库（推荐，支持多版本）
# 安装 Node.js 20.x
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# 验证
node -v
npm -v

# 方法二：使用 nvm（管理多版本）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts      # 安装最新版 LTS
nvm install 20        # 安装特定版本
nvm use 20             # 切换版本
nvm alias default 20  # 设为默认

# 全局安装常用工具
npm install -g pm2 yarn pnpm
```

---

## 数据库安装与优化

### MariaDB 初始配置

```bash
# 登录 MariaDB
sudo mysql -u root -p

# 创建数据库
CREATE DATABASE myapp_production CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

# 创建用户并授权
CREATE USER 'myapp_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON myapp_production.* TO 'myapp_user'@'localhost';
FLUSH PRIVILEGES;

# 查看数据库
SHOW DATABASES;
EXIT;
```

### MySQL/MariaDB 性能优化

```bash
# 查看当前配置
mysql -u root -p -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"

# 编辑 MySQL 配置
sudo nano /etc/mysql/mariadb.conf.d/99-performance.cnf
```

```ini
[mysqld]
# InnoDB 缓冲池大小（建议设为可用内存的 50-70%）
innodb_buffer_pool_size = 512M

# 临时表和排序缓冲区
tmp_table_size = 64M
max_heap_table_size = 64M
sort_buffer_size = 2M
read_buffer_size = 2M
read_rnd_buffer_size = 4M

# 连接数
max_connections = 150

# 慢查询日志
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# 二进制日志（用于主从复制和恢复）
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 7

# 字符集
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

```bash
# 重启 MySQL
sudo systemctl restart mariadb
```

### 数据库备份脚本

```bash
#!/bin/bash
# backup-mysql.sh — MySQL/MariaDB 自动备份脚本

set -euo pipefail

# 配置
DB_USER="myapp_user"
DB_PASS="StrongPassword123!"
DB_NAME="myapp_production"
BACKUP_DIR="/var/backups/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# 创建备份目录
mkdir -p "$BACKUP_DIR"

# 执行备份
echo "[$(date)] 开始备份数据库 $DB_NAME..."
mysqldump -u"$DB_USER" -p"$DB_PASS" --single-transaction \
    --routines --triggers --events \
    --master-data=2 \
    "$DB_NAME" | gzip > "$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz"

# 验证备份文件
if [ -f "$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz" ]; then
    SIZE=$(du -h "$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz" | cut -f1)
    echo "[$(date)] 备份成功！文件大小: $SIZE"
else
    echo "[$(date)] 备份失败！" >&2
    exit 1
fi

# 清理过期备份
find "$BACKUP_DIR" -name "${DB_NAME}_*.sql.gz" -mtime +$RETENTION_DAYS -delete
echo "[$(date)] 清理完成，保留最近 $RETENTION_DAYS 天的备份"

# 可选：上传到远程存储（S3/OSS/又拍云）
# aws s3 cp "$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz" s3://my-bucket/mysql/
```

---

## SSL 证书与 HTTPS

### 使用 Certbot 自动申请（Let's Encrypt 免费证书）

```bash
# 安装 Certbot
sudo apt install -y certbot python3-certbot-nginx

# 申请证书并自动配置 Nginx
sudo certbot --nginx -d your-domain.com -d www.your-domain.com

# 测试自动续期
sudo certbot renew --dry-run

# Certbot 自动续期任务（通常已配置好，可检查）
sudo systemctl status certbot.timer
```

### 手动配置 SSL（已有证书）

```nginx
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;
    return 301 https://$host$request_uri;  # 强制跳转 HTTPS
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name your-domain.com www.your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    
    # SSL 配置（Mozilla 现代兼容性配置）
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    
    # HSTS（启用前确保所有资源走 HTTPS）
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 1.1.1.1 8.8.8.8 valid=300s;
    resolver_timeout 5s;

    root /var/www/html;
    index index.php index.html;
    
    # ... 其余配置同上
}
```

---

## Docker 与容器化

### Docker 安装（Ubuntu 22.04+）

```bash
# 卸载旧版本
sudo apt remove -y docker docker-engine docker.io containerd runc

# 安装依赖
sudo apt install -y ca-certificates curl gnupg lsb-release

# 添加 Docker GPG 密钥和仓库
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 启动并设置开机自启
sudo systemctl enable --now docker

# 当前用户加入 docker 组（无需 sudo 执行 docker）
sudo usermod -aG docker $USER
newgrp docker

# 验证
docker run --rm hello-world
docker ps
docker --version
```

### Docker Compose 部署示例（WordPress）

```yaml
# docker-compose.yml
version: '3.9'

services:
  db:
    image: mariadb:10.11
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wp_user
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - wp_network

  wordpress:
    image: wordpress:6.3-php8.2-apache
    restart: always
    depends_on:
      - db
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wp_data:/var/www/html
    ports:
      - "8080:80"
    networks:
      - wp_network
    labels:
      - "com.centurylinklabs.watchtower.enable=true"

  caddy:
    image: caddy:2-alpine
    restart: always
    depends_on:
      - wordpress
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - wp_network

volumes:
  db_data:
  wp_data:
  caddy_data:
  caddy_config:

networks:
  wp_network:
    driver: bridge
```

```bash
# 启动
docker compose up -d

# 查看日志
docker compose logs -f

# 停止
docker compose down

# 更新
docker compose pull
docker compose up -d
```

---

## 监控与日志

### htop / btop 系统监控

```bash
# 安装 htop（交互式进程监控）
sudo apt install -y htop

# 安装 btop（更现代，支持图表）
sudo apt install -y btop

# 运行
htop
# 或
btop
```

### Uptime Kuma 自托管监控

```bash
# 使用 Docker 快速部署 Uptime Kuma
docker run -d \
  --name uptime-kuma \
  -p 3001:3001 \
  -v uptime-kuma-data:/app/data \
  --restart unless-stopped \
  louislam/uptime-kuma:1

# 访问 http://YOUR_VPS_IP:3001 完成初始化
# 添加监控项：HTTP(s) 监控、DNS 监控、端口监控、Ping 监控
```

### 日志管理

```bash
# journalctl 基本用法
sudo journalctl -u nginx --since "1 hour ago"    # 查看 nginx 日志
sudo journalctl -u mysql --since today          # 查看 MySQL 日志
sudo journalctl -f                              # 实时跟踪日志
sudo journalctl --disk-usage                     # 查看日志占用空间
sudo journalctl --vacuum-size=500M              # 限制日志大小

# Nginx 日志分析
sudo apt install -y goaccess
goaccess /var/log/nginx/access.log -o /var/www/html/report.html --log-format=COMBINED

# 清理日志文件（使用前请确认）
sudo journalctl --vacuum-time=7d    # 仅保留7天日志
sudo find /var/log -name "*.gz" -mtime +30 -delete  # 清理30天前的压缩日志
```

---

## 自动化运维

### 自动安全更新

```bash
# 安装 unattended-upgrades
sudo apt install -y unattended-upgrades

# 配置自动更新
sudo dpkg-reconfigure -plow unattended-upgrades
# 选择"是"启用自动更新

# 配置详细参数
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

```java
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";
    "${distro_id}:${distro_codename}-security";
    // "${distro_id}:${distro_codename}-updates";
};

Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::InstallOnShutdown "false";
Unattended-Upgrade::Mail "your_email@example.com";  // 通知邮箱
Unattended-Upgrade::MailReport "on-change";          // 仅变更时发送
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";        // 生产环境建议 false
// Unattended-Upgrade::Automatic-Reboot "true";      // 必要时自动重启
// Unattended-Upgrade::Automatic-Reboot-Time "02:00"; // 凌晨2点重启
```

### Crontab 定时任务

```bash
# 查看当前 crontab
crontab -l

# 编辑 crontab
crontab -e

# 添加以下任务：
```

```cron
# 系统维护
0 3 * * * /usr/bin/apt update && /usr/bin/apt upgrade -y >> /var/log/apt-upgrade.log 2>&1

# 数据库每日备份（凌晨2点）
0 2 * * * /root/scripts/backup-mysql.sh >> /var/log/mysql-backup.log 2>&1

# 日志清理（每周日凌晨3点）
0 3 * * 0 find /var/log -name "*.gz" -mtime +30 -delete

# SSL 证书续期检查（每周一凌晨4点）
0 4 * * 1 /usr/bin/certbot renew --renew-hook "/usr/sbin/service nginx reload" >> /var/log/letsencrypt-renewal.log 2>&1

# Docker 镜像清理（每周一凌晨5点）
0 5 * * 1 /usr/bin/docker system prune -f --volumes >> /var/log/docker-cleanup.log 2>&1

# 磁盘空间告警（每天早上9点）
0 9 * * * df -h | awk 'NR==1 || $5+0 > 80 {system("echo Disk alert on " $1 " at " $5 | mail -s VPS_Disk_Alert root")}'
```

---

## 故障排查手册

### 网络连接问题

```bash
# 1. 检查网络接口状态
ip addr
ip link show

# 2. 检查路由表
ip route
ip -6 route  # IPv6

# 3. 测试 DNS 解析
nslookup google.com
dig google.com

# 4. 测试网络连通性
ping -c 4 8.8.8.8          # Ping IP
ping -c 4 google.com       # Ping 域名
traceroute google.com      # 路由追踪
mtr google.com             # 实时路由监控

# 5. 检查端口连通性
nc -zv google.com 443      # 测试 TCP 连接
nc -zvu 8.8.8.8 53         # 测试 UDP 连接

# 6. 检查防火墙规则
sudo iptables -L -n -v
sudo ufw status verbose

# 7. 检查 systemd-networkd（某些系统）
systemctl status systemd-networkd
networkctl status
```

### 服务启动失败

```bash
# 1. 查看服务状态
sudo systemctl status nginx
sudo systemctl status mysql

# 2. 查看详细日志
sudo journalctl -u nginx --no-pager -n 50

# 3. 手动测试配置语法
sudo nginx -t
sudo php-fpm8.2 -t

# 4. 查看端口占用
sudo ss -tlnp | grep :80
sudo lsof -i :80

# 5. 常见错误处理
# Nginx: Address already in use
# → 多个 Nginx 进程，killall nginx 后重启
sudo killall nginx
sudo systemctl start nginx

# MySQL: Can't connect to local MySQL server
# → 检查 MySQL 是否运行
sudo systemctl status mysql
sudo systemctl restart mysql

# PHP-FPM: Connection refused
# → 检查 socket 文件是否存在
ls -la /run/php/php*-fpm.sock
```

### 磁盘空间不足

```bash
# 查看磁盘使用情况
df -h
df -i      # inode 使用情况

# 找出大文件
sudo du -sh /* 2>/dev/null | sort -h | tail -20

# 常见占用来源
du -sh /var/log/*      # 日志文件
du -sh /var/cache/*     # 缓存
du -sh /tmp/*           # 临时文件
du -sh /root/*          # root 用户目录

# Docker 清理
docker system df           # 查看 Docker 占用
docker system prune -a     # 清理所有未使用资源

# 大日志文件处理
sudo truncate -s 0 /var/log/nginx/access.log   # 清空日志（不删除文件）
sudo logrotate -f /etc/logrotate.conf           # 强制执行日志轮转
```

### 内存溢出 (OOM)

```bash
# 查看内存使用
free -h
cat /proc/meminfo

# 查看 OOM Killer 日志
dmesg | grep -i "out of memory"
journalctl -b | grep -i "out of memory"

# 找出内存占用最高的进程
ps aux --sort=-%mem | head -10

# 调整 swappiness（减少 swap 使用）
cat /proc/sys/vm/swappiness
sudo sysctl vm.swappiness=10
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
```

---

## 生产环境检查清单

### 上线前必查

```
安全检查：
☐ SSH 禁用密码登录，仅允许密钥
☐ SSH 端口改为非标准端口（>1024）
☐ root 登录禁用
☐ fail2ban 已启用并配置
☐ 防火墙已配置（仅开放必要端口）
☐ 系统和软件包已更新到最新
☐ sudo 用户已创建，普通用户无法直接 SSH
☐ 未使用默认密码
☐ SSL 证书已配置（HTTPS 强制）
☐ 无敏感文件（如 .env, config.php）被 Web 访问

性能检查：
☐ BBR 已启用
☐ 时区已设置正确
☐ NTP 时间同步已启用
☐ 磁盘 IO 正常（dd 测试 > 100MB/s）
☐ 网络延迟正常（到目标用户 < 150ms）
☐ 内存无泄漏（长期运行观察）
☐ 进程数正常（无异常高占用进程）

监控检查：
☐ 监控告警已配置（Uptime Kuma / 外链监控）
☐ 日志已配置（本地或远程）
☐ 备份策略已设置（数据库 + 文件）
☐ 恢复测试已执行（从备份恢复一次）
☐ DNS 已配置（A 记录 + CNAME）
☐ SSL 证书自动续期已配置

运维检查：
☐ 文档已更新（配置记录、密码记录）
☐ 应急联系方式已保存
☐ 定期维护日历已设置
☐ 成本监控已配置（防止意外超额）
```

---

## 总结

### 本仓库提供的核心价值

1. **完整的配置路径**：从 SSH 登录到生产环境的端到端指南
2. **安全加固体系**：SSH、防火墙、fail2ban 的系统化配置
3. **性能优化方案**：BBR、网络参数、内核调优的实战经验
4. **容器化部署**：Docker Compose 标准化部署方案
5. **自动化运维**：Crontab、自动更新、监控告警的完整生态

### 行动清单

- [ ] 初始化新购 VPS（系统更新、时区、用户）
- [ ] 配置 SSH 密钥登录 + 修改端口 + fail2ban
- [ ] 配置 UFW 防火墙
- [ ] 应用 BBR + 网络参数优化
- [ ] 部署 LEMP 或 Docker 环境
- [ ] 配置 SSL 证书（Certbot）
- [ ] 设置监控（Uptime Kuma）和备份脚本
- [ ] 执行生产环境检查清单

---

**VPS 与机场双向指南（保留全部推广入口）**：

> VPS 专业配置与运维教程 → [ClashHub](https://clashhub.net)  
> 订阅管理 / 机场节点优选 → [ClashVIP 导航站](https://nav.clashvip.net)  
> 高速稳定机场推荐 → [VPSVIP](https://vpsvip.net)  
> 代理协议与工具配置 → [Clash For Windows 官网](https://www.clash-for-windows.net)  
> 用户交流与技术讨论 → [ClashHub 论坛](https://bbs.clashhub.net)

---

*本手册内容基于公开技术资料整理，供学习参考。使用前请评估自身业务需求，合理配置服务器资源。*

*最后更新：2026-08-27 | 仓库：vps-review-20260409*
