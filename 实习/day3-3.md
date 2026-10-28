# Nginx 负载均衡、健康检查、Session 保持与 phpMyAdmin 实验笔记

> 本文根据当前 Rocky Linux 9.4 实验环境重写。 重点记录健康检查模块编译、Cookie/Session 现象、PHP 8.2 下 phpMyAdmin 部署，以及实际遇到的问题。

## 一、当前环境事实

### 1. 机器角色

192.168.75.10  lb01   Nginx 负载均衡、MariaDB 测试数据库  
192.168.75.20  web01  Nginx、PHP-FPM、phpMyAdmin  
192.168.75.30  web02  Nginx、PHP-FPM、phpMyAdmin

### 2. 当前软件版本

操作系统：Rocky Linux 9.4  
Nginx：1.20.1  
Nginx 编译时 OpenSSL：3.5.8  
PHP：8.2.35  
phpMyAdmin：5.2.3  
MariaDB：10.5.29

当前 Nginx 已使用第三方模块 `nginx_upstream_check_module` 编译，`nginx -V` 中可以看到：

--add-module=/root/nginx_upstream_check_module-master

当前访问链路：

浏览器  
  |  
  v  
php.test.com  
  |  
  v  
192.168.75.10:80  
  |  
  +--> 192.168.75.20:80  
  |  
  `--> 192.168.75.30:80  
          |  
          `--> 192.168.75.10:3306 MariaDB

## 二、rz 和 sz 文件传输

Rocky Linux 默认可能没有安装 `rz`、`sz`，先安装：

dnf install -y lrzsz

### 1. Windows 上传到 Linux 当前目录

rz  
rz -be  
rz -y

参数说明：

-b  使用 binary 模式  
-e  对控制字符进行转义  
-y  覆盖同名文件

推荐：

rz -be

### 2. Linux 下载到 Windows

sz filename  
sz -be filename  
sz file1 file2

注意：

- `rz`、`sz` 依赖终端支持 ZMODEM。
    
- 使用 Windows Terminal、部分 SSH 工具时可能不兼容。
    
- 不能使用时可以使用 `scp`、WinSCP 或共享目录。
    

## 三、Nginx 负载均衡主动健康检查

### 1. 官方模块与第三方模块

标准 Nginx 自带被动健康检查，例如：

server 192.168.75.20:80 max_fails=2 fail_timeout=10s;

这表示请求失败达到阈值后，Nginx 暂时认为节点不可用。但它不是独立周期的主动探测。

第三方模块项目：

https://github.com/yaoweibin/nginx_upstream_check_module

使用该模块后可以周期探测后端：

check interval=3000 rise=2 fall=3 timeout=1000 type=tcp;

### 2. 健康检查参数

interval=3000  
  每 3000 毫秒检查一次  
  
rise=2  
  连续成功 2 次后标记为 up  
  
fall=3  
  连续失败 3 次后标记为 down  
  
timeout=1000  
  单次检查超时时间为 1000 毫秒  
  
type=tcp  
  使用 TCP 连接检查

还可以使用 HTTP 检查：

check interval=3000 rise=2 fall=3 timeout=1000 type=http;  
check_http_send "HEAD / HTTP/1.0\r\nHost: php.test.com\r\n\r\n";  
check_http_expect_alive http_2xx http_3xx;

TCP 检查只能证明端口可连接，不能证明 PHP 应用正常。生产环境更适合 HTTP 健康检查。

## 四、Rocky Linux 9 编译 Nginx 与 OpenSSL 3 的问题

### 1. 为什么会编译失败

Rocky Linux 9 使用 OpenSSL 3.x。旧版 Nginx 源码中可能调用已经被标记为 deprecated 的 OpenSSL API。

如果编译参数把警告当成错误，会出现：

error: ... is deprecated  
make: *** [objs/...] Error

当前环境实际编译信息：

nginx version: nginx/1.20.1  
built with OpenSSL 3.5.8 25 Aug 2026  
TLS SNI support enabled

说明当前运行的 Nginx 已经成功编译，并且健康检查模块已经加载。如果只是继续做实验，不需要重新编译。

### 2. 方法一：继续使用 Nginx 1.20.1，并忽略废弃 API 警告

进入源码目录：

cd /root/nginx-1.20.1

清除旧编译产物：

make clean

重新配置时，在 `--with-cc-opt` 中增加：

-Wno-error=deprecated-declarations

示例：

./configure \  
  --prefix=/etc/nginx \  
  --sbin-path=/usr/sbin/nginx \  
  --modules-path=/usr/lib64/nginx/modules \  
  --conf-path=/etc/nginx/nginx.conf \  
  --error-log-path=/var/log/nginx/error.log \  
  --http-log-path=/var/log/nginx/access.log \  
  --pid-path=/var/run/nginx.pid \  
  --lock-path=/var/run/nginx.lock \  
  --http-client-body-temp-path=/var/cache/nginx/client_temp \  
  --http-proxy-temp-path=/var/cache/nginx/proxy_temp \  
  --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \  
  --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \  
  --http-scgi-temp-path=/var/cache/nginx/scgi_temp \  
  --user=nginx \  
  --group=nginx \  
  --with-compat \  
  --with-file-aio \  
  --with-threads \  
  --with-http_addition_module \  
  --with-http_auth_request_module \  
  --with-http_dav_module \  
  --with-http_flv_module \  
  --with-http_gunzip_module \  
  --with-http_gzip_static_module \  
  --with-http_mp4_module \  
  --with-http_random_index_module \  
  --with-http_realip_module \  
  --with-http_secure_link_module \  
  --with-http_slice_module \  
  --with-http_ssl_module \  
  --with-http_stub_status_module \  
  --with-http_sub_module \  
  --with-http_v2_module \  
  --with-mail \  
  --with-mail_ssl_module \  
  --with-stream \  
  --with-stream_realip_module \  
  --with-stream_ssl_module \  
  --with-stream_ssl_preread_module \  
  --add-module=/root/nginx_upstream_check_module-master \  
  --with-cc-opt="-O2 -g -pipe -Wall -Wno-error=deprecated-declarations -fexceptions -fstack-protector-strong -m64 -mtune=generic -fPIC" \  
  --with-ld-opt="-Wl,-z,relro -Wl,-z,now -pie"

编译安装：

make -j2  
make install

创建缓存目录：

mkdir -p /var/cache/nginx/client_temp  
mkdir -p /var/cache/nginx/proxy_temp  
mkdir -p /var/cache/nginx/fastcgi_temp  
mkdir -p /var/cache/nginx/uwsgi_temp  
mkdir -p /var/cache/nginx/scgi_temp

检查配置并重载：

nginx -t  
systemctl reload nginx

### 3. 方法二：使用较新的 Nginx 源码

旧笔记中写了 `nginx-1.14.2`，但下载命令实际是 `nginx-1.26.3`，这里需要统一。

建议使用较新的稳定版，例如：

cd /root  
wget http://nginx.org/download/nginx-1.26.3.tar.gz  
tar -zxvf nginx-1.26.3.tar.gz  
cd nginx-1.26.3

然后使用与该版本兼容的健康检查补丁。

注意：不能因为 Nginx 升级了就随意使用旧补丁。必须先检查模块目录中是否有匹配当前 Nginx 版本的 patch 文件。

ls /root/nginx_upstream_check_module-master

以下补丁示例只对应 Nginx 1.20.1，升级到 1.26.3 时不能直接照搬：

patch -p1 < /root/nginx_upstream_check_module-master/check_1.20.1+.patch

其中：

p1  忽略补丁路径中的第一层目录  
p0  不忽略目录层级

### 4. 编译时容易踩的坑

1. 反斜杠 `\` 后面不能有空格，否则命令会断开。
    
2. 如果使用环境变量中的 `CFLAGS=-Werror`，单纯设置 `-Wno-error=deprecated-declarations` 可能仍然不够，应同时检查 `--with-cc-opt`。
    
3. `make clean` 只清理当前源码目录，不会恢复 `/usr/sbin/nginx`。
    
4. `make install` 会覆盖当前 Nginx 二进制和默认配置，操作前应备份。
    
5. 自定义编译的 Nginx 可能与系统 RPM 包管理冲突，后续 `dnf update nginx` 可能覆盖自定义二进制。
    

## 五、当前负载均衡与健康检查配置

### 1. 修改前的状态

原配置只写了：

server_name node.test.com;

由于它是第一个加载的 80 端口 server 块，实际会成为默认虚拟主机。访问 `php.test.com` 时虽然没有显式匹配，也会被转入这个反向代理。

这种配置能够“碰巧工作”，但语义不清晰。

### 2. 当前配置

文件：

/etc/nginx/conf.d/node_proxy.conf

当前内容：

upstream node {  
    server 192.168.75.20:80 max_fails=2 fail_timeout=10s;  
    server 192.168.75.30:80 max_fails=2 fail_timeout=10s;  
    check interval=3000 rise=2 fall=3 timeout=1000 type=tcp;  
}  
  
server {  
    listen 80;  
    server_name node.test.com php.test.com;  
  
    location / {  
        proxy_pass http://node;  
        include proxy_params;  
    }  
  
    location /upstream_check {  
        check_status;  
    }  
}

如果旧 Node 项目不再使用，可以把 server_name 改为：

server_name php.test.com;

不要直接删除整个 upstream，因为当前它还负责 php.test.com 的负载均衡。

### 3. 当前健康检查结果

访问：

curl http://php.test.com/upstream_check

实际结果：

Check upstream server number: 2  
  
192.168.75.20:80  up  
192.168.75.30:80  up

状态字段说明：

up    节点正常  
down  节点异常

`rise`、`fall` 的次数会随着持续检查增长，不同时间查看到的次数不一定相同。

### 4. 生产环境建议

当前 `/upstream_check` 可以直接访问。生产环境建议限制来源：

location = /upstream_check {  
    allow 127.0.0.1;  
    allow 192.168.75.0/24;  
    deny all;  
    check_status;  
}

## 六、Cookie、Session 与负载均衡会话保持

### 1. Cookie 和 Session 的区别

Cookie 保存在浏览器端  
Session 默认保存在服务端

访问 phpMyAdmin 时，浏览器收到的响应头中包含：

Set-Cookie: pma_lang=en; Max-Age=2592000; path=/; HttpOnly; SameSite=Strict  
Set-Cookie: phpMyAdmin=<session-id>; path=/; HttpOnly; SameSite=Strict

实际抓取到的 Cookie：

pma_lang=en  
phpMyAdmin=<动态生成的 Session ID>

注意：

- `pma_lang` 记录语言选择，当前设置了约 30 天有效期。
    
- `phpMyAdmin` 是 PHP 会话 Cookie，通常作为浏览器会话 Cookie 使用。
    
- `HttpOnly` 表示 JavaScript 不能直接读取该 Cookie。
    
- `SameSite=Strict` 表示跨站请求不会携带该 Cookie。
    
- `phpMyAdmin` 的具体值每次可能不同，不能写死到笔记或脚本里。
    

### 2. 一次完整请求流程

1. 浏览器第一次访问 http://php.test.com/  
2. 负载均衡器把请求转发到 web01 或 web02  
3. PHP 在本地 /var/lib/php/session 创建 sess_<id>  
4. PHP 响应 Set-Cookie: phpMyAdmin=<id>  
5. 浏览器后续请求自动携带 Cookie: phpMyAdmin=<id>  
6. 服务端用该 ID 查找对应的 Session 文件

当前 PHP-FPM 配置：

user = apache  
group = apache  
listen = /run/php-fpm/www.sock  
php_value[session.save_path] = /var/lib/php/session

当前 Session 目录：

drwxrwx--- apache:apache /var/lib/php/session

### 3. 当前负载均衡没有 Session 保持

当前 upstream 没有 `ip_hash`，默认使用轮询：

upstream node {  
    server 192.168.75.20:80;  
    server 192.168.75.30:80;  
}

这表示同一个浏览器先后两次请求可能分别命中不同 Web 节点。

但两台机器当前使用各自本地的 Session 文件：

web01: /var/lib/php/session  
web02: /var/lib/php/session

这两个目录不是共享存储。

实际验证时：

浏览器第一次请求产生的 Cookie: phpMyAdmin=ktp3gdcaflgefhnrece16r15j2  
该 Session 文件只出现在 web01：  
/var/lib/php/session/sess_ktp3gdcaflgefhnrece16r15j2  
  
web02 上不存在同一个 Session 文件。

再次通过负载均衡访问时，`phpMyAdmin` Cookie 发生变化。这说明当前轮询链路不能可靠保证登录状态在多个节点之间保持。

典型现象：

第一次请求登录成功  
下一次请求被轮询到另一台 Web 服务器  
另一台机器没有相同 Session  
登录状态丢失，或者重新生成 Cookie

### 4. 解决方式一：ip_hash

根据客户端 IP 固定后端：

upstream node {  
    ip_hash;  
  
    server 192.168.75.20:80 max_fails=2 fail_timeout=10s;  
    server 192.168.75.30:80 max_fails=2 fail_timeout=10s;  
    check interval=3000 rise=2 fall=3 timeout=1000 type=tcp;  
}

优点：

- 配置简单。
    
- 同一客户端 IP 通常固定到同一后端。
    

缺点：

- 当客户端经过 NAT、代理或企业网关时，大量用户可能表现为同一个 IP。
    
- 后端故障切换后，原 Session 仍然可能丢失。
    
- 负载可能不均匀。
    

### 5. 解决方式二：共享 Session

常用方式：

Redis  
Memcached  
共享 NFS 目录  
数据库 Session

Redis 示例流程：

session.save_handler = redis  
session.save_path = "tcp://192.168.75.50:6379"

同时需要安装：

dnf install -y php-redis  
systemctl reload php-fpm

使用 NFS 共享 Session 时，两台 Web 节点的目录所有者、权限、SELinux 和挂载参数必须一致。生产环境更推荐 Redis。

## 七、PHP 与 phpMyAdmin 部署

### 1. 安装 PHP 环境

以下操作在 `web01` 和 `web02` 都执行。

清理可能影响依赖的旧仓库：

rm -f /etc/yum.repos.d/eagle.repo  
mv /etc/yum.repos.d/Rocky-Media.repo /etc/yum.repos.d/Rocky-Media.repo.disabled

安装 Remi 和 EPEL：

dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm  
dnf install -y epel-release

查看并启用 PHP 8.2：

dnf module list php  
dnf module enable php:remi-8.2 -y

安装 PHP-FPM 和常用扩展：

dnf install -y \  
  php \  
  php-fpm \  
  php-mysqlnd \  
  php-mbstring \  
  php-xml \  
  php-gd \  
  php-opcache \  
  php-json \  
  php-zip \  
  php-intl \  
  php-redis

这里不需要安装 `httpd`，因为 Web 服务器使用 Nginx。

启动 PHP-FPM：

systemctl enable --now php-fpm  
systemctl status php-fpm --no-pager  
php -v

当前版本：

PHP 8.2.35

### 2. 安装 phpMyAdmin

当前环境已经安装：

phpMyAdmin 5.2.3  
程序目录：/usr/share/phpMyAdmin  
配置文件：/etc/phpMyAdmin/config.inc.php

重新安装使用：

dnf install -y phpMyAdmin

不要使用旧笔记中的 phpMyAdmin 4.8.4：

phpMyAdmin-4.8.4-all-languages.zip

该版本与 PHP 8.2 不兼容，会报：

PHP Fatal error:  
Unparenthesized `a ? b : c ? d : e` is not supported  
DatabaseInterface.php on line 615

如果实验要求使用压缩包，应使用支持 PHP 8.2 的 5.2.x 版本，并将 `root` 指向实际解压目录。

### 3. Nginx PHP 配置

文件：

/etc/nginx/conf.d/php.conf

当前正确配置：

server {  
    listen 80;  
    server_name php.test.com;  
    root /usr/share/phpMyAdmin;  
  
    location / {  
        index index.php index.html;  
    }  
  
    location ~ \.php$ {  
        fastcgi_pass unix:/run/php-fpm/www.sock;  
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;  
        include fastcgi_params;  
    }  
}

关键点：

root 必须以 / 开头  
FastCGI 必须连接 PHP-FPM 实际监听的地址

当前 PHP-FPM 监听：

listen = /run/php-fpm/www.sock

因此不能写成：

fastcgi_pass 127.0.0.1:9000;

### 4. 配置 phpMyAdmin 连接远程数据库

当前使用 RPM 安装，修改：

vi /etc/phpMyAdmin/config.inc.php

核心配置：

$cfg['Servers'][$i]['auth_type'] = 'cookie';  
$cfg['Servers'][$i]['host'] = '192.168.75.10';  
$cfg['Servers'][$i]['port'] = '3306';  
$cfg['Servers'][$i]['AllowNoPassword'] = false;

`blowfish_secret` 必须替换为至少 32 个字符的随机字符串：

$cfg['blowfish_secret'] = '请替换为足够长的随机字符串';

如果使用手动解压版，配置文件才放在 phpMyAdmin 程序根目录：

cd /path/to/phpMyAdmin  
cp config.sample.inc.php config.inc.php

### 5. PHP Session 权限

错误做法：

chown -R nginx:nginx /var/lib/php/

当前 PHP-FPM 使用 `apache` 用户运行，因此 Session 目录应归属 `apache`：

chown apache:apache /var/lib/php/session  
chmod 770 /var/lib/php/session

不要无差别递归修改整个 `/var/lib/php`。

权限错误时页面会出现：

session_start(): open(SESSION_FILE, O_RDWR) failed:  
Permission denied (13)  
files (path: /var/lib/php/session)

### 6. 检查并重载 Nginx

nginx -t  
systemctl reload nginx

## 八、在 lb01 部署 MariaDB

### 1. 安装和启动

在 `192.168.75.10` 上执行：

dnf install -y mariadb-server mariadb  
systemctl enable --now mariadb

检查状态：

systemctl is-active mariadb  
ss -lntp | grep 3306  
mysqladmin ping

当前实测结果：

mariadb.service: active  
MariaDB 监听：*:3306  
mysqladmin: mysqld is alive

### 2. 设置 root 密码

如果 root 还没有密码：

mysqladmin password '设置一个强密码'

不要重复执行该命令。密码已经设置时再次执行可能失败，或者覆盖现有密码。

### 3. 创建 phpMyAdmin 测试用户

推荐创建专用用户，不建议使用 root：

CREATE USER 'pma_test'@'192.168.75.%' IDENTIFIED BY '设置一个强密码';  
GRANT ALL PRIVILEGES ON *.* TO 'pma_test'@'192.168.75.%' WITH GRANT OPTION;  
FLUSH PRIVILEGES;

当前数据库中已有远程 root 用户：

root@192.168.75.%

这只是实验环境的现有配置，不适合直接用于生产。

### 4. 当前 MariaDB 远程访问情况

当前 `.10` 的 3306 端口可连接，配置中没有限制只监听 localhost。

检查：

ss -lntp | grep 3306  
firewall-cmd --state  
firewall-cmd --list-ports

如果启用了 firewalld，只允许两台 Web 节点访问：

firewall-cmd --permanent \  
  --add-rich-rule='rule family=ipv4 source address=192.168.75.20/32 port port=3306 protocol=tcp accept'  
  
firewall-cmd --permanent \  
  --add-rich-rule='rule family=ipv4 source address=192.168.75.30/32 port port=3306 protocol=tcp accept'  
  
firewall-cmd --reload

## 九、访问验证

### 1. 检查健康状态

curl http://192.168.75.10/upstream_check

期望两台节点均为 `up`。

### 2. 从后端本机验证

在 web01、web02 分别执行：

curl -I -H 'Host: php.test.com' http://127.0.0.1/

期望：

HTTP/1.1 200 OK

### 3. 从负载均衡器验证

curl -I -H 'Host: php.test.com' http://127.0.0.1/

### 4. 从 Windows 验证

curl.exe -I http://php.test.com/

当前结果：

HTTP/1.1 200 OK  
Server: nginx/1.20.1  
X-Powered-By: PHP/8.2.35

### 5. 获取并回放 Cookie

保存 Cookie：

curl -c /tmp/php.cookies -D - http://php.test.com/ -o /tmp/php.html

携带 Cookie 再访问：

curl -b /tmp/php.cookies -c /tmp/php.cookies \  
  -D - http://php.test.com/ -o /dev/null

浏览器中查看：

F12  
Application / Storage  
Cookies  
http://php.test.com

应看到：

phpMyAdmin  
pma_lang

## 十、故障排查表

### 1. 返回 502 Bad Gateway

优先检查 FastCGI 地址：

grep -n fastcgi_pass /etc/nginx/conf.d/php.conf  
nginx -t  
tail -50 /var/log/nginx/error.log

如果看到：

connect() failed (111: Connection refused)  
upstream: fastcgi://127.0.0.1:9000

说明 Nginx 在连接 TCP 9000，但 PHP-FPM 没有监听 9000。

检查 PHP-FPM：

grep -n '^listen' /etc/php-fpm.d/www.conf  
ss -lxnp | grep php  
systemctl status php-fpm

常见修复：

fastcgi_pass unix:/run/php-fpm/www.sock;

### 2. 返回 500，页面提示 Session 权限错误

ls -ld /var/lib/php/session  
grep -nE '^user|^group' /etc/php-fpm.d/www.conf  
tail -50 /var/log/php-fpm/www-error.log

如果 PHP-FPM 用户是 `apache`：

chown apache:apache /var/lib/php/session  
chmod 770 /var/lib/php/session

### 3. 返回 500，日志出现旧三元表达式错误

日志：

Unparenthesized `a ? b : c ? d : e` is not supported

同时检查：

php -v  
nginx -T | grep -n 'root '  
rpm -q phpMyAdmin

如果 PHP 是 8.2，而程序目录是 phpMyAdmin 4.8.4，应切换到：

root /usr/share/phpMyAdmin;

### 4. 页面能打开，但登录状态反复丢失

检查：

grep -Rni 'ip_hash' /etc/nginx  
ls -ld /var/lib/php/session

当前负载均衡默认轮询，且两台 Web 节点各自保存 Session，登录状态不能保证共享。

解决：

增加 ip_hash  
或者使用 Redis/NFS 等共享 Session

### 5. 健康检查显示 down

curl -I http://192.168.75.20/  
curl -I http://192.168.75.30/  
ss -lntp | grep ':80'  
tail -50 /var/log/nginx/error.log

先确认后端 80 端口，再检查健康检查参数是否配置错误。

## 十一、关键坑位总结

1. `node.test.com` 写成第一个 server 块时会成为默认虚拟主机，`php.test.com` 可能被隐式匹配。
    
2. Nginx 502 不一定是网络不通，也可能是 FastCGI 上游连接错误。
    
3. PHP-FPM 当前监听 Unix Socket，不是 TCP 9000。
    
4. PHP-FPM 运行用户是 `apache`，Session 目录也必须允许 `apache` 写入。
    
5. `chown -R nginx:nginx /var/lib/php/` 是错误配置，会再次触发 Session 权限问题。
    
6. PHP 8.2 不能使用 phpMyAdmin 4.8.4，应使用 5.2.x。
    
7. Cookie 只是浏览器保存的 Session ID，真正的 Session 数据在服务端。
    
8. 当前轮询模式没有 Session 保持，登录状态可能因切换后端而丢失。
    
9. TCP 健康检查只能检查端口，不能保证 PHP 页面返回正常。
    
10. Rocky Linux 9 的 OpenSSL 3 与旧 Nginx 源码编译可能冲突，使用警告抑制或升级源码前都要确认模块兼容性。
    
11. 长编译命令中反斜杠后不能有空格。
    
12. 使用 `make install` 覆盖系统 Nginx 前应先备份配置和二进制。