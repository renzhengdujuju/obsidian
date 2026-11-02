# php.test.com 访问故障排查与修复记录

## 一、故障现象

从 Windows 主机访问：

http://php.test.com

无法正常打开页面，负载均衡器 `192.168.75.10` 返回：

502 Bad Gateway

排查过程中还依次出现了：

HTTP 500 Internal Server Error

以及 phpMyAdmin 页面无法加载的问题。

## 二、系统与访问链路

相关拓扑如下：

客户端  
  |  
  v  
php.test.com  
  |  
  v  
192.168.75.10（lb01，Nginx 负载均衡）  
  |-- 192.168.75.20（web01，Nginx + PHP-FPM + phpMyAdmin）  
  `-- 192.168.75.30（web02，Nginx + PHP-FPM + phpMyAdmin）

系统环境：

操作系统：Rocky Linux 9.4  
负载均衡：Nginx 1.20.1  
PHP：PHP 8.2.35  
PHP-FPM：php-fpm 8.2.35  
原 phpMyAdmin：4.8.4  
可用的新 phpMyAdmin：5.2.3

## 三、排查过程

### 1. 确认域名解析

检查结果：

php.test.com -> file.test.com -> 192.168.75.10

同时 Windows 的 hosts 文件中存在：

192.168.75.10 file.test.com node.test.com php.test.com

因此 DNS 和域名解析不是本次故障的主要原因。

### 2. 检查负载均衡器端口

在 `192.168.75.10` 上检查端口：

22 端口：可连接  
80 端口：可连接  
443 端口：不可连接

说明当前服务只通过 HTTP 80 端口提供访问，没有配置 HTTPS。

### 3. 复现 502

直接向负载均衡器发送带有 php.test.com Host 头的请求：

curl -I -H "Host: php.test.com" http://192.168.75.10/

返回：

HTTP/1.1 502 Bad Gateway  
Server: nginx/1.20.1

### 4. 检查 `.10` 的 Nginx 配置

原配置为：

upstream node {  
    server 192.168.75.20:80 max_fails=2 fail_timeout=10s;  
    server 192.168.75.30:80 max_fails=2 fail_timeout=10s;  
    check interval=3000 rise=2 fall=3 timeout=1000 type=tcp;  
}  
  
server {  
    listen 80;  
    server_name node.test.com;  
  
    location / {  
        proxy_pass http://node;  
        include proxy_params;  
    }  
}

这里存在两个问题：

- `php.test.com` 没有显式出现在 `server_name` 中。
    
- `node.test.com` 是第一个加载的 server 块，因此也会成为默认虚拟主机，导致未显式匹配的 `php.test.com` 被转发到 `.20` 和 `.30`。
    

不过，这个配置本身能够完成后端转发，真正导致 502 的原因还在后端。

### 5. 检查 `.20` 和 `.30` 后端服务

直接访问两台后端的默认站点：

curl -I http://192.168.75.20/  
curl -I http://192.168.75.30/

均返回：

HTTP/1.1 200 OK

但是携带 php.test.com Host 头访问时：

curl -I -H "Host: php.test.com" http://192.168.75.20/  
curl -I -H "Host: php.test.com" http://192.168.75.30/

两台都返回：

HTTP/1.1 502 Bad Gateway

因此可以确定，网络连通性和 80 端口监听正常，问题出在两台后端针对 `php.test.com` 的 Nginx/PHP 配置。

### 6. 找到第一个根因：FastCGI 地址错误

`.20` 和 `.30` 上的原配置为：

location ~ \.php$ {  
    fastcgi_pass 127.0.0.1:9000;  
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;  
    include fastcgi_params;  
}

但 PHP-FPM 的实际配置是：

listen = /run/php-fpm/www.sock

检查结果：

- `/run/php-fpm/www.sock` 正在监听。
    
- `127.0.0.1:9000` 没有任何进程监听。
    

Nginx 错误日志：

connect() failed (111: Connection refused) while connecting to upstream,  
upstream: "fastcgi://127.0.0.1:9000"

结论：

Nginx 尝试通过 TCP 9000 连接 PHP-FPM，  
但 PHP-FPM 实际只监听 Unix Socket，  
因此 Nginx 无法把 PHP 请求交给 PHP-FPM，最终返回 502。

### 7. 找到第二个根因：PHP Session 目录权限错误

修复 FastCGI 后，502 消失，但页面开始返回 500。

PHP 错误内容为：

session_start(): open(SESSION_FILE, O_RDWR) failed: Permission denied (13)  
session_start(): Failed to read session data:  
files (path: /var/lib/php/session)

检查结果：

PHP-FPM 运行用户：apache  
/var/lib/php/session 原所有者：nginx:nginx  
/var/lib/php/session 原权限：770

由于目录属于 `nginx:nginx`，运行 PHP-FPM 的 `apache` 用户没有读写权限，因此 phpMyAdmin 无法创建或读取会话文件。

### 8. 找到第三个根因：phpMyAdmin 版本与 PHP 版本不兼容

修复 Session 目录权限后，仍然出现 500。

PHP 错误日志显示：

PHP Fatal error:  
Unparenthesized `a ? b : c ? d : e` is not supported.  
Use either `(a ? b : c) ? d : e` or `a ? b : (c ? d : e)`  
in /code/phpMyAdmin-4.8.4-all-languages/libraries/classes/DatabaseInterface.php  
on line 615

原站点使用的是 2018 年发布的 phpMyAdmin 4.8.4，而当前 PHP 版本是 8.2.35。phpMyAdmin 4.8.4 不兼容 PHP 8，因此产生致命语法错误。

系统中已经安装了兼容的新版本：

phpMyAdmin 5.2.3  
安装目录：/usr/share/phpMyAdmin

## 四、实际执行的修复操作

### 1. 修改负载均衡器 `.10`

文件：

/etc/nginx/conf.d/node_proxy.conf

修改内容：

- server_name node.test.com;  
+ server_name node.test.com php.test.com;

作用：

- 明确让 `php.test.com` 使用该反向代理配置。
    
- 避免依赖“第一个 server 块自动成为默认虚拟主机”的隐式行为。
    

备份文件：

/etc/nginx/conf.d/node_proxy.conf.bak.20261028131127

### 2. 修改两台后端 `.20` 和 `.30`

文件：

/etc/nginx/conf.d/php.conf

修改内容：

- root /code/phpMyAdmin-4.8.4-all-languages;  
+ root /usr/share/phpMyAdmin;  
  
- fastcgi_pass 127.0.0.1:9000;  
+ fastcgi_pass unix:/run/php-fpm/www.sock;

作用：

- 把 FastCGI 从错误的 TCP 9000 改到实际监听的 Unix Socket。
    
- 解决 Nginx 无法连接 PHP-FPM 导致的 502。
    
- 把站点切换到与 PHP 8.2 兼容的 phpMyAdmin 5.2.3。
    
- 解决 phpMyAdmin 4.8.4 在 PHP 8.2 下产生的致命错误。
    

两台机器的备份文件：

/etc/nginx/conf.d/php.conf.bak.20261028131124

### 3. 修复 PHP Session 目录权限

在 `192.168.75.20` 和 `192.168.75.30` 上分别执行：

chown apache:apache /var/lib/php/session  
chmod 770 /var/lib/php/session

作用：

- 让 PHP-FPM 的实际运行用户 `apache` 拥有会话目录读写权限。
    
- 解决 phpMyAdmin 启动 Session 时返回 500 的问题。
    

### 4. 检查并重载 Nginx

三台机器上均执行了配置检查：

nginx -t

检查结果均为：

nginx: configuration file /etc/nginx/nginx.conf syntax is ok  
nginx: configuration file /etc/nginx/nginx.conf test is successful

随后执行：

systemctl reload nginx

作用：

- 在不中断 Nginx 监听的情况下加载新配置。
    

## 五、修复后的验证结果

### 1. 验证两台后端

在 `.10` 上分别请求两台后端：

curl -H "Host: php.test.com" http://192.168.75.20/  
curl -H "Host: php.test.com" http://192.168.75.30/

结果：

192.168.75.20 -> HTTP 200  
192.168.75.30 -> HTTP 200

### 2. 验证域名访问

从当前 Windows 主机访问：

http://php.test.com/

结果：

HTTP 200  
页面标题：phpMyAdmin  
页面内容：Welcome to phpMyAdmin

连续发起 6 次请求，全部返回 200，远端地址均为负载均衡器：

192.168.75.10

两台后端 Nginx 访问日志也确认收到了来自 `192.168.75.10` 的成功请求。

## 六、当前仍需要注意的问题

### 1. MariaDB 服务未启动

两台后端虽然安装了 MariaDB，但当前状态为：

mariadb.service: disabled  
mariadb.service: inactive

因此当前情况是：

- phpMyAdmin 登录页面可以正常打开。
    
- 如果要在 phpMyAdmin 中登录数据库，还需要启动并正确配置 MariaDB。
    

phpMyAdmin 当前配置的数据库地址是：

localhost

### 2. HTTPS 未配置

负载均衡器 `192.168.75.10` 当前只监听 80 端口，443 端口不可连接。

因此：

http://php.test.com  可以访问  
https://php.test.com 当前不可访问

如果需要 HTTPS，还需要配置 443 监听、证书和私钥。

## 七、最终结论

本次访问失败不是 DNS 解析问题，也不是 `.20`、`.30` 的 80 端口不通，而是多个配置问题叠加造成：

1. `php.test.com` 在 `.10` 上没有显式的 server_name 配置。
    
2. 两台后端的 Nginx 使用 TCP `127.0.0.1:9000` 连接 PHP-FPM，但 PHP-FPM 实际监听 Unix Socket，导致 502。
    
3. PHP Session 目录属于 `nginx:nginx`，而 PHP-FPM 使用 `apache` 用户运行，导致 Session 写入失败并返回 500。
    
4. phpMyAdmin 4.8.4 不兼容 PHP 8.2，触发致命错误。
    

完成上述配置修改、目录权限修复和 Nginx 重载后，`http://php.test.com/` 已经可以通过 `192.168.75.10` 负载均衡正常访问 `192.168.75.20` 和 `192.168.75.30`，并返回 phpMyAdmin 5.2.3 登录页面。

