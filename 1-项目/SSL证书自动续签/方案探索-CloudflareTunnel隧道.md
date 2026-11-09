# 方案探索：Cloudflare Tunnel 隧道（服务器零证书、零公网端口）

> **状态**：调研中，未实施（2026-11-09）
> **定位**：SSL 证书自动续签项目的**第三条路线**——和前两条（[[执行方案-乐此加密|乐此加密]] CNAME 委派、[[备选方案-阿里云RAM角色|阿里云 RAM 角色]] TXT 直连）思路完全不同：**你的服务器从头到尾不碰证书**，证书在 Cloudflare 边缘，流量走加密隧道进来。
> **原理背景**：[[从域名验证到证书认证-完整闭环梳理]]
> **阅读位置**：按顺序读到这里——[[个人网站搭建|建站前传]] → [[从域名验证到证书认证-完整闭环梳理|背景原理]] → [[实战记录-RAM子账号AK方案|实战记录]] → 方案手册一（乐此）/ 方案手册二（RAM 角色）→ 本文（第三条路线调研）

---

## 一、和前两条方案的根本区别

前两条路线的模型是：**证书装在你的服务器上**，续签 = 服务器自己找 CA 签新证书，所以服务器必须有"改 DNS"或"开 80 端口"的资格。

这条路线把模型反过来：

- 证书放在 **Cloudflare 边缘**，自动签发、自动续期，跟你无关；
- 你的服务器只提供 HTTP 内容，通过一条**出站加密隧道**和 Cloudflare 相连；
- 不需要公网开放 80/443，甚至不需要公网 IP（NAT 后面也行）。

---

## 二、流量路径

```
客户端 ──HTTPS──> Cloudflare 边缘 ──加密隧道(QUIC/HTTP2)──> cloudflared ──HTTP──> 本机 Nginx (127.0.0.1:80)
        <──HTTPS(边缘证书)──                <────隧道────              <──HTTP 响应──
```

1. **客户端 → Cloudflare**：TLS 握手和证书验证发生在客户端和 Cloudflare 边缘之间，客户端看到的是 Cloudflare 的证书，跟你的服务器无关。
2. **Cloudflare → 你的服务器**：边缘把请求经 `cloudflared` 建立的**出站加密隧道**转发进来（隧道本身加密，不是裸 HTTP 走公网），落地后以 HTTP 交给本机 Nginx。
3. **Nginx → Cloudflare → 客户端**：Nginx 返回纯 HTTP 响应，沿隧道回到边缘，边缘用它自己的证书以 HTTPS 返回客户端。

**一个容易被误解的点**：不是服务器"主动发送"给 Cloudflare，而是 Cloudflare 通过隧道"来取"。因为隧道是服务器**主动向外建立**的，所以哪怕服务器在 NAT 后、没公网 IP、一个入站端口都不开，Cloudflare 也能把请求送进来——这就是"不需要开放 80/443"的原理。

**证书在哪**：客户端看到的证书 = Cloudflare 边缘的（自动续期）；你的 Nginx 上**不需要任何 SSL 证书**，之前 `ssl_certificate` 那套配置可以全部去掉，Nginx 只监听 80（甚至只 `127.0.0.1:80`）。

> 一句话总结：客户端和 Cloudflare 之间是 HTTPS（Cloudflare 的证书）；Cloudflare 和服务器之间是加密隧道；隧道落地到 Nginx 是 HTTP。服务器从头到尾不碰 TLS。

---

## 三、关键前提：域名 NS 必须迁到 Cloudflare（最大的决策）

Tunnel 要绑定自己的域名（`renzhengdejuju.online`），前提是域名的 **NS 记录指向 Cloudflare**。操作：Cloudflare 添加站点 → 回阿里云域名控制台把 NS 服务器改成 Cloudflare 分配的两个地址。免费，但**DNS 解析从此由 Cloudflare 管理**。

**不想动 NS？** Cloudflare 也提供临时随机域名（`*.trycloudflare.com`），但每次重启 `cloudflared` 域名都变，只适合临时测试，不适合正式站点。

### ⚠️ 迁移 NS 的三个连锁影响（原文没提，动手前必看）

1. **现有的 acme.sh 自动续期会静默失效**：域名 zone 迁走后，阿里云云解析不再是权威 DNS，`dns_ali` 插件调阿里云 API 写 TXT 记录就写了个寂寞，`acme.sh --cron` 每天照跑但永远签不出新证书。迁移后要么删掉旧的续期 cron，要么把插件换成 `dns_cf`（用 Cloudflare API Token）。
2. **备案审核期间绝对不要动 NS**：备案正在阿里云初审/管局审核中，阿里云核查时会检查域名解析状态，中途换 NS 解析商很可能导致核验失败、备案退回。**等备案下来再考虑迁移**。
3. **阿里云解析里的记录要在 Cloudflare 重建**：现在解析在阿里云（`@` / `www` 两条 A 记录指向 8.159.145.12），迁移后这些记录要在 Cloudflare 重新配置——不过走 Tunnel 方案的话，它们会被隧道自动生成的 CNAME 取代，倒省事了。

---

## 四、操作步骤（在现有阿里云 ECS 上，Alibaba Cloud Linux 3）

### 1. 安装 cloudflared

```bash
curl -fsSl https://pkg.cloudflare.com/cloudflared.repo | sudo tee /etc/yum.repos.d/cloudflared.repo
sudo yum update && sudo yum install cloudflared
cloudflared --version
```

### 2. 登录 Cloudflare 并授权域名

```bash
cloudflared tunnel login
```

终端给出链接，复制到浏览器登录 Cloudflare 账号，选择授权域名（`renzhengdejuju.online`），服务器上会生成 `cert.pem` 授权文件。

### 3. 创建命名隧道

```bash
cloudflared tunnel create renzheng-tunnel
```

生成隧道 ID 和凭证 JSON 文件，记下隧道 ID。

### 4. 配置隧道指向本地 Nginx

创建 `/etc/cloudflared/config.yml`：

```yaml
tunnel: 你的隧道ID
credentials-file: /root/.cloudflared/你的隧道ID.json
ingress:
  - hostname: renzhengdejuju.online
    service: http://localhost:80
  - hostname: www.renzhengdejuju.online
    service: http://localhost:80
  - service: http_status:404
```

`service: http://localhost:80` 指向本机 Nginx。Nginx 甚至可以改成只监听 `127.0.0.1:80`，更安全。

### 5. 把域名路由到隧道

```bash
cloudflared tunnel route dns renzheng-tunnel renzhengdejuju.online
cloudflared tunnel route dns renzheng-tunnel www.renzhengdejuju.online
```

自动在 Cloudflare DNS 里创建指向隧道的 CNAME 记录。

### 6. 安装为系统服务并启动

```bash
sudo cloudflared service install
sudo systemctl enable --now cloudflared
```

服务器重启后隧道自动恢复。

### 7. 清理服务器上原来的 SSL 配置（实施后做）

SSL 由 Cloudflare 边缘负责后，服务器上这些就是多余的，可以清理：

- Nginx 配置（`/etc/nginx/nginx.conf` 第 50~51 行）里的 `ssl_certificate` / `ssl_certificate_key` 两行和 443 监听块；
- `/etc/nginx/ssl/` 目录下的证书和私钥文件；
- acme.sh 的续期 cron（`crontab -l` 里 `acme.sh --cron` 那行）——zone 已经不在阿里云，留着它每天空跑还会静默失败；
- 注意：当前服务器用的是 acme.sh（不是 Certbot），所以不存在 `/var/www/certbot/` 之类的东西，清上述几样即可。

---

## 五、代价与限制（认真权衡）

**国内访问速度（最大的代价）**：Cloudflare 在国内没有合规边缘节点，国内用户访问流量通常绕海外（圣何塞、香港等），延迟和稳定性不可控，实测首屏可能 1.2~1.8 秒，晚高峰更差。网站若主要面向国内用户，这个体验下降要认真掂量。

**免费版限制**：单个请求体上限 100MB；不允许用 Tunnel 串流视频（Plex/Jellyfin 这类）。普通网页和 API 不受影响。

**⚠️ 合规提醒（最重要）**：服务器在中国内地，**ICP 备案义务不因为流量走隧道而消失**。用隧道让"未备案/审核中"的域名经内地服务器对外提供服务，属于绕开服务商合规检查的行为，存在被阿里云检测到后**封停服务器**的风险。如果真要走 Cloudflare 路线，合理的搭配是：**备案完成后再实施**，或者把网站迁到香港/海外服务器（免备案）+ Cloudflare。

---

## 六、三条路线总览（本项目全局视角）

- **方案一 乐此加密**：证书仍在自己服务器上，续签靠 CNAME 委派给第三方代写 TXT。约 ¥99 一次性，不碰阿里云 API，服务器要开 80/443。
- **方案二 阿里云 RAM 角色 / 子账号 AK**（已实战 ✅）：证书在自己服务器上，acme.sh 直连阿里云 DNS 写 TXT。0 元自主可控，服务器要开 80/443。
- **方案三 Cloudflare Tunnel**（本文，调研中）：证书根本不在自己服务器上，在 CF 边缘；服务器零证书、可以零端口。代价：NS 迁移 + 国内速度 + 合规敏感期。

一句话选择：**要国内访问速度和合规稳妥 → 方案一/二；要彻底不碰证书、不在意国内速度（或服务器在海外）→ 方案三**。

---

## 参考链接

- [Cloudflare Tunnel 官方文档](https://developers.cloudflare.com/tunnel/)
- [Locally-managed tunnels: create local tunnel](https://developers.cloudflare.com/tunnel/features/locally-managed-tunnels/create-local-tunnel/)
- [cloudflared 部署笔记（GitLab）](https://gitlab.com/jerrybai/notes/-/blob/3f3d9530b2bf17330ad3a4103af68fc6df086719/cloudflare-tunnel.md)
- [aiinfra-atlas 的 Cloudflare 文档（GitHub）](https://github.com/AI-as-Infrastructure/aiinfra-atlas/blob/fe8370e491554c0b1d4cb35a9ed29fc892a374d6/docs/cloudflare.md)
- [openalgo 的 cloudflared 配置示例（GitHub）](https://github.com/marketcalls/openalgo/blob/eb4cce0a117224107ce26b263813808d49817de8/.cloudflared/README.md)
- [Cloudflare 国内访问实测讨论](https://www.locdd.com/t/topic/49806/4)
- [NS 迁移操作参考（CSDN）](https://riat-.blog.csdn.net/article/details/159392012)
