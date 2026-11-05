# 实战记录：RAM 子账号 AK 方案续签证书（acme.sh + dns_ali）

> **日期**：2026-11-05　**耗时**：约 1 小时　**结果**：✅ 证书签发并部署成功，自动续期已生效
> **路线**：RAM 子账号 `acme-dns` + 长期 AccessKey + acme.sh 自带 `dns_ali` 插件
> **前情**：[[个人网站搭建|个人网站搭建]] 上线时申请的免费证书 90 天到期——这是第一次实战续签演练
> **路线定位**：即《备选方案-阿里云RAM角色》附录 12.1 的"退而求其次"路线，与手册主路线（ECS RAM 角色 + STS）的区别见文末对比
> **原始对话存档**：[[千问-阿里 AI 助手.html]]（和千问 AI 边操作边问的完整记录）

---

## 🗺️ 流程总览

```
① 创建 RAM 子账号 acme-dns（踩了 3 个坑）
② 保存 AccessKey（Secret 只显示一次）
③ 授予 AliyunDNSFullAccess 权限（忘了授权 → 申请必失败）
④ 服务器安装 acme.sh
⑤ 密钥写入 ~/.bashrc 永久化（export 转眼就失效的坑）
⑥ 一条命令签发证书（curl 35/28 不是错误）
⑦ 部署到 Nginx（fullchain 信任链 + 路径替换）
⑧ 确认自动续期（自带 cron，别手动加 --force 任务）
```

---

## 第 1 步：创建 RAM 子账号（踩了 3 个坑）

路径：阿里云控制台 → **RAM 访问控制** → **身份管理 → 用户 → 创建用户**。登录名 `acme-dns`，显示名"SSL证书DNS验证专用"。

![[Pasted image 20261105115251.png]]
> 图：RAM 创建用户页面

**坑 1：保存下来的 ak 文件是空的**
创建时没勾"为子账户生成访问密钥"（阿里云新版默认不直接生成，因为网页直出的永久 AK 泄露风险高）。解决：进用户详情页 → **认证管理** → **创建 AccessKey** 手动生成。

![[Pasted image 20261105122125.png]]
> 图：保存结果下载的 ak 文件为空——说明创建时没有生成 AccessKey

![[Pasted image 20261105122221.png]]
> 图：进入 acme-dns 账号的"认证管理"页手动创建 AccessKey

**坑 2：弹出"使用场景"确认**
选哪个都不影响最终密钥；服务器脚本场景选"在阿里云 ECS 上运行的服务"或"本地开发环境"都可以。

![[Pasted image 20261105122236.png]]
> 图：AccessKey 使用场景确认弹窗——勾选"我确认必须创建 AccessKey"后继续

**坑 3：误勾"控制台访问 + MFA 必须开启"**
这是给"需要网页登录的账号"用的。我们要给服务器上的脚本用，**什么都不用勾**；勾了 MFA 脚本会直接失败（脚本没法输手机验证码）。

![[Pasted image 20261105122246.png]]
> 图：错误的创建方式——勾了控制台访问和 MFA，对脚本场景完全不适用

---

## 第 2 步：保存 AccessKey（只有一次机会）

创建 AccessKey 后弹出的 **AccessKey ID / AccessKey Secret 必须立刻复制保存**——Secret 关掉弹窗就再也找不回，只能重新生成。

![[Pasted image 20261105122526.png]]
> 图：创建成功后复制出的凭证信息，其中 AccessKey ID 和 Secret 是 acme.sh 要用的核心

---

## 第 3 步：授予 DNS 权限（忘了这步 = 申请必失败）

用户详情页 → **权限管理 → 添加权限** → 搜索并选中 **`AliyunDNSFullAccess`**（管理云解析 DNS 的权限）→ 确定。

**踩坑记录：忘记附加权限策略**

- **现象**：跑签发命令时证书申请失败。
- **原因**：子账号手里只有钥匙（AK），没有门上的锁芯（权限），无法调 DNS API 加 TXT 记录。
- **解决**：回控制台补上 `AliyunDNSFullAccess` 即可。

![[Pasted image 20261105122611.png]]
> 图：申请失败后回头检查——发现创建用户时漏了附加权限策略这一步

> 💡 **权限和密钥是独立的**：补授权后**不需要**重新生成 AK，之前执行的 `export` 也依然有效——权限是"锁芯"，AK 是"钥匙"，换锁芯不用换钥匙，且立即生效。

---

## 第 4 步：服务器安装 acme.sh

```bash
curl https://get.acme.sh | sh -s email=my@example.com
source ~/.bashrc
```

> acme.sh 默认 CA 是 **ZeroSSL**（不是 Let's Encrypt）——两者都是免费、浏览器全信任、90 天有效期，续期命令通用，无需特意切换。

---

## 第 5 步：配置密钥（export 转眼失效的坑）

**踩坑记录：报错 `You don't specify aliyun api key and secret yet.`**

- **现象**：明明 export 过密钥，签发命令却说没找到。
- **原因**：`export` 只对当前终端会话有效——粘贴时断行、或中途重连 SSH，环境变量就没了。
- **解决**：把密钥**永久写入** `~/.bashrc`：

```bash
# 写入（把两个"你的xxx"替换成真实值）
echo 'export Ali_Key="你的AccessKeyId"' >> ~/.bashrc
echo 'export Ali_Secret="你的AccessKeySecret"' >> ~/.bashrc
# 生效
source ~/.bashrc
# 验证（能打印出密钥才算设置成功）
echo $Ali_Key
echo $Ali_Secret
```

![[Pasted image 20261105122638.png]]
> 图：签发报错——"You don't specify aliyun api key and secret yet."，密钥环境变量没生效

---

## 第 6 步：一条命令签发证书

```bash
acme.sh --issue --dns dns_ali -d renzhengdejuju.online -d www.renzhengdejuju.online
```

这条命令背后自动完成了 9 件事：

1. 选定 CA：ZeroSSL；
2. 识别双域名（主域名 + www，签成一张 SAN 证书）；
3. 调阿里云 DNS API **自动添加两条 TXT 验证记录**（`_acme-challenge.主` 和 `_acme-challenge.www`）——能加这条记录就证明了域名归你管；
4. 等待 DNS 全球同步（自动 sleep 20 秒）；
5. 向公共 DNS 查询验证（通过）；
6. **自动删除**临时 TXT 记录，解析恢复干净；
7. 向 CA 提交签名请求；
8. 签发成功，证书存入 `~/.acme.sh/renzhengdejuju.online_ecc/`（含 `.cer` 域名证书、`.key` 私钥、`ca.cer` 中间证书、`fullchain.cer` 完整链）；
9. 自动配置续期 cron，**下次续签时间 2027-01-20**（从 ARI 窗口选取）。

![[Pasted image 20261105123323.png]]
> 图：签发成功汇总——四个证书文件路径、覆盖域名、下次续签时间，以及 install-cert 部署命令建议

> 💡 **终端里的 curl "报错"不用管**：`error code 35`（SSL 握手中断）和 `error code 28`（请求超时）是访问阿里云 DNS API 时的网络波动，acme.sh 内置自动重试，重试后全部成功——这不是错误，无需修复。

---

## 第 7 步：部署到 Nginx

### 7.1 install-cert 把证书装进 Nginx 目录

```bash
acme.sh --install-cert -d renzhengdejuju.online \
  --key-file /etc/nginx/ssl/renzhengdejuju.online.key \
  --fullchain-file /etc/nginx/ssl/renzhengdejuju.online.fullchain.cer \
  --reloadcmd "systemctl reload nginx"
```

![[Pasted image 20261105123343.png]]
> 图：证书已自动部署—— Installing key/cert to /etc/nginx/ssl/ 目录

### 7.2 修改 Nginx 配置：用完整证书链 + 开 443

先定位配置：

```bash
grep -rn "renzhengdejuju.online" /etc/nginx/
```

![[Pasted image 20261105123352.png]]
> 图：grep 定位到 /etc/nginx/nginx.conf 第 50~51 行的 ssl_certificate 配置

编辑 `/etc/nginx/nginx.conf`，把证书路径从单公钥 `.pem` 换成**完整证书链** `.fullchain.cer`，并确保 `server` 块里有 `listen 443 ssl;`：

```nginx
ssl_certificate /etc/nginx/ssl/renzhengdejuju.online.fullchain.cer;
ssl_certificate_key /etc/nginx/ssl/renzhengdejuju.online.key;
```

> **为什么要换成 fullchain**：HTTPS 握手时浏览器不止要你的域名证书，还要**中间 CA 证书**拼出完整信任链。单公钥 `.pem` 缺中间证书，部分浏览器（尤其手机）会判为不安全；`fullchain.cer` = 你的证书 + 中间 CA 证书，全平台都认。

### 7.3 验证并重载

```bash
nginx -t
systemctl reload nginx
```

![[Pasted image 20261105123421.png]]
> 图：vim 保存退出后 nginx -t 语法检查通过，systemctl reload nginx 平滑重载

浏览器访问 `https://renzhengdejuju.online` —— 小锁出现，续签部署完成 ✅

---

## 第 8 步：确认自动续期（不要画蛇添足）

acme.sh 安装时**已自动写入 cron**（每天凌晨检查）：

```
0 0 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null
```

它每天自动检查证书有效期 → 快过期自动续签 → 按 `--install-cert` 记录的路径**自动覆盖** `/etc/nginx/ssl/` 下的文件 → 自动执行 `reloadcmd` 重载 Nginx。Nginx 配置里写的路径不变，所以**文件一换、重载一执行，新证书就生效**——这就是"自动续签"的全部原理。

**⚠️ 不要手动再加一条 `--renew --force` 的 cron**：

- 重复重载：acme.sh 自己的 cron 续完你手动再 reload 一次，Nginx 无意义重启两次；
- `--force` 风险：证书还剩 30 天以上也强制签，反复强签会触发 CA 限流封禁。

`crontab -l` 能看到 `acme.sh --cron` 那一行，就说明自动续期已在工作，**什么都不用再加**。

---

## 🔍 两种路线对比：本次实战（子账号 AK）vs 备选方案（ECS RAM 角色）

本实战用的就是《备选方案-阿里云RAM角色》附录 12.1 的"退而求其次"路线。两条路线完整对比如下：

**身份与凭证**
- 本实战：RAM 用户 `acme-dns` + **长期 AccessKey**，写入服务器 `~/.bashrc`，永不过期
- 角色方案：ECS 绑定 RAM 角色（临时工牌），每次续签从元数据服务刷 **STS 临时凭证**，几小时自动过期

**权限范围**
- 本实战：`AliyunDNSFullAccess`——整个云解析 DNS 的**全部**权限
- 角色方案：自建最小权限策略，只有查域名、查记录、加记录、删记录 **4 个动作**

**实现插件**
- 本实战：acme.sh **自带** `dns_ali` 插件，读环境变量 `Ali_Key` / `Ali_Secret`
- 角色方案：**自写** `dns_ali_role` 插件，桥接 aliyun CLI 刷工牌（必须放 `~/.acme.sh/dnsapi/`、实现 `_add`/`_rm` 两个函数）

**泄露后果**
- 本实战：AK 长期有效，一旦泄露，对方可持续控制你的 DNS 解析，直到手动换 Key
- 角色方案：泄露的只是几小时后自动作废的临时凭证，损失有限

**部署难度（实测）**
- 本实战：**低**——今天约 1 小时跑通全流程
- 角色方案：**中高**——建角色/建策略/绑 ECS/写插件，约 2~4 小时

**定位与建议**
- 本实战（AK）：**快速跑通、练手理解原理**的路线，今天已经验证可行
- 角色方案：**长期生产**路线——网站真正稳定服役后，值得按手册升级过去，把长期 AK 从服务器上消掉

> 💡 两条路线的**续签效果完全一样**：都是 acme.sh 自动加 TXT → 验证 → 签发 → 覆盖证书文件 → reload Nginx。区别只在"acme.sh 拿什么凭证去改 DNS"——一把长期钥匙，还是每次现刷的临时工牌。

---

## 原始对话存档

![[千问-阿里 AI 助手.html]]
