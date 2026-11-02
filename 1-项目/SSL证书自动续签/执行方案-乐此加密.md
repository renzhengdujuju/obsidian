# SSL 证书自动续签（最终执行版：乐此加密）

  

> 整理日期：2026-11-01

> 方案确定：2026-11-02

> 最终方案：**乐此加密（letsssl.cn）CNAME 委派 + 自动续期客户端**

> 费用：证书免费；自动续期公开资料为 **¥30 一次性买断**（以控制台实际显示为准）

> 说明：文中的 `ff.example.com` 是占位示例，实操时全部替换成真实完整域名（FQDN）。

  

---

  

# 第一部分：最终执行方案（乐此加密）

  

## 1. 先回答：要花多少钱？

  

| 项目 | 费用 | 说明 |

|------|------|------|

| SSL 证书本身（Let's Encrypt / Google Trust Services / ZeroSSL） | **免费** | 90 天有效期，手动续签也免费 |

| 自动续期功能 | **约 ¥30 一次性买断，永久有效** | 来自公开资料/公众号文章；官方控制台可能显示不同套餐，以实际为准 |

| 域名、服务器 | 已有，不额外收费 | |

| 是否需要“充值余额” | 一般不需要 | 更像购买一次性权益，按控制台提示支付即可 |

  

> **必须注意**：¥30 这个数字不是从你们账号里看到的，而是公开文章写的。明天登录 `console.letsssl.cn`，在「我的证书 → 一键部署 & 自动续期」页面确认实际价格；如果价格不同，以控制台为准，先截图给老板确认再付款。

>

> 如果老板不想付这 ¥30：可以继续免费用手动续签，但那就回到“每 80 天手动一次”的老问题，不符合本次目标。

  

---

  

## 2. 方案原理（30 秒版）

  

1. 你在阿里云 DNS 保留一条 CNAME：

   `_acme-challenge.ff.example.com → xxxx.cname.letsssl.cn`

2. 以后 Let's Encrypt 要验证域名时，会顺着这条 CNAME 去问乐此加密的服务器。

3. 乐此加密在自己那边写 TXT 暗号，替你完成 DNS-01 验证。

4. 服务器上装一个乐此加密的续期客户端，它定期检查证书到期时间，自动续签、写入证书文件、执行你配置的 reload 命令。

5. 整个过程不需要登录阿里云控制台，也不需要给服务器阿里云 API 权限。

  

> 这条 CNAME 是“验证通道”，不是证书本身。删掉它，已经签发的证书不会立刻失效，但乐此加密以后就无法自动续期了。**所以本方案中不要删它。**

  

---

  

## 3. 前置确认（动手前必须确认）

  

| # | 确认项 | 怎么查 | 不确定时 |

|---|--------|--------|---------|

| 1 | 乐此加密账号是谁的、能不能登录 | 问老板/看浏览器保存的账号 | 先别付款，先确认账号归属 |

| 2 | 证书要覆盖的完整域名（FQDN） | 阿里云 DNS 控制台看 zone + 主机记录 | 记下 zone，例如 `example.com`，再拼出 `ff.example.com` |

| 3 | 是否要泛域名 `*.ff.example.com`，有没有三级域名 | 问开发/看业务 | 泛域名不覆盖 `a.b.ff.example.com` |

| 4 | 证书部署到哪台服务器 | 截图里 `website` 是 47.102.116.218，`*.ff` 是 180.175.161.67，可能是两台 | 问老板/看资产表；每台终止 TLS 的机器都要部署 |

| 5 | Nginx 是宿主机还是 Docker | `ps -ef \| grep nginx`、`docker ps` | 决定 reload 命令和证书路径 |

| 6 | 当前证书和私钥的路径 | `nginx -T \| grep -E "ssl_certificate"` | 抄下来，配置客户端时要填 |

| 7 | 服务器 OS 和 CPU 架构 | `cat /etc/os-release`、`uname -m` | AMD64 / ARM64 选错客户端跑不起来 |

| 8 | 是否有 root/SSH 权限 | 直接登录试 | 没有就没法装客户端 |

| 9 | 服务器能否访问 letsssl.cn | `curl -I https://letsssl.cn` | 不能出网就要配代理/白名单 |

| 10 | 8899 端口的安全策略 | 安全组/防火墙 | 优先用 SSH 隧道，不要暴露公网 |

| 11 | ¥30 是否可报销 | 问老板/财务 | 先确认再付款 |

| 12 | 现有证书是否已备份 | 见 5.1 | 没备份先备份，别直接覆盖 |

| 13 | 现有 CNAME 是否和平台要求一致 | `dig +short CNAME _acme-challenge.ff.example.com` | 不一致就按平台给的值改 |

  

---

  

## 4. 平台侧操作（约 20 分钟）

  

### 4.1 登录

  

- 打开 `https://console.letsssl.cn/`（或从 `letsssl.cn` 点“登录/控制台”）。

- 用老板给的账号登录；没有账号就先注册（一般邮箱/手机号）。

  

### 4.2 添加域名 / 申请证书

  

进入「我的证书」→「申请证书」：

  

| 选项 | 建议值 |

|------|--------|

| 域名 | `ff.example.com` 和 `*.ff.example.com`（两条都加，泛域名不包含主域名） |

| CA | **Let's Encrypt**（兼容性最好，90 天） |

| 验证方式 | **DNS 验证** |

| 证书类型 | 泛域名/多域名，按实际选 |

  

提交后，平台会显示需要添加的 DNS 记录，通常是：

  

```

主机记录：_acme-challenge.ff

类型：CNAME

值：xxxx.cname.letsssl.cn

```

  

### 4.3 核对现有的 CNAME

  

你们截图里已经有这条记录：

  

```

_acme-challenge.ff  CNAME  5244...cname.letsssl.cn

```

  

在服务器上查一下：

  

```bash

dig +short CNAME _acme-challenge.ff.example.com

```

  

- 如果和平台显示的值**一致**：什么都不用改，直接等签发。

- 如果**不一致**：以平台当前显示的值为准，去阿里云 DNS 修改；改完等 10–30 分钟（截图里 TTL 是 10 分钟）再继续。

- 如果平台要求的是 TXT 而不是 CNAME：说明这个域名没走委派模式，停下来问平台客服，不要自己乱加（会冲突）。

  

> **不要删除这条 CNAME。** 本方案靠它续命。

  

### 4.4 购买/开通自动续期

  

证书签发成功后：

  

- 进入「我的证书 → 一键部署 & 自动续期」。

- 如果提示需要开通自动续期，按控制台账单支付（公开资料约 ¥30 一次性，永久有效）。

- 支付前截图价格给老板确认；支付后保留订单截图/发票。

  

### 4.5 生成部署命令

  

在同一页面填写：

  

| 配置项 | 示例值 | 说明 |

|--------|--------|------|

| 服务器类型 | Nginx | 如果是 Apache/Tomcat/IIS 就选对应项 |

| 操作系统 | Linux | |

| CPU 架构 | AMD64 或 ARM64 | 用 `uname -m` 确认：`x86_64`→AMD64，`aarch64`→ARM64 |

| 私钥文件路径 | `/etc/nginx/ssl/ff.example.com.key` | 填 Nginx 当前使用的 key 路径 |

| 证书文件路径 | `/etc/nginx/ssl/ff.example.com.cer` | 填 Nginx 当前使用的 fullchain 路径 |

| 应用重载命令 | `nginx -t && nginx -s reload` | 先 `nginx -t` 再 reload，避免坏配置直接上线 |

| 部署模式 | **Web 后台服务模式** | 可视化，推荐；CLI 模式适合脚本化 |

  

> 如果 Nginx 在 Docker 里：

> - reload 命令改成 `docker exec <容器名> nginx -t && docker exec <容器名> nginx -s reload`

> - 证书路径要填**宿主机上挂载给容器的目录**，否则容器里看不到新证书。

  

点「生成部署命令」，复制命令（里面含 `TOKEN`，不要发到群里/提交到代码库）。

  

---

  

## 5. 服务器侧操作（约 20 分钟）

  

### 5.1 先备份（必须）

  

```bash

# 证书和私钥

mkdir -p /root/ssl-backup-$(date +%F)

cp -a /etc/nginx/ssl/. /root/ssl-backup-$(date +%F)/ 2>/dev/null || true

  

# Nginx 配置

mkdir -p /root/nginx-conf-backup-$(date +%F)

cp -a /etc/nginx/conf.d/. /root/nginx-conf-backup-$(date +%F)/ 2>/dev/null || true

nginx -T > /root/nginx-full-config-$(date +%F).txt 2>/dev/null || true

  

# 记录当前证书到期日，方便对比

find /etc/nginx -name '*.pem' -o -name '*.cer' -o -name '*.crt' 2>/dev/null | while read f; do

  echo "== $f"; openssl x509 -in "$f" -noout -subject -issuer -dates 2>/dev/null

done

```

  

### 5.2 确认环境

  

```bash

# 架构

uname -m

  

# Nginx 是否在跑、是否 Docker

systemctl status nginx --no-pager 2>/dev/null

docker ps 2>/dev/null | grep -i nginx

  

# 当前证书路径

nginx -T 2>/dev/null | grep -E "ssl_certificate|ssl_certificate_key"

  

# 确认 CNAME

dig +short CNAME _acme-challenge.ff.example.com

```

  

### 5.3 下载并检查安装脚本

  

不要直接 `curl | bash`，先下载看一眼：

  

```bash

curl -f https://letsssl.cn/download/letsssl-renew/install.sh -o /tmp/letsssl-install.sh

less /tmp/letsssl-install.sh

```

  

重点看：

  

- 下载的二进制 URL 是不是 `letsssl.cn` 域名；

- 有没有执行奇怪的系统修改；

- 安装路径是不是 `/usr/local/bin`；

- systemd 服务名是不是 `letsssl`；

- 有没有把你的 TOKEN 写到明文文件里。

  

确认没问题再执行。

  

### 5.4 执行安装

  

用平台生成的命令（把 `TOKEN` 换成你的）：

  

```bash

sudo TOKEN=xxxx SERVER_TYPE=nginx bash /tmp/letsssl-install.sh

```

  

安装完成后，终端会打印类似：

  

```

服务状态：active (running)

访问地址：http://<服务器IP>:8899/

首次部署认证口令：xxxxxxxx xxxxxxxx

```

  

记下这个一次性口令。

  

### 5.5 打开管理面板（推荐 SSH 隧道）

  

**不要把 8899 直接暴露公网。** 在你自己的电脑上建 SSH 隧道：

  

```bash

ssh -L 8899:127.0.0.1:8899 root@<服务器IP>

```

  

然后浏览器访问：

  

```

http://127.0.0.1:8899/

```

  

如果必须远程访问，至少在安全组/防火墙里只放行办公 IP。

  

### 5.6 初始化并配置

  

1. 用终端打印的“首次部署认证口令”设置管理员密码（强密码）。

2. 登录后按提示「导入原有定时任务」。

3. 在任务列表里核对：

   - 域名是否正确；

   - 私钥路径、证书路径是否正确；

   - reload 命令是否正确；

   - 检查周期（一般是每 15 天检查一次，到期前自动续）；

   - 是否开启旧证书备份。

  

### 5.7 先手动验证 reload 命令

  

在服务器上执行一次：

  

```bash

nginx -t && nginx -s reload

```

  

必须成功。否则客户端续期后会卡在 reload 这一步。

  

---

  

## 6. 验证（必须做）

  

```bash

# 1. 服务状态

systemctl status letsssl --no-pager

  

# 2. 实时日志

journalctl -u letsssl -f

  

# 3. 证书文件内容

openssl x509 -in /etc/nginx/ssl/ff.example.com.cer -noout -subject -issuer -dates

  

# 4. 线上实际返回的证书

echo | openssl s_client -connect ff.example.com:443 -servername ff.example.com 2>/dev/null \

  | openssl x509 -noout -subject -issuer -dates

  

# 5. HTTPS 是否正常

curl -I https://ff.example.com

  

# 6. 文件更新时间

stat -c '%y %n' /etc/nginx/ssl/ff.example.com.key /etc/nginx/ssl/ff.example.com.cer

```

  

浏览器再打开一次网站，点地址栏的小锁 → 查看证书：

  

- 颁发者应该是 Let's Encrypt / Google Trust Services / ZeroSSL；

- 有效期应该是新的 90 天；

- 域名包含 `ff.example.com` 和 `*.ff.example.com`。

  

---

  

## 7. 自动续期演练（建议做一次）

  

在面板里找「立即检查 / 立即续期」之类的按钮，点一次，然后：

  

```bash

journalctl -u letsssl -f

```

  

观察日志是否出现：

  

- 检查证书到期时间；

- 未到期 → 跳过（正常）；

- 到期 → 申请新证书 → 写文件 → 执行 reload。

  

同时对比证书序列号/到期日有没有变化：

  

```bash

openssl x509 -in /etc/nginx/ssl/ff.example.com.cer -noout -serial -dates

```

  

> 不要反复点“立即续期”，会触发 CA 频率限制。点一次确认链路通即可。

  

如果面板没有强制续期按钮，至少确认：

  

```bash

systemctl list-timers | grep -i letsssl

# 或看面板里的任务下次执行时间

```

  

---

  

## 8. 监控与告警（强烈建议）

  

自动续期也可能失败（CNAME 被误删、平台故障、网络问题）。加一层兜底：

  

### 方案一：Uptime Kuma / 外部 HTTPS 监控

  

- 添加 HTTPS 监控，开启证书到期提醒（提前 20 天）。

- 这是最省事的，不用写代码。

  

### 方案二：服务器上的到期检查脚本

  

```bash

#!/usr/bin/env bash

# /usr/local/bin/check-ssl-expiry.sh

DOMAIN="ff.example.com"

WARN_DAYS=20

WEBHOOK="https://oapi.dingtalk.com/robot/send?access_token=替换成你的"

  

END=$(echo | openssl s_client -connect "$DOMAIN:443" -servername "$DOMAIN" 2>/dev/null \

  | openssl x509 -noout -enddate | cut -d= -f2)

[ -z "$END" ] && exit 1

  

END_TS=$(date -d "$END" +%s)

NOW_TS=$(date +%s)

LEFT=$(( (END_TS - NOW_TS) / 86400 ))

  

if [ "$LEFT" -lt "$WARN_DAYS" ]; then

  curl -s -H 'Content-Type: application/json' \

    -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"$DOMAIN 证书剩余 ${LEFT} 天，请检查乐此加密续期是否正常\"}}" \

    "$WEBHOOK"

fi

```

  

加 cron：

  

```bash

(crontab -l 2>/dev/null; echo '30 9 * * * /usr/local/bin/check-ssl-expiry.sh') | crontab -

```

  

### 方案三：日历提醒

  

在证书到期前 15 天设一个日历提醒，作为最后兜底。

  

---

  

## 9. 安全加固

  

| 项目 | 要求 |

|------|------|

| 8899 端口 | 不要暴露公网；优先 SSH 隧道；必须远程时只放行办公 IP |

| 面板密码 | 强密码，不用默认/弱口令 |

| TOKEN | 视同密码；不要发群、不要提交代码库 |

| 命令历史 | 安装命令含 TOKEN，执行后可用 `history -d` 删除对应行，或命令前加空格并设置 `HISTCONTROL=ignorespace` |

| 客户端权限 | 它以 root 运行，装之前确认公司安全规范允许 |

| 旧证书备份 | 保留至少一个续期周期 |

| CNAME | 不要删；删了自动续期就断 |

  

---

  

## 10. 回滚与卸载

  

### 回滚（续期出问题时）

  

```bash

# 1. 停掉客户端，防止它继续改文件

sudo systemctl stop letsssl

sudo systemctl disable letsssl

  

# 2. 恢复旧证书

cp -a /root/ssl-backup-<日期>/. /etc/nginx/ssl/

  

# 3. 检查并重载

nginx -t && nginx -s reload

```

  

### 彻底卸载（确认不再使用后）

  

```bash

sudo systemctl stop letsssl

sudo systemctl disable letsssl

sudo rm -f /etc/systemd/system/letsssl.service

sudo systemctl daemon-reload

  

# 以下路径先确认再删，不要盲删

ls -l /usr/local/bin/letsencrypt-renew

ls -ld /usr/local/letsssl-data

# sudo rm -f /usr/local/bin/letsencrypt-renew

# sudo rm -rf /usr/local/letsssl-data

```

  

CNAME 是否删除：如果以后都不再用乐此加密，可以删；如果只是暂时停用，保留它。

  

---

  

## 11. 常见问题排查

  

| 现象 | 可能原因 | 处理 |

|------|---------|------|

| 8899 打不开 | 服务没起 / 安全组没放行 / 只监听了 127.0.0.1 | `systemctl status letsssl`；用 SSH 隧道；检查防火墙 |

| 安装脚本下载失败 | 服务器不能出网 / DNS 问题 | `curl -I https://letsssl.cn`；检查代理/安全组 |

| 客户端启动失败 | 架构选错 / 权限不足 | `uname -m` 重新选 AMD64/ARM64；用 root |

| 续期失败 | CNAME 被删/改、平台账号异常、域名过期 | `dig` 查 CNAME；登录平台看证书状态 |

| 续期成功但网站还是旧证书 | reload 命令失败 / 路径不对 / Docker 没挂载 | 手动执行 reload；`nginx -T` 看实际路径；检查容器挂载 |

| Nginx reload 报错 | 配置本身有问题 | 先 `nginx -t` 看报错，修好再 reload |

| 面板密码忘了 | —— | 查官方文档/重置流程；必要时重装客户端（先备份证书） |

| 证书到期但没续 | 任务没导入 / 客户端没运行 / TOKEN 失效 | 看面板任务列表；`journalctl -u letsssl`；重新生成 TOKEN |

| 多台服务器只有一台更新 | 客户端只装了一台 | 每台终止 TLS 的服务器都装，或把证书同步过去 |

  

---

  

## 12. 明天执行清单（打勾）

  

- [ ] 确认乐此加密账号能登录

- [ ] 确认真实 FQDN（zone + 主机记录）

- [ ] 确认要覆盖 `ff.example.com` 和 `*.ff.example.com`

- [ ] 确认证书部署到哪台/哪几台服务器

- [ ] 确认 Nginx 是宿主机还是 Docker

- [ ] 记录当前证书/私钥路径

- [ ] 备份现有证书、私钥、Nginx 配置

- [ ] `dig` 确认 `_acme-challenge.ff` CNAME 存在且与平台一致

- [ ] 平台申请/确认证书已签发

- [ ] 控制台确认自动续期价格（约 ¥30 一次性），截图给老板

- [ ] 付款/开通自动续期

- [ ] 填写私钥路径、证书路径、reload 命令

- [ ] 生成并复制部署命令

- [ ] 服务器下载安装脚本并检查内容

- [ ] 执行安装

- [ ] SSH 隧道打开 8899，设置管理员密码

- [ ] 导入/确认续期任务

- [ ] 手动执行 `nginx -t && nginx -s reload` 验证

- [ ] 验证证书颁发者、有效期、域名

- [ ] 触发一次续期检查，看日志

- [ ] 配置到期监控/告警

- [ ] 记录回滚步骤和备份路径

- [ ] 给老板交付说明

  

---

  

## 13. 还需要问老板/确认的问题

  

1. 乐此加密账号是谁注册的？以后续费用哪个账号登录？

2. 自动续期 ¥30 一次性费用，能否报销？支付方式是什么？

3. 证书要部署到几台服务器？`*.ff` 的 180.175.161.67 和 `website` 的 47.102.116.218 分别是什么角色？

4. 是否有三级域名（`a.b.ff.example.com`）？泛域名不覆盖三级。

5. 公司安全规范是否允许安装第三方客户端（root 运行、开 8899 端口）？

6. 是否需要把证书同步到 CDN / SLB / WAF？

7. 变更窗口是什么时候？reload 虽然平滑，但配置错误会影响业务。

8. 有没有现成的监控平台（Uptime Kuma / Zabbix / 钉钉告警）可以接证书到期提醒？

  

---

  

## 14. 参考链接

  

- [乐此加密官网](https://letsssl.cn/)

- [自动续期客户端部署与使用指南](https://letsssl.cn/pages/auto-renewal-guide.html)

- [乐此加密功能说明](https://letsssl.cn/pages/features.html)

- [乐此加密关于页面](https://letsssl.cn/pages/about.html)

- [30 元永久自动续期的第三方说明（微信文章）](http://mp.weixin.qq.com/s?__biz=MzkwOTIxNTc2NQ==&mid=2247498349&idx=1&sn=018afc717fbba41e4d44fa10aa684da7)

  

> 价格、页面名称、安装参数可能随平台更新变化，**以控制台和官方指南当前显示为准**。

  

---

  

# 第二部分：通用方案调研（备查）

  

## 一、核心结论

  

1. **“免费 + 自动续签”本质上只有一条路**：ACME 协议 + 免费 CA（Let's Encrypt / ZeroSSL / Google Trust Services），客户端使用 `acme.sh`、`certbot`、`Caddy`、`win-acme` 等工具。

2. **国内云厂商的“免费证书”基本都不支持 ACME 自动申请**，仍然要进控制台手动操作，解决不了 80 天手动签一次的问题。它们只是“免费的证书”，不是“免费的自动化”。

3. **完全可以不碰阿里云账号**：

   - 服务器 80 端口公网可达 → 走 **HTTP-01** 验证，全程不需要任何云账号；

   - 需要泛域名，或 80 端口被封/未备案 → 走 **DNS-01**，只需一个 DNS 服务商的 API（Cloudflare、DNSPod、华为云都行，不一定是阿里云）。

4. **越早自动化越好**：2026-03-15 起所有公网 TLS 证书最长 **200 天**，2027-03-15 降到 **100 天**，2029 年 **47 天**。手动续签只会越来越频繁。

  

---

  

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

  

---

  

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

  

---

  

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

  --key-file       /etc/nginx/ssl/example.com/privkey.pem \

  --fullchain-file /etc/nginx/ssl/example.com/fullchain.pem \

  --reloadcmd      "systemctl reload nginx"

  

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

  

---

  

### 方案 B：DNS-01（泛域名 / 80 端口不可用）

  

泛域名（`*.example.com`）只能走 DNS-01。以 Cloudflare 为例：

  

```bash

export CF_Token="你的API令牌"   # 权限只需 Zone:Read + DNS:Edit

  

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

  

---

  

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

  

---

  

### 方案 D：Caddy（最省心，但占用 80/443）

  

```bash

sudo apt install -y caddy   # 或按官方源安装

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

  

---

  

### 方案 E：图形化面板（不想碰命令行）

  

#### 宝塔面板

  

网站 → SSL → Let's Encrypt → 勾选域名 → 申请 → 开启“自动续签”。

  

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

  

---

  

### 方案 F：Windows Server + IIS（win-acme）

  

1. 从 win-acme.com 下载 x64 版，解压到 `C:\win-acme`；

2. 以管理员身份运行 `wacs.exe`；

3. 交互菜单选 `M`（Create certificate, full options）；

4. 输入域名 → 验证方式选 `SelfHosting`（HTTP-01，需 80 端口）或 DNS 插件；

5. 存储选 PEM / IIS → 自动绑定 IIS 站点；

6. 程序会自动创建计划任务 `win-acme renew`，每天检查，到期前 30 天自动续期并重新绑定 IIS。

  

测试时加 `--test` 参数走 staging 环境。

  

> Windows 也可以用 certbot，但对 IIS 的支持不如 win-acme 完整。

  

---

  

### 方案 G：Cloudflare（真正“零续签”）

  

**做法**

  

1. 把域名 NS 迁到 Cloudflare（免费）；

2. 开启 **Universal SSL**：CF 边缘证书自动签发、自动轮换，永远不用管；

3. 源站再签一张 **Cloudflare Origin CA**（有效期 15 年，仅 CF 回源信任），源站也不需要续签。

  

**国内场景的三个坑**

  

- 免费版流量走境外节点，大陆访问延迟和稳定性不如国内 CDN；

- 国内服务器 + 备案环境下，CF 代理可能带来访问异常，通常只能“仅 DNS”（灰云）；

- 灰云状态下 Universal SSL 不生效，又得回到源站 ACME；

- Cloudflare 中国网络（京东云合作节点）只有企业版才有。

  

**结论**：适合“域名本来就在 CF、访问以海外为主”的站点；纯国内业务不推荐作为主方案。

  

---

  

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

  

---

  

## 五、选型决策图

  

```

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

  

---

  

## 六、上线验证与排错清单

  

### 验证命令

  

```bash

# 查看证书签发者、有效期

echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \

  | openssl x509 -noout -subject -issuer -dates

  

# 确认自动续期任务存在

crontab -l | grep acme                  # acme.sh

systemctl list-timers | grep certbot    # certbot

  

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

  

---

  

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

  

1. “不通过阿里云账号”具体指：

   - ☐ 不用阿里云的 SSL 证书服务，但服务器/DNS 仍在阿里云；

   - ☐ 完全拿不到阿里云账号（连安全组、DNS 都改不了）。

2. 服务器 80 端口能否放行？如果拿不到安全组权限，是否接受走 DNS-01？

3. 域名 DNS 的控制权在谁手里？能否创建 API 子账号/令牌？

4. 现有站点是否允许短暂停服或改 Nginx 配置？

5. 是否需要泛域名证书（`*.example.com`）？

6. 是否有监控/告警渠道（钉钉、飞书、企业微信、邮件）可接入续期失败通知？

7. 后续证书是否需要部署到多台服务器或 CDN？如果是，考虑 AllinSSL/Certd 统一管理。

  

---

  

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

  

---

  

## 附录：常用命令速查

  

```bash

# acme.sh

~/.acme.sh/acme.sh --list                                  # 列出证书

~/.acme.sh/acme.sh --renew -d example.com --force          # 强制续期

~/.acme.sh/acme.sh --cron                                  # 手动跑一次续期检查

~/.acme.sh/acme.sh --install-cert ... --reloadcmd "..."    # 安装/更新部署钩子

~/.acme.sh/acme.sh --revoke -d example.com                 # 吊销

  

# certbot

certbot certificates                                       # 列出证书

certbot renew --dry-run                                    # 续期演练

certbot renew                                              # 执行续期

certbot delete --cert-name example.com                     # 删除证书

  

# 查看证书

openssl x509 -in /etc/nginx/ssl/example.com/fullchain.pem -noout -dates -issuer

```

  

---

  

## 参考来源

  

- [Buypass 停止签发 Go SSL 公告](https://community.buypass.com/t/y4y130p/buypass-terminates-issuance-of-gossl-certificates)

- [Sectigo：200 天有效期新规说明](https://sectigo.cn/blog/200-day-certificate-expiration-begins/)

- [阿里云：CA/B TLS 证书最大有效期变化通知](https://help.aliyun.com/zh/sfoa/product-overview/notice-regarding-changes-to-the-maximum-validity-period-of-ca-b-tls-certificates-and-preparations)

- [火山引擎：SSL 证书有效期缩短通知](https://docs.volcengine.com/docs/CertificateCenter/NoticeReductioninSSLCertificateValidityPeriod?lang=zh)

- [AllinSSL 官网](https://allinssl.com/)

- [acme.sh 官方仓库](https://github.com/acmesh-official/acme.sh)

- [来此加密 / FreeSSL 自动续签流程](https://blog.csdn.net/lpx666168/article/details/158312760)