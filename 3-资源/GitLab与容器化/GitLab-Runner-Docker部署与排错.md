# GitLab + Runner Docker 部署与产物测试排错笔记
在 Windows 本机通过 Docker Desktop 部署 GitLab 和 GitLab Runner，并进行 CI/CD 产物（Artifacts）测试。整个过程踩了不少坑，主要集中在**Windows 文件系统兼容性**和**容器网络通信**上。以下是完整的踩坑记录与解决方案。
---
## 一、 整体遇到的核心错误
### 1. `502 Waiting for GitLab to boot` 或 `ERR_EMPTY_RESPONSE`
*   **现象**：启动容器后，浏览器访问 `http://localhost` 报错或一直转圈。
*   **原因**：GitLab 首次启动需要初始化数据库（PostgreSQL 建表等），非常消耗资源，在 Windows 下通常需要 5-10 分钟。此时内部的 Nginx 还没准备好接收请求。
*   **解决**：耐心等待。通过 `docker logs -f gitlab` 查看日志，直到出现 `Reconfigured!` 字样。期间可去获取初始密码。
### 2. `fatal: unable to access 'http://localhost/...': Failed to connect to localhost port 80`
*   **现象**：流水线启动后，Job 瞬间失败（exit code 128）。
*   **原因**：Runner 在拉取代码时，使用了 GitLab 默认生成的 `localhost` 地址。但在 Docker 容器内部，`localhost` 指向的是容器自己，而不是宿主机上的 GitLab。
*   **解决**：
    *   修改 Runner 的 `config.toml` 文件。
    *   在 `[[runners]]` 层级下加入：`clone_url = "http://host.docker.internal"`。
### 3. `500 Internal Server Error` (上传产物失败)
*   **现象**：脚本执行成功，但在最后一步 `Uploading artifacts...` 时报 500，提示 `upload failed with no response from coordinator`。
*   **原因**：这是 Windows 下最坑的问题。最初使用了本地路径挂载（Bind Mount，如 `C:\gitlab-test\data:/var/opt/gitlab`）。Windows 的 NTFS 文件系统与 Linux 容器的文件权限不兼容，导致 GitLab 无法正常写入产物文件。
*   **解决**：放弃本地路径挂载，改用 **Docker 命名卷（Named Volumes）**，让 Docker 在 WSL2 内部管理数据。
### 4. `Failed to pull image "alpine:latest"` 或 `Failed to fetch anonymous token`
*   **现象**：Runner 使用 `docker` 执行器时，尝试去 Docker Hub 拉取镜像，因网络原因超时或被阻断。
*   **原因**：国内网络访问 Docker Hub 受限。
*   **解决**：不要让它去网上拉。修改 Runner 的 `config.toml`，加上 `pull_policy = "if-not-present"`（有本地镜像就用本地的）。或者直接更换执行器。
### 5. `Pending` 状态卡死不动
*   **现象**：流水线一直显示排队中，Runner 明明在线却不接单。
*   **原因**：Windows 下 Docker Desktop 配合 `docker` 执行器（需要调用宿主机 socket）时，经常会遇到 socket 权限问题，导致 Runner 内部崩溃或卡死。
*   **解决**：改用 **`shell` 执行器**。让 Runner 在自己的容器内直接运行脚本，彻底绕开调用宿主机 Docker 的兼容性深坑。
### 6. `PANIC: decoding configuration file: toml: line 9 ... expected value`
*   **现象**：注册或启动 Runner 时报 TOML 格式错误。
*   **原因**：手动修改 `config.toml` 时，等号、引号写错，或者用命令行 `sed` 插入时发生语法折叠。
*   **解决**：直接删掉配置卷/配置文件，使用干净的交互式命令重新注册。
---
## 二、 最终正确的配置与步骤
### 1. `docker-compose.yml` 配置（使用命名卷）
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

### 2. 注册 Runner（使用 `shell` 执行器）

bash

docker run --rm -it -v C:\gitlab-test\runner-config:/etc/gitlab-runner gitlab/gitlab-runner register

_(交互式填写 URL、Token、名字，执行器填入 `shell`)_

### 3. 修复克隆地址

找到挂载出的本地 `runner-config/config.toml`，在 `[[runners]]` 下添加：

toml

clone_url = "http://host.docker.internal"

然后重启 Runner：`docker restart gitlab-runner`

### 4. 测试用 `.gitlab-ci.yml`

yaml

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

---

## 三、 经验总结

1. **Windows 下玩 Docker 跑服务，坚决避开 Bind Mount（本地路径挂载）**，尤其是数据库和文件系统交互频繁的应用（如 GitLab）。**永远首选 Named Volumes（命名卷）**，这是避免权限 500 错误的最有效手段。
    
2. **Windows 下 GitLab Runner 的 `docker` 执行器极易卡死**。如果只是跑简单脚本或测试，直接用 `shell` 执行器最稳。
    
3. **容器内的 `localhost` 永远指向容器自己**。容器与宿主机通信，Windows 下用 `host.docker.internal`。
    
4. **不要盲目用 `sed` 或命令行改 TOML 配置**，如果引号或缩进不对直接报 PANIC。用 `nano`（Debian系）、`vi` 或直接挂载出来用记事本改最稳妥。
    
5. 遇到 `Pending` 或 `500`，第一时间看 `docker logs gitlab` 和 `docker logs gitlab-runner`。
   
在这行 `clone_url = "http://host.docker.internal"` 加在 `[runners]`（实际上是 `[[runners]]`）下面，核心目的是：**强制改变 Runner 下载（Clone）代码时使用的 Git 仓库地址。**

为了让你彻底明白，我们来看一下如果不加这行，会发生什么：

### 1. 默认的冲突场景（不加这行）

当你在 GitLab 网页上点击运行流水线时，GitLab 会告诉 Runner：“去 `http://localhost/root/my-project.git` 这个地址把代码拉下来”。

- **你的 Windows 电脑**：访问 `localhost` 就是访问你自己的电脑（宿主机），能正确找到 GitLab。
    
- **Runner 容器**：它是一个独立运行的 Linux 小系统。当它去访问 `localhost` 时，它是在**找它自己内部**的 80 端口，而它自己内部根本没有 GitLab。所以直接报错：`Failed to connect to localhost port 80`。
    

### 2. 加上这行后的变化

加上 `clone_url = "http://host.docker.internal"` 后，就相当于给 Runner 强行下了一道指令：

> “以后不管 GitLab 告诉你代码地址是什么，你统统把前缀替换成 `http://host.docker.internal` 去拉取！”

`host.docker.internal` 是 Docker Desktop 专门为 Windows/Mac 提供的一个**魔法域名**，它永远指向你的宿主机（也就是你的 Windows 电脑）。  
所以，Runner 最终实际去访问的地址变成了：`http://host.docker.internal/root/my-project.git`。  
这个地址在 Runner 容器里一解析，就完美连上了你 Windows 电脑上的 80 端口，代码就成功拉下来了。