**1. 准备阿里云“临时通行证” (RAM 子账号)**

- **做什么**：登录阿里云控制台，创建一个 RAM 子账号，只给它一个权限：`AliyunDNSFullAccess`（DNS解析管理权限）。
- **得到什么**：拿到这个子账号的 **AccessKey ID** 和 **AccessKey Secret**。

**2. 服务器上安装 acme.sh**

- 登录到你们的服务器，执行一键安装命令（国内服务器使用 Gitee 镜像以防下载失败）：
    
    ```
    git clone https://gitee.com/neilpang/acme.sh.git && cd acme.sh && ./acme.sh --install -m your-email@example.com
    ```
    
- 安装后，建议退出终端重新连接，让 `acme.sh` 命令生效。
    

**3. 申请并签发 SSL 证书**

- 把第一步拿到的密钥告诉 `acme.sh`，并开始签发（假设你们的域名是 `ff.com`）：
    ```
    export Ali_Key="你的AccessKey ID"
    export Ali_Secret="你的AccessKey Secret"
    
    # 签发通配符证书（支持 *.ff.com 和 ff.com）
    acme.sh --issue --dns dns_ali -d ff.com -d *.ff.com
    ```
    
- _注：此时脚本会自动调用阿里云接口去创建和删除那条 `_acme-challenge` 验证记录。_
    

**4. 安装证书并配置自动重载**

- 证书签发好后，不能直接用它默认的路径，需要拷贝到 Nginx 的指定目录下，并告诉它如何重载：

    
    ```
    acme.sh --install-cert -d ff.com -d *.ff.com \
    --key-file       /etc/nginx/ssl/ff.com.key  \
    --fullchain-file /etc/nginx/ssl/ff.com.cer \
    --reloadcmd     "nginx -s reload"
    ```
    
- **完成！** `acme.sh` 会在安装这一步自动创建一个每天定时检查的 Cron 任务，后续完全不需要人工干预。
    

---

###  第二步：核心难点与解决方案

在实际部署中，经常会遇到“手动操作能成功，但自动续期却静默失败”的情况，以下是三个最常见的致命难点及排坑方案：

#### 难点一：Cron 定时任务“静默失效”

- **表现**：手动执行 `acme.sh --cron` 一切正常，证书也能成功续期，但每天定时任务就是无法自动续期，直到证书过期。
    
- **原因**：Cron 定时任务的执行环境极其精简，它没有读取到你在服务器终端里 `export` 的阿里云密钥（Ali_Key 和 Ali_Secret），导致它没有权限去修改 DNS 验证记录。
    
- **解决方案**：  
    将阿里云密钥直接写入 `acme.sh` 的全局配置文件中，永久保存：
    
    ```
    acme.sh --register-account -m your-email@example.com \
    --server letsencrypt \
    --issue --dns dns_ali \
    --yes-I-know-dns-manual-mode-enough-go-ahead-please \
    --log \
    --force --debug 2
    ```
    
    _(实际上，在签发证书成功后，`acme.sh` 通常会自动将凭证缓存到 `~/.acme.sh/account.conf` 中，但需确保该文件权限正常。最稳妥的是在服务器重启后测试定时任务是否生效。)_
    

#### 难点二：续期成功，但网站证书没更新（Nginx 没重载）

- **表现**：SSL 证书文件的时间戳更新了，但用户打开浏览器看，证书依然是过期的。
- **原因**：证书文件虽然更新了，但 Nginx 服务器还加载着旧的证书。`--install-cert` 里的 `reloadcmd` 命令写错了（比如路径不对，或者 Docker 环境下没有用 `docker exec`），导致 Nginx 没有平滑重载。
- **解决方案**：
    - 如果是传统服务器，确保 `nginx -s reload` 能直接运行。
    - 如果是 **Docker** 部署，需要改为类似：`--reloadcmd "docker exec -i nginx_container nginx -s reload"`。

#### 难点三：触发官方限流，无法签发

- **表现**：在执行签发时，报错提示 `rate limit`（速率限制），导致签发失败。
- **原因**：Let's Encrypt 等证书机构有严格限制，同一域名 7 天内最多只能成功签发 5 次。如果在排查问题时频繁使用 `--force` 强制续签，就会触发限流，且工具通常不会给你明确提示。
- **解决方案**：  
    一旦触发限流，除了等待 7 天解封外别无他法。因此，**排查问题时千万不要频繁强制续期**，可以用测试域名测试脚本，确认无误后再用正式域名。