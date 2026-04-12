这份文档已经为你整理并优化了排版。我修正了原稿中的一些逻辑顺序（例如：先移动目录再执行 composer），统一了代码块格式，并增加了更清晰的步骤指引，方便你直接复制使用或保存为 .md 文件。
# Ubuntu 22.04 安装 SSPanel-UIM 详细指南
本指南适用于 **Ubuntu 22.04 LTS**，包含 Nginx、PHP 8.4、MariaDB 11.8 及 Redis 的完整配置。
## 一、 系统初始化
首先更新系统并安装基础依赖工具，设置时区并关闭防火墙。
```bash
apt update && apt upgrade -y
apt install -y curl wget git unzip software-properties-common apt-transport-https ca-certificates gnupg lsb-release
timedatectl set-timezone Asia/Shanghai
ufw disable
# 更新系统后建议重启一次
# reboot

```
## 二、 安装 Nginx
使用 Nginx 官方源安装最新稳定版。
```bash
# 添加官方签名及源
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" > /etc/apt/sources.list.d/nginx.list

# 安装并启动
apt update && apt install -y nginx
systemctl start nginx && systemctl enable nginx

# 修改 Nginx 运行用户为 www-data 以匹配 PHP-FPM
sed -i 's/^user  *nginx;/user www-data;/' /etc/nginx/nginx.conf
systemctl restart nginx

```
## 三、 安装 PHP 8.4
安装 PHP 8.4 及其必要的扩展组件。
```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
sudo apt install -y php8.4-{bcmath,bz2,cli,common,curl,fpm,gd,gmp,intl,mbstring,mysql,opcache,readline,soap,xml,zip,redis,yaml}

# 优化 PHP 配置参数
sed -i 's/^max_execution_time.*/max_execution_time = 300/' /etc/php/8.4/fpm/php.ini
sed -i 's/^memory_limit.*/memory_limit = 256M/' /etc/php/8.4/fpm/php.ini
sed -i 's/^post_max_size.*/post_max_size = 50M/' /etc/php/8.4/fpm/php.ini
sed -i 's/^upload_max_filesize.*/upload_max_filesize = 50M/' /etc/php/8.4/fpm/php.ini
sed -i 's/^;date.timezone.*/date.timezone = Asia\/Shanghai/' /etc/php/8.4/fpm/php.ini

# 配置 PHP-FPM Socket 权限
sed -i 's/^;listen.owner.*/listen.owner = www-data/' /etc/php/8.4/fpm/pool.d/www.conf
sed -i 's/^;listen.group.*/listen.group = www-data/' /etc/php/8.4/fpm/pool.d/www.conf
sed -i 's/^;listen.mode.*/listen.mode = 0660/' /etc/php/8.4/fpm/pool.d/www.conf

systemctl restart php8.4-fpm && systemctl enable php8.4-fpm

```
## 四、 安装 MariaDB 11.8
使用官方脚本安装指定的 11.8 版本。
```bash
curl -LsS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version="mariadb-11.8"
sudo apt update
sudo apt install -y mariadb-server mariadb-client
sudo systemctl start mariadb && sudo systemctl enable mariadb

# 安全初始化
sudo mysql_secure_installation
# 提示说明：Switch to unix_socket(n), Change root password(n), 其余全部选 y

```
### 创建数据库
```sql
-- 进入数据库
mariadb -u root

-- 执行以下 SQL
CREATE DATABASE sspanel CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'sspanel'@'127.0.0.1' IDENTIFIED BY '9999zhen';
GRANT ALL PRIVILEGES ON sspanel.* TO 'sspanel'@'127.0.0.1';
FLUSH PRIVILEGES;
EXIT;

```
## 五、 安装 Redis & Composer
```bash
# 安装 Redis
curl -fsSL https://packages.redis.io/gpg | gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/redis.list
apt update && apt install -y redis
systemctl restart redis-server && systemctl enable redis-server

# 安装 Composer
php -r "copy('https://getcomposer.org/installer', '/tmp/composer-setup.php');"
php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer
ln -s /usr/local/bin/composer /usr/bin/composer

```
## 六、 部署 SSPanel-UIM 源码
```bash
mkdir -p /var/www && cd /var/www
git clone https://github.com/Anankke/SSPanel-UIM.git sspanel
cd sspanel
git config --global --add safe.directory /var/www/sspanel

# 安装依赖
composer install --no-dev --optimize-autoloader

# 配置文件准备
cp config/.config.example.php config/.config.php
cp config/appprofile.example.php config/appprofile.php

```
### 关键配置修改
使用 nano config/.config.php 修改以下内容：
 * $_ENV['baseUrl'] : 修改为你的域名（如 https://tlte.top）
 * $_ENV['db_host'] : 必须填 127.0.0.1
 * $_ENV['db_password'] : 填写刚才设置的 9999zhen
 * $_ENV['muKey'] : 设置你的通讯密钥
## 七、 设置目录权限
```bash
chown -R www-data:www-data /var/www/sspanel
find /var/www/sspanel -type d -exec chmod 755 {} \;
find /var/www/sspanel -type f -exec chmod 644 {} \;

# 特殊目录权限
chmod -R 777 /var/www/sspanel/storage
chmod 775 /var/www/sspanel/public/clients
mkdir -p /var/www/sspanel/storage/framework/smarty/{cache,compile}
mkdir -p /var/www/sspanel/storage/framework/twig/cache
chmod -R 777 /var/www/sspanel/storage/framework
chmod 664 /var/www/sspanel/config/.config.php

```
## 八、 配置 Nginx 站点
```bash
tee /etc/nginx/conf.d/sspanel.conf > /dev/null << 'NGINXEOF'
server {
    listen 80;
    listen [::]:80;
    server_name dc.tlte.top; # 修改为你的域名

    root /var/www/sspanel/public;
    index index.php;

    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
NGINXEOF

nginx -t && systemctl reload nginx

```
## 九、 初始化数据库与 SSL
### 初始化面板
```bash
cd /var/www/sspanel
php xcat Migration new      # 初始化数据库
php xcat Migration latest   # 更新到最新
php xcat Tool importSetting # 导入默认配置
php xcat Tool createAdmin   # 创建管理员

```
### 申请 SSL 证书
```bash
apt install -y certbot python3-certbot-nginx
certbot --nginx -d dc.tlte.top
systemctl enable certbot-renew.timer

```
## 十、 定时任务与验收
### 配置 Cron
执行 crontab -e 并添加：
```text
*/5 * * * * /usr/bin/php /var/www/sspanel/xcat Cron >> /var/log/sspanel-cron.log 2>&1

```
### 检查服务状态
```bash
systemctl status nginx php8.4-fpm mariadb redis-server
crontab -l

```
