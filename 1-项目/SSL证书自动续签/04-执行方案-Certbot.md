# Certbot 自动签发与续签 Let's Encrypt 免费 SSL 证书

> 📍 **本文定位**：HTTP-01 验证的标准流程手册（Certbot + Nginx 插件）。阅读顺序：[[01-个人网站搭建|建站前传]] → [[02-从域名验证到证书认证-完整闭环梳理|背景原理]] → 本文 → 其他方案手册。
>
> ⚠️ **11-09 实战结果**：本手册的 HTTP-01 验证在阿里云 ECS 上受阻——Let's Encrypt 多视角验证节点被阿里云返回空响应，反复 403（非配置问题），最终改用 DNS-01 手动模式。全过程见 [[08-实战记录-Certbot+HTTP-01方案|实战记录：Certbot+HTTP-01方案]]。本手册保留作为 HTTP-01 标准流程参考，**照做前先看实战记录**。

## 1. 本文目标

本文用于在以下环境中，为 `renzhengdejuju.online` 配置免费的 HTTPS 证书，并实现自动续签和自动加载 Nginx：

- 操作系统：Alibaba Cloud Linux 3.2104 LTS 64 位
- Web 服务：Nginx 1.24.0
- 域名：`renzhengdejuju.online`
- 证书颁发机构：Let's Encrypt
- 域名验证方式：HTTP-01
- 证书部署位置：Nginx

最终效果：

```text
Let's Encrypt 免费证书
        ↓
Certbot 自动申请和续签
        ↓
自动更新 Nginx 证书配置
        ↓
续签成功后平滑 reload Nginx
```

本方案不需要阿里云 DNS AccessKey，因为 HTTP-01 验证通过公网 80 端口完成。

> 如果以后要申请 `*.renzhengdejuju.online` 泛域名证书，不能使用本文的 HTTP-01 方式，需要改用 DNS-01，并配置阿里云 DNS API。

---

## 2. 重要说明

### 2.1 证书和软件费用

以下内容本身不需要购买 SSL 证书：

- Let's Encrypt：免费证书颁发机构；
- Certbot：开源免费软件；
- Nginx：开源软件。

服务器、域名、云安全组和网络流量费用不属于证书费用。

### 2.2 不要使用手动 DNS 方式

下面这种方式需要每次手动添加 TXT 记录，**不能实现无人值守自动续签**：

```bash
certbot certonly --manual --preferred-challenges dns
```

本文使用的是：

```bash
certbot --nginx
```

它使用 HTTP-01 验证，并由 Certbot 自动续签。

### 2.3 不要频繁强制续签

日常不要把下面命令放进定时任务：

```bash
certbot renew --force-renewal
```

频繁强制申请可能触发 Let's Encrypt 频率限制。日常使用：

```bash
sudo certbot renew
```

Certbot 会自动判断证书是否接近到期，只有需要续签时才真正申请新证书。

---

## 3. 第一步：确认 DNS 和公网访问

在服务器上执行：

```bash
getent hosts renzhengdejuju.online

sudo ss -lntp | grep -E ':(80|443)[[:space:]]' || true

sudo nginx -t
```

预期结果：

1. `getent hosts` 能返回域名对应的公网 IP；
2. Nginx 正在监听 `0.0.0.0:80`；
3. `nginx -t` 返回：

```text
syntax is ok
test is successful
```

然后检查 Nginx 是否已经配置了这个域名：

```bash
sudo nginx -T 2>/dev/null | grep -nE 'server_name|listen'
```

配置中应当能看到类似内容：

```nginx
server_name renzhengdejuju.online;
listen 80;
```

如果还需要 `www`，必须先确认 DNS 中存在 `www.renzhengdejuju.online` 的解析，并且 Nginx 中也配置了：

```nginx
server_name renzhengdejuju.online www.renzhengdejuju.online;
```

当前文档只申请主域名，因此后续命令只使用：

```text
renzhengdejuju.online
```

### 3.1 检查 HTTP 端口是否能从外部访问

在服务器上执行：

```bash
curl -I --connect-timeout 10 http://renzhengdejuju.online
```

返回 `200`、`301`、`302` 或 `404` 都说明 HTTP 请求已经到达服务器；如果出现 `Connection refused` 或超时，需要检查：

- 阿里云 ECS 安全组是否允许 TCP 80；
- 系统防火墙是否允许 HTTP；
- 域名 A 记录是否指向这台服务器；
- 域名前面是否接入了 CDN、WAF 或其他代理。

如果使用 `firewalld`，可检查：

```bash
sudo firewall-cmd --state 2>/dev/null || true
sudo firewall-cmd --list-ports 2>/dev/null || true
```

如果确认使用 firewalld 且 80 端口未放行，可以执行：

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

如果服务器没有启用 firewalld，不需要执行这两条放行命令。

---

## 4. 第二步：备份 Nginx 配置

Certbot 的 `--nginx` 插件会修改 Nginx 配置。申请前先备份：

```bash
sudo cp -a /etc/nginx "/etc/nginx.backup.$(date +%Y%m%d%H%M%S)"
sudo nginx -t
```

如果后续需要查看备份目录：

```bash
ls -ld /etc/nginx.backup.*
```

---

## 5. 第三步：安装 Certbot

Alibaba Cloud Linux 3 属于 RHEL/Anolis 兼容系统，本文使用 Certbot 官方推荐的 Snap 安装方式。

### 5.1 安装并启动 snapd

```bash
sudo dnf install -y snapd
sudo systemctl enable --now snapd.socket
```

建立 classic snap 所需的路径：

```bash
if [ ! -e /snap ]; then
  sudo ln -s /var/lib/snapd/snap /snap
fi
```

等待 Snap 初始化：

```bash
sudo snap wait system seed.loaded
```

如果这条命令暂时没有返回，等待几秒后再执行一次。

### 5.2 安装 Certbot

```bash
sudo snap install core
sudo snap refresh core
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/local/bin/certbot
```

检查版本：

```bash
certbot --version
```

应当能看到类似：

```text
certbot 版本号
```

检查 Nginx 插件：

```bash
sudo certbot plugins
```

输出中应包含：

```text
nginx
```

---

## 6. 第四步：首次申请并安装证书

先再次检查 Nginx：

```bash
sudo nginx -t
```

然后执行首次申请：

```bash
sudo certbot --nginx \
  -d renzhengdejuju.online \
  --email 你的邮箱地址 \
  --agree-tos \
  --no-eff-email \
  --redirect
```

请把 `你的邮箱地址` 替换为真实邮箱，例如：

```bash
--email admin@example.com
```

参数说明：

- `--nginx`：使用 Nginx 插件，自动完成验证并配置证书；
- `-d renzhengdejuju.online`：申请主域名证书；
- `--email`：用于接收重要的证书通知；
- `--agree-tos`：同意 Let's Encrypt 服务条款；
- `--no-eff-email`：不向 EFF 订阅推广邮件；
- `--redirect`：将 HTTP 请求重定向到 HTTPS。

执行过程中如果询问是否将 HTTP 重定向到 HTTPS，选择启用重定向即可。

成功时通常会看到类似内容：

```text
Successfully received certificate.
Certificate is saved at:
/etc/letsencrypt/live/renzhengdejuju.online/fullchain.pem
Key is saved at:
/etc/letsencrypt/live/renzhengdejuju.online/privkey.pem
```

证书目录中的文件是符号链接，实际文件位于 `/etc/letsencrypt/archive/`。Nginx 应使用 `live` 目录中的路径，不要直接写死 `archive` 路径。

---

## 7. 第五步：检查 HTTPS 是否生效

检查 Nginx 配置：

```bash
sudo nginx -t
```

检查 HTTPS 响应：

```bash
curl -I --connect-timeout 10 https://renzhengdejuju.online
```

检查证书信息和有效期：

```bash
echo | openssl s_client \
  -connect renzhengdejuju.online:443 \
  -servername renzhengdejuju.online 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

预期能看到：

- `subject` 包含 `renzhengdejuju.online`；
- `issuer` 为 Let's Encrypt 相关颁发者；
- `notBefore` 为生效时间；
- `notAfter` 为到期时间。

查看 Certbot 管理的证书：

```bash
sudo certbot certificates
```

---

## 8. 第六步：配置续签成功后的 Nginx reload

证书续签成功后，Nginx 需要重新读取新证书。创建 Certbot 部署钩子：

```bash
sudo install -d -m 755 /etc/letsencrypt/renewal-hooks/deploy
```

创建脚本：

```bash
sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

/usr/sbin/nginx -t
/usr/bin/systemctl reload nginx
EOF
```

设置权限：

```bash
sudo chmod 750 /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

说明：

- `nginx -t` 先检查配置；
- 配置检查失败时，脚本不会 reload；
- 只有证书成功续签后，Certbot 才会执行 `deploy` 钩子；
- `reload` 是平滑重载，不会像 `restart` 那样直接停止再启动服务。

---

## 9. 第七步：测试自动续签

执行模拟续签：

```bash
sudo certbot renew --dry-run
```

成功时通常会出现：

```text
Congratulations, all simulated renewals succeeded
```

这一步不会替换生产证书，但会验证续签流程是否可以正常完成。

如果还要同时验证部署钩子，可以执行：

```bash
sudo certbot renew --dry-run --run-deploy-hooks
```

这会在模拟流程成功后执行 Nginx reload 钩子。执行前确认：

```bash
sudo nginx -t
```

---

## 10. 第八步：确认自动定时任务

检查 Certbot 是否已经安装定时任务：

```bash
systemctl list-timers --all | grep -Ei 'certbot|snap.certbot' || true
```

也可以检查 Snap 服务：

```bash
sudo snap services certbot 2>/dev/null || true
```

Certbot 的定时任务通常每天运行一次或按系统配置运行；`certbot renew` 会自行判断证书是否需要续签，不会每次都重新申请证书。

手动执行一次普通续签检查：

```bash
sudo certbot renew
```

如果证书距离到期时间还较远，看到“未到续签时间”属于正常结果。

> 如果系统已经存在 Certbot 的 systemd timer，不要再额外创建 cron，避免同一时间重复执行。

---

## 11. 常用检查命令

### 查看证书

```bash
sudo certbot certificates
```

### 查看证书文件

```bash
sudo ls -l /etc/letsencrypt/live/renzhengdejuju.online/
```

### 查看 Nginx 使用的证书路径

```bash
sudo nginx -T 2>/dev/null | grep -nE 'server_name|ssl_certificate|ssl_certificate_key'
```

### 检查 Nginx 配置

```bash
sudo nginx -t
```

### 平滑 reload Nginx

```bash
sudo systemctl reload nginx
```

### 查看 Certbot 日志

```bash
sudo tail -n 100 /var/log/letsencrypt/letsencrypt.log
```

### 查看定时器日志

```bash
sudo journalctl -u snap.certbot.renew.service --no-pager -n 100 2>/dev/null || true
sudo journalctl -u certbot.service --no-pager -n 100 2>/dev/null || true
```

---

## 12. 常见问题

### 12.1 `Connection refused` 或连接超时

重点检查：

```bash
sudo ss -lntp | grep ':80 '
sudo firewall-cmd --list-all 2>/dev/null || true
sudo nginx -t
```

同时检查阿里云 ECS 安全组是否允许：

```text
TCP 80  来源 0.0.0.0/0
TCP 443 来源 0.0.0.0/0
```

### 12.2 域名解析到了错误的服务器

检查：

```bash
getent hosts renzhengdejuju.online
```

从其他网络环境查询 A 记录，确认结果指向当前 Nginx 服务器。Let's Encrypt 是从公网访问域名，不是只看服务器本机解析结果。

### 12.3 Nginx 中没有对应的 `server_name`

Certbot 的 Nginx 插件需要找到对应域名的站点配置。先定位配置：

```bash
sudo nginx -T 2>/dev/null | grep -n 'server_name'
```

在实际站点的 HTTP `server` 配置中加入：

```nginx
server_name renzhengdejuju.online;
```

修改后执行：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 12.4 端口 80 已经强制跳转 HTTPS

首次申请时，如果 443 证书还没有准备好，过早强制跳转可能影响验证。可以先暂时保留一个正常监听 80 的 HTTP server，让 Certbot 完成申请，成功后再启用 HTTPS 重定向。

### 12.5 使用 CDN、WAF 或反向代理

HTTP-01 验证请求必须最终到达这台 Nginx。如果域名前面有 CDN、WAF、负载均衡或代理，需要确认：

- `/.well-known/acme-challenge/` 请求不会被拦截；
- 请求可以转发到当前服务器；
- 80 端口对公网可访问。

如果无法保证这一点，应改用 DNS-01。

### 12.6 申请泛域名失败

以下域名不能使用本文的 HTTP-01 方式：

```text
*.renzhengdejuju.online
```

泛域名必须使用 DNS-01。届时需要配置阿里云 DNS API，或者由 DNS 服务商提供 CNAME 委派验证。

### 12.7 `certbot renew --dry-run` 失败

不要先强制续签。先收集以下信息：

```bash
certbot --version
sudo certbot certificates
sudo nginx -t
sudo nginx -T 2>/dev/null | grep -nE 'server_name|listen|ssl_certificate'
sudo tail -n 100 /var/log/letsencrypt/letsencrypt.log
```

把错误信息中涉及域名、命令和日志行发出来即可。不要发送邮箱密码、AccessKey、私钥或完整私密配置。

---

## 13. 最终验收清单

以下项目全部满足，即表示配置完成：

- [ ] `certbot --version` 可以正常执行；
- [ ] `sudo certbot certificates` 能看到 `renzhengdejuju.online`；
- [ ] `/etc/letsencrypt/live/renzhengdejuju.online/fullchain.pem` 存在；
- [ ] `/etc/letsencrypt/live/renzhengdejuju.online/privkey.pem` 存在；
- [ ] `sudo nginx -t` 成功；
- [ ] `curl -I https://renzhengdejuju.online` 返回正常 HTTP 状态；
- [ ] `openssl s_client` 能看到正确的证书主题和有效期；
- [ ] `sudo certbot renew --dry-run` 成功；
- [ ] `/etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh` 存在且可执行；
- [ ] 系统存在 Certbot 自动定时任务；
- [ ] 没有使用 `--manual`；
- [ ] 没有把 `--force-renewal` 放进日常定时任务。

---

## 14. 方案边界

本文方案适用于：

```text
renzhengdejuju.online
```

以及其他能通过公网 80 端口访问到目标 Nginx 的普通域名。

本文方案不适用于：

- 只允许内网访问的域名；
- 没有公网 80 端口的服务器；
- 需要泛域名证书的场景；
- DNS 由 CDN 或代理完全接管、且 HTTP 验证请求无法回源的场景。

这些场景需要改用 DNS-01 自动验证。
