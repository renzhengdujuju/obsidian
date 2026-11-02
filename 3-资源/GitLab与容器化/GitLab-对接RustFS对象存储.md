# GitLab 对接 RustFS 对象存储实战排错笔记

## 一、核心目标与架构变更

- **目标**：将 GitLab CI/CD 的流水线产物（Artifacts）从本地磁盘存储，剥离并转移到独立的 **RustFS** 对象存储中。
- **架构原理**：GitLab 跑完流水线后，不再把产物存自己肚子里，而是通过 S3 协议（走 9000 端口），把文件作为对象存入 RustFS 的 `gitlab-artifacts` 桶（Bucket）中。

## 二、遇到的四大"深坑"及解决方法

### 坑 1：Windows 本地路径挂载导致产物上传报 `500 Internal Server Error`

- **现象**：流水线脚本执行成功，但 `Uploading artifacts...` 环节一直报 500 错误（无响应）。
- **原因**：Windows Docker Desktop 下，直接挂载本地文件夹（Bind Mount，如 `./vdata:/var/opt/gitlab`）给 Linux 容器用。NTFS 与 Linux 权限模型不兼容，GitLab 写入产物文件被拒绝，服务端崩溃。
- **解决办法**：
  1. **首选**：改用 Docker 命名卷（Named Volumes），让 Docker 在 WSL2 内部自己管理文件。
  2. **公司规范方案**：如果必须用本地路径挂载，需在 Docker Desktop 的 `Settings -> Resources -> File Sharing` 把本地文件夹加入共享白名单并赋读写权限。

### 坑 2：RustFS 容器无限重启（`Restarting`）

- **现象**：启动 RustFS 容器后状态一直是 `Restarting`，服务起不来。
- **日志报错**：`local erasure endpoints must use distinct physical disks`
- **原因**：RustFS 默认用纠删码（Erasure Coding）保证数据安全，要求数据节点位于不同物理硬盘。单机 Docker 测试下多个目录都在同一块虚拟盘上，触发了它的安全保护。
- **解决办法**：在 `docker-compose-rustfs.yml` 的 `environment` 里加 `RUSTFS_UNSAFE_BYPASS_DISK_CHECK=true`，跳过物理盘检查（仅限本地测试/CI）。

### 坑 3：下载产物时浏览器报 `host.docker.internal 拒绝连接`

- **现象**：在 GitLab 网页点下载产物，浏览器跳转到 `http://host.docker.internal:9000/...` 并报错。
- **原因**：GitLab 默认"直连模式"下载，生成下载链接时用了自己容器内配置的 RustFS 地址（`host.docker.internal`），但浏览器跑在 Windows 宿主机上，解析不了 Docker 内部域名。
- **解决办法**：
  1. 进入 GitLab 容器，改 `/etc/gitlab/gitlab.rb`。
  2. 把 `gitlab_rails['artifacts_object_store_proxy_download']` 改为 `true`（开启代理下载）。
  3. 执行 `gitlab-ctl reconfigure` 生效。这样用户点下载时，由 GitLab 代为从 RustFS 取文件流式传给浏览器。

### 坑 4：Runner 配置文件格式损坏导致解析失败（`PANIC` / `FATAL`）

- **现象**：改 `config.toml` 后 Runner 容器不断崩溃重启。
- **日志报错**：`toml: line 9 (last key "runners.clone_url"): expected value but found "http" instead`
- **原因**：在 Windows PowerShell 里用 `sed`/`echo` 改 TOML 时，反斜杠和引号转义错乱，等号后出现非法字符，解析器崩溃。
- **解决办法**：清空坏配置（或命名卷）重新注册；或在容器内用 Linux 原生语法精确修改。

## 三、最终跑通的完整配置参考

### 1. RustFS 启动配置（`docker-compose-rustfs.yml`）

```yaml
services:
  rustfs:
    image: rustfs/rustfs:latest
    container_name: rustfs
    ports:
      - "9000:9000" # S3 API 端口
      - "9001:9001" # 控制台端口
    environment:
      - RUSTFS_UNSAFE_BYPASS_DISK_CHECK=true # 本地单机测试跳过磁盘检查
      - RUSTFS_ACCESS_KEY=rustfsadmin
      - RUSTFS_SECRET_KEY=rustfsadmin
    volumes:
      - rustfs_data:/data
    restart: unless-stopped
volumes:
  rustfs_data:
```

### 2. GitLab 对接 RustFS 的核心配置（`docker-compose.yml`）

```yaml
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://localhost'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222

        # 开启产物并连接 RustFS
        gitlab_rails['artifacts_enabled'] = true
        gitlab_rails['artifacts_object_store_enabled'] = true
        gitlab_rails['artifacts_object_store_proxy_download'] = true # 强制 GitLab 代理下载
        gitlab_rails['artifacts_object_store_remote_directory'] = "gitlab-artifacts"
        gitlab_rails['artifacts_object_store_connection'] = {
          'provider' => 'AWS',
          'endpoint' => 'http://host.docker.internal:9000',
          'region' => 'us-east-1',
          'aws_access_key_id' => 'rustfsadmin',
          'aws_secret_access_key' => 'rustfsadmin',
          'path_style' => true,
          'enable_signature_v4_streaming' => false
        }
```

### 3. 验证用的 `.gitlab-ci.yml`

```yaml
stages:
  - build
  - test
build:
  stage: build
  script:
    - mkdir aaaa
    - echo 1 > aaaa/2.txt
  artifacts:
    paths:
      - aaaa/
    expire_in: 1 day
test:
  stage: test
  dependencies:
    - build
  script:
    - cd aaaa
    - ls
```

## 四、核心经验总结（Takeaways）

1. **对象存储的优势**：彻底解决单台服务器磁盘容量瓶颈，数据与计算节点解耦，支持水平扩展，是大型企业 CI/CD 的标配。
2. **代理下载 vs 直连下载**：跨越网络边界（宿主机 ↔ 容器，或内网 ↔ 外网）时，`proxy_download = true` 是最简单粗暴且稳妥的排错方案。
3. **容器内的 `localhost`**：Docker 容器里的 `localhost` 指容器自己；要访问宿主机，Windows/Mac 下用 `host.docker.internal`。
4. **排错三把斧**：看日志（`docker logs`）→ 查状态（`docker ps`）→ 进容器排查（`docker exec`）。不要盲猜，日志里的报错代码（500、128、FATAL）就是最准的指南针。
