## 一、最适合先玩的项目

### 1. Ollama：在本地运行 AI 模型

地址：[https://github.com/ollama/ollama](https://github.com/ollama/ollama)

可以在自己的电脑上运行开源大模型，例如 Qwen、Llama、Mistral 等。

可以体验：

- 本地聊天
- 不依赖网页端使用 AI
- 通过命令行运行模型
- 配合 Open WebUI 使用成一个本地版 ChatGPT
- 让其他软件调用本地模型

难度：低  
适合程度：★★★★★

如果你对 AI 感兴趣，建议优先玩这个。

---

### 2. Open WebUI：给本地 AI 加一个漂亮网页界面

地址：[https://github.com/open-webui/open-webui](https://github.com/open-webui/open-webui)

Open WebUI 本身不是模型，它是一个聊天界面，通常配合 Ollama 使用。

安装 Ollama 后，可以通过 Open WebUI 实现：

- 网页聊天
- 管理多个模型
- 上传文件并提问
- 知识库问答
- 保存聊天记录
- 多用户使用

难度：低  
适合程度：★★★★★

简单理解就是：

> Ollama 负责运行模型，Open WebUI 负责提供操作界面。

---

### 3. AnythingLLM：给自己的文件建立 AI 知识库

地址：[https://github.com/Mintplex-Labs/anything-llm](https://github.com/Mintplex-Labs/anything-llm)

可以把 PDF、Word、Markdown、网页资料等导入进去，然后直接向这些资料提问。

适合用来做：

- 公司制度问答
- 个人资料库
- 技术文档搜索
- 项目资料整理
- 本地文档问答

难度：低到中  
适合程度：★★★★★

如果你想体验“让 AI 读取自己的资料”，这个项目比单纯聊天更有意思。

---

### 4. ComfyUI：可视化生成图片和视频

地址：[https://github.com/comfyanonymous/ComfyUI](https://github.com/comfyanonymous/ComfyUI)

ComfyUI 使用节点连接的方式生成图片、图片变化和视频。

可以玩：

- 文生图
- 图生图
- 人物换装
- 图片放大
- 局部重绘
- 工作流组合
- AI 视频相关流程

难度：中  
适合程度：★★★★★

它的特点是自由度很高，但刚开始会觉得节点比较多。适合喜欢研究图片生成和工作流的人。

---

### 5. Stable Diffusion WebUI Forge：更容易上手的 AI 绘图界面

地址：[https://github.com/lllyasviel/stable-diffusion-webui-forge](https://github.com/lllyasviel/stable-diffusion-webui-forge)

这是一个相对成熟的 AI 绘图 WebUI，可以通过浏览器操作。

可以体验：

- 文生图
- 图生图
- ControlNet
- LoRA
- 图片修复
- 人物姿态控制
- 图片放大

难度：中  
适合程度：★★★★☆

如果你想直接开始 AI 绘图，可以先研究这个；如果想深入理解工作流，再尝试 ComfyUI。

---

## 二、适合做实用工具

### 6. n8n：可视化自动化工作流

地址：[https://github.com/n8n-io/n8n](https://github.com/n8n-io/n8n)

n8n 可以把不同网站、接口和工具连接起来，做成自动化流程。

例如：

- 收到邮件后自动整理内容
- 定时读取网页信息
- 自动调用 AI 总结文本
- 表单提交后写入数据库
- 自动发送企业微信或邮件通知
- 定时备份文件

难度：中  
适合程度：★★★★★

它有点像“程序员版的自动化平台”，不一定要先会写很多代码。

---

### 7. Stirling-PDF：本地 PDF 工具箱

地址：[https://github.com/Stirling-Tools/Stirling-PDF](https://github.com/Stirling-Tools/Stirling-PDF)

这是一个功能比较完整的 PDF 工具，可以自己部署。

支持：

- PDF 合并
- PDF 拆分
- PDF 转 Word
- PDF 转图片
- 添加水印
- OCR
- 旋转页面
- 压缩 PDF
- 修改 PDF 信息

难度：低  
适合程度：★★★★☆

适合先部署一个实用项目，成功率比较高。

---

### 8. Paperless-ngx：建立自己的文档管理系统

地址：[https://github.com/paperless-ngx/paperless-ngx](https://github.com/paperless-ngx/paperless-ngx)

可以把纸质文件、扫描件、PDF、合同和票据集中管理，并通过 OCR 和标签进行搜索。

可以体验：

- 上传文件自动识别文字
- 按标签分类
- 搜索文档内容
- 管理合同和票据
- 建立个人资料库

难度：中  
适合程度：★★★★☆

如果你手上有很多 PDF 和扫描资料，这个项目很实用。

---

### 9. Immich：自己搭建照片管理系统

地址：[https://github.com/immich-app/immich](https://github.com/immich-app/immich)

可以搭建一个类似 Google Photos 或 iCloud Photos 的私人照片管理平台。

支持：

- 手机自动备份照片
- 相册管理
- 人脸识别
- 地点和时间整理
- 图片搜索
- 视频管理

难度：中  
适合程度：★★★★★

不过它比较依赖硬盘空间，正式使用前需要注意备份问题。

---

### 10. Syncthing：电脑和手机之间自动同步文件

地址：[https://github.com/syncthing/syncthing](https://github.com/syncthing/syncthing)

Syncthing 可以让多台设备之间自动同步文件，不需要把资料上传到第三方网盘。

可以用来：

- 同步电脑和笔记本文件
- 同步手机照片
- 同步 Markdown 笔记
- 在多台设备之间传输资料

难度：低到中  
适合程度：★★★★☆

它更像一个“自己控制的文件同步工具”。

---

### 11. LocalSend：局域网文件传输

地址：[https://github.com/localsend/localsend](https://github.com/localsend/localsend)

安装到电脑和手机后，可以在同一个局域网里快速传文件。

适合：

- 手机传照片到电脑
- 电脑传文件到手机
- 不想使用网盘时临时传输文件
- 局域网内传大文件

难度：低  
适合程度：★★★★★

这个项目非常适合先安装体验，操作简单，也比较直观。

---

## 三、适合开发和技术实验

### 12. OpenHands：AI 编程助手

地址：[https://github.com/All-Hands-AI/OpenHands](https://github.com/All-Hands-AI/OpenHands)

OpenHands 可以尝试让 AI 在项目目录中分析代码、修改文件、运行命令和完成开发任务。

可以体验：

- 让 AI 阅读项目
- 自动修改代码
- 运行测试
- 分析报错
- 生成代码
- 处理简单开发任务

难度：中到高  
适合程度：★★★★☆

使用时一定要限制它可以访问的目录，不要直接让它操作重要项目或包含敏感信息的文件夹。

---

### 13. Aider：命令行 AI 编程工具

地址：[https://github.com/Aider-AI/aider](https://github.com/Aider-AI/aider)

Aider 在命令行中运行，可以配合不同的 AI 模型修改代码。

适合体验：

- 让 AI 修改指定文件
- 根据需求生成代码
- 解释代码
- 自动生成提交内容
- 处理小型重构

难度：中  
适合程度：★★★★☆

如果你正在使用 Claude Code、Codex 一类工具，Aider 也值得对比体验。

---

### 14. Continue：编辑器里的 AI 编程助手

地址：[https://github.com/continuedev/continue](https://github.com/continuedev/continue)

Continue 可以安装到 VS Code 或 JetBrains 等编辑器中，提供：

- 代码补全
- 代码解释
- 修改代码
- 对话式编程
- 连接本地模型
- 连接远程 AI 服务

难度：低到中  
适合程度：★★★★☆

它适合想把本地 Ollama 模型接入 VS Code 的情况。

---

### 15. Gitea：自己搭建 Git 代码仓库

地址：[https://github.com/go-gitea/gitea](https://github.com/go-gitea/gitea)

Gitea 是轻量级的代码托管平台，可以理解为自己部署的 GitHub。

支持：

- 创建代码仓库
- 管理分支
- 提交代码
- Issues
- Wiki
- Pull Request
- 用户权限管理

难度：中  
适合程度：★★★★☆

适合用来了解公司内部代码平台是怎么运行的。

---

## 四、适合做创意和网页实验

### 16. Excalidraw：手绘风格在线白板

地址：[https://github.com/excalidraw/excalidraw](https://github.com/excalidraw/excalidraw)

可以自己部署一个在线白板，用来画：

- 流程图
- 系统架构图
- 思维导图
- 页面草图
- 项目方案图

难度：低到中  
适合程度：★★★★☆

---

### 17. Penpot：开源 UI 设计工具

地址：[https://github.com/penpot/penpot](https://github.com/penpot/penpot)

Penpot 类似 Figma，是一个可以自己部署的网页设计工具。

可以用来：

- 设计网页
- 设计 APP 界面
- 制作原型
- 管理组件
- 协作设计

难度：中  
适合程度：★★★★☆

---

### 18. Hoppscotch：网页端 API 调试工具

地址：[https://github.com/hoppscotch/hoppscotch](https://github.com/hoppscotch/hoppscotch)

它类似 Postman，可以用来调试：

- GET 请求
- POST 请求
- REST API
- GraphQL
- WebSocket
- 请求头和 Token

难度：低  
适合程度：★★★★☆

如果你以后要测试接口，这个项目很值得安装。

## 我建议你的体验顺序

根据你前面在使用 Claude、Obsidian 和公司导航网站，我建议按这个顺序尝试：

1. `LocalSend`：最容易成功，先熟悉 GitHub 项目的下载和安装方式。
2. `Ollama`：体验本地 AI。
3. `Open WebUI`：给本地 AI 添加网页界面。
4. `AnythingLLM`：让 AI 读取自己的文档。
5. `n8n`：体验自动化工作流。
6. `Stirling-PDF`：部署一个真正能长期使用的工具。
7. `ComfyUI`：体验 AI 图片生成。
8. `OpenHands` 或 `Aider`：体验 AI 编程。

如果只选三个，我建议：

- 想玩 AI：`Ollama + Open WebUI + AnythingLLM`
- 想玩自动化：`n8n`
- 想玩图片生成：`ComfyUI`
- 想先体验简单项目：`LocalSend`
- 想部署实用工具：`Stirling-PDF`