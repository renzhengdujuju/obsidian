# 阿里云 RAM 角色 + acme.sh 自动续签

![[Pasted image 20261102003345.png]]

![[Pasted image 20261102003407.png]]

> 整理日期：2026-11-02
> 方案类型：自建 ACME 自动续签（Let's Encrypt）
> 增量费用：**0 元**
> 说明：文中 `ff.example.com` 是占位示例，实操时全部替换成你们的真实域名。

---

## 一、这个方案解决什么问题

**现状**：SSL 证书 90 天到期，每到期就要手动去申请、下载、替换、重启 Nginx，忘了就出事故。

**目标**：装好这套东西之后，服务器自己续证书，你不用再管。

**做法一句话**：给服务器发一张阿里云"临时工牌"（RAM 角色），装一个续证书的小工具（acme.sh），工具续证书时刷工牌自动改 DNS 验证记录，全程不需要任何长期密码。

---

## 二、开工前必须懂的 5 个名词

不懂这些名词，后面每一步都会懵。先花 2 分钟读完：

| 名词            | 大白话解释                                                                      |
| ------------- | -------------------------------------------------------------------------- |
| **RAM 角色**    | 给服务器发的"临时工牌"。阿里云不把万能钥匙（长期密码 AccessKey）发给服务器，而是发一张限定范围的工牌——只能管 DNS 记录，干不了别的 |
| **STS 临时凭证**  | 刷工牌换来的临时通行证，**几小时后自动过期**，过期了再刷一张。就算泄露，损失也有限                                |
| **acme.sh**   | 装在服务器上的一个开源小工具，专门干"向 Let's Encrypt 申请/续期 SSL 证书"这件事                        |
| **DNS-01 验证** | Let's Encrypt（CA) 发证书前要确认"域名真的是你的"。验证方式：往 DNS 里写一条 TXT 记录，它来查。查到了就发证书      |
| **TXT 记录**    | DNS 电话簿里的"备注栏"，贴一段文字，谁查都能读到。原理详见 [[从域名验证到证书认证-完整闭环梳理\|从域名验证到证书认证]]         |

---

## 三、全流程地图（先看清全貌再动手）

总共 **4 个阶段 12 步**，操作地点只有两个：**阿里云网页控制台**和**服务器 SSH 终端**。

| 阶段    | 步骤     | 在哪操作    | 做什么                            | 大约耗时  |
| ----- | ------ | ------- | ------------------------------ | ----- |
| 准备    | 第 0 步  | 服务器     | 检查前提、备份现有证书                    | 5 分钟  |
| 一：控制台 | 第 1 步  | 🖥️ 浏览器 | 创建 RAM 角色（发工牌）                 | 3 分钟  |
| 一：控制台 | 第 2 步  | 🖥️ 浏览器 | 创建最小权限策略（规定工牌能干什么）             | 3 分钟  |
| 一：控制台 | 第 3 步  | 🖥️ 浏览器 | 把角色绑定到 ECS（把工牌挂到服务器上）          | 2 分钟  |
| 二：装工具 | 第 4 步  | 💻 SSH  | 安装并配置阿里云 CLI                   | 5 分钟  |
| 二：装工具 | 第 5 步  | 💻 SSH  | 安装 acme.sh                     | 2 分钟  |
| 三：签发  | 第 6 步  | 💻 SSH  | 写 DNS 插件（教 acme.sh 怎么改阿里云 DNS） | 10 分钟 |
| 三：签发  | 第 7 步  | 💻 SSH  | 测试签发（用测试环境，不影响正式证书）            | 10 分钟 |
| 三：签发  | 第 8 步  | 💻 SSH  | 正式签发                           | 5 分钟  |
| 四：自动化 | 第 9 步  | 💻 SSH  | 把证书装进 Nginx 并重载                | 2 分钟  |
| 四：自动化 | 第 10 步 | 💻 SSH  | 配置定时任务，以后全自动                   | 3 分钟  |
| 四：自动化 | 第 11 步 | 💻 SSH  | 最终验证                           | 5 分钟  |

⚠️ **最重要的前提**：如果之前配过乐此加密（DNS 里有一条 `_acme-challenge` 的 CNAME 记录），**必须先删掉它**，否则本方案必失败。原因见第 0 步。

---

## 四、第 0 步：开工前的检查与备份

### 0.1 逐项检查前提（缺一项都做不下去）

| #   | 检查项                             | 怎么查                                            | 不满足怎么办              |
| --- | ------------------------------- | ---------------------------------------------- | ------------------- |
| 1   | 域名 DNS 和 ECS 在**同一个阿里云账号**      | 两边控制台都能看到同一批域名/实例                              | 跨账号做不了，见附录 12.1     |
| 2   | ECS 是 VPC 网络                    | ECS 控制台 → 实例详情 → 网络类型                          | 经典网络不支持 RAM 角色      |
| 3   | 你的 RAM 子账号**能创建角色、创建策略、绑定 ECS** | 直接试第 1 步，报权限错误就说明不行                            | 找有主账号权限的同事操作第 1~3 步 |
| 4   | 服务器有 root/SSH 权限                | 直接登录                                           | 没有就没法装任何东西          |
| 5   | 服务器能访问外网                        | `curl -I https://acme-v02.api.letsencrypt.org` | 配代理或白名单             |

### 0.2 备份现有证书和 Nginx 配置（必做，1 分钟的事，出事能救命）

**💻 在服务器上执行：**

```bash
# 创建备份目录，$(date +%F) 会自动替换成今天的日期，如 2026-11-02
mkdir -p /root/ssl-backup-$(date +%F)
# 把 Nginx 证书目录整体拷贝进去（cp -a = 连权限一起原样复制）
cp -a /etc/nginx/ssl/. /root/ssl-backup-$(date +%F)/ 2>/dev/null || true

# 备份 Nginx 站点配置
mkdir -p /root/nginx-conf-backup-$(date +%F)
cp -a /etc/nginx/conf.d/. /root/nginx-conf-backup-$(date +%F)/ 2>/dev/null || true

# 导出 Nginx 完整生效配置，留档
nginx -T > /root/nginx-full-config-$(date +%F).txt 2>/dev/null || true
```

> `2>/dev/null || true` 的意思：即使某个目录不存在报错，也不要停下来，继续往下执行。

**再记录一下当前证书的信息（到期日等），方便之后对比：**

```bash
# find 找出 /etc/nginx 下所有证书文件，逐个读取有效期
find /etc/nginx -type f \( -name '*.pem' -o -name '*.cer' -o -name '*.crt' \) 2>/dev/null | while read f; do
  echo "== $f"
  openssl x509 -in "$f" -noout -subject -issuer -dates 2>/dev/null
done
```

> - `find ... \( -name A -o -name B \)` = 找文件名是 .pem 或 .cer 或 .crt 的文件
> - `openssl x509 -in 文件 -noout -dates` = 只打印这张证书的生效/到期时间

### 0.3 处理乐此加密的 CNAME 冲突（关键！）

**为什么必须删**：DNS 规定同一个名字下 CNAME 和 TXT 不能共存（原理见 [[从域名验证到证书认证-完整闭环梳理|从域名验证到证书认证]]）。乐此加密的 CNAME 占着 `_acme-challenge.ff` 这个位置，acme.sh 就写不进 TXT，验证必失败。

**💻 在服务器上先查现状：**

```bash
dig +short CNAME _acme-challenge.ff.example.com
```

> `dig` = 查询 DNS 记录的命令；`+short` = 只显示结果不显示细节

- **有输出**（指向 `xxx.cname.letsssl.cn`）→ 乐此委派还在，走本方案前需要：① 和老板确认乐此停用 → ② 在阿里云 DNS 控制台删除这条 CNAME → ③ 等 10~30 分钟（TTL 生效）→ ④ 再用上面的命令确认没有输出了
- **没有输出** → 干净的，直接开工
- 删 CNAME **不会**让已签发的证书立刻失效，证书会用到到期日，不用担心断网

---

## 五、阶段一：在阿里云控制台操作（🖥️ 浏览器）

### 第 1 步：创建 RAM 角色

**在哪里**：浏览器打开阿里云控制台 → 搜索"RAM 访问控制" → 左侧菜单 **身份管理 → 角色 → 创建角色**

按页面依次填：

1. **信任主体类型**：选 **云服务**
2. **信任主体名称**：选 **云服务器 ECS**
3. **角色名称**：填 `EcsAutoSslRenew`（后面所有步骤都用这个名字，抄下来别写错）
4. 点创建

> **这一步在干什么**：相当于在阿里云前台给服务器办了一张空白工牌。第 2 步规定工牌权限，第 3 步把工牌挂到服务器上。

### 第 2 步：创建最小权限策略

**在哪里**：RAM 控制台 → **权限管理 → 权限策略 → 创建权限策略 → 脚本编辑**

名称填 `EcsAutoSslRenewPolicy`，脚本内容粘贴：

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

**这个 JSON 每一行是什么意思**：

| 内容 | 含义 |
|------|------|
| `"Effect": "Allow"` | 允许（而不是拒绝） |
| `alidns:DescribeDomains` | 查看**有哪些域名**（插件要靠它找到域名属于哪个 DNS zone） |
| `alidns:DescribeDomainRecords` | 查看**某域名下现有的解析记录**（删 TXT 前要先找到它） |
| `alidns:AddDomainRecord` | **添加**解析记录（写 TXT 验证值用，核心动作） |
| `alidns:DeleteDomainRecord` | **删除**解析记录（验证完清理用） |
| `"Resource": "*"` | 对账号下所有域名生效（阿里云 DNS 的权限没法精确到单个域名，只能放宽到这里；但动作已被上面 4 条限死，只能动解析记录） |

> 为什么不用现成的 `AliyunDNSFullAccess`？那个权限太大（连域名增删都给）。这里只给插件实际需要的 4 个动作，最小权限原则。

创建完策略后，**回到第 1 步创建的角色 `EcsAutoSslRenew` → 授权 → 把这个策略授给它**。

### 第 3 步：把角色绑定到 ECS

**在哪里**：ECS 控制台 → 找到目标实例 → **实例设置 → 授予/收回 RAM 角色** → 选 `EcsAutoSslRenew`

**💻 绑定完回到服务器上验证工牌挂上了没有：**

```bash
curl -s http://100.100.100.200/latest/meta-data/ram/security-credentials/
```

> - `curl -s` = 命令行里发 HTTP 请求，`-s` 静默模式
> - `100.100.100.200` = 阿里云每台 ECS **内置**的"元数据服务"地址，只能本机访问，相当于服务器的"个人档案柜"
> - `.../ram/security-credentials/` = 问档案柜："我身上挂了哪些工牌？"

**✅ 成功标志**：输出一行 `EcsAutoSslRenew`。

**再看一眼临时凭证能不能换到：**

```bash
curl -s http://100.100.100.200/latest/meta-data/ram/security-credentials/EcsAutoSslRenew
```

**✅ 成功标志**：输出一段 JSON，里面有 `AccessKeyId`、`SecurityToken`、`Expiration`（过期时间）。注意看：**这些凭证是临时的**，到 Expiration 时间就作废——这就是"工牌"机制的核心。

> ⚠️ 如果返回 404 或空：角色没绑上 / 实例不是 VPC / 元数据服务被禁用。另外**不要把这段输出贴到聊天或日志里**，它是凭证。

---

## 六、阶段二：在服务器上装工具（💻 SSH）

### 第 4 步：安装并配置阿里云 CLI

```bash
# 先看装没装过
aliyun version
```

没装的话，按官方文档装（建议 3.3.0 以上）：https://help.aliyun.com/zh/cli/ ，装完放到 `/usr/local/bin/aliyun`。

**配置 CLI 使用工牌（工牌刷卡模式）：**

```bash
aliyun configure --mode EcsRamRole --profile EcsAutoSslRenew
```

> - `--mode EcsRamRole` = 认证方式选"刷服务器上的工牌"，而不是输入账号密码
> - `--profile EcsAutoSslRenew` = 给这套配置起个名字（profile 就像浏览器里的"用户配置文件"），以后所有命令都用 `--profile` 指定它

按提示填写：

| 提示 | 填什么 |
|------|--------|
| Ecs Ram Role | `EcsAutoSslRenew`（直接回车留空也行，会自动发现） |
| Default Region Id | `cn-hangzhou`（按你们实际区域填） |
| Default Output Format | `json` |

**验证链路通不通：**

```bash
# 验证 1：刷工牌能不能换到身份（返回 ARN 说明工牌有效）
aliyun sts GetCallerIdentity --profile EcsAutoSslRenew

# 验证 2：用工牌列出你账号下所有域名
aliyun alidns DescribeDomains \
  --profile EcsAutoSslRenew \
  --region cn-hangzhou \
  --output json \
  | jq -r '.Domains.Domain[].DomainName'
```

> - `sts GetCallerIdentity` = "我是谁？"，能答上来说明认证通了
> - `alidns DescribeDomains` = 列出 DNS 域名
> - 管道符 `|` 把上一条命令的输出喂给下一条；`jq` 是 JSON 解析工具，`-r '.Domains.Domain[].DomainName'` = 从 JSON 里只抽出域名那一列

**✅ 成功标志**：第二条命令列出了你们的域名。**"工牌 → CLI → DNS"这条链路就通了**——这是整个方案的心脏，通了后面都是顺水推舟。

### 第 5 步：安装 acme.sh

```bash
# 从 gitee 克隆（国内访问 github 不稳）
git clone https://gitee.com/neilpang/acme.sh.git
cd acme.sh
# 安装，-m 后面填你的邮箱（Let's Encrypt 用来发到期提醒）
./acme.sh --install -m ops@example.com
# 让环境变量生效
source ~/.bashrc

# 把默认 CA 改成 Let's Encrypt（acme.sh 3.x 默认是 ZeroSSL）
~/.acme.sh/acme.sh --set-default-ca --server letsencrypt

# 确认版本
~/.acme.sh/acme.sh --version
```

> acme.sh 装在 `~/.acme.sh/` 目录（root 用户就是 `/root/.acme.sh/`），以后所有命令都用全路径调用它。

---

## 七、阶段三：写 DNS 插件 + 签发证书（💻 SSH）

### 第 6 步：写 DNS 插件（本方案唯一的"代码"，10 分钟）

**为什么要写插件**：acme.sh 自带的阿里云插件只认长期密码（Ali_Key/Ali_Secret），不认我们的工牌模式。所以要写一个小插件教它：改 DNS 的时候，去调 CLI，CLI 会自动刷工牌。

**先搞懂 acme.sh 插件的 3 条规矩**（网上很多教程栽在这里）：

1. 插件文件**必须**放在 `~/.acme.sh/dnsapi/` 目录下
2. 文件名**必须**是 `dns_开头.sh`（我们用 `dns_ali_role.sh`，那插件名就是 `dns_ali_role`）
3. 文件里**必须**实现两个函数：`dns_ali_role_add`（加 TXT）和 `dns_ali_role_rm`（删 TXT）——acme.sh 验证前调用 add，验证后调用 rm，**不是**命令行子命令

**创建文件：**

```bash
mkdir -p ~/.acme.sh/dnsapi
vim ~/.acme.sh/dnsapi/dns_ali_role.sh
```

**粘贴以下完整代码**（先贴，逐段讲解在后面）：

```sh
#!/usr/bin/env sh
# acme.sh DNS plugin: Aliyun DNS via ECS RAM role (STS, no long-term AK)
# Usage: acme.sh --issue --dns dns_ali_role -d example.com -d '*.example.com'
# Deps: aliyun CLI v3 (EcsRamRole profile), jq

# 插件说明信息，acme.sh --help 时会显示
dns_ali_role_info='Aliyun DNS via ECS RAM role (aliyun CLI + STS)
Options:
 ALI_ROLE_PROFILE  aliyun CLI profile name (default: EcsAutoSslRenew)
 ALI_ROLE_REGION   Alidns region (default: cn-hangzhou)
'

# 三个可配置变量：没设就用默认值（${VAR:-默认值} 语法）
ALI_ROLE_PROFILE="${ALI_ROLE_PROFILE:-EcsAutoSslRenew}"
ALI_ROLE_REGION="${ALI_ROLE_REGION:-cn-hangzhou}"
ALIYUN_BIN="${ALIYUN_BIN:-$(command -v aliyun 2>/dev/null || echo /usr/local/bin/aliyun)}"

# ---------- 工具函数 1：统一封装"刷工牌调 API" ----------
_ali_role_api() {
  _product="$1"     # 产品名，如 alidns
  _action="$2"      # 动作名，如 AddDomainRecord
  shift 2           # 丢掉前两个参数，剩下的是 API 的键值参数
  "$ALIYUN_BIN" "$_product" "$_action" \
    --profile "$ALI_ROLE_PROFILE" \
    --region "$ALI_ROLE_REGION" \
    "$@"
}

# ---------- 工具函数 2：从完整域名反查它属于哪个 zone ----------
# 例：传入 _acme-challenge.ff.example.com，返回 example.com
_ali_role_zone() {
  _fqdn="$1"
  # 列出账号下所有 DNS zone
  _zones=$("$ALIYUN_BIN" alidns DescribeDomains \
    --profile "$ALI_ROLE_PROFILE" \
    --region "$ALI_ROLE_REGION" \
    --PageSize 100 --output json 2>/dev/null \
    | jq -r '.Domains.Domain[]?.DomainName' 2>/dev/null)
  if [ -z "$_zones" ]; then
    echo "dns_ali_role: cannot list Aliyun DNS zones (role/profile/CLI?)" >&2
    return 1
  fi
  # 在所有 zone 里找"是 _fqdn 后缀的最长 zone"——防止 ff.example.com 被误配到 example.net 之类
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

# ---------- 工具函数 3：按"名字+值"精确查找 TXT 记录，返回记录 ID ----------
_ali_role_find() {
  _fqdn="$1"
  _txt="$2"
  _zone=$(_ali_role_zone "$_fqdn") || return 1
  _rr="${_fqdn%.$_zone}"    # 去掉 zone 后缀，得到主机记录名，如 _acme-challenge.ff
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

# ---------- acme.sh 会调用的函数 1：添加 TXT ----------
dns_ali_role_add() {
  fulldomain="$1"    # acme.sh 传入完整域名，如 _acme-challenge.ff.example.com
  txtvalue="$2"      # acme.sh 传入要写的验证值
  _info "dns_ali_role: adding TXT record for $fulldomain"
  _zone=$(_ali_role_zone "$fulldomain") || return 1
  _rr="${fulldomain%.$_zone}"
  if [ -z "$_rr" ] || [ "$_rr" = "$fulldomain" ]; then
    _err "dns_ali_role: cannot split $fulldomain using zone $_zone"
    return 1
  fi
  # 防重复：已存在同名同值记录就不重复加
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

# ---------- acme.sh 会调用的函数 2：删除 TXT ----------
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

**这个插件是怎么工作的（跟着数据走一遍）**：

1. 你执行 `acme.sh --issue --dns dns_ali_role -d ff.example.com -d '*.ff.example.com'`
2. acme.sh 找到 `~/.acme.sh/dnsapi/dns_ali_role.sh`，调用里面的 `dns_ali_role_add "_acme-challenge.ff.example.com" "随机验证值"`
3. `add` 函数：先问"这域名属于哪个 zone"（函数 2）→ 算出主机记录 `_acme-challenge.ff` → 查一下有没有重复（函数 3）→ 调阿里云 API 添加 TXT（函数 1，内部刷工牌拿临时凭证）
4. Let's Encrypt 来查 TXT → 读到验证值 → 发证书
5. acme.sh 调用 `dns_ali_role_rm`，函数 3 按"名字+值"精确找到那条记录的 ID，删掉

**为什么删除时要按"值"精确匹配**：泛域名 + 主域名会在同一个 `_acme-challenge.ff` 下写**两条不同值**的 TXT。如果像很多老教程那样"找到第一条就删"，删错一条另一条就验证失败。`select(.RR == $rr and .Value == $v)` 保证删的就是刚才加的那条。

**保存并赋权：**

```bash
# chmod 700 = 只有 root 能读写执行（这文件涉及凭证，别放开权限）
chmod 700 ~/.acme.sh/dnsapi/dns_ali_role.sh
```

### 第 7 步：测试签发（staging，不会产生正式证书）

```bash
~/.acme.sh/acme.sh --issue --dns dns_ali_role --dnssleep 180 \
  -d "ff.example.com" -d "*.ff.example.com" --staging
```

> - `--dns dns_ali_role` = 用我们刚写的插件（插件名去掉 `.sh`）
> - `--dnssleep 180` = 写完 TXT 后等 180 秒再让 Let's Encrypt 来查（等 DNS 传播）
> - `-d` = 要签给哪个域名；**泛域名 `*.ff.example.com` 必须加引号**，不然 shell 会把 `*` 展开成文件名
> - `--staging` = 用 Let's Encrypt 的**测试环境**，签出的证书浏览器不信任，但流程完全一样——专门用来演练，避免把正式环境的签发次数限额用光（同一域名每周只有 5 次正式签发机会，滥用会被封 7 天）

这一步会依次：写 TXT → 等 180 秒 → 验证 → 删 TXT → 签出测试证书。**看到"Cert success"就算全链路通过。**

失败了看日志：

```bash
tail -n 100 ~/.acme.sh/acme.sh.log
```

### 第 8 步：正式签发

测试通过后，去掉 `--staging` 再跑一次：

```bash
~/.acme.sh/acme.sh --issue --dns dns_ali_role --dnssleep 180 \
  -d "ff.example.com" -d "*.ff.example.com"
```

**✅ 成功标志**：输出证书路径，位于 `~/.acme.sh/ff.example.com_ecc/` 下。**注意：先别动它，证书不能直接从这个目录引用**——下一步把正式文件装到 Nginx 目录。

---

## 八、阶段四：部署证书 + 全自动化（💻 SSH）

### 第 9 步：安装证书到 Nginx 并重载

```bash
mkdir -p /etc/nginx/ssl

~/.acme.sh/acme.sh --install-cert -d "ff.example.com" \
  --key-file       /etc/nginx/ssl/ff.example.com.key \
  --fullchain-file /etc/nginx/ssl/ff.example.com.cer \
  --reloadcmd      "nginx -t && nginx -s reload"
```

> - `--install-cert` = 把签好的证书**拷贝**到指定路径（这一步还会被记录下来，以后每次自动续期后都自动重新拷贝）
> - `--key-file` = 私钥放哪；`--fullchain-file` = 证书链放哪（**这两个路径必须和 Nginx 配置里 `ssl_certificate` 写的一致**，用 `nginx -T | grep ssl_certificate` 先查一下现状）
> - `--reloadcmd "nginx -t && nginx -s reload"` = 证书更新后自动执行的命令。`nginx -t` 先检查配置有没有写错，没错才 reload——防止坏配置直接上线

**如果 Nginx 跑在 Docker 里**，reload 命令换成：

```bash
--reloadcmd "docker exec <容器名> nginx -t && docker exec <容器名> nginx -s reload"
```

> 同时证书路径要填**宿主机上挂载进容器的目录**，否则容器里看不到新文件。

### 第 10 步：配置定时任务（自动化核心）

acme.sh 安装时通常会自动写入 cron，先确认：

```bash
crontab -l | grep acme
```

> `crontab -l` = 列出当前用户的所有定时任务

如果没有，手动加一条（每天 3:17 检查一次）：

```bash
(crontab -l 2>/dev/null; echo '17 3 * * * /root/.acme.sh/acme.sh --cron --home /root/.acme.sh >> /var/log/acme.log 2>&1') | crontab -
```

> - `17 3 * * *` = 每天 3 点 17 分（错开整点高峰）
> - `--cron` = acme.sh 的检查模式：证书距到期还有 30 天以上就什么都不做，不足 30 天自动续
> - `>> /var/log/acme.log 2>&1` = 所有输出追加到日志文件，出问题有据可查

**必须处理 PATH 问题**（最容易翻车的坑）：cron 执行任务时环境变量极简，很可能找不到 `aliyun` 命令 → 表现为"手动跑成功，定时任务静默失败"。两种解法任选：

```bash
# 解法 A：crontab -e 打开，在文件最顶部加一行
PATH=/usr/local/bin:/usr/bin:/bin

# 解法 B：在插件文件顶部写死 aliyun 的绝对路径
export ALIYUN_BIN=/usr/local/bin/aliyun
```

### 第 11 步：最终验证

```bash
# 1. 证书列表：能看到域名和"下次续期时间"就说明自动化已挂上
~/.acme.sh/acme.sh --list

# 2. 本地证书文件内容：颁发者 Let's Encrypt，90 天有效期
openssl x509 -in /etc/nginx/ssl/ff.example.com.cer -noout -subject -issuer -dates

# 3. 线上实际返回的证书：和本地一致
echo | openssl s_client -connect ff.example.com:443 -servername ff.example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# 4. HTTPS 正常返回
curl -I https://ff.example.com

# 5. 手动跑一次 cron（未到期不会真续，但能验证脚本能跑起来、PATH 没问题）
/root/.acme.sh/acme.sh --cron --home /root/.acme.sh
tail -n 50 /var/log/acme.log
```

> - `openssl s_client -connect 域名:443` = 模拟浏览器连上去看服务器出示的证书；`echo |` 是给它喂一个回车让它主动退出
> - 浏览器最终验收：打开网站 → 点地址栏小锁 → 看证书，颁发者 Let's Encrypt、有效期 90 天、域名含 `ff.example.com` 和 `*.ff.example.com`

---

## 九、出问题了怎么办（按现象查）

| 现象 | 最可能的原因 | 处理 |
|------|-------------|------|
| `--dns dns_ali_role` 报插件不存在 | 文件不在 `~/.acme.sh/dnsapi/`，或文件名/函数名不对 | 确认 `dns_ali_role.sh`，且函数叫 `dns_ali_role_add` / `dns_ali_role_rm` |
| `cannot list Aliyun DNS zones` | 工牌没绑上 / CLI profile 不对 / cron 的 PATH 问题 | 服务器上跑 `aliyun sts GetCallerIdentity --profile EcsAutoSslRenew` 验证；定时任务失败就加 PATH |
| `Forbidden.RAM` 权限不足 | 第 2 步的策略没授给角色，或 Action 少了 | 回控制台检查授权，补齐 4 个 alidns 动作 |
| 添加 TXT 报冲突 | `_acme-challenge` 上还有乐此的 CNAME | 删 CNAME，等 10~30 分钟，`dig` 确认消失再试 |
| 验证一直超时 | TXT 没传播 / CNAME 还在 / 域名写错 | `dig TXT _acme-challenge.ff.example.com` 亲自查；加大 `--dnssleep` |
| 证书更新了但网站还是旧的 | `--reloadcmd` 没执行成功 | 手动跑 `nginx -t && nginx -s reload` 看报什么错；Docker 场景检查挂载路径 |
| cron 不执行 | PATH / crond 没起 | `crontab -l` 确认任务在；`systemctl status crond`；看 `/var/log/acme.log` |
| 触发 Let's Encrypt 限流 | 反复 `--force` 正式签发 | 停 7 天等解封；以后测试一律加 `--staging` |
| 元数据返回 404 | 工牌没绑 / 不是 VPC / IMDS 被禁 | 回 ECS 控制台重新授予角色 |
| `jq: command not found` | 没装 jq | `yum install -y jq` 或 `apt install -y jq` |

---

## 十、如何回滚

```bash
# 1. 用第 0 步的备份恢复证书
cp -a /root/ssl-backup-<日期>/. /etc/nginx/ssl/
nginx -t && nginx -s reload

# 2. 如果决定彻底放弃本方案，删掉定时任务
crontab -l | grep -v acme.sh | crontab -
```

想切回[[执行方案-乐此加密|乐此加密]]：停掉 acme.sh 的 cron → 删掉 acme.sh 写的 TXT → 重新添加乐此给的 CNAME → 乐此平台重新验证。

---

## 十一、给老板看的一页纸

| 维度 | 乐此加密 | 本方案（RAM 角色 + acme.sh） |
|------|---------|------------------------------|
| 证书费用 | 免费 | 免费 |
| 自动续期费用 | 约 ¥30 一次性买断 | **0 元** |
| 续签时是否调用阿里云 | 不调用（只保留一条 CNAME） | 调用阿里云 DNS API |
| 是否依赖第三方平台 | 是（letsssl.cn 及其客户端） | 否，全在自己手里 |
| 服务器是否存长期密码 | — | **否**，每次刷工牌换临时凭证 |
| 部署难度 | 低（约 30–60 分钟） | 中高（约 2–4 小时，按本手册走） |
| 适合 | 想最快上线、续签完全不碰阿里云 | 要求 0 成本、自主可控、有阿里云权限 |

**一句话**：本方案不花一分钱，但要动阿里云账号的 RAM 角色和 DNS；照本手册配置一次，续签全自动。

**两条路线不能同时用**：[[执行方案-乐此加密|乐此]]靠 CNAME、本方案靠 TXT，二者在 DNS 里占同一个位置，互斥（原理见 [[从域名验证到证书认证-完整闭环梳理|从域名验证到证书认证]]）。

---

## 十二、附录

### 12.1 权限不够时的替代方案

- **让主账号协助（推荐）**：第 1~3 步（创建角色/策略/绑定）让有主账号的同事做，之后的步骤你在服务器上独立完成
- **退而求其次**：建一个只授 Alidns 权限的 RAM 用户，用它的 AccessKey 配合 acme.sh 自带 `dns_ali` 插件（`export Ali_Key=... Ali_Secret=...`）。缺点：长期密码存在服务器上，安全性不如工牌，需 `chmod 600` 并定期轮换
- **跨账号**（域名和 ECS 不同账号）：ECS 工牌只能管本账号 DNS，跨账号要么用对方账号的 RAM 用户 AK，要么走 RAM 跨账号信任（复杂，不推荐为这事上）
- **连 RAM 权限都没有**：只能走乐此加密，或把 DNS 迁到 Cloudflare/DNSPod 用它们的 API

### 12.2 多服务器 / Docker / CDN

- **多台 ECS**：同一个工牌可以绑给多台实例，每台各装 acme.sh 各自续签（推荐，无单点）；或一台签发后 scp 同步（注意私钥传输安全）
- **Docker 里的 Nginx**：证书写宿主机挂载目录 + reload 用 `docker exec`，先用 `docker inspect <容器名> | grep -A5 Mounts` 确认路径映射
- **CDN / SLB / WAF**：acme.sh 只写本机文件，上传到 CDN 需额外脚本；有这类需求建议评估 AllinSSL / Certd

### 12.3 安全说明

| 项目 | 说明 |
|------|------|
| 无长期 AK | 服务器只通过元数据拿临时凭证，自动过期 |
| 最小权限 | 工牌只授予 4 个 DNS 读写动作 |
| 残余风险 | Alidns 权限没法精确到单域名，工牌理论上能改本账号所有域名的解析；服务器本身的安全要做好 |
| 凭证文件 | `~/.acme.sh/account.conf` 可能含账号信息，`chmod 600` |
| 备份 | 定期备份 `~/.acme.sh` 整个目录（含账户密钥，丢了就要重新注册） |
| 日志纪律 | 不要把 STS 凭证、私钥、TXT 值贴到聊天/工单 |
| 测试纪律 | 一律先 `--staging`，避免触发 Let's Encrypt 正式环境限流 |

### 12.4 参考链接

- [acme.sh 官方仓库](https://github.com/acmesh-official/acme.sh)
- [acme.sh DNS API 开发指南](https://github.com/acmesh-official/acme.sh/wiki/DNS-API-Dev-Guide)
- [acme.sh dns_ali 不支持 STS 的 Issue #6118](https://github.com/acmesh-official/acme.sh/issues/6118)
- [阿里云 CLI 凭证模式与环境变量](https://help.aliyun.com/zh/cli/environment-variables)
- [阿里云 RAM 角色文档](https://help.aliyun.com/zh/ram/user-guide/ram-roles)
- [阿里云 Alidns API 文档](https://help.aliyun.com/zh/dns/api-alidns-2015-01-09/)
- [Let's Encrypt 频率限制](https://letsencrypt.org/docs/rate-limits/)
- [APNIC：CNAME 委派验证原理](https://blog.apnic.net/2023/07/03/delegated-domain-verification/)
