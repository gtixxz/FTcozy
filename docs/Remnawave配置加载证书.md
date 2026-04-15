第一步：找到证书在哪里
docker exec caddy find /data/caddy/certificates -name "*.crt" -o -name "*.key" 2>/dev/null

第一步：复制证书到固定目录
mkdir -p /opt/remnawave/ssl

docker cp caddy:/data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/panel.tlte.top/panel.tlte.top.crt /opt/remnawave/ssl/fullchain.pem

docker cp caddy:/data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/panel.tlte.top/panel.tlte.top.key /opt/remnawave/ssl/privkey.key

第二步：把证书挂载到节点容器
编辑节点的 docker-compose.yml：
nano /opt/remnanode/docker-compose.yml
在 volumes 里加一行：
    volumes:
      - '/var/log/remnanode:/var/log/remnanode'
      - '/opt/remnawave/ssl:/var/lib/remnawave/configs/xray/ssl:ro'

			
重启节点：
cd /opt/remnanode && docker compose down && docker compose up -d

第三步：设置证书自动更新
Caddy 每 60 天自动续期，续期后需要重新复制。创建一个定时任务：
cat > /opt/remnawave/renew-ssl.sh << 'EOF'
#!/bin/bash
docker cp caddy:/data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/panel.tlte.top/panel.tlte.top.crt /opt/remnawave/ssl/fullchain.pem
docker cp caddy:/data/caddy/certificates/acme-v02.api.letsencrypt.org-directory/panel.tlte.top/panel.tlte.top.key /opt/remnawave/ssl/privkey.key
EOF

chmod +x /opt/remnawave/renew-ssl.sh

# 每天凌晨3点自动同步证书
(crontab -l 2>/dev/null; echo "0 3 * * * /opt/remnawave/renew-ssl.sh") | crontab -

