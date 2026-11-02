# GitLab + Runner Docker 部署与产物测试排错笔记

在 Windows 本机通过 Docker Desktop 部署 GitLab 和 GitLab Runner，并进行 CI/CD 产物（Artifacts）测试。整个过程踩了不少坑，主要集中在 **Windows 文件系统兼容性** 和 **容器网络通信** 上。以下是踩坑记录与解决方案。

## 一、整体遇到的核心错误

### 1. `502 Waiting for GitLab to boot` 或 `ERR_EMPTY_RESPONSE`

- **现象**：启动容器后，浏览器访问 `http://localhost` 报错或一直转圈。
- **原因**：GitLab 首次启动需要初始化数据库（PostgreSQL 建表等），非常消耗资源，在 Windows 下通常需要 5–10 分钟，此时内部的 Nginx 还没准备好接收请求。
- **解决**：耐心等待，通过 `docker logs -f gitlab` 查看日志，直到出现 `Reconfigured!` 字样。期间可去获取初始密码。

### 2. `fatal: unable to access 'http://localhost/...': Failed to connect to localhost port 80`

- **现象**：流水线启动后，Job 瞬间失败（exit code 128）。
- **原因**：Runner 拉取代码时使用了 GitLab 默认生成的 `localhost` 地址。但在 Docker 容器内部，`localhost` 指向容器自己，而不是宿主机上的 GitLab。
- **解决**：修改 Runner 的 `config.toml`，在 `[[runners]]` 层级下加一行 `clone_url = "http://host.docker.internal"`。

> 原理：`host.docker.internal` 是 Docker Desktop 为 Windows/Mac 提供的特殊域名，永远指向宿主机。加这行后，Runner 不管 GitLab 给的代码地址是什么，都把前缀替换成它去拉取。

### 3. `500 Internal Server Error`（上传产物失败）

- **现象**：脚本执行成功，但最后一步 `Uploading artifacts...` 报 500，提示 `upload failed with no response from coordinator`。
- **原因**：Windows 下最坑的问题。最初用了本地路径挂载（Bind Mount，如 `C:\gitlab-test\data:/var/opt/gitlab`），Windows 的 NTFS 与 Linux 容器的文件权限不兼容，导致 GitLab 无法写入产物文件。
- **解决**：放弃本地路径挂载，改用 **Docker 命名卷（Named Volumes）**，让 Docker 在 WSL2 内部管理数据。

### 4. `Failed to pull image "alpine:latest"` 或 `Failed to fetch anonymous token`

- **现象**：Runner 使用 `docker` 执行器时，尝试去 Docker Hub 拉取镜像，因网络原因超时或被阻断。
- **原因**：国内网络访问 Docker Hub 受限。
- **解决**：修改 `config.toml`，加 `pull_policy = "if-not-present"`（有本地镜像就用本地）；或直接更换执行器。

### 5. `Pending` 状态卡死不动

- **现象**：流水线一直排队，Runner 在线却不接单。
- **原因**：Windows 下 Docker Desktop 配合 `docker` 执行器（需要调用宿主机 socket）时常遇到 socket 权限问题，导致 Runner 崩溃或卡死。
- **解决**：改用 **`shell` 执行器**，让 Runner 在自己容器内直接跑脚本，绕开调用宿主机 Docker 的兼容性深坑。

### 6. `PANIC: decoding configuration file: toml: line 9 ... expected value`

- **现象**：注册或启动 Runner 时报 TOML 格式错误。
- **原因**：手动改 `config.toml` 时等号、引号写错，或用命令行 `sed` 插入时发生语法折叠。
- **解决**：直接删掉配置卷/配置文件，用干净的交互式命令重新注册。

## 二、最终正确的配置与步骤

### 1. `docker-compose.yml`（使用命名卷）

```yaml
services:
  gitlab:
    image: 'gitlab/gitlab-ce:latest'
    container_name: gitlab
    restart: always
    hostname: 'localhost'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://localhost'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
    ports:
      - '80:80'
      - '443:443'
      - '2222:22'
    volumes:
      - gitlab-config:/etc/gitlab
      - gitlab-logs:/var/log/gitlab
      - gitlab-data:/var/opt/gitlab
    shm_size: '256m'
  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: always
    volumes:
      - runner-config:/etc/gitlab-runner
volumes:
  gitlab-config:
  gitlab-logs:
  gitlab-data:
  runner-config:
```

### 2. 注册 Runner（使用 `shell` 执行器）

```bash
docker run --rm -it -v C:\gitlab-test\runner-config:/etc/gitlab-runner gitlab/gitlab-runner register
```

交互式填写 URL、Token、名字，执行器填 `shell`。

### 3. 修复克隆地址

找到挂载出的本地 `runner-config/config.toml`，在 `[[runners]]` 下添加：

```toml
clone_url = "http://host.docker.internal"
```

然后重启 Runner：`docker restart gitlab-runner`

### 4. 测试用 `.gitlab-ci.yml`

```yaml
stages:
  - build
test-artifact:
  stage: build
  script:
    - echo "这是构建产物内容" > artifact.txt
    - echo "测试构建成功！"
  artifacts:
    paths:
      - artifact.txt
```

## 三、经验总结

1. **Windows 下跑 Docker 服务，坚决避开 Bind Mount（本地路径挂载）**，尤其是数据库和文件系统交互频繁的应用（如 GitLab）。**永远首选命名卷（Named Volumes）**，这是避免权限 500 错误的最有效手段。
2. **Windows 下 GitLab Runner 的 `docker` 执行器极易卡死**。只跑简单脚本或测试时，直接用 `shell` 执行器最稳。
3. **容器内的 `localhost` 永远指向容器自己**。容器与宿主机通信，Windows 下用 `host.docker.internal`。
4. **不要盲目用 `sed` 或命令行改 TOML 配置**，引号或缩进不对直接报 PANIC。用 `nano`/`vi` 或挂载出来用记事本改最稳妥。
5. 遇到 `Pending` 或 `500`，第一时间看 `docker logs gitlab` 和 `docker logs gitlab-runner`。
