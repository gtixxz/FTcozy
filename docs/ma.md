
Marzban VPN 面板

Ubuntu 系统完整安装教程

域名：s.tlte.top  |  系统：Ubuntu 22.04  |  2026年整理

 

 

第一章：安装前准备（必须先做）
⚠️ 跳过此步骤是 SSL 证书申请失败的最常见原因，请务必先执行！
1.1 更新系统
apt update && apt upgrade -y
apt install -y curl socat wget nano ufw
reboot
1.2 开放防火墙端口
ufw allow 22      # SSH
ufw allow 80      # HTTP（acme.sh 申请证书需要）
ufw allow 443     # HTTPS / VLESS Reality
ufw allow 8000    # Marzban 面板（临时）
ufw allow 8443    # VLESS Reality（备用）
ufw allow 2053    # VMess WS+TLS
ufw allow 2083    # VLESS WS+TLS
ufw allow 2096    # Trojan TCP+TLS
ufw allow 1080    # Shadowsocks
ufw enable
ufw status
1.3 确认域名 DNS 已解析
在执行证书申请前，必须确认域名 A 记录已指向服务器 IP：
ping s.tlte.top
# 确认返回的 IP 是你的服务器 IP
⚠️ 注意：如果使用 Cloudflare，申请证书期间需要将小云朵设为「仅 DNS（灰色）」，申请成功后再改回橙色。
第二章：安装 Marzban
2.1 执行官方一键安装脚本
sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban.sh)" @ install
安装完成后会自动显示日志，按 Ctrl+C 停止即可。
2.2 创建管理员账号
marzban cli admin create --sudo
按提示输入用户名和密码，创建完成后可通过以下地址临时访问面板：
http://服务器IP:8000/dashboard/
第三章：申请 SSL 证书
3.1 安装 acme.sh
curl https://get.acme.sh | sh -s email=aiweixxz@gmail.com
source ~/.bashrc
3.2 停止占用 80 端口的服务
systemctl stop nginx 2>/dev/null
systemctl stop apache2 2>/dev/null
3.3 申请证书
export DOMAIN=s.tlte.top
 
mkdir -p /var/lib/marzban/certs
 
~/.acme.sh/acme.sh \
 --issue --force --standalone -d "$DOMAIN" \
 --fullchain-file "/var/lib/marzban/certs/$DOMAIN.cer" \
 --key-file "/var/lib/marzban/certs/$DOMAIN.cer.key"
成功后会看到：Your cert is in: /var/lib/marzban/certs/s.tlte.top.cer
3.4 证书失败排查清单
问题

检查命令 / 解决方法

DNS 未解析

ping s.tlte.top 是否返回服务器 IP

80 端口被占用

netstat -tlnp | grep :80

防火墙拦截

ufw allow 80

系统时间不对

timedatectl 检查时间

Cloudflare 代理开着

申请时将小云朵改为灰色（仅DNS）

 

第四章：配置 .env 启用 HTTPS
nano /opt/marzban/.env
在文件末尾添加或修改以下内容：
UVICORN_PORT = 443
UVICORN_SSL_CERTFILE = "/var/lib/marzban/certs/s.tlte.top.cer"
UVICORN_SSL_KEYFILE = "/var/lib/marzban/certs/s.tlte.top.cer.key"
XRAY_SUBSCRIPTION_URL_PREFIX = https://s.tlte.top
保存：Ctrl+S，退出：Ctrl+X，然后重启：
marzban restart
现在可以通过 https://s.tlte.top/dashboard/ 访问面板。
第五章：配置节点协议（xray_config.json）
5.1 生成 VLESS Reality 密钥
docker exec $(docker ps | grep marzban | grep -v node | awk '{print $1}') xray x25519
输出示例（请保存好你自己生成的密钥）：
Private key: GBfyebMQiujmpsUOYWaylfUJvBe7WM8DZgIrEn1UVh8
Public key:  Z-AUAn4sErBulVz42RHvS-FUpfxhMUJwcm9-udoacQ4
⚠️ 注意：只需要 Private key，Public key 不需要填入配置。
5.2 完整 xray_config.json 配置
进入面板 → Core Settings → 清空全部内容 → 粘贴以下配置（已填入你的密钥和域名）：
{
 "log": { "loglevel": "warning" },
 "dns": { "servers": ["8.8.8.8", "8.8.4.4"] },
 "inbounds": [
   {
     "tag": "VLESS_REALITY",
     "listen": "0.0.0.0",
     "port": 8443,
     "protocol": "vless",
     "settings": { "clients": [], "decryption": "none" },
     "streamSettings": {
       "network": "tcp",
       "security": "reality",
       "realitySettings": {
         "show": false,
         "dest": "www.microsoft.com:443",
         "xver": 0,
         "serverNames": ["www.microsoft.com"],
         "privateKey": "GBfyebMQiujmpsUOYWaylfUJvBe7WM8DZgIrEn1UVh8",
         "shortIds": ["abcd1234"]
       }
     },
     "sniffing": { "enabled": true,
       "destOverride": ["http","tls","quic"] }
   },
   {
     "tag": "VMESS_WS_TLS",
     "listen": "0.0.0.0", "port": 2053,
     "protocol": "vmess",
     "settings": { "clients": [] },
     "streamSettings": {
       "network": "ws", "security": "tls",
       "tlsSettings": { "minVersion": "1.2",
         "certificates": [{
           "certificateFile": "/var/lib/marzban/certs/s.tlte.top.cer",
           "keyFile": "/var/lib/marzban/certs/s.tlte.top.cer.key"
         }]
       },
       "wsSettings": { "path": "/vmess", "headers": {} }
     },
     "sniffing": { "enabled": true,
       "destOverride": ["http","tls"] }
   },
   {
     "tag": "VLESS_WS_TLS",
     "listen": "0.0.0.0", "port": 2083,
     "protocol": "vless",
     "settings": { "clients": [], "decryption": "none" },
     "streamSettings": {
       "network": "ws", "security": "tls",
       "tlsSettings": { "minVersion": "1.2",
         "certificates": [{
           "certificateFile": "/var/lib/marzban/certs/s.tlte.top.cer",
           "keyFile": "/var/lib/marzban/certs/s.tlte.top.cer.key"
         }]
       },
       "wsSettings": { "path": "/vless", "headers": {} }
     },
     "sniffing": { "enabled": true,
       "destOverride": ["http","tls"] }
   },
   {
     "tag": "TROJAN_TCP_TLS",
     "listen": "0.0.0.0", "port": 2096,
     "protocol": "trojan",
     "settings": { "clients": [] },
     "streamSettings": {
       "network": "tcp", "security": "tls",
       "tlsSettings": { "minVersion": "1.2",
         "certificates": [{
           "certificateFile": "/var/lib/marzban/certs/s.tlte.top.cer",
           "keyFile": "/var/lib/marzban/certs/s.tlte.top.cer.key"
         }]
       }
     },
     "sniffing": { "enabled": true,
       "destOverride": ["http","tls"] }
   },
   {
     "tag": "SHADOWSOCKS_TCP",
     "listen": "0.0.0.0", "port": 1080,
     "protocol": "shadowsocks",
     "settings": { "clients": [], "network": "tcp,udp" }
   }
 ],
 "outbounds": [
   { "tag": "direct", "protocol": "freedom", "settings": {} },
   { "tag": "block", "protocol": "blackhole", "settings": {} }
 ],
 "routing": {
   "domainStrategy": "IPIfNonMatch",
   "rules": [
     { "type": "field", "ip": ["geoip:private"],
       "outboundTag": "block" },
     { "type": "field", "ip": ["geoip:cn"],
       "outboundTag": "block" }
   ]
 }
}
5.3 协议端口汇总
协议

标签

端口

加密方式

备注

VLESS

VLESS_REALITY

8443

Reality

最强，无需证书

VMess

VMESS_WS_TLS

2053

TLS + WS

需要域名证书

VLESS

VLESS_WS_TLS

2083

TLS + WS

需要域名证书

Trojan

TROJAN_TCP_TLS

2096

TLS

需要域名证书

Shadowsocks

SHADOWSOCKS_TCP

1080

无

兼容性最好

 

第六章：添加用户并连接
6.1 配置节点 Host 地址（重要！）
⚠️ 重要：这是节点连不上的最常见原因，必须检查！
面板 → Hosts → 每个协议的 Address 都要填写 s.tlte.top，端口对应如下：
协议标签

Address

Port

VLESS_REALITY

s.tlte.top

8443

VMESS_WS_TLS

s.tlte.top

2053

VLESS_WS_TLS

s.tlte.top

2083

TROJAN_TCP_TLS

s.tlte.top

2096

SHADOWSOCKS_TCP

s.tlte.top

1080

 

6.2 添加用户
• 面板 → Users → Add User
• 填写 Username（任意）
• Data Limit：流量限制（0 = 无限）
• Expire Date：到期时间（不填 = 永不过期）
• 点 Save 保存
6.3 获取订阅链接
用户保存后，点击用户名旁的链接图标，复制订阅链接，导入到客户端：
客户端

平台

导入方式

Shadowrocket（小火箭）

iOS

添加 → 从 URL 导入

V2rayNG

Android

扫码或复制订阅链接

Clash / ClashX

Mac / PC

添加订阅链接

Nekoray

PC

从剪贴板导入

 

第七章：常用命令速查
命令

作用

marzban status

查看运行状态

marzban restart

重启 Marzban

marzban logs

查看实时日志

marzban update

更新到最新版本

marzban stop

停止服务

marzban start

启动服务

nano /opt/marzban/.env

编辑配置文件

ls /var/lib/marzban/certs/

查看证书文件

 

7.1 证书续期
acme.sh 会自动续期，手动续期命令：
~/.acme.sh/acme.sh --renew -d s.tlte.top --force
marzban restart
7.2 查看 Xray 是否启动成功
marzban logs
# 看到 Xray core started 表示正常
第八章：Node1 连接失败处理
日志出现 Unable to connect to Node1 node 时：
方案一：不需要多节点 → 直接删除（推荐）
• 面板 → Nodes → 找到 Node1 → 点删除
marzban restart
方案二：需要多节点 → 在第二台服务器安装 marzban-node
sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban-node.sh)" @ install
 

本文档根据实际安装调试过程整理 · 官方文档：https://gozargah.github.io/marzban/en/
