# Remnawave 完整安装部署教程

**Remnawave 是一款基于 Xray-core 的代理管理面板， 专为多节点管理、用户订阅和流量监控而设计。**  它采用 TypeScript 全栈开发（后端 NestJS、前端 React），搭配 PostgreSQL 和 Valkey（Redis 兼容）数据库，通过 Docker 一键部署。相比 3X-UI 等单机面板，Remnawave 的核心优势在于**多节点集中管理、精细的用户流量控制和完善的 REST API**。 本教程基于官方文档（https://docs.rw/）及中文社区实战经验整理，适合零基础用户从头搭建。

-----

## 一、架构概览与系统要求

Remnawave 由三个独立组件构成，理解这一架构是成功部署的关键。

**Remnawave Panel（面板）** 是核心管理中枢，提供 Web 管理界面和 REST API，处理用户管理、节点通信和配置下发。面板本身**不包含 Xray-core**，仅负责管理和调度。  面板监听 `127.0.0.1:3000`，必须通过反向代理暴露到公网。  

**Remnawave Node（节点）** 是轻量容器，内置 Xray-core，以 `network_mode: host` 运行直接暴露端口。 节点通过 HTTP API（SECRET_KEY 认证）接收面板下发的配置，是实际处理代理流量的组件。 

**Subscription Page（订阅页，可选）** 是独立的用户订阅展示页面，运行在 `127.0.0.1:3010`，用于隐藏面板真实地址。 

### 硬件要求

|组件   |操作系统               |内存                 |CPU          |存储   |
|-----|-------------------|-------------------|-------------|-----|
|面板服务器|Ubuntu / Debian（推荐）|最低 **2 GB**，推荐 4 GB|最低 2 核，推荐 4 核|20 GB|
|节点服务器|Ubuntu / Debian（推荐）|最低 **1 GB**        |最低 1 核       |10 GB|

两台服务器都需要安装 **Docker 和 Docker Compose 插件**。 面板和节点可以部署在同一台机器上，也可以分开部署（推荐分开）。

### 你需要提前准备

- 一个域名（用于面板访问和订阅链接），并将 DNS 解析到面板服务器 IP 
- 一台面板服务器和至少一台节点服务器（可以是同一台）
- SSH 终端工具（如 PuTTY、Termius 或系统自带终端）
- （可选）Telegram Bot Token（用于通知推送）
- （可选）Cloudflare 账号（如使用 CF Tunnel）

-----

## 二、面板安装：从零到运行只需三步

### 第 0 步：安装 Docker

SSH 登录面板服务器，执行以下命令一键安装 Docker：

```bash
sudo curl -fsSL https://get.docker.com | sh
```

> **中国大陆用户提示**：如果 Docker 官方源下载缓慢，可以使用国内镜像源。安装完成后可通过 `docker --version` 验证安装是否成功。

### 第 1 步：下载配置文件

创建项目目录并下载官方的 `docker-compose.yml` 和 `.env` 文件： 

```bash
mkdir /opt/remnawave && cd /opt/remnawave

# 下载生产环境 docker-compose 文件
curl -o docker-compose.yml https://raw.githubusercontent.com/remnawave/backend/refs/heads/main/docker-compose-prod.yml

# 下载环境变量模板
curl -o .env https://raw.githubusercontent.com/remnawave/backend/refs/heads/main/.env.sample
```

 

### 第 2 步：配置 .env 文件

这是最关键的一步。先自动生成安全密钥，再手动修改域名配置。

**自动生成 JWT 密钥（必须执行）：** 

```bash
sed -i "s/^JWT_AUTH_SECRET=.*/JWT_AUTH_SECRET=$(openssl rand -hex 64)/" .env && \
sed -i "s/^JWT_API_TOKENS_SECRET=.*/JWT_API_TOKENS_SECRET=$(openssl rand -hex 64)/" .env
```

 

**自动生成 Metrics 和 Webhook 密码：**

```bash
sed -i "s/^METRICS_PASS=.*/METRICS_PASS=$(openssl rand -hex 64)/" .env && \
sed -i "s/^WEBHOOK_SECRET_HEADER=.*/WEBHOOK_SECRET_HEADER=$(openssl rand -hex 64)/" .env
```

 

**修改 PostgreSQL 默认密码（强烈推荐）：** 

```bash
pw=$(openssl rand -hex 24) && \
sed -i "s/^POSTGRES_PASSWORD=.*/POSTGRES_PASSWORD=$pw/" .env && \
sed -i "s|^\(DATABASE_URL=\"postgresql://postgres:\)[^\@]*\(@.*\)|\1$pw\2|" .env
```

**手动编辑域名相关配置：**

```bash
nano .env
```

找到以下两行并修改为你的实际域名：

```ini
# 面板访问域名（用于 CORS 跨域设置）
FRONT_END_DOMAIN=panel.yourdomain.com

# 订阅地址（不带 http/https，末尾不加 /）
SUB_PUBLIC_DOMAIN=panel.yourdomain.com/api/sub
```

按 `Ctrl+O` 保存，`Ctrl+X` 退出。

### 第 3 步：启动容器

```bash
docker compose up -d && docker compose logs -f -t
```

看到日志中出现 `Nest application successfully started` 即表示面板后端启动成功。按 `Ctrl+C` 退出日志查看。

> ⚠️ **重要安全提示**：面板绑定在 `127.0.0.1:3000`，**绝对不要**将 3000 端口直接暴露到公网。你必须配置反向代理才能从外部访问面板。 

-----

## 三、反向代理配置：让面板可从公网访问

Remnawave 支持 Caddy、Nginx、Traefik 和 Cloudflare Tunnel 四种反向代理方案。** Caddy 最简单**（自动申请 SSL），Cloudflare Tunnel 最安全（无需开放任何端口）。

### 方案 A：Caddy（推荐新手使用）

```bash
mkdir -p /opt/remnawave/caddy && cd /opt/remnawave/caddy
```

**创建 Caddyfile：**

```bash
nano Caddyfile
```

写入以下内容（把 `REPLACE_WITH_YOUR_DOMAIN` 替换为你的实际域名）：

```
https://REPLACE_WITH_YOUR_DOMAIN {
    reverse_proxy * http://remnawave:3000
}

:443 {
    tls internal
    respond 204
}
```

 

**创建 Caddy 的 docker-compose.yml：**

```bash
nano docker-compose.yml
```

```yaml
services:
  caddy:
    image: caddy:2.9
    container_name: 'caddy'
    hostname: caddy
    restart: always
    ports:
      - '0.0.0.0:443:443'
      - '0.0.0.0:80:80'
    networks:
      - remnawave-network
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy-ssl-data:/data

networks:
  remnawave-network:
    name: remnawave-network
    driver: bridge
    external: true

volumes:
  caddy-ssl-data:
    driver: local
    external: false
    name: caddy-ssl-data
```

 

启动 Caddy：

```bash
docker compose up -d && docker compose logs -f -t
```

Caddy 会自动为你的域名申请 Let’s Encrypt SSL 证书。稍等片刻后访问 `https://你的域名` 即可看到面板登录页。

### 方案 B：Nginx（需手动管理证书）

```bash
mkdir -p /opt/remnawave/nginx && cd /opt/remnawave/nginx
```

**先用 acme.sh 申请 SSL 证书：**

```bash
sudo apt-get install -y cron socat
curl https://get.acme.sh | sh -s email=你的邮箱@example.com && source ~/.bashrc

acme.sh --issue --standalone -d '你的域名' \
  --key-file /opt/remnawave/nginx/privkey.key \
  --fullchain-file /opt/remnawave/nginx/fullchain.pem \
  --alpn --tlsport 8443
```

**创建 Nginx 的 docker-compose.yml：** 

```yaml
services:
  remnawave-nginx:
    image: nginx:1.28
    container_name: remnawave-nginx
    hostname: remnawave-nginx
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./fullchain.pem:/etc/nginx/ssl/fullchain.pem:ro
      - ./privkey.key:/etc/nginx/ssl/privkey.key:ro
    restart: always
    ports:
      - '0.0.0.0:443:443'
    networks:
      - remnawave-network

networks:
  remnawave-network:
    name: remnawave-network
    driver: bridge
    external: true
```

### 方案 C：Cloudflare Tunnel（无需开放端口，最安全）

这是中文社区推荐的方案，特别适合不想暴露服务器 IP 的场景。  

在 Cloudflare Zero Trust 控制台创建 Tunnel，获取 Token（以 `ey` 开头的长字符串），然后在 `.env` 中添加： 

```ini
CLOUDFLARE_TOKEN=ey你的CF_Tunnel_Token
```

在面板的 `docker-compose.yml` 中添加 cloudflared 服务：

```yaml
  remnawave-cloudflared:
    image: cloudflare/cloudflared:latest
    env_file: .env
    networks:
      - remnawave-network
    restart: always
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TOKEN}
    depends_on:
      - remnawave
```

在 Cloudflare Tunnel 的 Public Hostnames 中配置：面板域名 → `http://remnawave:3000`，订阅域名 → `http://remnawave-subscription-page:3010`。 

-----

## 四、节点部署：让代理真正跑起来

面板装好后还不能代理上网，你需要至少部署一个节点。 节点可以和面板在同一台服务器上，也可以在不同的 VPS 上。

### 第 1 步：节点服务器安装 Docker

SSH 登录节点服务器：

```bash
sudo curl -fsSL https://get.docker.com | sh
mkdir /opt/remnanode && cd /opt/remnanode
```

### 第 2 步：在面板中添加节点

登录面板 Web 界面，进入 **Nodes → Management**，点击右上角 **「+」** 按钮添加新节点。  填写表单时注意：

- **Name**：节点名称（自定义，如 “美国节点-1”）
- **Address**：节点服务器的公网 IP 或域名
- **Node Port**：面板与节点通信的 API 端口（默认 **2222**，不是代理端口）  

填写完成后点击 **「Copy docker-compose.yml」**，系统会自动生成包含 SECRET_KEY 的配置文件。  

### 第 3 步：创建节点配置文件

在节点服务器上创建 docker-compose.yml：

```bash
cd /opt/remnanode && nano docker-compose.yml
```

粘贴从面板复制的内容，格式类似：

```yaml
services:
  remnanode:
    container_name: remnanode
    hostname: remnanode
    image: remnawave/node:latest
    restart: always
    network_mode: host
    environment:
      - NODE_PORT=2222
      - SECRET_KEY="面板自动生成的密钥"
    volumes:
      - '/var/log/remnanode:/var/log/remnanode'
```

### 第 4 步：启动节点

```bash
docker compose up -d && docker compose logs -f -t
```

### 第 5 步：在面板中完成节点配置

回到面板界面，点击 **「Next」**，选择一个 **Config Profile**（Xray 配置模板），点击 **「Create」** 完成。  

> ⚠️ **防火墙安全**：**NODE_PORT（如 2222）必须在防火墙中限制**，仅允许面板服务器 IP 访问。  代理端口（如 443、8443）则需要对所有用户开放。

### 节点日志管理（推荐配置）

```bash
mkdir -p /var/log/remnanode
sudo apt update && sudo apt install -y logrotate
nano /etc/logrotate.d/remnanode
```

写入：

```
/var/log/remnanode/*.log {
    size 50M
    rotate 5
    compress
    missingok
    notifempty
    copytruncate
}
```

-----

## 五、Xray 配置模板：中国用户实战推荐

官方提供的配置模板主要面向俄罗斯用户，**中国用户需要根据实际网络环境自行编写 Xray 配置**。 以下是社区验证过的常用配置。

### VLESS-TCP-Reality（最流行，免域名免证书）

```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [{
    "tag": "VLESS_TCP_REALITY",
    "port": 443,
    "protocol": "vless",
    "settings": { "clients": [], "decryption": "none" },
    "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] },
    "streamSettings": {
      "network": "raw",
      "security": "reality",
      "realitySettings": {
        "show": false,
        "xver": 0,
        "target": "www.cloudflare.com:443",
        "shortIds": [""],
        "privateKey": "面板生成的私钥",
        "serverNames": ["www.cloudflare.com"]
      }
    }
  }],
  "outbounds": [
    { "tag": "DIRECT", "protocol": "freedom" },
    { "tag": "BLOCK", "protocol": "blackhole" }
  ],
  "routing": {
    "rules": [
      { "ip": ["geoip:private"], "outboundTag": "BLOCK" }
    ],
    "domainStrategy": "IPIfNonMatch"
  }
}
```

> **关于 shortIds 和 privateKey**：可以在面板的 Config Profile 编辑界面生成。 如果不熟悉，也可以先在 3X-UI 中创建一个 Reality 节点来随机生成这些值。 `target` 和 `serverNames` 推荐使用 `www.cloudflare.com` 或 `osxapps.itunes.apple.com` 等大型网站。

### Shadowsocks（最简单，适合快速测试）

```json
{
  "log": { "loglevel": "info" },
  "inbounds": [{
    "tag": "SS_IN",
    "port": 51888,
    "protocol": "shadowsocks",
    "settings": { "clients": [], "network": "tcp,udp" },
    "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] }
  }],
  "outbounds": [
    { "tag": "DIRECT", "protocol": "freedom" },
    { "tag": "BLOCK", "protocol": "blackhole" }
  ],
  "routing": { "rules": [] }
}
```

### VLESS-XHTTP-Reality（上下行分离，适合大流量）

```json
{
  "log": { "loglevel": "error" },
  "inbounds": [{
    "tag": "REALITY_XHTTP",
    "port": 80,
    "protocol": "vless",
    "settings": { "clients": [], "decryption": "none" },
    "sniffing": { "enabled": true, "destOverride": ["http", "tls", "quic"] },
    "streamSettings": {
      "network": "xhttp",
      "security": "reality",
      "xhttpSettings": { "host": "", "mode": "auto", "path": "/your_path" },
      "realitySettings": {
        "target": "osxapps.itunes.apple.com:443",
        "shortIds": ["b2c86d5449d237fa"],
        "privateKey": "替换为你的私钥",
        "serverNames": ["osxapps.itunes.apple.com"]
      }
    }
  }],
  "outbounds": [
    { "tag": "direct", "protocol": "freedom" },
    { "tag": "block", "protocol": "blackhole" }
  ]
}
```

-----

## 六、面板初始化：第一次登录后要做什么

面板部署完成后，访问 `https://你的域名`，首次打开会进入注册/登录页面。**第一个注册的用户自动成为超级管理员**。 登录后按以下顺序操作：

**正确的配置逻辑顺序非常重要**，否则用户无法正常使用订阅： 

1. **创建 Config Profile（配置模板）**：进入 Config Profiles 页面，创建一个新的 Xray 配置（粘贴上面的 JSON 模板），保存后系统会自动解析出 Inbound Tag。
1. **添加 Node（节点）**：进入 Nodes → Management，添加节点并关联刚才的 Config Profile。这一步在上面”第四章”已详细说明。
1. **创建 Internal Group（内部分组）**：进入分组管理，创建一个分组（如”默认分组”），并**绑定对应的 Inbound Tag**（如 `VLESS_TCP_REALITY`）。  
1. **创建 Host（主机）**：进入 Hosts 页面，创建主机条目，选择对应的 Inbound Config，填写节点服务器的公网 IP 或域名作为 Address，启用开关。  
1. **创建 User（用户）**：进入 Users 页面，添加用户，选择流量限制和分组。创建后系统自动生成订阅链接，用户导入客户端即可使用。

### API Token 生成（用于订阅页等外部集成）

进入 **Remnawave Settings → API Tokens**，创建新的 API Token，用于订阅页面或其他第三方集成。 

-----

## 七、订阅页面部署（可选但推荐）

订阅页面为用户提供更友好的订阅展示界面，且可以隐藏面板真实地址。

```bash
mkdir -p /opt/remnawave/subscription && cd /opt/remnawave/subscription
```

**创建 docker-compose.yml：**

```yaml
services:
  remnawave-subscription-page:
    image: remnawave/subscription-page:latest
    container_name: remnawave-subscription-page
    hostname: remnawave-subscription-page
    restart: always
    environment:
      - APP_PORT=3010
      - REMNAWAVE_PANEL_URL=http://remnawave:3000
      - REMNAWAVE_API_TOKEN=从面板生成的API_Token
      - META_TITLE=我的订阅
      - META_DESCRIPTION=订阅管理页面
    ports:
      - '127.0.0.1:3010:3010'
    networks:
      - remnawave-network

networks:
  remnawave-network:
    driver: bridge
    external: true
```

 

启动：

```bash
docker compose up -d
```

然后在反向代理中为订阅页面添加一个新的域名解析。如果使用 Caddy，在 Caddyfile 中添加：

```
https://sub.yourdomain.com {
    reverse_proxy * http://remnawave-subscription-page:3010
}
```

 

同时回到面板 `.env`，更新 `SUB_PUBLIC_DOMAIN=sub.yourdomain.com`，然后重启面板容器：

```bash
cd /opt/remnawave && docker compose down remnawave && docker compose up -d
```

-----

## 八、完整的 .env 配置参考

以下是所有重要环境变量的说明，按功能分组：

|变量名                                |说明                     |默认值                                                        |
|-----------------------------------|-----------------------|-----------------------------------------------------------|
|`APP_PORT`                         |面板监听端口                 |`3000`                                                     |
|`METRICS_PORT`                     |Prometheus 指标端口        |`3001`                                                     |
|`API_INSTANCES`                    |API 进程数（`max`/数字/`-1`） |`1`                                                        |
|`DATABASE_URL`                     |PostgreSQL 连接字符串       |`postgresql://postgres:postgres@remnawave-db:5432/postgres`|
|`REDIS_HOST`                       |Redis 主机名              |`remnawave-redis`                                          |
|`REDIS_PORT`                       |Redis 端口               |`6379`                                                     |
|`JWT_AUTH_SECRET`                  |认证 JWT 密钥（≥64 字符）      |必须修改                                                       |
|`JWT_API_TOKENS_SECRET`            |API Token JWT 密钥       |必须修改                                                       |
|`JWT_AUTH_LIFETIME`                |登录有效期（小时，12-168）       |`12`                                                       |
|`FRONT_END_DOMAIN`                 |面板域名（CORS）             |`*`                                                        |
|`SUB_PUBLIC_DOMAIN`                |公开订阅地址                 |必须修改                                                       |
|`IS_TELEGRAM_NOTIFICATIONS_ENABLED`|启用 TG 通知               |`false`                                                    |
|`TELEGRAM_BOT_TOKEN`               |TG Bot Token           |—                                                          |
|`TELEGRAM_NOTIFY_USERS`            |用户通知（chat_id:thread_id）|—                                                          |
|`TELEGRAM_NOTIFY_NODES`            |节点通知                   |—                                                          |
|`SHORT_UUID_LENGTH`                |订阅短链长度（16-64）          |`16`                                                       |
|`HWID_DEVICE_LIMIT_ENABLED`        |硬件 ID 设备限制             |`false`                                                    |

-----

## 九、升级与维护

### 升级面板

```bash
cd /opt/remnawave && docker compose pull && docker compose down && docker compose up -d && docker compose logs -f
```

### 升级节点

```bash
cd /opt/remnanode && docker compose pull && docker compose down && docker compose up -d && docker compose logs -f
```

### 升级订阅页

```bash
cd /opt/remnawave/subscription && docker compose pull && docker compose down && docker compose up -d
```

### 清理旧镜像

```bash
docker image prune
```

> **注意**：修改 `.env` 中的环境变量后，必须执行 `docker compose down && docker compose up -d` 重建容器才能生效，仅 `docker compose restart` **不会**更新环境变量。  

### 一键安装脚本（懒人方案）

社区维护了多个自动安装脚本，适合不想手动配置的用户：

- **DigneZzZ 脚本**（最完善，支持面板/节点/备份）：`sudo bash -c "$(curl -sL https://github.com/DigneZzZ/remnawave-scripts/raw/main/remnawave.sh)" @ install` 
- **RemnaSetup**：`bash <(curl -fsSL raw.githubusercontent.com/Capybara-z/RemnaSetup/refs/heads/main/install.sh)` 
- **AsanFillter AutoSetup**：`wget -O start.sh https://raw.githubusercontent.com/AsanFillter/Remnawave-AutoSetup/main/start.sh && chmod +x start.sh && ./start.sh` 

-----

## 十、常见问题与避坑指南

**面板启动后无法访问？** 确认反向代理已正确配置且启动，检查域名 DNS 是否解析到面板服务器 IP。运行 `docker compose logs remnawave` 查看后端日志是否有报错。

**节点显示离线？** 检查面板服务器能否访问节点的 NODE_PORT（如 2222）。 在面板服务器上执行 `curl http://节点IP:2222` 测试连通性。确认节点防火墙已放行该端口给面板 IP。

**用户订阅链接为空？** 这是最常见的新手问题。必须按照正确顺序完成配置：Config Profile → Node（关联 Config Profile） → Internal Group（绑定 Inbound Tag） → Host（填写节点地址） → User（分配分组）。** 任何一步缺失都会导致订阅为空。** 

**SSL 证书配置在哪里？** 如果使用 VLESS-Vision-TLS（非 Reality），SSL 证书要挂载到**面板容器**而不是节点容器。面板会自动将证书推送到所有节点。 在面板的 docker-compose.yml 中添加：`volumes: - '/opt/remnawave/nginx:/var/lib/remnawave/configs/xray/ssl'`。 

**Docker Hub 下载慢怎么办？** 中国大陆用户可配置 Docker 镜像加速器。编辑 `/etc/docker/daemon.json`，添加国内可用的镜像源后重启 Docker 服务。

**Remnawave 能否对接 SSPanel-UIM？** 经查证，Remnawave 是**完全独立的项目**，不支持与 SSPanel-UIM 直接对接。但它提供了完善的 REST API、Python SDK 和 TypeScript SDK，技术上可以自行开发集成。官方目前仅提供从 Marzban 面板的迁移工具（`remnawave/migrate`）。 

**与 3X-UI 相比该选哪个？** 如果你只有一台服务器、需求简单，3X-UI 更轻量易用。如果你管理**多台节点、需要用户流量计费、想用 XHTTP 上下行分离**等高级功能，Remnawave 明显更强大。

-----

## 结论

Remnawave 的部署虽然比 3X-UI 等单机面板复杂，但其**面板-节点分离的架构**在多节点场景下优势巨大。整个部署过程的核心就是三件事：面板安装（Docker Compose 一键起）、反向代理配置（Caddy 最省心）、节点添加（面板自动生成配置）。中国用户需要特别注意自行编写适合国内网络环境的 Xray 配置模板，官方默认模板不一定适用。建议先用 Shadowsocks 配置跑通全流程，确认面板→节点→订阅链路正常后，再切换到 VLESS-Reality 等高级协议。所有官方文档和 API 规范可在 **https://docs.rw/** 查阅，项目 GitHub 地址为 **https://github.com/remnawave**，Telegram 群组 `@remnawave` 也有活跃的社区支持。
