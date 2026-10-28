# php.test.com 故障分析与排查思路说明

## 一、`node_proxy.conf` 是配置不完整，不是语法错误

原来的配置为：

server {  
    listen 80;  
    server_name node.test.com;  
​  
    location / {  
        proxy_pass http://node;  
    }  
}

该配置可以通过 `nginx -t`，所以它并不是语法错误。主要问题是：

- 没有显式加入 `php.test.com`。
    
- 当请求携带 `Host: php.test.com` 时，没有找到完全匹配的 `server_name`。
    
- 该配置又是第一个加载的 80 端口 server 块，因此成为默认虚拟主机。
    
- 所以 `php.test.com` 最终还是进入了这个反向代理块，并被转发给 `.20` 和 `.30`。
    

因此需要区分：

缺少 php.test.com 的 server_name

和：

php.test.com 完全无法进入这个 Nginx 配置

本次实际情况更接近第二种：

php.test.com 进入了反向代理，  
但不是因为明确匹配了 php.test.com，  
而是因为 node.test.com 的 server 块成了默认虚拟主机。

这种写法可能“碰巧工作”，但不够明确。如果以后有其他域名、默认站点或其他 upstream，就可能把无关请求错误转发出去。

因此修复时加入了：

server_name node.test.com php.test.com;

这个操作的作用是把路由关系写清楚，并不是本次 502 的直接修复点。

## 二、不是访问不到，而是访问到了但后端返回 502

前面的判断是正确的：

直接访问 .20/.30 可以访问  
带 Host 头访问时返回 502  
说明网络连通性没有问题

直接访问：

curl -I http://192.168.75.20/  
curl -I http://192.168.75.30/

均返回 `200`，说明：

- 网络是通的。
    
- 80 端口正在监听。
    
- 后端 Nginx 正常运行。
    

但是携带域名 Host 头访问：

curl -I -H "Host: php.test.com" http://192.168.75.20/  
curl -I -H "Host: php.test.com" http://192.168.75.30/

两台都返回 `502`。

原因是同一个 IP 上可以根据 `Host` 选择不同的 Nginx 虚拟主机：

curl http://192.168.75.20/  
  -> 进入默认站点，返回 web01，HTTP 200  
  
curl -H "Host: php.test.com" http://192.168.75.20/  
  -> 进入 php.test.com 虚拟主机  
  -> 请求交给 PHP 处理  
  -> Nginx 连接 PHP-FPM 失败  
  -> 返回 HTTP 502

命令是 `curl -I`，大写字母 `I`，用于查看响应头；`curl -l` 是小写字母 `L`，含义不同。

## 三、502 和 500 的区别

排查时可以这样理解：

连接不上 80 端口  
  -> Connection refused 或 timeout  
  
Nginx 能访问，但无法连接它的上游  
  -> 502 Bad Gateway  
  
Nginx 成功把请求交给 PHP，但 PHP 程序自己报错  
  -> 500 Internal Server Error

最初返回 502，因为 Nginx 配置为：

fastcgi_pass 127.0.0.1:9000;

但 PHP-FPM 实际监听的是：

listen = /run/php-fpm/www.sock

因此 Nginx 去连接一个没有程序监听的 9000 端口，日志显示：

connect() failed (111: Connection refused)  
upstream: "fastcgi://127.0.0.1:9000"

将配置改为：

fastcgi_pass unix:/run/php-fpm/www.sock;

后，Nginx 已经能够把 PHP 请求交给 PHP-FPM，因此 502 消失。随后出现 500，说明反向代理和 FastCGI 链路已经打通，问题进入了应用层。

## 四、如何发现 PHP Session 权限问题

修复 FastCGI 后，除了查看状态码，还查看了响应具体内容和 PHP 日志。

查看页面响应：

curl -sS -D - -H "Host: php.test.com" http://127.0.0.1/

页面中出现了：

Error during session start  
session_start(): open(SESSION_FILE, O_RDWR) failed:  
Permission denied (13)  
files (path: /var/lib/php/session)

该错误已经直接指出三个信息：

- 错误发生在 `session_start()`。
    
- 错误类型是权限拒绝。
    
- Session 路径是 `/var/lib/php/session`。
    

接着确认 PHP-FPM 使用哪个用户运行：

grep -n '^user\|^group' /etc/php-fpm.d/www.conf

发现：

user = apache  
group = apache

再检查 Session 目录：

ls -ld /var/lib/php/session

发现：

drwxrwx--- nginx nginx /var/lib/php/session

也就是说：

目录属于：nginx  
PHP-FPM 使用：apache  
目录权限：770

`apache` 用户既不是目录所有者，也不在允许访问的用户组中，因此无法创建 Session 文件。

修复命令：

chown apache:apache /var/lib/php/session  
chmod 770 /var/lib/php/session

修复后，页面不再报 Session 权限错误。

PHP-FPM 的错误日志位置也可以通过配置确认：

grep -Rni 'error_log' /etc/php.ini /etc/php-fpm.d

实际日志位于：

/var/log/php-fpm/www-error.log

## 五、如何发现 phpMyAdmin 与 PHP 版本不兼容

Session 权限修复后，页面仍然返回 500。这时继续查看最新的 PHP-FPM 错误日志：

tail -80 /var/log/php-fpm/www-error.log

日志中出现：

PHP Fatal error:  
Unparenthesized `a ? b : c ? d : e` is not supported.  
in /code/phpMyAdmin-4.8.4-all-languages/  
libraries/classes/DatabaseInterface.php on line 615

随后核对 PHP 版本：

php -v

得到：

PHP 8.2.35

站点根目录指向：

/code/phpMyAdmin-4.8.4-all-languages

而系统已经安装：

phpMyAdmin 5.2.3  
/usr/share/phpMyAdmin

phpMyAdmin 4.8.4 是旧版本，不支持 PHP 8.2。日志中的嵌套三元表达式写法在 PHP 8 中已经被禁止，因此执行到该文件时直接产生致命错误。

所以将 Nginx 根目录：

root /code/phpMyAdmin-4.8.4-all-languages;

改为：

root /usr/share/phpMyAdmin;

作用是切换到系统已经安装的 phpMyAdmin 5.2.3，解决 PHP 8.2 不兼容问题。

## 六、最终结论

本次不是 DNS 问题，也不是 `.20`、`.30` 网络或 80 端口不通，而是多个配置问题叠加造成：

1. `node_proxy.conf` 没有显式配置 `php.test.com`，只是依靠默认虚拟主机偶然转发，配置语义不严谨。
    
2. 后端 Nginx 连接 PHP-FPM 时使用了错误的 TCP 9000，但 PHP-FPM 实际监听 Unix Socket，导致 502。
    
3. PHP Session 目录属于 `nginx`，PHP-FPM 却使用 `apache` 用户运行，导致 Session 权限错误并返回 500。
    
4. 实际运行的 phpMyAdmin 4.8.4 与 PHP 8.2 不兼容，导致进一步的 PHP 致命错误。
    
5. 切换到系统已安装的 phpMyAdmin 5.2.3 后，`php.test.com` 通过 `.10` 负载均衡访问 `.20/.30` 恢复正常，返回 200。
    

排查的核心思路是把问题分层：

网络是否通  
  -> 端口是否监听  
  -> Host 是否正确路由到虚拟主机  
  -> 反向代理能否连接上游  
  -> PHP-FPM 能否被 FastCGI 访问  
  -> PHP 应用是否报错  
  -> 应用依赖目录和服务是否正常

本次排查正是沿着这条链路，从 502 查到 FastCGI 连接错误，再查 Session 权限，最后查到 phpMyAdmin 版本不兼容。