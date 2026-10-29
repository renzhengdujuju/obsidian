### 一、需求与问题汇总

|问题|风险/影响|解决方式|
|---|---|---|

|问题|风险/影响|解决方式|
|---|---|---|
|SSL 证书有效期短，人工续期容易忘|证书过期会导致网站提示不安全、服务中断|使用 `acme.sh` 自动续签|
|通配符证书必须用 DNS 验证|HTTP 验证无法签发 `*.domain.com`|使用阿里云 DNS-01 验证|
|不能长期写死 AccessKey/Secret|密钥泄露后可能被长期滥用|使用 ECS RAM 角色 + STS 临时凭证|
|`acme.sh` 的 `dns_ali` 插件默认依赖 `Ali_Key/Ali_Secret`|不能直接走 ECS RAM 角色|用阿里云 CLI 的 `EcsRamRole` 模式配合自定义 DNS Hook 桥接|
|证书更新后 Nginx 不生效|用户浏览器仍看到旧证书|在部署阶段绑定 `reloadcmd`，续期后自动重载 Nginx|
|定时任务环境缺少权限|手动成功、自动失败|通过 CLI Profile 自动从 ECS 元数据获取临时凭证|

### 二、方案说明

服务器是阿里云 ECS，域名解析在阿里云 DNS。  
给 ECS 绑定一个只允许管理 DNS 记录的 RAM 角色，服务器上不再保存任何长期 AccessKey/Secret。  
每次续签时，`acme.sh` 调用自定义脚本，脚本通过阿里云 CLI 的 `EcsRamRole` 模式自动获取 STS 临时凭证，再调用阿里云 DNS API 添加/删除 `_acme-challenge` TXT 记录。  
续签成功后，证书自动部署到 Nginx 并触发重载。

### 三、完整落地步骤

假设域名是 `example.com`，需要覆盖 `example.com` 和 `*.example.com`。请替换成你们实际域名。

### 1. 在阿里云控制台创建 RAM 角色

进入 RAM 控制台，选择 **身份管理 > 角色 > 创建角色**。

- 信任主体类型：选择 **云服务**
- 信任主体名称：选择 **云服务器 ECS**
- 角色名称：例如 `EcsAutoSslRenew`

创建后，为该角色添加权限：

- 系统策略：`AliyunDNSFullAccess`
- 更稳妥的做法是自建最小权限策略，只允许：

json

编辑

```
{
  "Version": "1",
  "Statement": [
    {
      "Action": [
        "alidns:DescribeDomains",
        "alidns:DescribeDomainRecords",
        "alidns:AddDomainRecord",
        "alidns:UpdateDomainRecord",
        "alidns:DeleteDomainRecord"
      ],
      "Resource": "*",
      "Effect": "Allow"
    }
  ]
}
```

### 2. 把 RAM 角色绑定到 ECS

在 ECS 控制台找到目标实例，进入 **实例设置 > 授予/收回 RAM 角色**，选择 `EcsAutoSslRenew` 并授予。

登录服务器后验证是否绑定成功：

bash

编辑

```
curl http://100.100.100.200/latest/meta-data/ram/security-credentials/
```

如果返回角色名称，说明绑定成功。

### 3. 安装 acme.sh

国内服务器建议用 Gitee 镜像：

bash

编辑

```
git clone https://gitee.com/neilpang/acme.sh.git
cd acme.sh
./acme.sh --install -m your-email@example.com
source ~/.bashrc
```

然后切换默认 CA：

bash

编辑

```
acme.sh --set-default-ca --server letsencrypt
```

### 4. 安装阿里云 CLI

bash

编辑

```
aliyun version
```

如果未安装，按系统安装。阿里云 CLI 版本建议不低于 `3.3.0`。

### 5. 配置 CLI 使用 ECS RAM 角色

bash

编辑

```
aliyun configure --mode EcsRamRole --profile EcsAutoSslRenew
```

按提示输入：

- Ecs Ram Role：`EcsAutoSslRenew`
- Default Region Id：例如 `cn-shanghai`

验证：

bash

编辑

```
aliyun sts get-caller-identity --profile EcsAutoSslRenew
```

能返回 ARN 信息，说明临时凭证可用。

### 6. 创建 DNS Hook 脚本

`acme.sh` 的 `dns_ali` 插件本身依赖 `Ali_Key/Ali_Secret`，所以这里用自定义 Hook 替代，让它通过 CLI 调用阿里云 DNS API。

创建脚本：

bash

编辑

```
mkdir -p /root/.acme.sh/dns_hook
vim /root/.acme.sh/dns_hook/aliyun_ecs_role.sh
```

写入：

bash

编辑

```
#!/bin/bash
set -e

ROLE_PROFILE="EcsAutoSslRenew"
REGION="cn-shanghai"

add_dns_record() {
  local domain="$1"
  local txt="$2"

  aliyun alidns AddDomainRecord \
    --profile "$ROLE_PROFILE" \
    --region "$REGION" \
    --DomainName "$domain" \
    --RR "_acme-challenge" \
    --Type TXT \
    --Value "$txt"
}

update_dns_record() {
  local record_id="$1"
  local txt="$2"

  aliyun alidns UpdateDomainRecord \
    --profile "$ROLE_PROFILE" \
    --region "$REGION" \
    --RecordId "$record_id" \
    --RR "_acme-challenge" \
    --Type TXT \
    --Value "$txt"
}

delete_dns_record() {
  local record_id="$1"

  aliyun alidns DeleteDomainRecord \
    --profile "$ROLE_PROFILE" \
    --region "$REGION" \
    --RecordId "$record_id"
}

find_record_id() {
  local domain="$1"

  aliyun alidns DescribeDomainRecords \
    --profile "$ROLE_PROFILE" \
    --region "$REGION" \
    --DomainName "$domain" \
    --RRKeyWord "_acme-challenge" \
    --Type TXT \
    --output json
}

case "$1" in
  add)
    add_dns_record "$2" "$3"
    ;;
  clean)
    RECORD_JSON=$(find_record_id "$2")
    RECORD_ID=$(echo "$RECORD_JSON" | grep -o '"RecordId":"[^"]*"' | head -1 | cut -d'"' -f4)

    if [ -n "$RECORD_ID" ]; then
      delete_dns_record "$RECORD_ID"
    fi
    ;;
  *)
    echo "Usage: $0 {add|clean} domain txt_value"
    exit 1
    ;;
esac
```

赋予执行权限：

bash

编辑

```
chmod +x /root/.acme.sh/dns_hook/aliyun_ecs_role.sh
```

### 7. 首次签发证书

bash

编辑

```
acme.sh --issue \
  --dns /root/.acme.sh/dns_hook/aliyun_ecs_role.sh \
  -d example.com \
  -d *.example.com
```

如果第一次解析传播较慢，可以增加等待时间：

bash

编辑

```
acme.sh --issue \
  --dns /root/.acme.sh/dns_hook/aliyun_ecs_role.sh \
  --dnssleep 60 \
  -d example.com \
  -d *.example.com
```

### 8. 安装证书到 Nginx

先确认证书目录：

bash

编辑

```
mkdir -p /etc/nginx/ssl
```

然后执行：

bash

编辑

```
acme.sh --install-cert -d example.com \
  --key-file /etc/nginx/ssl/example.com.key \
  --fullchain-file /etc/nginx/ssl/example.com.cer \
  --reloadcmd "nginx -s reload"
```

如果是 Docker 里的 Nginx，改成：

bash

编辑

```
--reloadcmd "docker exec nginx_container nginx -s reload"
```

之后修改 Nginx 配置，指向这两个证书文件，并重载 Nginx。

### 9. 配置自动续签

`acme.sh` 安装后通常会自动创建 cron 任务：

bash

编辑

```
crontab -l | grep acme
```

如果没有，可以手动添加：

bash

编辑

```
(crontab -l 2>/dev/null; echo "0 2 * * * /root/.acme.sh/acme.sh --cron --home /root/.acme.sh >> /var/log/acme.log 2>&1") | crontab -
```

因为自定义 Hook 脚本里已经通过 `--profile EcsAutoSslRenew` 使用 ECS RAM 角色，所以 cron 环境不需要再设置任何 AccessKey。

### 10. 验证自动续签是否可用

先干跑一次：

bash

编辑

```
/root/.acme.sh/acme.sh --cron --home /root/.acme.sh
```

再检查：

bash

编辑

```
acme.sh --list
```

查看证书域名、签发时间、下一次续期时间是否正常。

### 四、常见问题与处理

|问题|原因|处理|
|---|---|---|

|问题|原因|处理|
|---|---|---|
|`404 Not Found`|ECS 未绑定 RAM 角色|在 ECS 控制台重新授予角色|
|CLI 报权限不足|角色权限不够|补充 `AliyunDNSFullAccess` 或自定义 DNS 权限|
|DNS 记录添加失败|域名不在当前账号或 Region 不对|检查域名所属阿里云账号和 CLI Region|
|续签成功但网站未更新|Nginx 未重载|检查 `--reloadcmd` 是否正确|
|Docker 环境 reload 失败|宿主机没有 nginx 服务|改为 `docker exec` 重载|
|`_acme-challenge` 残留|上次验证中断|在 DNS 控制台删除残留 TXT 记录后重试|
|触发 CA 限流|频繁强制续签|暂停 7 天，先用测试域名验证脚本|

### 五、交付时可以这样说明

> 当前方案不使用任何长期 AccessKey/Secret。ECS 已绑定 RAM 角色，服务器每次续签时通过阿里云 CLI 自动获取 STS 临时凭证，完成 DNS 验证后凭证会自动失效。证书由 `acme.sh` 每日自动检查，到期前自动续签并重新部署到 Nginx，无需人工介入。