# Remnawave 面板安装教程（整理版）

## 1. 系统要求
- Ubuntu 22.04 / 24.04
- 1C1G 起步（推荐 2C2G）
- Docker + Docker Compose
- 域名（可选但推荐）

---

## 2. 安装 Docker

```bash
curl -fsSL https://get.docker.com | sh
docker -v
docker compose version
```

---

## 3. 创建目录

```bash
mkdir -p /opt/remnawave
cd /opt/remnawave
```

---

## 4. 下载配置文件

```bash
curl -o docker-compose.yml https://raw.githubusercontent.com/remnawave/backend/refs/heads/main/docker-compose-prod.yml
curl -o .env https://raw.githubusercontent.com/remnawave/backend/refs/heads/main/.env.sample
```

---

## 5. 生成安全密钥

```bash
sed -i "s/^JWT_AUTH_SECRET=.*/JWT_AUTH_SECRET=$(openssl rand -hex 64)/" .env
sed -i "s/^JWT_API_TOKENS_SECRET=.*/JWT_API_TOKENS_SECRET=$(openssl rand -hex 64)/" .env
sed -i "s/^METRICS_PASS=.*/METRICS_PASS=$(openssl rand -hex 64)/" .env
sed -i "s/^WEBHOOK_SECRET_HEADER=.*/WEBHOOK_SECRET_HEADER=$(openssl rand -hex 64)/" .env
```

---

## 6. 数据库密码（可选）

```bash
pw=$(openssl rand -hex 24)
sed -i "s/^POSTGRES_PASSWORD=.*/POSTGRES_PASSWORD=$pw/" .env
```

---

## 7. 配置域名

编辑 `.env`：

```
FRONT_END_DOMAIN=yourdomain.com
SUB_PUBLIC_DOMAIN=yourdomain.com/api/sub
```

---

## 8. 启动服务

```bash
docker compose up -d
docker compose logs -f
```

---

## 9. 访问面板

```
http://服务器IP:3000
```

或使用域名 + 反向代理：

```
https://yourdomain.com
```

---

## 10. 反向代理建议

推荐：
- Nginx
- Caddy
- Cloudflare Tunnel

不要直接暴露 3000 端口

---

## 11. 节点部署

建议使用：

- 3x-ui 管理 Xray 节点

安装：

```bash
bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

---

## 12. 总体架构

Remnawave Panel（控制中心）
↓
Xray Node（代理节点）
↓
用户订阅链接自动生成
