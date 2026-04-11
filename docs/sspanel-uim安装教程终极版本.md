# SSPanel-UIM 完整安装教程（终极修正版）

**Ubuntu 22.04 · MariaDB 11.8 · PHP 8.4 · Nginx 官方源**

> ℹ️ 本教程基于实际部署验证，已修正所有踩坑问题：目录错误、管道符复制失败、花括号展开失败、数据库用户权限、数据库未创建等。可逐步复制粘贴直接执行。

-----

## 系统要求

|项目      |要求                  |
|--------|--------------------|
|操作系统    |Ubuntu 22.04 LTS（推荐）|
|CPU / 内存|最低 1核1GB，推荐 2核2GB   |
|数据库     |MariaDB 11.8（必须）    |
|PHP     |8.4（必须明确指定版本）       |
|Nginx   |1.24+（官方源安装）        |
|Composer|2.7+（官网脚本安装）        |

-----

## 第一步：系统初始化

```bash
apt update && apt upgrade -y

apt install -y curl wget git unzip software-properties-common apt-transport-https ca-certificates gnupg lsb-release

timedatectl set-timezone Asia/Shanghai

ufw disable
```

-----

## 第二步：安装 Nginx（官方源）

> ⚠️ 不能直接 `apt install nginx`，Ubuntu 自带源版本过旧。必须添加 Nginx 官方源。

```bash
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg
```

```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" > /etc/apt/sources.list.d/nginx.list
```

```bash
apt update && apt install -y nginx
systemctl start nginx && systemctl enable nginx
```

```bash
# 验证版本（应为 1.24+）
nginx -v
```

### 修改 Nginx 运行用户（安装后立即执行）

> ⚠️ Nginx 官方源默认以 `nginx` 用户运行，而 PHP-FPM 和文件所有者是 `www-data`，不改会导致 502。必须在配置站点前就改掉。

```bash
sed -i 's/^user  *nginx;/user www-data;/' /etc/nginx/nginx.conf
systemctl restart nginx
```

-----

## 第三步：安装 PHP 8.4（必须指定版本号）

> ⚠️ Ubuntu 22.04 即使添加 PPA，直接 `apt install php` 仍会装 8.1。所有包名必须带 `8.4`！
> 
> ⚠️ 花括号展开 `php8.4-{xxx}` 在某些终端复制粘贴时会失败，下面使用展开后的完整命令，确保不出错。

```bash
add-apt-repository ppa:ondrej/php -y
apt update
```

```bash
apt install -y php8.4-bcmath php8.4-bz2 php8.4-cli php8.4-common php8.4-curl php8.4-fpm php8.4-gd php8.4-gmp php8.4-intl php8.4-mbstring php8.4-mysql php8.4-opcache php8.4-readline php8.4-redis php8.4-soap php8.4-xml php8.4-yaml php8.4-zip
```

```bash
# 验证版本（必须是 8.4.x）
php -v
```

### 配置 PHP 参数

```bash
sed -i 's/^max_execution_time.*/max_execution_time = 300/' /etc/php/8.4/fpm/php.ini
sed -i 's/^memory_limit.*/memory_limit = 256M/' /etc/php/8.4/fpm/php.ini
sed -i 's/^post_max_size.*/post_max_size = 50M/' /etc/php/8.4/fpm/php.ini
sed -i 's/^upload_max_filesize.*/upload_max_filesize = 50M/' /etc/php/8.4/fpm/php.ini
sed -i 's/^;date.timezone.*/date.timezone = Asia\/Shanghai/' /etc/php/8.4/fpm/php.ini
```

### 配置 PHP-FPM Socket 权限

```bash
sed -i 's/^;listen.owner.*/listen.owner = www-data/' /etc/php/8.4/fpm/pool.d/www.conf
sed -i 's/^;listen.group.*/listen.group = www-data/' /etc/php/8.4/fpm/pool.d/www.conf
sed -i 's/^;listen.mode.*/listen.mode = 0660/' /etc/php/8.4/fpm/pool.d/www.conf
```

```bash
systemctl restart php8.4-fpm && systemctl enable php8.4-fpm
```

-----

## 第四步：安装 MariaDB 11.8（核心步骤）

> ⚠️ 必须安装 11.8，Ubuntu 自带的 10.6 不支持 `inet4` 类型，会导致数据库初始化失败。

```bash
mkdir -p /etc/apt/keyrings
curl -o /etc/apt/keyrings/mariadb-keyring.pgp 'https://mariadb.org/mariadb_release_signing_key.pgp'
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/mariadb-keyring.pgp] https://deb.mariadb.org/11.8/ubuntu jammy main" > /etc/apt/sources.list.d/mariadb.list
```

```bash
apt update && apt install -y mariadb-server mariadb-client
systemctl start mariadb && systemctl enable mariadb
```

```bash
# 验证版本（必须是 11.8.x）
mariadb --version
```

### 安全初始化

```bash
mariadb-secure-installation
```

按提示依次输入：

- Switch to unix_socket authentication → **n**
- Change root password → **n**
- Remove anonymous users → **y**
- Disallow root login remotely → **y**
- Remove test database → **y**
- Reload privilege tables → **y**

### 创建数据库和用户

> ⚠️ 必须先创建数据库，否则后续 Migration 会报 `Unknown database 'sspanel'`。
> 
> ⚠️ 必须同时创建 `127.0.0.1` 和 `localhost` 两个用户。`localhost` 走 Unix socket，`127.0.0.1` 走 TCP，只建一个会导致连接失败。
> 
> ⚠️ 下面的 `你的密码` 请替换为你自己的密码，后续配置文件中要用到。

```bash
mariadb -u root -e "
CREATE DATABASE sspanel CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'sspanel'@'127.0.0.1' IDENTIFIED BY '你的密码';
CREATE USER 'sspanel'@'localhost' IDENTIFIED BY '你的密码';
GRANT ALL PRIVILEGES ON sspanel.* TO 'sspanel'@'127.0.0.1';
GRANT ALL PRIVILEGES ON sspanel.* TO 'sspanel'@'localhost';
FLUSH PRIVILEGES;
"
```

验证能否连接（将 `你的密码` 替换为实际密码）：

```bash
mariadb -u sspanel -p你的密码 -h 127.0.0.1 -e "SELECT 1;"
```

如果输出一个表格包含数字 1，说明数据库配置成功。

-----

## 第五步：安装 Redis

```bash
curl -fsSL https://packages.redis.io/gpg | gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
```

```bash
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/redis.list
```

```bash
apt update && apt install -y redis
```

```bash
sed -i 's/^# bind 127.0.0.1/bind 127.0.0.1/' /etc/redis/redis.conf
echo "maxmemory 256mb" >> /etc/redis/redis.conf
echo "maxmemory-policy allkeys-lru" >> /etc/redis/redis.conf
```

```bash
systemctl restart redis-server && systemctl enable redis-server
```

-----

## 第六步：安装 Composer（官网最新版）

> ⚠️ 不要用 `apt install composer`，Ubuntu 源里是旧版 2.2.x。必须用官网脚本安装。

```bash
php -r "copy('https://getcomposer.org/installer', '/tmp/composer-setup.php');"
php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer
rm /tmp/composer-setup.php
```

```bash
# 验证版本（应为 2.7+）
composer --version
```

```bash
# 如提示找不到命令，执行：
ln -s /usr/local/bin/composer /usr/bin/composer
```

-----

## 第七步：部署 SSPanel-UIM

> ⚠️ 必须直接 `cd /var/www` 后 clone，不要提前 `mkdir` 目标目录，否则 git clone 会失败或 clone 到错误位置。

```bash
cd /var/www
git clone https://github.com/Anankke/SSPanel-UIM.git sspanel
cd sspanel
```

```bash
git config --global --add safe.directory /var/www/sspanel
```

```bash
# 删除 composer.lock（PHP 8.4 兼容性处理）
rm -f composer.lock
```

```bash
# 安装依赖
composer install --no-dev --optimize-autoloader
```

```bash
# 确认安装成功（必须存在此文件）
ls vendor/autoload.php
```

```bash
# 复制配置文件
cp config/.config.example.php config/.config.php
cp config/appprofile.example.php config/appprofile.php
```

-----

## 第八步：配置 SSPanel

```bash
nano /var/www/sspanel/config/.config.php
```

修改以下关键项（将域名和密码替换为你自己的）：

```php
$_ENV['baseUrl'] = 'https://你的域名';

// 必须用 127.0.0.1，不能用 localhost
$_ENV['db_host'] = '127.0.0.1';
$_ENV['db_database'] = 'sspanel';
$_ENV['db_username'] = 'sspanel';
$_ENV['db_password'] = '第四步设置的密码';
$_ENV['db_port'] = '3306';
```

> ⚠️ `db_host` 必须填 `127.0.0.1`，填 `localhost` 会导致连接失败。

-----

## 第九步：设置目录权限

```bash
chown -R www-data:www-data /var/www/sspanel
find /var/www/sspanel -type d -exec chmod 755 {} \;
find /var/www/sspanel -type f -exec chmod 644 {} \;
```

```bash
chmod -R 777 /var/www/sspanel/storage
chmod 775 /var/www/sspanel/public/clients
```

```bash
mkdir -p /var/www/sspanel/storage/framework/smarty/{cache,compile}
mkdir -p /var/www/sspanel/storage/framework/twig/cache
chmod -R 777 /var/www/sspanel/storage/framework
```

```bash
chmod 664 /var/www/sspanel/config/.config.php
chmod 664 /var/www/sspanel/config/appprofile.php
```

-----

## 第十步：配置 Nginx 站点

> ℹ️ 将下面的 `你的域名` 替换为你的实际域名（如 `panel.tlte.top`）。

```bash
tee /etc/nginx/conf.d/sspanel.conf > /dev/null << 'NGINXEOF'
server {
    listen 80;
    listen [::]:80;
    server_name 你的域名;

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
```

> ⚠️ 粘贴后需要手动把 `server_name` 后面的 `你的域名` 改成实际域名，或者粘贴前先修改好。

```bash
nginx -t && systemctl reload nginx
```

-----

## 第十一步：初始化数据库

> ℹ️ 执行前确认：
> 
> 1. `vendor/autoload.php` 存在
> 1. `config/.config.php` 已填写正确的数据库信息
> 1. 数据库 `sspanel` 已创建（第四步）
> 1. 数据库用户可以连接（第四步末尾已验证）

> ⚠️ 如果报 `Database Error`，先开启 debug 模式查看具体原因：
> 
> ```bash
> sed -i "s/\$_ENV\['debug'\] = false;/\$_ENV['debug'] = true;/" /var/www/sspanel/config/.config.php
> ```
> 
> 修复后记得改回 `false`。

```bash
cd /var/www/sspanel
php xcat Migration new 2>&1
```

```bash
php xcat Migration latest
```

```bash
php xcat Tool importSetting
```

```bash
# 创建管理员账户（按提示输入邮箱和密码）
php xcat Tool createAdmin
```

初始化完成后，关闭 debug 模式：

```bash
sed -i "s/\$_ENV\['debug'\] = true;/\$_ENV['debug'] = false;/" /var/www/sspanel/config/.config.php
```

-----

## 第十二步：申请 SSL 证书

> ⚠️ 执行前确认域名已解析到本服务器 IP，否则证书申请会失败。

```bash
apt install -y certbot python3-certbot-nginx
certbot --nginx -d 你的域名
systemctl enable certbot-renew.timer
```

按提示操作：输入邮箱 → 同意条款输入 y → 强制 HTTPS 选 2

-----

## 第十三步：配置定时任务

```bash
crontab -e
```

选编辑器时选 1（nano），在文件末尾添加：

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

|前台地址        |后台地址              |
|------------|------------------|
|https://你的域名|https://你的域名/admin|

-----

## 常见错误速查

|错误信息                        |原因                      |解决方法                                              |
|----------------------------|------------------------|--------------------------------------------------|
|`Unknown database 'sspanel'`|忘记创建数据库                 |`mariadb -u root -e "CREATE DATABASE sspanel ..."`|
|`Access denied for user`    |数据库用户未创建或密码错误           |重新 DROP + CREATE USER（见第四步）                       |
|`inet4` 类型错误                |MariaDB < 10.10         |安装 MariaDB 11.8                                   |
|`BLOB/TEXT default` 错误      |MySQL 8.0 不支持           |换用 MariaDB 11.8                                   |
|`user` 表不存在                 |Migration 中途失败          |清空数据库重跑 `Migration new`                           |
|数据库连接失败                     |`db_host` 写了 `localhost`|改为 `127.0.0.1`                                    |
|`Database Error`（无详情）       |debug 模式关闭              |开启 debug 查看具体原因                                   |
|502 Bad Gateway             |Nginx/PHP-FPM 用户不一致     |`nginx.conf` 改 user 为 `www-data`                  |
|PHP 装成 8.1                  |包名未指定版本号                |包名改为 `php8.4-xxx`                                 |
|PHP 包安装失败                   |花括号展开失败                 |使用展开后的完整包名（见第三步）                                  |
|Composer 2.2.x              |用了 `apt install`        |改用官网脚本安装                                          |
|Nginx 版本过旧                  |用了 Ubuntu 自带源           |改用 Nginx 官方源                                      |
|git clone 到错误目录             |先 mkdir 再 clone         |不要提前建目录，直接 `cd /var/www && git clone`             |
|config 目录为空                 |clone 到了其他位置            |确认 `/var/www/sspanel/config/` 存在配置文件              |

-----

## 后续维护建议

- 定期备份数据库：`mariadb-dump -u root sspanel > backup.sql`
- 及时更新 SSPanel：`cd /var/www/sspanel && git pull && composer install --no-dev`
- SSL 证书自动续期已配置（certbot-renew.timer），无需手动操作
- 查看定时任务日志：`tail -f /var/log/sspanel-cron.log`

-----

## 本版修正记录

相比原始教程，本版做了以下关键修正：

1. **PHP 包安装命令改为展开写法**：避免花括号 `{}` 在终端复制粘贴时展开失败
1. **所有管道符统一为半角 `|`**：原文档中部分 `|` 为全角 `│`，复制会报错
1. **每条长命令单独一个代码块**：避免多行反斜杠 `\` 换行在复制时丢失
1. **第七步去掉 `mkdir -p`**：避免目录已存在导致 git clone 失败或 clone 到错误位置
1. **Nginx 用户修改提前到第二步**：安装后立即改，避免后续出现 502
1. **第四步合并创建数据库和用户为一条命令**：避免遗漏建库导致 `Unknown database` 错误
1. **第四步增加连接验证**：建完用户后立即测试，确保后续不出问题
1. **第十一步增加 debug 开关说明**：遇到 `Database Error` 时可快速定位原因
1. **常见错误表新增实际踩坑项**：`Unknown database`、`Access denied`、花括号展开、目录错误等
