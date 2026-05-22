# 腾讯云部署指南

将 Token Gateway 部署到腾讯云轻量应用服务器 / CVM。

---

## 1. 服务器选购建议

| 项目 | 推荐配置 | 最低配置 |
|------|---------|---------|
| **CPU** | 2 核+ | 2 核 |
| **内存** | 4 GB+ | 2 GB |
| **系统盘** | 60 GB SSD | 40 GB |
| **系统** | Ubuntu 24.04 LTS | Ubuntu 22.04 LTS |
| **带宽** | 5 Mbps+ | 3 Mbps |

> 💡 轻量应用服务器性价比高，适合中小规模；CVM 更灵活（可选按量计费先测试）。

---

## 2. 安全组配置

在腾讯云控制台 → 安全组，开放以下端口：

| 端口 | 协议 | 来源 | 说明 |
|------|------|------|------|
| 22 | TCP | 你的 IP | SSH 管理 |
| 80 | TCP | 0.0.0.0/0 | HTTP（Let's Encrypt 验证） |
| 443 | TCP | 0.0.0.0/0 | HTTPS（API 服务） |
| 8080 | TCP | 127.0.0.1 | 后端（仅本地访问，通过 Nginx 反代） |

---

## 3. SSH 登录 & 基础环境

```bash
ssh root@<你的服务器IP>
```

### 3.1 更新系统 & 安装基础工具

```bash
apt update && apt upgrade -y
apt install -y curl wget git vim ufw
```

### 3.2 安装 Docker

```bash
curl -fsSL https://get.docker.com | bash
systemctl enable docker
docker --version
```

### 3.3 安装 Docker Compose

```bash
apt install -y docker-compose-plugin
docker compose version
```

---

## 4. 部署 Token Gateway

### 4.1 克隆仓库

```bash
cd /opt
git clone https://github.com/PeterKZhao/token-gateway.git
cd token-gateway/deploy
```

### 4.2 配置环境变量

```bash
cp .env.example .env

# 用 vim 编辑 .env
vim .env
```

**必须修改的配置：**

```bash
# 生成随机密码（复制下面三条命令的输出）
openssl rand -hex 32   # → 填到 POSTGRES_PASSWORD
openssl rand -hex 32   # → 填到 JWT_SECRET
openssl rand -hex 32   # → 填到 TOTP_ENCRYPTION_KEY

# .env 示例
POSTGRES_PASSWORD=abc123def456...
JWT_SECRET=abc123def456...
TOTP_ENCRYPTION_KEY=abc123def456...

# 管理员账号（首次自动创建）
ADMIN_EMAIL=admin@你的域名.com
ADMIN_PASSWORD=你的安全密码

# 时区
TZ=Asia/Shanghai
```

### 4.3 启动服务

```bash
docker compose -f docker-compose.local.yml up -d
```

### 4.4 检查状态

```bash
docker compose -f docker-compose.local.yml ps
docker compose -f docker-compose.local.yml logs -f sub2api
```

看到 `Server starting on 0.0.0.0:8080` 即启动成功。

---

## 5. 配置 Nginx 反向代理 + HTTPS

### 5.1 安装 Nginx & Certbot

```bash
apt install -y nginx certbot python3-certbot-nginx
```

### 5.2 先配置 HTTP（用于证书验证）

```bash
vim /etc/nginx/sites-available/token-gateway
```

```nginx
server {
    listen 80;
    server_name api.你的域名.com;   # 替换为你的域名

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;

        # 关键：允许带下划线的 header（sticky session 需要）
        underscores_in_headers on;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/token-gateway /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```

### 5.3 申请 SSL 证书

```bash
certbot --nginx -d api.你的域名.com
```

选 `2` 自动重定向 HTTP → HTTPS。

---

## 6. 访问管理后台

浏览器打开 `https://api.你的域名.com`

用 `.env` 中设置的 `ADMIN_EMAIL` 和 `ADMIN_PASSWORD` 登录。

### 初始化步骤：
1. 登录管理后台
2. 在「设置」中配置站点名称、Logo
3. 在「账号管理」中添加上游 API 账号（你的 Claude/OpenAI 订阅）
4. 在「用户管理」中创建用户/API Key
5. 配置支付（如需）：见 `docs/PAYMENT_CN.md`

---

## 7. 日常运维

### 查看日志
```bash
docker compose -f /opt/token-gateway/deploy/docker-compose.local.yml logs -f --tail=100
```

### 重启服务
```bash
cd /opt/token-gateway/deploy
docker compose -f docker-compose.local.yml restart
```

### 更新代码
```bash
cd /opt/token-gateway
git pull
cd deploy
docker compose -f docker-compose.local.yml up -d --build
```

### 数据备份（PostgreSQL）
```bash
# 备份
docker exec sub2api-postgres pg_dump -U sub2api sub2api > backup_$(date +%Y%m%d).sql

# 恢复
docker exec -i sub2api-postgres psql -U sub2api sub2api < backup_20250101.sql
```

建议配置 crontab 每日自动备份：
```bash
crontab -e
# 每天凌晨 3 点备份，保留 7 天
0 3 * * * docker exec sub2api-postgres pg_dump -U sub2api sub2api > /opt/backups/backup_$(date +\%Y\%m\%d).sql && find /opt/backups/ -name "*.sql" -mtime +7 -delete
```

---

## 8. 监控建议

- **腾讯云云监控**：免费，CPU/内存/磁盘/网络
- **Uptime Kuma**（可选）：自部署的 uptime 监控
- **日志**：`docker compose logs` + journald

---

## 9. 常见问题

### 端口被占用
```bash
lsof -i :8080  # 查看谁在用
```

### 容器启动失败
```bash
docker compose -f docker-compose.local.yml logs sub2api | tail -50
```

### 数据库连接失败
检查 `.env` 中 `POSTGRES_PASSWORD` 是否与 docker-compose 中一致。

---

> 📚 更多文档见原项目 [Wei-Shaw/sub2api](https://github.com/Wei-Shaw/sub2api)
