
# SSPanel-UIM 完整安装教程

> **环境：** Ubuntu 22.04 · MariaDB 11.8 · PHP 8.4 · Nginx 官方源  
> **域名示例：** tlte.top  
> 本教程已实际验证，解决了 inet4 类型错误、数据库连接失败、PHP 版本错误、Composer 版本过旧、Nginx 用户权限等所有常见坑。

-----

## 系统要求

|项目      |要求                                        |
|--------|------------------------------------------|
|操作系统    |Ubuntu 20.04 / 22.04 LTS（推荐 22.04）        |
|CPU / 内存|最低 1核 1GB，推荐 2核 2GB                       |
|数据库     |**MariaDB 11.8**（必须，不可用 MySQL 或旧版 MariaDB）|
|PHP     |**8.4**（必须明确指定版本）                         |
|Nginx   |**1.24+**（必须用官方源安装）                       |
|Composer|**2.7+**（必须用官网脚本安装）                       |

-----

## 第一步：系统初始化

```bash
apt update && apt upgrade -y

apt install -y curl wget git unzip software-properties-common \
  apt-transport-https ca-certificates gnupg lsb-release

timedatectl set-timezone Asia/Shanghai

ufw disable
```

-----

## 第二步：安装 Nginx（官方源）

> ⚠️ **不能直接 `apt install nginx`**，Ubuntu 自带源版本过旧。必须添加 Nginx 官方源。

```bash
# 添加官方签名
curl -fsSL https://nginx.org/keys/nginx_signing.key | \
  gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg

# 添加官方源
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
  http://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" \
  > /etc/apt/sources.list.d/nginx.list

# 安装
apt update && apt install -y nginx

systemctl start nginx && systemctl enable nginx

# 验证版本（应为 1.24+）
nginx -v
```

-----

## 第三步：安装 PHP 8.4（必须指定版本号）

> ⚠️ Ubuntu 22.04 即使添加了 PPA，直接 `apt install php` 仍会装 **8.1**。  
> **所有包名必须带 `8.4`，否则装出来的是 8.1。**

```bash
# 添加 PPA 源
add-apt-repository ppa:ondrej/php -y
apt update

# 关键：所有包名指定 8.4
apt install -y php8.4-{bcmath,bz2,cli,common,curl,fpm,gd,gmp,intl,mbstring,mysql,opcache,readline,redis,soap,xml,yaml,zip}

# 验证版本（必须是 8.4.x）
php -v

# 配置 PHP 参数
sed -i 's/^max_execution_time.*/max_execution_time = 300/' /etc/php/8.4/fpm/php.ini
sed -i 's/^memory_limit.*/memory_limit = 256M/' /etc/php/8.4/fpm/php.ini
sed -i 's/^post_max_size.*/post_max_size = 50M/' /etc/php/8.4/fpm/php.ini
sed -i 's/^upload_max_filesize.*/upload_max_filesize = 50M/' /etc/php/8.4/fpm/php.ini
sed -i 's/^;date.timezone.*/date.timezone = Asia\/Shanghai/' /etc/php/8.4/fpm/php.ini

# 配置 PHP-FPM socket 权限
sed -i 's/^;listen.owner.*/listen.owner = www-data/' /etc/php/8.4/fpm/pool.d/www.conf
sed -i 's/^;listen.group.*/listen.group = www-data/' /etc/php/8.4/fpm/pool.d/www.conf
sed -i 's/^;listen.mode.*/listen.mode = 0660/' /etc/php/8.4/fpm/pool.d/www.conf

systemctl restart php8.4-fpm && systemctl enable php8.4-fpm
```

-----

## 第四步：安装 MariaDB 11.8（核心步骤）

> ⚠️ **必须安装 11.8**，Ubuntu 自带的 10.6 不支持 `inet4` 类型，会导致数据库初始化失败。

```bash
# 添加官方源
mkdir -p /etc/apt/keyrings
curl -o /etc/apt/keyrings/mariadb-keyring.pgp \
  'https://mariadb.org/mariadb_release_signing_key.pgp'

echo "deb [signed-by=/etc/apt/keyrings/mariadb-keyring.pgp] \
  https://deb.mariadb.org/11.8/ubuntu jammy main" \
  > /etc/apt/sources.list.d/mariadb.list

apt update && apt install -y mariadb-server mariadb-client
systemctl start mariadb && systemctl enable mariadb

# 验证版本（必须是 11.8.x）
mariadb --version
```

### 安全初始化

```bash
mariadb-secure-installation
```

按提示依次输入：

- Switch to unix_socket authentication → `n`
- Change root password → `n`
- Remove anonymous users → `y`
- Disallow root login remotely → `y`
- Remove test database → `y`
- Reload privilege tables → `y`

### 创建数据库和用户

```bash
mariadb -u root
```

```sql
CREATE DATABASE sspanel CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'sspanel'@'127.0.0.1' IDENTIFIED BY '你的密码';
CREATE USER 'sspanel'@'localhost' IDENTIFIED BY '你的密码';

GRANT ALL PRIVILEGES ON sspanel.* TO 'sspanel'@'127.0.0.1';
GRANT ALL PRIVILEGES ON sspanel.* TO 'sspanel'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```

> ⚠️ 必须同时创建 `127.0.0.1` 和 `localhost` 两个用户。`localhost` 走 Unix socket，`127.0.0.1` 走 TCP，只建一个会导致 PHP 连接失败。

-----

## 第五步：安装 Redis

```bash
curl -fsSL https://packages.redis.io/gpg | \
  gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] \
  https://packages.redis.io/deb $(lsb_release -cs) main" \
  | tee /etc/apt/sources.list.d/redis.list

apt update && apt install -y redis

sed -i 's/^# bind 127.0.0.1/bind 127.0.0.1/' /etc/redis/redis.conf
echo "maxmemory 256mb" >> /etc/redis/redis.conf
echo "maxmemory-policy allkeys-lru" >> /etc/redis/redis.conf

systemctl restart redis-server && systemctl enable redis-server
```

-----

## 第六步：安装 Composer（官网最新版）

> ⚠️ **不要用 `apt install composer`**，Ubuntu 源里是旧版 2.2.x。  
> 必须用官网脚本安装，才能得到 2.7+ 最新版。

```bash
# 从官网下载安装脚本
php -r "copy('https://getcomposer.org/installer', '/tmp/composer-setup.php');"

# 安装到全局路径
php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer
rm /tmp/composer-setup.php

# 验证版本（应为 2.7+）
composer --version

# 如提示找不到命令，执行：
ln -s /usr/local/bin/composer /usr/bin/composer
```

-----

## 第七步：部署 SSPanel-UIM

```bash
mkdir -p /var/www/sspanel && cd /var/www
git clone https://github.com/Anankke/SSPanel-UIM.git sspanel
cd sspanel

git config --global --add safe.directory /var/www/sspanel

# 删除 composer.lock（PHP 8.4 兼容性处理，避免依赖冲突）
rm -f composer.lock

# 安装依赖
composer install --no-dev --optimize-autoloader

# 确认安装成功（必须存在此文件）
ls vendor/autoload.php

# 复制配置文件
cp config/.config.example.php config/.config.php
cp config/appprofile.example.php config/appprofile.php
```

-----

## 第八步：配置 SSPanel

```bash
nano /var/www/sspanel/config/.config.php
```

修改以下内容：

```php
$_ENV['baseUrl'] = 'https://tlte.top';

// ⚠️ 必须用 127.0.0.1，不能用 localhost
$_ENV['db_host'] = '127.0.0.1';
$_ENV['db_database'] = 'sspanel';
$_ENV['db_username'] = 'sspanel';
$_ENV['db_password'] = '第四步设置的密码';
$_ENV['db_port'] = '3306';
```

-----

## 第九步：设置目录权限

```bash
chown -R www-data:www-data /var/www/sspanel
find /var/www/sspanel -type d -exec chmod 755 {} \;
find /var/www/sspanel -type f -exec chmod 644 {} \;

chmod -R 777 /var/www/sspanel/storage
chmod 775 /var/www/sspanel/public/clients

mkdir -p /var/www/sspanel/storage/framework/smarty/{cache,compile}
mkdir -p /var/www/sspanel/storage/framework/twig/cache
chmod -R 777 /var/www/sspanel/storage/framework

chmod 664 /var/www/sspanel/config/.config.php
chmod 664 /var/www/sspanel/config/appprofile.php
```

-----

## 第十步：配置 Nginx

> ℹ️ 用 `tee` 写配置文件，避免直接粘贴时内容截断。

```bash
tee /etc/nginx/conf.d/sspanel.conf > /dev/null << 'NGINXEOF'
server {
    listen 80;
    listen [::]:80;
    server_name tlte.top;

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

### 修改 Nginx 运行用户（必须执行）

> ⚠️ Nginx 官方源安装后默认以 `nginx` 用户运行，而 PHP-FPM 和网站文件所有者是 `www-data`，不一致会导致 502 错误或权限报错。必须改为 `www-data`。

```bash
nano /etc/nginx/nginx.conf
```

找到文件顶部的：

```nginx
user nginx;
```

改为：

```nginx
user www-data;
```

保存后重启：

```bash
systemctl restart nginx
```

-----

## 第十一步：初始化数据库

> ℹ️ 执行前确认：`vendor/autoload.php` 存在，`config/.config.php` 已填写正确的数据库信息。

```bash
cd /var/www/sspanel

# 初始化（全新安装，数据库必须为空）
php xcat Migration new 2>&1

# 更新到最新版本
php xcat Migration latest

# 导入默认配置
php xcat Tool importSetting

# 创建管理员账户（按提示输入邮箱和密码）
php xcat Tool createAdmin
```

-----

## 第十二步：申请 SSL 证书

> ⚠️ 确保域名 `tlte.top` 已解析到本服务器 IP，否则证书申请会失败。

```bash
apt install -y certbot python3-certbot-nginx
certbot --nginx -d tlte.top
systemctl enable certbot-renew.timer
```

按提示操作：输入邮箱 → 同意条款输入 `y` → 强制 HTTPS 选 `2`

-----

## 第十三步：配置定时任务

```bash
crontab -e
```

选编辑器时选 `1`（nano），在文件末尾添加：

```
*/5 * * * * /usr/bin/php /var/www/sspanel/xcat Cron >> /var/log/sspanel-cron.log 2>&1
```

保存：`Ctrl+X` → `Y` → 回车

-----

## 验收检查

```bash
systemctl status nginx
systemctl status php8.4-fpm
systemctl status mariadb
systemctl status redis-server
crontab -l
```

|前台地址            |后台地址                  |
|----------------|----------------------|
|https://tlte.top|https://tlte.top/admin|

-----

## 常见错误速查

|错误信息                                         |原因                       |解决方法                              |
|---------------------------------------------|-------------------------|----------------------------------|
|`Unknown data type: 'inet4'`                 |MariaDB 版本低于 10.10       |必须用官方源安装 MariaDB 11.8             |
|`BLOB/TEXT column can't have a default value`|MySQL 8.0 不支持此语法         |换用 MariaDB 11.8，不能用 MySQL         |
|`Table 'sspanel.user' doesn't exist`         |Migration 中途失败           |清空数据库重跑 `Migration new`           |
|数据库连接失败                                      |`db_host` 写的 `localhost` |配置文件改为 `127.0.0.1`                |
|`502 Bad Gateway`                            |Nginx 用户与 PHP-FPM 不一致    |`nginx.conf` 改 `user` 为 `www-data`|
|PHP 装成了 8.1                                  |包名未指定版本号                 |包名全部改为 `php8.4-xxx`               |
|Composer 版本 2.2.x                            |用了 `apt install composer`|改用官网脚本安装                          |
|Nginx 版本过旧 / 安装报错                            |用了 Ubuntu 自带源            |改用 Nginx 官方源安装                    |

-----

> 整理于 2026年4月，基于 Ubuntu 22.04 + MariaDB 11.8 + PHP 8.4 + Nginx 官方源 实际安装验证。
