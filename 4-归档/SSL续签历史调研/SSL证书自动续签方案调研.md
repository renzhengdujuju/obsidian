# SSL 证书自动续签方案调研

> 整理日期：2026-11-01

> 背景：现有免费证书约 90 天有效期，每 80 天需手动申请一次，运维成本高；且要求**不通过阿里云账号**完成自动续签。

> 用途：明天对照本文档确认实际环境，然后确定落地方案。

> 已落地：最终确定的两个方案见 [[执行方案-乐此加密|方案一：乐此加密]] 和 [[备选方案-阿里云RAM角色|方案二：阿里云 RAM + acme.sh]]。

## 一、核心结论

1. **"免费 + 自动续签"本质上只有一条路**：ACME 协议 + 免费 CA（Let's Encrypt / ZeroSSL / Google Trust Services），客户端使用 `acme.sh`、`certbot`、`Caddy`、`win-acme` 等工具。

2. **国内云厂商的"免费证书"基本都不支持 ACME 自动申请**，仍然要进控制台手动操作，解决不了 80 天手动签一次的问题。它们只是"免费的证书"，不是"免费的自动化"。

3. **完全可以不碰阿里云账号**：
   - 服务器 80 端口公网可达 → 走 **HTTP-01** 验证，全程不需要任何云账号；
   - 需要泛域名，或 80 端口被封/未备案 → 走 **DNS-01**，只需一个 DNS 服务商的 API（Cloudflare、DNSPod、华为云都行，不一定是阿里云）。

4. **越早自动化越好**：2026-03-15 起所有公网 TLS 证书最长 **200 天**，2027-03-15 降到 **100 天**，2029 年 **47 天**。手动续签只会越来越频繁。

## 二、两个必须先知道的事实

### 1. 证书有效期正在持续缩短（CA/B Forum SC-081v3）

| 时间节点 | 单张证书最长有效期 |
|----------|------------------|
| 2026-03-15 之前 | 398 天 |
| **2026-03-15 起（当前）** | **200 天** |
| 2027-03-15 起 | 100 天 |
| 2029-03-15 起 | 47 天 |

说明：

- 已签发的旧证书不受影响，续签/重签才按新规执行。
- 国内厂商（阿里云、腾讯云、火山引擎、百度智能云、天翼云、CFCA 等）也都已同步调整。
- 结论：**买一年期证书的时代已经结束**，无论免费还是付费，都必须走自动化。

### 2. Buypass 180 天免费证书已经停服

- 曾是免费 ACME 生态中唯一能签 180 天的 CA。
- 2025-08-18 宣布退出 TLS/SSL 业务；
- 2025-10-31 最后一次签发；
- 2026-04-15 ACME 服务正式终止（原 ACME 目录已返回 404）；
- 目前 CAA 里还写着 `buypass.com` / `buypass.no` 的域名需要改成新 CA 或删除记录。

**结论：不要再找 180 天的免费 ACME 证书了，现在没有。**

## 三、方案总览

| # | 方案 | 适合场景 | 需要账号 | 续签方式 | 难度 |
|---|------|---------|---------|---------|------|
| A | **acme.sh + Let's Encrypt（HTTP-01）** | Linux 服务器，80 端口可用 | **不需要任何云账号** | cron 自动 | ★★ |
| B | **acme.sh + DNS-01** | 泛域名、80 端口被封、未备案 | 一个 DNS API | cron 自动 | ★★★ |
| C | **certbot + nginx** | Ubuntu/Debian + Nginx | 不需要 | systemd timer 自动 | ★ |
| D | **Caddy** | 新项目 / 可替换 Web 服务器 | 不需要 | 内置自动 | ★ |
| E | **宝塔 / 1Panel / AllinSSL** | 不想碰命令行 | 不需要（面板自身账号） | 面板定时任务 | ★ |
| F | **win-acme（WACS）** | Windows Server + IIS | 不需要 | 计划任务自动 | ★★ |
| G | **Cloudflare 通用 SSL** | 域名可迁到 Cloudflare | CF 免费账号 | **根本不用续** | ★ |
| H | 国内云免费证书（腾讯/华为/火山/百度） | —— | 各家账号 | **仍需手动** | ★ |
| I | 国内 ACME 兼容服务（来此加密 / FreeSSL.cn） | Let's Encrypt 连不上时的备选 | 注册其账号 | 自动 | ★★★ |

## 四、各方案详细执行流程

### 方案 A：acme.sh + Let's Encrypt（HTTP-01）——首选

**前提条件**

- 域名已解析到服务器；
- 服务器 80 端口公网可达（安全组、firewalld/ufw、云防火墙都要放行）；
- 大陆服务器需已完成 ICP 备案，否则 80 会被拦截，HTTP-01 必然失败；
- Nginx 站点根目录可写。

**执行步骤**

```bash
# 1. 安装（国内建议 gitee 源，避免 GitHub 访问不稳定）
git clone https://gitee.com/neilpang/acme.sh.git
cd acme.sh && ./acme.sh --install -m ops@example.com

# 或使用官方一键脚本
curl https://get.acme.sh | sh -s email=ops@example.com

# 2. 默认 CA 改为 Let's Encrypt（acme.sh 3.x 默认是 ZeroSSL）
~/.acme.sh/acme.sh --set-default-ca --server letsencrypt

# 3. 先用 staging 测试，避免触发 Let's Encrypt 频率限制
~/.acme.sh/acme.sh --issue -d example.com -d www.example.com \
  -w /var/www/html --keylength ec-256 --staging

# 4. 测试通过后正式签发（staging 证书需要 --force 重签）
~/.acme.sh/acme.sh --issue -d example.com -d www.example.com \
  -w /var/www/html --keylength ec-256 --force

# 5. 安装到 Nginx 目录，并让 acme.sh 在续期后自动 reload
mkdir -p /etc/nginx/ssl/example.com
~/.acme.sh/acme.sh --install-cert -d example.com --ecc \
  --key-file       /etc/nginx/ssl/example.com/privkey.pem \
  --fullchain-file /etc/nginx/ssl/example.com/fullchain.pem \
  --reloadcmd      "systemctl reload nginx"

# 6. 检查定时任务与证书列表
crontab -l | grep acme
~/.acme.sh/acme.sh --list
```

**Nginx 配置要点**

确保 `/.well-known/acme-challenge/` 不会被 301 重定向吃掉：

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }
    location / {
        return 301 https://$host$request_uri;
    }
}
```

**续签机制**

- `--install` 时 acme.sh 会自动往 crontab 写入一条每天执行的 `--cron`；
- 证书剩余约 30 天时自动续期；
- 续期后按 `--reloadcmd` 自动重载 Nginx；
- 装完之后基本不需要人工干预。

### 方案 B：DNS-01（泛域名 / 80 端口不可用）

泛域名（`*.example.com`）只能走 DNS-01。以 Cloudflare 为例：

```bash
export CF_Token="你的API令牌"   # 权限只需 Zone:Read + DNS:Edit

~/.acme.sh/acme.sh --issue --dns dns_cf \
  -d example.com -d '*.example.com' --keylength ec-256

~/.acme.sh/acme.sh --install-cert -d example.com --ecc \
  --key-file /etc/nginx/ssl/example.com/privkey.pem \
  --fullchain-file /etc/nginx/ssl/example.com/fullchain.pem \
  --reloadcmd "systemctl reload nginx"
```

**常见 DNS 服务商参数**

| 服务商 | `--dns` 参数 | 环境变量 |
|--------|-------------|---------|
| Cloudflare | `dns_cf` | `CF_Token` |
| DNSPod / 腾讯云 | `dns_dp` | `DP_Id`、`DP_Key` |
| 腾讯云 API3 | `dns_tencent` | `Tencent_SecretId`、`Tencent_SecretKey` |
| 华为云 | `dns_huaweicloud` | `HUAWEICLOUD_Username`、`HUAWEICLOUD_Password`、`HUAWEICLOUD_DomainName` |
| 阿里云 | `dns_ali` | `Ali_Key`、`Ali_Secret` |
| GoDaddy | `dns_gd` | `GD_Key`、`GD_Secret` |

**安全要点**

- 不要使用主账号 AK；
- Cloudflare 使用限定范围的 API Token；
- 阿里云如非用不可，建 RAM 子账号只授 `AliyunDNSFullAccess`；
- 腾讯云使用子用户，只给 DNSPod 权限；
- 凭证保存在 `~/.acme.sh/account.conf`，建议 `chmod 600` 并纳入备份。

### 方案 C：certbot + Nginx（Ubuntu/Debian 最省事）

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx

# certbot 会自动修改 Nginx 配置、申请并安装证书
sudo certbot --nginx -d example.com -d www.example.com

# 验证自动续期流程（不会真的签发）
sudo certbot renew --dry-run

# 确认 systemd 定时器已启用（安装包时自动创建）
systemctl list-timers | grep certbot
```

- `certbot.timer` 每天执行两次，剩余 30 天内自动续期并 reload Nginx；
- 适合已有 Nginx、不想额外安装 acme.sh 的场景。

### 方案 D：Caddy（最省心，但占用 80/443）

```bash
sudo apt install -y caddy   # 或按官方源安装
```

`/etc/caddy/Caddyfile`：

```caddyfile
example.com, www.example.com {
    root * /var/www/html
    file_server
    # 反代后端时改为：reverse_proxy 127.0.0.1:8080
}
```

```bash
sudo systemctl reload caddy
```

特点：

- 自动申请、自动续期、自动热加载，**没有任何 cron 需要维护**；
- 需要自己监听 80/443，已有 Nginx 时要么替换，要么让 Caddy 做前置反代；
- 泛域名需要 DNS 插件，必须用 `xcaddy` 自行编译。

### 方案 E：图形化面板（不想碰命令行）

#### 宝塔面板

网站 → SSL → Let's Encrypt → 勾选域名 → 申请 → 开启"自动续签"。

#### 1Panel

证书 → 创建 ACME 账户 → 申请证书 → 开启自动续签，支持 Cloudflare、DNSPod 等 DNS 账号。

#### AllinSSL（宝塔出品的开源 SSL 全生命周期平台）

- 开源免费（AGPL-3.0），Go + lego 实现，支持 Let's Encrypt / ZeroSSL / Google Trust Services；
- 支持部署到 SSH、宝塔、1Panel、阿里云/腾讯云 CDN；
- 支持钉钉/飞书/邮件/Webhook 通知。

```bash
curl -sSO http://allinssl.bt.cn/install_allinssl.sh
bash install_allinssl.sh allinssl

# 默认访问：http://服务器IP:7979/allinssl
# 默认账号：allinssl / 密码：allinssldocker
```

使用流程：添加 DNS API → 新建证书（选 ACME）→ 新建部署（SSH/宝塔/1Panel/CDN）→ 设置续签与通知。

⚠️ 这类面板默认密码是公开的，装完第一件事：改密码、改端口、前面套 Nginx 反代加 HTTPS。

同类开源工具：**Certd**，也支持局域网部署给公网服务器签发证书。

### 方案 F：Windows Server + IIS（win-acme）

1. 从 win-acme.com 下载 x64 版，解压到 `C:\win-acme`；
2. 以管理员身份运行 `wacs.exe`；
3. 交互菜单选 `M`（Create certificate, full options）；
4. 输入域名 → 验证方式选 `SelfHosting`（HTTP-01，需 80 端口）或 DNS 插件；
5. 存储选 PEM / IIS → 自动绑定 IIS 站点；
6. 程序会自动创建计划任务 `win-acme renew`，每天检查，到期前 30 天自动续期并重新绑定 IIS。

测试时加 `--test` 参数走 staging 环境。

> Windows 也可以用 certbot，但对 IIS 的支持不如 win-acme 完整。

### 方案 G：Cloudflare（真正"零续签"）

**做法**

1. 把域名 NS 迁到 Cloudflare（免费）；
2. 开启 **Universal SSL**：CF 边缘证书自动签发、自动轮换，永远不用管；
3. 源站再签一张 **Cloudflare Origin CA**（有效期 15 年，仅 CF 回源信任），源站也不需要续签。

**国内场景的三个坑**

- 免费版流量走境外节点，大陆访问延迟和稳定性不如国内 CDN；
- 国内服务器 + 备案环境下，CF 代理可能带来访问异常，通常只能"仅 DNS"（灰云）；
- 灰云状态下 Universal SSL 不生效，又得回到源站 ACME；
- Cloudflare 中国网络（京东云合作节点）只有企业版才有。

**结论**：适合"域名本来就在 CF、访问以海外为主"的站点；纯国内业务不推荐作为主方案。

### 方案 H：国内云免费证书的真实情况

| 厂商 | 免费额度 | 有效期 | 能否 ACME 自动续 |
|------|---------|--------|-----------------|
| 阿里云 | 个人测试证书约 20 张/年 | 90 天 | ❌ 控制台手动 |
| 腾讯云 | 免费 DV 约 20 张/年 | 90 天（TrustAsia） | ❌ 手动 |
| 华为云 | 免费 DV | 90 天 | ❌ 手动 |
| 火山引擎 | 免费 DV | 90 天 | ❌ 手动 |
| 百度智能云 | 免费 DV | 已按新规调整 | ❌ 手动 |

> 额度与有效期以各家官网当前公告为准。

共同问题：

- 需要账号；
- 需要控制台操作；
- 一般不支持通配符；
- 不支持 ACME 自动申请。

**合规说明**：国内服务器使用 Let's Encrypt 等国外 CA 完全合法，浏览器信任即可。只有政企/等保项目可能额外要求国产 CA 或国密 SM2 证书，那种场景另议。

#### 备选：国内 ACME 兼容服务（来此加密 / FreeSSL.cn）

- 通过 EAB 凭证注册 ACME 账户；
- ACME 地址：`https://acme.litessl.com/acme/v2/directory`；
- 配合 `--eab-kid` / `--eab-hmac-key` 使用；
- 适合服务器访问 Let's Encrypt 网络不稳定的场景；
- **上线前务必在浏览器中确认证书链受信任**。

```bash
acme.sh --register-account \
  --server https://acme.litessl.com/acme/v2/directory \
  --eab-kid <kid> --eab-hmac-key <hmac> -m ops@example.com
```

## 五、选型决策图

```text
服务器 80 端口公网可达，且已备案？
├─ 是 → 方案 A（acme.sh HTTP-01）或方案 C（certbot）→ 不需要任何云账号
└─ 否 / 需要泛域名
   ├─ 有 Cloudflare / DNSPod / 华为云 DNS 的 API → 方案 B（DNS-01）
   ├─ DNS 在阿里云、但只能用受限权限 → RAM 子账号只授 DNS 权限，仍走方案 B
   └─ 完全拿不到 DNS 控制权 → 只能手动加 TXT（每 60 天一次），
                               或把域名 NS 迁到 Cloudflare 走方案 G
服务器是 Windows + IIS → 方案 F（win-acme）
不想碰命令行 → 方案 E（宝塔 / 1Panel / AllinSSL）
```

**初步建议**：如果站点现在 HTTP/HTTPS 访问正常，说明 80/443 和备案都没问题，直接上 **方案 A（acme.sh + HTTP-01）**——不依赖阿里云证书服务、不依赖任何 DNS API、不依赖控制台，装完只管收续期成功通知。若后续需要泛域名或多子域，再把同一个 acme.sh 切换为 DNS-01。

## 六、上线验证与排错清单

### 验证命令

```bash
# 查看证书签发者、有效期
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# 确认自动续期任务存在
crontab -l | grep acme                  # acme.sh
systemctl list-timers | grep certbot    # certbot

# 手动触发一次续期演练
~/.acme.sh/acme.sh --renew -d example.com --force --staging
```

### 常见坑

| 问题 | 原因 | 处理 |
|------|------|------|
| HTTP-01 验证失败 | 80 端口未放行 | 检查安全组、firewalld/ufw、云防火墙三层；拿不到控制台权限则改 DNS-01 |
| 验证超时 / 连接被拒 | 域名未备案 | 大陆服务器 80/443 会被拦截，改用 DNS-01 |
| 触发 Let's Encrypt 频率限制 | 测试用了生产环境 | 一律用 `--staging` 测试；同一注册域名每周 50 张、重复证书每周 5 张 |
| 证书更新了但浏览器还是旧的 | 漏了 `--reloadcmd` | 补上 `--reloadcmd "systemctl reload nginx"` 并重新 `--install-cert` |
| 换机后无法续期 | 没备份 ACME 账户 | 备份 `~/.acme.sh`（含账户密钥和全部证书） |
| 静默失败无人知晓 | 没有告警 | 配置 `--notify-hook`（邮件/Telegram/Slack），或用 Uptime Kuma 监控证书到期 |

### 安全与备份

- `~/.acme.sh/account.conf` 内含 DNS API 凭证，权限设为 `600`；
- 定期备份 `~/.acme.sh` 整个目录；
- 面板类工具（AllinSSL/宝塔/Certd）务必改默认密码、改端口、加 HTTPS 反代。

## 七、明天需要确认的环境信息（现场填写）

> 以下信息决定最终方案，确认后可直接落地。

| 项目 | 实际情况 | 备注 |
|------|---------|------|
| 服务器操作系统 | ☐ Linux（发行版：______） ☐ Windows Server（版本：______） | |
| Web 服务器 / 反向代理 | ☐ Nginx ☐ Apache ☐ IIS ☐ Caddy ☐ 其他：______ | |
| 服务器所在云 / IDC | ☐ 阿里云 ECS ☐ 腾讯云 ☐ 华为云 ☐ 其他：______ | |
| 域名数量 / 是否需要泛域名 | ______ 个域名；泛域名：☐ 需要 ☐ 不需要 | |
| 域名 DNS 托管在哪家 | ☐ 阿里云 ☐ DNSPod/腾讯云 ☐ Cloudflare ☐ 华为云 ☐ 其他：______ | |
| 是否有该 DNS 的 API 权限 | ☐ 有 ☐ 没有 ☐ 不清楚 | |
| 80 端口是否公网可达 | ☐ 是 ☐ 否 ☐ 不清楚 | |
| 域名是否已备案 | ☐ 是 ☐ 否 ☐ 不清楚 | |
| 是否使用 CDN / 负载均衡 | ☐ 否 ☐ 是：______ | 证书需部署到 CDN |
| 当前证书来源与到期日 | ______ | |
| 服务器 SSH / 远程权限 | ☐ 有 root ☐ 有 sudo ☐ 仅普通用户 ☐ 无 | |
| 能否接受安装第三方面板 | ☐ 可以 ☐ 不可以 | 决定是否走方案 E |

### 需要向对方确认的问题

1. "不通过阿里云账号"具体指：
   - ☐ 不用阿里云的 SSL 证书服务，但服务器/DNS 仍在阿里云；
   - ☐ 完全拿不到阿里云账号（连安全组、DNS 都改不了）。
2. 服务器 80 端口能否放行？如果拿不到安全组权限，是否接受走 DNS-01？
3. 域名 DNS 的控制权在谁手里？能否创建 API 子账号/令牌？
4. 现有站点是否允许短暂停服或改 Nginx 配置？
5. 是否需要泛域名证书（`*.example.com`）？
6. 是否有监控/告警渠道（钉钉、飞书、企业微信、邮件）可接入续期失败通知？
7. 后续证书是否需要部署到多台服务器或 CDN？如果是，考虑 AllinSSL/Certd 统一管理。

## 八、初步推荐（待环境确认后调整）

| 环境情况 | 推荐方案 |
|---------|---------|
| Linux + Nginx + 80 可达 + 已备案 | **方案 A：acme.sh + HTTP-01** |
| Linux + Nginx，但 80 不可达 / 未备案 / 要泛域名 | **方案 B：acme.sh + DNS-01（Cloudflare 或 DNSPod）** |
| Ubuntu/Debian，希望改动最小 | **方案 C：certbot --nginx** |
| 可以换 Web 服务器 / 新项目 | **方案 D：Caddy** |
| 不想碰命令行 | **方案 E：宝塔 / 1Panel / AllinSSL** |
| Windows Server + IIS | **方案 F：win-acme** |
| 域名可迁 Cloudflare、访问以海外为主 | **方案 G：Cloudflare Universal SSL** |

## 附录：常用命令速查

```bash
# acme.sh
~/.acme.sh/acme.sh --list                                  # 列出证书
~/.acme.sh/acme.sh --renew -d example.com --force          # 强制续期
~/.acme.sh/acme.sh --cron                                  # 手动跑一次续期检查
~/.acme.sh/acme.sh --install-cert ... --reloadcmd "..."    # 安装/更新部署钩子
~/.acme.sh/acme.sh --revoke -d example.com                 # 吊销

# certbot
certbot certificates                                       # 列出证书
certbot renew --dry-run                                    # 续期演练
certbot renew                                              # 执行续期
certbot delete --cert-name example.com                     # 删除证书

# 查看证书
openssl x509 -in /etc/nginx/ssl/example.com/fullchain.pem -noout -dates -issuer
```

## 参考来源

- [Buypass 停止签发 Go SSL 公告](https://community.buypass.com/t/y4y130p/buypass-terminates-issuance-of-gossl-certificates)
- [Sectigo：200 天有效期新规说明](https://sectigo.cn/blog/200-day-certificate-expiration-begins/)
- [阿里云：CA/B TLS 证书最大有效期变化通知](https://help.aliyun.com/zh/sfoa/product-overview/notice-regarding-changes-to-the-maximum-validity-period-of-ca-b-tls-certificates-and-preparations)
- [火山引擎：SSL 证书有效期缩短通知](https://docs.volcengine.com/docs/CertificateCenter/NoticeReductioninSSLCertificateValidityPeriod?lang=zh)
- [AllinSSL 官网](https://allinssl.com/)
- [acme.sh 官方仓库](https://github.com/acmesh-official/acme.sh)
- [来此加密 / FreeSSL 自动续签流程](https://blog.csdn.net/lpx666168/article/details/158312760)
