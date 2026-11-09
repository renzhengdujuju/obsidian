
# Certbot 在阿里云 ECS 上申请 SSL 证书踩坑笔记
## 一、环境信息
- 系统：Alibaba Cloud Linux 3 (OpenAnolis Edition)
- Web 服务器：Nginx 1.24.0（yum 安装）
- 域名：`renzhengdejuju.online` / `www.renzhengdejuju.online`
- 目标：从 acme.sh 切换到 Certbot 做 SSL 自动续签
- SELinux：已 disabled
---
## 二、遇到的问题时间线
### 阶段 1：直接 `certbot --nginx` 申请 → 403
```bash
sudo certbot --nginx -d renzhengdejuju.online -d www.renzhengdejuju.online

报错：
Type: unauthorized
Detail: 8.159.145.12: Invalid response from
http://renzhengdejuju.online/.well-known/acme-challenge/xxx: 403

**排查方向**：Nginx 配置里可能有拒绝 `.` 开头路径的规则，或者目录权限/SELinux 拦截。

---

### 阶段 2：改用 `--webroot` 方式 → 仍然 403

创建 `/var/www/certbot` 作为验证目录，在 `conf.d/` 下新建 `certbot-webroot.conf`：

nginx

server {
    listen 80;
    server_name renzhengdejuju.online www.renzhengdejuju.online;
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    location / {
        return 301 http://$host$request_uri;
    }
}

重新申请 → 还是 403。

**关键发现**：`nginx -t` 出现警告：

text

nginx: [warn] conflicting server name "renzhengdejuju.online" on 0.0.0.0:80, ignored

说明 `nginx.conf` 里已经有 80 端口的 `server` 块占用了这个域名，`conf.d/certbot-webroot.conf` 被**直接忽略**，配置根本没生效。

---

### 阶段 3：把验证规则写进 `nginx.conf` 主配置 → 404

修改 `nginx.conf` 的 80 端口 `server` 块：

nginx

location ^~ /.well-known/acme-challenge/ {
    root /usr/share/nginx/html;
    default_type "text/plain";
}

结果 `curl` 返回 **404**。

**原因**：`root` 会把完整 URI 拼在根目录后，Nginx 实际去找的是：

text

/usr/share/nginx/html/.well-known/acme-challenge/testfile

这个路径不存在，所以 404。

**修复**：把 `root` 改成 `alias`：

nginx

location ^~ /.well-known/acme-challenge/ {
    alias /var/www/certbot/;
    default_type "text/plain";
    allow all;
}

`alias` 会把 `/.well-known/acme-challenge/testfile` 精确映射到 `/var/www/certbot/testfile`。

---

### 阶段 4：清理冲突文件后，本机 `curl` 返回 200

移走 `conf.d/certbot-webroot.conf` 后，`nginx -t` 不再有冲突警告，本机测试：

bash

echo "hello-certbot" | sudo tee /var/www/certbot/testfile
curl -I http://renzhengdejuju.online/.well-known/acme-challenge/testfile

返回：

text

HTTP/1.1 200 OK
Content-Type: text/plain

**本机验证路径完全正常。**

---

### 阶段 5：再次申请 Certbot → 又回到 403

bash

sudo certbot certonly --webroot -w /var/www/certbot -d renzhengdejuju.online -d www.renzhengdejuju.online

仍然 403。

---

## 三、最终定位：不是配置问题，是阿里云网络层问题

检查 Nginx 访问日志 `access.log`，找到 Let's Encrypt 验证服务器的请求记录：

text

23.178.112.210 - - [09/Nov/2026:09:19:59 +0800] "GET /.well-known/acme-challenge/Q7nqYvpQ... HTTP/1.1" 200 0 "-" "Mozilla/5.0 (compatible; Let's Encrypt validation server; ...)"

**关键点：`200 0`** —— Let's Encrypt 收到了 200 状态码，但**响应体是 0 字节**。

同时 `error.log` 显示：

text

open() "/var/www/certbot/f51G5lr2..." failed (2: No such file or directory)

这说明：

1. Certbot 确实把验证文件写到了 `/var/www/certbot/`
    
2. Nginx 的 `alias` 也正确映射到了这个路径
    
3. 但 Let's Encrypt 从某些验证节点收到的仍然是**空响应**
    

### 根因

Let's Encrypt 现在使用**多视角验证（Multi-Perspective Validation）**，会从全球多个节点（包括大量 AWS IP 段）同时发起验证请求。

**阿里云 ECS 对来自部分 AWS IP 的请求返回了"200 但 body 为 0 字节"的异常响应**，导致 Let's Encrypt 判定验证失败，最终返回 403。

你本机 `curl` 能通，是因为请求不经过"阿里云 ↔ AWS"这条链路，所以复现不了问题。

### 为什么 acme.sh 以前能成功？

- Let's Encrypt 的多视角验证是逐步铺开的，之前用 acme.sh 时可能还没触发到这个机制
    
- 或者当时网络路径恰好没有出问题
    
- 工具从 acme.sh 换成 Certbot，只是让这个隐藏的网络问题暴露了出来
    

**这不是 Nginx 配置问题，继续调配置没有意义。**

---

## 四、解决方案：改用 DNS-01 验证

绕开 HTTP 网络路径，直接让 Let's Encrypt 检查 DNS 记录。

bash

sudo certbot certonly --manual \
  -d renzhengdejuju.online \
  -d www.renzhengdejuju.online \
  --preferred-challenges dns

执行后：

1. Certbot 提示为 `_acme-challenge.renzhengdejuju.online` 创建 **TXT 记录**
    
2. 去阿里云 DNS 控制台添加该 TXT 记录
    
3. 本地确认生效：
    
    bash
    
    nslookup -type=TXT _acme-challenge.renzhengdejuju.online
    
4. 回终端按回车，证书申请成功
    

**注意**：`--manual` 模式**不会自动续期**，90 天后需手动重复。如需自动续期，可考虑：

- 配置 certbot 的阿里云 DNS API 插件
    
- 或换回 acme.sh 配合阿里云 DNS API（对国内 DNS 服务商支持更成熟）
    

---

## 五、关键经验总结

|问题|原因|解决|
|---|---|---|
|`conflicting server name` 警告|`conf.d/` 和 `nginx.conf` 里有重复的 `server_name`|移走冲突文件，只保留一处配置|
|`root` 导致 404|`root` 会拼接完整 URI|改用 `alias` 精确映射目录|
|本机 curl 200 但 Certbot 403|阿里云对部分验证节点返回空响应|改用 DNS-01 验证|
|多视角验证失败|Let's Encrypt 从全球多节点验证，网络层被干扰|放弃 HTTP-01，走 DNS-01|

### 排查命令备忘

bash

# 配置检查
sudo nginx -t
ls -l /etc/nginx/conf.d/
# 验证路径测试
echo "test" | sudo tee /var/www/certbot/testfile
curl -I http://你的域名/.well-known/acme-challenge/testfile
# 日志排查（重点看 Let's Encrypt 请求的返回码和 body 大小）
sudo tail -50 /var/log/nginx/access.log
sudo tail -50 /var/log/nginx/error.log
# SELinux 状态
sestatus
# 目录权限
ls -ld /var/www/certbot
ps aux | grep nginx

### 核心教训

1. **先看日志再改配置**：`access.log` 里 Let's Encrypt 请求的 `200 0` 直接暴露了网络层问题，比反复调 Nginx 高效得多。
    
2. **`conflicting server name` 警告不能忽略**：它意味着你的配置被静默丢弃了。
    
3. **`root` vs `alias`**：验证路径映射用 `alias` 更精确，避免 URI 拼接导致 404。
    
4. **阿里云 + Let's Encrypt HTTP-01 可能不稳定**：遇到 403 且本机 curl 正常时，优先怀疑网络层，直接转 DNS-01。
    
5. **acme.sh 能成功不代表 Certbot 也能成功**：Let's Encrypt 验证机制在变化，工具切换可能暴露之前隐藏的网络问题。