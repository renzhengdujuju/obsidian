# 阿里云 ECS RAM 角色 + acme.sh 自动续签方案

![[Pasted image 20261102003345.png]]

  ![[Pasted image 20261102003407.png]]

> 整理日期：2026-11-02

> 方案类型：自建 ACME 自动续签（Let's Encrypt）

> 增量费用：**0 元**

> 前提：域名 DNS 和 ECS 在同一个阿里云账号；RAM 有权限；服务器是 VPC 网络

> 说明：文中的 `ff.example.com` 是占位示例，实操时替换成真实完整域名（FQDN）。

  

---

  

## 0. 一页结论（给老板看）

  

| 项目 | 内容 |

|------|------|

| 方案 | ECS 绑定 RAM 角色 + STS 临时凭证 + acme.sh + Let's Encrypt |

| 证书费用 | **0 元** |

| 工具费用 | **0 元**（acme.sh、aliyun CLI、Let's Encrypt 都免费） |

| 是否用阿里云账号 | **是**。但续签全自动，不需要人工登录控制台 |

| 服务器是否保存长期 AccessKey | **否**。每次续签从 ECS 元数据服务获取 STS 临时凭证 |

| 自动续期 | 每天 cron 检查，到期前约 30 天自动续签，自动 reload Nginx |

| 主要前提 | 域名 DNS 与 ECS 同账号；ECS 是 VPC；RAM 有建角色/绑 ECS 权限；**删除现有 `_acme-challenge` CNAME** |

| 与乐此相比 | 免费、自主可控、无第三方；但部署更复杂，且仍依赖阿里云账号 |

  

**一句话**：这条方案不花一分钱，但要动阿里云账号的 RAM 角色和 DNS；配置一次之后，续签全自动。

  

---

  

## 1. 两个方案对比（老板选型用）

  

| 维度 | 乐此加密 | 阿里云 RAM 角色 + acme.sh |

|------|---------|--------------------------|

| 证书费用 | 免费 | **免费** |

| 自动续期费用 | **约 ¥30 一次性买断** | **0 元** |

| 续签时是否调用阿里云 | 不调用（只保留一条 CNAME） | **调用阿里云 DNS API** |

| 是否依赖第三方平台 | 是（letsssl.cn 及其客户端） | 否 |

| 私钥控制 | 需确认本地生成/复用还是平台下发 | **本地生成，自己保管** |

| 安全模型 | CNAME 验证权委托给第三方；客户端 root 运行 | RAM 角色只给 Alidns 权限；STS 临时凭证；无长期 AK |

| 部署难度 | 低（平台 + 客户端，约 30–60 分钟） | 中高（角色 + CLI + 插件，约 2–4 小时） |

| 日常维护 | 低 | 低（cron 自动） |

| 出问题排查 | 看平台面板和客户端日志 | 看 acme.sh 日志、CLI、RAM 权限 |

| 适合 | 想最快上线、续签完全不碰阿里云、接受 ¥30 和第三方 | 必须 0 成本、要自主可控、有阿里云账号权限 |

  

### 选型建议

  

- **选乐此**：老板愿意付约 ¥30，想当天搞定，且明确要求续签过程不碰阿里云。

- **选阿里云**：老板要求完全免费、不想依赖第三方，且能拿到阿里云 RAM 权限。

- **不能两个同时用**：乐此靠 `_acme-challenge` 的 CNAME，阿里云方案靠同名的 TXT，DNS 规定二者不能共存。详见第 2.3 节。

  

---

  

## 2. 原理

  

### 2.1 整体架构

  

```

ECS 实例

├── 绑定 RAM 角色 EcsAutoSslRenew（只允许 Alidns 读写）

├── aliyun CLI（EcsRamRole 模式）

│     └── 从 100.100.100.200 元数据服务自动获取 STS 临时凭证

├── acme.sh

│     └── 自定义插件 dns_ali_role.sh

│           └── 调 aliyun CLI 添加/删除 _acme-challenge TXT

└── Nginx

      └── 证书更新后 nginx -t && nginx -s reload

```

  

### 2.2 续签流程

  

1. acme.sh 每天由 cron 触发，检查证书剩余天数。

2. 剩余约 30 天时，向 Let's Encrypt 申请续签。

3. Let's Encrypt 要求做 DNS-01 验证，给一个随机值。

4. acme.sh 调用自定义插件，插件通过 aliyun CLI（RAM 角色 → STS）在阿里云 DNS 写 TXT。

5. Let's Encrypt 查询 TXT，验证通过，签发新证书。

6. 插件删除 TXT；acme.sh 把新证书写到 Nginx 目录。

7. 执行 `--reloadcmd`，Nginx 平滑加载新证书。

  

### 2.3 关键前提：必须删掉乐此的 CNAME

  

如果 `_acme-challenge.ff.example.com` 上还有 CNAME（指向 `cname.letsssl.cn`）：

  

- DNS 规定 CNAME 不能和 TXT 同名共存；

- 阿里云会拒绝添加 TXT，或者解析时 CNAME 优先，Let's Encrypt 看不到你写的 TXT；

- 结果是 acme.sh 验证必失败。

  

所以执行本方案前必须：

  

1. 确认乐此加密不再使用（或接受它以后不能自动续期）；

2. 删除 `_acme-challenge.ff` 的 CNAME；

3. 等 10–30 分钟（TTL 生效）；

4. `dig` 确认 CNAME 已消失；

5. 再执行 acme.sh 签发。

  

> 删 CNAME 不会让已签发的证书立刻失效，证书会用到到期日；只是乐此以后续不了。

  

---

  

## 3. 前置条件与确认清单

  

| # | 确认项 | 怎么查 | 不满足时 |

|---|--------|--------|---------|

| 1 | 域名 DNS zone 与 ECS 在同一个阿里云账号 | 阿里云 DNS 控制台看域名；ECS 控制台看实例 | 跨账号无法直接用 ECS 角色，见第 6 节 |

| 2 | ECS 是 VPC 网络 | ECS 实例详情 → 网络类型 | 经典网络不支持实例 RAM 角色 |

| 3 | RAM 子账号能创建角色、创建策略、绑定 ECS | RAM 控制台看授权，或直接试 | 让主账号操作，见第 6 节 |

| 4 | 服务器有 root/SSH 权限 | 直接登录 | 没有就无法安装 |

| 5 | 服务器能访问阿里云 API 和 Let's Encrypt | `curl -I https://acme-v02.api.letsencrypt.org` | 配代理/白名单 |

| 6 | 已安装或能安装 `jq` | `jq --version` | `yum install -y jq` / `apt install -y jq` |

| 7 | 阿里云 CLI 版本 ≥ 3.3.0 | `aliyun version` | 升级/重装 |

| 8 | Nginx 路径、证书路径、reload 命令 | `nginx -T \| grep ssl_certificate` | 记录后填入 |

| 9 | 证书部署到哪几台服务器 | 看资产表/DNS 记录 | 每台终止 TLS 的服务器都要处理 |

| 10 | 现有 `_acme-challenge` CNAME 是否还能删 | `dig +short CNAME _acme-challenge.ff.example.com` | 先和老板确认乐此是否停用 |

| 11 | 现有证书是否已备份 | 见 4.0 | 先备份再动手 |

| 12 | 目标 FQDN 和泛域名范围 | DNS 控制台 + 业务确认 | 泛域名不覆盖三级域名 |

  

---

  

## 4. 完整落地步骤

  

### 4.0 第 0 步：备份 + 处理 CNAME 冲突

  

```bash

# 备份证书和 Nginx 配置

mkdir -p /root/ssl-backup-$(date +%F)

cp -a /etc/nginx/ssl/. /root/ssl-backup-$(date +%F)/ 2>/dev/null || true

mkdir -p /root/nginx-conf-backup-$(date +%F)

cp -a /etc/nginx/conf.d/. /root/nginx-conf-backup-$(date +%F)/ 2>/dev/null || true

nginx -T > /root/nginx-full-config-$(date +%F).txt 2>/dev/null || true

  

# 记录当前证书信息

find /etc/nginx -type f \( -name '*.pem' -o -name '*.cer' -o -name '*.crt' \) 2>/dev/null | while read f; do

  echo "== $f"

  openssl x509 -in "$f" -noout -subject -issuer -dates 2>/dev/null

done

  

# 确认 CNAME 现状（有输出说明乐此委派还在）

dig +short CNAME _acme-challenge.ff.example.com

  

# 记录 CNAME 的值，万一要恢复乐此

# 然后在阿里云 DNS 控制台删除 _acme-challenge.ff 的 CNAME

# 等 10-30 分钟后再确认

dig +short CNAME _acme-challenge.ff.example.com

```

  

### 4.1 第 1 步：创建 RAM 角色

  

阿里云 RAM 控制台：

  

1. **身份管理 → 角色 → 创建角色**

2. 信任主体类型：**云服务**

3. 信任主体名称：**云服务器 ECS**

4. 角色名称：`EcsAutoSslRenew`

  

### 4.2 第 2 步：创建最小权限策略

  

**权限管理 → 权限策略 → 创建权限策略 → 脚本编辑**，粘贴：

  

```json

{

  "Version": "1",

  "Statement": [

    {

      "Effect": "Allow",

      "Action": [

        "alidns:DescribeDomains",

        "alidns:DescribeDomainRecords",

        "alidns:AddDomainRecord",

        "alidns:DeleteDomainRecord"

      ],

      "Resource": "*"

    }

  ]

}

```

  

策略名称例如 `EcsAutoSslRenewPolicy`，然后把它授予 `EcsAutoSslRenew` 角色。

  

> 为什么不用 `AliyunDNSFullAccess`：那个权限更大；这里只保留插件实际需要的 4 个动作。

> 注意：AliDNS 的 RAM 策略很难精确到单个域名，`Resource: "*"` 是常见做法；但动作已经限制在 DNS 读写。

  

### 4.3 第 3 步：把角色绑定到 ECS

  

ECS 控制台 → 目标实例 → **实例设置 → 授予/收回 RAM 角色**（控制台菜单名可能随版本变化）→ 选择 `EcsAutoSslRenew`。

  

服务器上验证：

  

```bash

# 应该返回角色名

curl -s http://100.100.100.200/latest/meta-data/ram/security-credentials/

  

# 应该返回 AccessKeyId / AccessKeySecret / SecurityToken / Expiration

curl -s http://100.100.100.200/latest/meta-data/ram/security-credentials/EcsAutoSslRenew

```

  

> 返回 404 或空：角色没绑上、实例不是 VPC、或元数据服务被禁用。

> 不要把返回内容贴到聊天或日志里。

  

### 4.4 第 4 步：安装并配置阿里云 CLI

  

```bash

# 确认是否已安装

aliyun version

```

  

如果没有，按官方文档安装（建议 v3.3.0+）：`https://help.aliyun.com/zh/cli/`

也可以从 GitHub Releases 下载 `aliyun-cli-linux-amd64.tgz`，解压后把 `aliyun` 放到 `/usr/local/bin`。

  

**用 root 配置**（acme.sh 的 cron 也是 root）：

  

```bash

aliyun configure --mode EcsRamRole --profile EcsAutoSslRenew

```

  

按提示填写：

  

| 提示 | 填写 |

|------|------|

| Ecs Ram Role | `EcsAutoSslRenew`（留空则自动发现实例角色） |

| Default Region Id | `cn-hangzhou` |

| Default Output Format | `json` |

  

验证：

  

```bash

aliyun sts GetCallerIdentity --profile EcsAutoSslRenew

  

aliyun alidns DescribeDomains \

  --profile EcsAutoSslRenew \

  --region cn-hangzhou \

  --output json \

  | jq -r '.Domains.Domain[].DomainName'

```

  

第二条能列出你的域名，说明“角色 → CLI → DNS”链路通了。

  

### 4.5 第 5 步：安装 acme.sh

  

```bash

# 国内建议 gitee 源

git clone https://gitee.com/neilpang/acme.sh.git

cd acme.sh

./acme.sh --install -m ops@example.com

source ~/.bashrc

  

# 默认 CA 改成 Let's Encrypt（acme.sh 3.x 默认是 ZeroSSL）

~/.acme.sh/acme.sh --set-default-ca --server letsencrypt

  

# 确认版本

~/.acme.sh/acme.sh --version

```

  

### 4.6 第 6 步：写 DNS 插件（完整代码）

  

创建 `~/.acme.sh/dnsapi/dns_ali_role.sh`，内容如下：

  

```sh

#!/usr/bin/env sh

# acme.sh DNS plugin: Aliyun DNS via ECS RAM role (STS, no long-term AK)

# File: ~/.acme.sh/dnsapi/dns_ali_role.sh

# Usage: acme.sh --issue --dns dns_ali_role -d example.com -d '*.example.com'

# Deps: aliyun CLI v3 (EcsRamRole profile), jq

  

dns_ali_role_info='Aliyun DNS via ECS RAM role (aliyun CLI + STS)

Options:

 ALI_ROLE_PROFILE  aliyun CLI profile name (default: EcsAutoSslRenew)

 ALI_ROLE_REGION   Alidns region (default: cn-hangzhou)

'

  

ALI_ROLE_PROFILE="${ALI_ROLE_PROFILE:-EcsAutoSslRenew}"

ALI_ROLE_REGION="${ALI_ROLE_REGION:-cn-hangzhou}"

ALIYUN_BIN="${ALIYUN_BIN:-$(command -v aliyun 2>/dev/null || echo /usr/local/bin/aliyun)}"

  

_ali_role_api() {

  _product="$1"

  _action="$2"

  shift 2

  "$ALIYUN_BIN" "$_product" "$_action" \

    --profile "$ALI_ROLE_PROFILE" \

    --region "$ALI_ROLE_REGION" \

    "$@"

}

  

_ali_role_zone() {

  _fqdn="$1"

  _zones=$("$ALIYUN_BIN" alidns DescribeDomains \

    --profile "$ALI_ROLE_PROFILE" \

    --region "$ALI_ROLE_REGION" \

    --PageSize 100 --output json 2>/dev/null \

    | jq -r '.Domains.Domain[]?.DomainName' 2>/dev/null)

  if [ -z "$_zones" ]; then

    echo "dns_ali_role: cannot list Aliyun DNS zones (role/profile/CLI?)" >&2

    return 1

  fi

  _best=""

  for _z in $_zones; do

    case "$_fqdn" in

      *".$_z")

        if [ "${#_z}" -gt "${#_best}" ]; then

          _best="$_z"

        fi

        ;;

    esac

  done

  if [ -z "$_best" ]; then

    echo "dns_ali_role: no Aliyun DNS zone matches $_fqdn" >&2

    return 1

  fi

  printf '%s\n' "$_best"

}

  

_ali_role_find() {

  _fqdn="$1"

  _txt="$2"

  _zone=$(_ali_role_zone "$_fqdn") || return 1

  _rr="${_fqdn%.$_zone}"

  [ -z "$_rr" ] && return 1

  "$ALIYUN_BIN" alidns DescribeDomainRecords \

    --profile "$ALI_ROLE_PROFILE" \

    --region "$ALI_ROLE_REGION" \

    --DomainName "$_zone" \

    --RRKeyWord "$_rr" \

    --TypeKeyWord TXT \

    --PageSize 100 --output json 2>/dev/null \

    | jq -r --arg rr "$_rr" --arg v "$_txt" \

        '.DomainRecords.Record[]? | select(.RR == $rr and .Value == $v) | .RecordId' 2>/dev/null \

    | head -n 1

}

  

dns_ali_role_add() {

  fulldomain="$1"

  txtvalue="$2"

  _info "dns_ali_role: adding TXT record for $fulldomain"

  _zone=$(_ali_role_zone "$fulldomain") || return 1

  _rr="${fulldomain%.$_zone}"

  if [ -z "$_rr" ] || [ "$_rr" = "$fulldomain" ]; then

    _err "dns_ali_role: cannot split $fulldomain using zone $_zone"

    return 1

  fi

  _existing=$(_ali_role_find "$fulldomain" "$txtvalue") || return 1

  if [ -n "$_existing" ]; then

    _info "dns_ali_role: record already exists, skip add"

    return 0

  fi

  _debug _zone "$_zone"

  _debug _rr "$_rr"

  _ali_role_api alidns AddDomainRecord \

    --DomainName "$_zone" \

    --RR "$_rr" \

    --Type TXT \

    --Value "$txtvalue" \

    --TTL 600

}

  

dns_ali_role_rm() {

  fulldomain="$1"

  txtvalue="$2"

  _info "dns_ali_role: removing TXT record for $fulldomain"

  _rid=$(_ali_role_find "$fulldomain" "$txtvalue") || return 1

  if [ -z "$_rid" ]; then

    _info "dns_ali_role: record not found, skip"

    return 0

  fi

  _ali_role_api alidns DeleteDomainRecord --RecordId "$_rid"

}

```

  

赋予权限：

  

```bash

chmod 700 ~/.acme.sh/dnsapi/dns_ali_role.sh

```

  

> 这个插件和网上很多“自定义脚本”的区别：

> 1. 它必须放在 `~/.acme.sh/dnsapi/`，名字必须是 `dns_ali_role.sh`；

> 2. 它实现的是 `dns_ali_role_add` / `dns_ali_role_rm` 两个函数，不是 `add/clean` 子命令；

> 3. 删除时按 TXT 的**值**精确匹配，能正确处理泛域名 + 主域名两条记录。

  

### 4.7 第 7 步：先用 staging 测试

  

```bash

~/.acme.sh/acme.sh --issue --dns dns_ali_role --dnssleep 180 \

  -d "ff.example.com" -d "*.ff.example.com" --staging

```

  

这一步会：

  

1. 用 Let's Encrypt 测试环境申请；

2. 通过插件在阿里云 DNS 写 TXT；

3. 等 180 秒；

4. 验证；

5. 删除 TXT；

6. 签发一张**不被浏览器信任**的测试证书（不要安装到生产路径）。

  

失败时看日志：

  

```bash

tail -n 100 ~/.acme.sh/acme.sh.log

```

  

### 4.8 第 8 步：正式签发

  

```bash

~/.acme.sh/acme.sh --issue --dns dns_ali_role --dnssleep 180 \

  -d "ff.example.com" -d "*.ff.example.com"

```

  

> 泛域名一定要加引号：`-d "*.ff.example.com"`。

  

### 4.9 第 9 步：安装证书并 reload

  

```bash

mkdir -p /etc/nginx/ssl

  

~/.acme.sh/acme.sh --install-cert -d "ff.example.com" \

  --key-file       /etc/nginx/ssl/ff.example.com.key \

  --fullchain-file /etc/nginx/ssl/ff.example.com.cer \

  --reloadcmd      "nginx -t && nginx -s reload"

```

  

如果 Nginx 在 Docker 里：

  

```bash

--reloadcmd "docker exec <容器名> nginx -t && docker exec <容器名> nginx -s reload"

```

  

> 证书文件必须写到宿主机上**挂载进容器**的目录，否则容器里看不到。

> 如果签发时用了 ECC（例如 `--keylength ec-256`），`--install-cert` 要加 `--ecc`；本文默认 RSA，不需要。

  

### 4.10 第 10 步：配置 cron 和 PATH

  

```bash

# 看 acme.sh 是否自动加了任务

crontab -l | grep acme

  

# 如果没有，手动加

(crontab -l 2>/dev/null; echo '17 3 * * * /root/.acme.sh/acme.sh --cron --home /root/.acme.sh >> /var/log/acme.log 2>&1') | crontab -

```

  

**必须处理 PATH**，否则 cron 里找不到 `aliyun`：

  

```bash

crontab -e

```

  

在文件顶部加：

  

```

PATH=/usr/local/bin:/usr/bin:/bin

```

  

或者在插件里固定：

  

```sh

export ALIYUN_BIN=/usr/local/bin/aliyun

```

  

> “手动执行成功、定时任务失败”最常见的原因就是 PATH。

  

### 4.11 第 11 步：验证

  

```bash

# 证书列表和下次续期时间

~/.acme.sh/acme.sh --list

  

# 证书文件内容

openssl x509 -in /etc/nginx/ssl/ff.example.com.cer -noout -subject -issuer -dates

  

# 线上实际返回的证书

echo | openssl s_client -connect ff.example.com:443 -servername ff.example.com 2>/dev/null \

  | openssl x509 -noout -subject -issuer -dates

  

# HTTPS 是否正常

curl -I https://ff.example.com

  

# 手动跑一次 cron（未到期时不会续，但能验证脚本能跑起来）

/root/.acme.sh/acme.sh --cron --home /root/.acme.sh

tail -n 50 /var/log/acme.log

```

  

**真正的端到端验证**：用测试子域（例如 `test-renew.example.com`）走一遍 staging，确认“写 TXT → 验证 → 删 TXT → 签发”全链路正常。不要用生产域名反复 `--force`，会触发 Let's Encrypt 频率限制。

  

### 4.12 第 12 步：回滚

  

```bash

# 恢复旧证书

cp -a /root/ssl-backup-<日期>/. /etc/nginx/ssl/

nginx -t && nginx -s reload

  

# 如果决定不再用 acme.sh，再删 cron；否则保留

# crontab -l | grep -v acme.sh | crontab -

```

  

如果要从本方案切回乐此加密：

  

1. 停掉 acme.sh 的 cron；

2. 删除 acme.sh 写在 `_acme-challenge.ff` 的 TXT；

3. 重新添加乐此给的 CNAME；

4. 在乐此平台重新验证/续期。

  

---

  

## 5. 多服务器、Docker、CDN 场景

  

### 5.1 多台 ECS

  

- 每台终止 TLS 的 ECS 都可以绑定同一个 RAM 角色（一个角色可以授予多个实例）。

- 每台都装 acme.sh 和插件，各自续签；或者只在一台签发，再用 scp/配置管理同步证书并 reload。后者要注意私钥传输安全。

- 如果多台服务器共享同一张泛域名证书，建议每台都自动续，避免单点故障。

  

### 5.2 Docker 里的 Nginx

  

- 证书写到宿主机挂载目录；

- reload 用 `docker exec`；

- 先确认容器内路径和宿主机路径的映射：`docker inspect <容器名> | grep -A5 Mounts`；

- `docker exec <容器名> nginx -t` 必须成功。

  

### 5.3 CDN / SLB / WAF

  

acme.sh 默认只把证书写到本机文件。如果还要上传到阿里云 CDN/SLB，需要额外的 API 脚本或部署钩子。本文不展开；如果你们有 CDN，建议单独评估 AllinSSL/Certd 这类支持多目标部署的工具。

  

---

  

## 6. 权限不够时的替代方案

  

### 6.1 让主账号协助（推荐）

  

如果 RAM 子账号不能创建角色/绑定 ECS，让主账号或管理员做第 4.1–4.3 步；之后的 acme.sh 配置可以由你在服务器上完成。

  

### 6.2 退而求其次：只用 DNS 权限的 RAM 用户 AK

  

如果实在拿不到“绑定 ECS 角色”的权限，可以建一个**只授予 Alidns 权限的 RAM 用户**，用它的 AccessKey 配合 acme.sh 自带的 `dns_ali` 插件：

  

```bash

export Ali_Key="RAM用户的AccessKeyId"

export Ali_Secret="RAM用户的AccessKeySecret"

  

~/.acme.sh/acme.sh --issue --dns dns_ali --dnssleep 180 \

  -d "ff.example.com" -d "*.ff.example.com"

```

  

缺点：这是**长期 AK**，会保存在 `~/.acme.sh/account.conf`，需要 `chmod 600` 并定期轮换；安全性不如 RAM 角色 + STS。仅作为权限受限时的过渡方案。

  

### 6.3 如果域名和 ECS 不在同一个账号

  

ECS 实例角色只能管理**本账号**的 DNS。跨账号有两种做法：

  

1. 在 DNS 所属账号建一个只读 Alidns 权限的 RAM 用户 AK，用 `dns_ali`（长期 AK，见 6.2）；

2. 使用 RAM 角色的跨账号信任（`RamRoleArn` 模式），配置更复杂，一般不建议为这个场景上。

  

### 6.4 如果连 RAM 权限都没有

  

那就只能走乐此加密（需要 ¥30 自动续期），或者把 DNS 迁到 Cloudflare/DNSPod 用它们的 API。

  

---

  

## 7. 安全说明

  

| 项目 | 说明 |

|------|------|

| 无长期 AK | 服务器只通过实例元数据获取 STS 临时凭证，凭证自动过期 |

| 最小权限 | 角色只授予 `DescribeDomains`、`DescribeDomainRecords`、`AddDomainRecord`、`DeleteDomainRecord` |

| 权限范围 | Alidns 的 RAM 策略难以精确到单域名，角色理论上能改本账号所有域名的解析；这是本方案的主要残余风险 |

| 元数据服务 | ECS 上任何进程都能访问 100.100.100.200；如果服务器被入侵，攻击者也能拿到临时凭证。因此要同时做好主机安全 |

| 凭证文件 | `~/.acme.sh/account.conf` 可能含账号信息，`chmod 600` |

| 备份 | 备份 `~/.acme.sh`（含 ACME 账户密钥和证书） |

| 日志 | 不要把 STS 凭证、私钥、TXT 值贴到聊天/工单 |

| 测试 | 一律先用 `--staging`，避免触发 Let's Encrypt 频率限制 |

  

---

  

## 8. 排错表

  

| 现象 | 可能原因 | 处理 |

|------|---------|------|

| `--dns dns_ali_role` 报插件不存在 | 文件不在 `~/.acme.sh/dnsapi/`，或名字/函数名不对 | 确认 `dns_ali_role.sh` 和 `dns_ali_role_add/rm` |

| `cannot list Aliyun DNS zones` | 角色没绑、CLI profile 不对、cron PATH 问题 | `aliyun sts GetCallerIdentity` 验证；检查 `/root/.aliyun/config.json`；加 PATH |

| `Forbidden.RAM` / 权限不足 | 角色策略缺 action | 补 4 个 Alidns action；确认策略已授予角色 |

| 添加 TXT 报冲突 | `_acme-challenge` 上还有 CNAME | 删除 CNAME，等 TTL，再重试 |

| 验证一直超时 | TXT 没传播、CNAME 还在、FQDN 写错 | `dig TXT` 查；确认 `--dnssleep`；核对域名 |

| 证书更新但网站还是旧的 | `--reloadcmd` 没执行/失败 | 手动 `nginx -t && nginx -s reload`；检查路径和 Docker 挂载 |

| cron 不执行 | PATH、crond 没起、脚本权限 | `crontab -l`；加 PATH；`systemctl status crond`；看 `/var/log/acme.log` |

| 触发 Let's Encrypt 限流 | 反复 `--force` 生产签发 | 停 7 天；以后一律 staging 测试 |

| 元数据 404 | 角色没绑 / 不是 VPC / IMDS 被禁用 | ECS 控制台重新授予角色；确认网络类型 |

| `jq: command not found` | 没装 jq | `yum install -y jq` / `apt install -y jq` |

| `no Aliyun DNS zone matches` | 域名不在该账号，或 zone 拼写错误 | 核对域名所属账号和 zone |

  

---

  

## 9. 明天执行清单（打勾）

  

- [ ] 确认域名 DNS 和 ECS 同账号

- [ ] 确认 ECS 是 VPC

- [ ] 确认 RAM 子账号权限，或找主账号协助

- [ ] 记录真实 FQDN 和泛域名范围

- [ ] 备份现有证书、私钥、Nginx 配置

- [ ] 和老板确认乐此停用，删除 `_acme-challenge` CNAME

- [ ] 等 TTL，`dig` 确认 CNAME 已消失

- [ ] 创建 RAM 角色 `EcsAutoSslRenew`

- [ ] 创建并授予最小权限策略

- [ ] 把角色绑定到 ECS

- [ ] 验证元数据能返回角色凭证

- [ ] 安装/配置 aliyun CLI（EcsRamRole）

- [ ] `aliyun sts GetCallerIdentity` 验证

- [ ] `DescribeDomains` 能列出域名

- [ ] 安装 acme.sh，默认 CA 设为 Let's Encrypt

- [ ] 写 `dns_ali_role.sh` 插件

- [ ] staging 签发成功

- [ ] 正式签发成功

- [ ] `--install-cert` 写入 Nginx 目录并 reload

- [ ] 线上证书检查通过

- [ ] cron 和 PATH 配置完成

- [ ] 用测试子域做一次端到端演练

- [ ] 配置到期监控/告警

- [ ] 记录回滚步骤和备份路径

- [ ] 给老板交付说明

  

---

  

## 10. 需要确认的问题

  

1. 域名 DNS 和 ECS 是否同一个阿里云账号？主账号是谁？

2. 当前 RAM 子账号能否创建角色、创建策略、给 ECS 授予角色？不能的话谁能操作？

3. ECS 是不是 VPC 网络？

4. 证书要覆盖哪些完整域名？是否需要 `*.ff.example.com`？有没有三级域名？

5. 证书部署到几台服务器？`*.ff` 的 180.175.161.67 和 `website` 的 47.102.116.218 分别是什么角色？

6. Nginx 是宿主机还是 Docker？证书路径和 reload 命令是什么？

7. 乐此加密是否可以停用？删掉 CNAME 会不会影响其他业务？

8. 变更窗口是什么时候？reload 虽然平滑，但配置错误会影响业务。

9. 是否有 CDN/SLB/WAF 需要同步证书？

10. 是否有监控平台可以接证书到期告警？

  

---

  

## 11. 参考链接

  

- [acme.sh 官方仓库](https://github.com/acmesh-official/acme.sh)

- [acme.sh DNS API 开发指南](https://github.com/acmesh-official/acme.sh/wiki/DNS-API-Dev-Guide)

- [acme.sh dns_ali 不支持 STS 的 Issue #6118](https://github.com/acmesh-official/acme.sh/issues/6118)

- [阿里云 CLI 凭证模式与环境变量](https://help.aliyun.com/zh/cli/environment-variables)

- [阿里云 RAM 角色文档](https://help.aliyun.com/zh/ram/user-guide/ram-roles)

- [阿里云 Alidns API 文档](https://help.aliyun.com/zh/dns/api-alidns-2015-01-09/)

- [Let's Encrypt 频率限制](https://letsencrypt.org/docs/rate-limits/)

- [APNIC：CNAME 委派验证原理](https://blog.apnic.net/2023/07/03/delegated-domain-verification/)