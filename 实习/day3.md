# 第三天学习记录：将 Edge 浏览器扩展迁移到 Chrome

## 操作目标
尝试将 Edge 中已经使用的第三方扩展迁移到 Google Chrome，并记录每一步操作。

## 已完成的步骤

1. 确认了 Edge 的默认用户数据目录：
   `C:\Users\DZ-05-001\AppData\Local\Microsoft\Edge\User Data\Default\Extensions`
2. 检查了 Edge 中扩展文件夹。每个扩展通常按照“扩展 ID\版本号\manifest.json”的结构保存。
3. 确认目标扩展为 **Qshot-子弹搜索**，扩展 ID 为：
   `mhmemfgmeacjninbhlilbcaimmcallnl`
4. 确认 `manifest.json` 位于：
   `C:\Users\DZ-05-001\AppData\Local\Microsoft\Edge\User Data\Default\Extensions\mhmemfgmeacjninbhlilbcaimmcallnl\1.2.1_0\manifest.json`
5. 打开了 Chrome 的扩展管理页面：
   `chrome://extensions/`
6. 确认 Chrome 的“开发者模式”已经开启，页面上可以看到“加载已解压的扩展程序”按钮。
7. 为避免修改 Edge 原始扩展文件，已将 Qshot 复制到独立目录：
   `C:\Users\DZ-05-001\Documents\ChromeExtensions\Qshot-子弹搜索\1.2.1_0`
8. 已确认副本目录中存在 `manifest.json`。
9. 尝试点击 Chrome 的“加载已解压的扩展程序”按钮，但系统文件夹选择窗口没有成功显示，Chrome 当前仍在运行。

## 当前状态

Edge 原扩展未被修改，Chrome 原有扩展未被覆盖。目标扩展的副本已经准备好，剩余步骤是：在 Chrome 扩展管理页点击“加载已解压的扩展程序”，选择：

`C:\Users\DZ-05-001\Documents\ChromeExtensions\Qshot-子弹搜索\1.2.1_0`

选择后确认加载，Chrome 扩展列表中应出现 Qshot-子弹搜索。

## 注意事项

- `manifest.json` 是文件，Chrome 中要选择它所在的上一级文件夹，即 `1.2.1_0`。
- 不要选择 Edge 的 `Extensions` 总目录，也不要只选择扩展 ID 目录。
- 不要直接移动或修改 Edge 原始扩展目录。
- 这是本地加载方式，扩展后续更新通常需要手动替换副本并在 Chrome 扩展页点击“重新加载”。
- Qshot 具有读取网页内容等浏览器权限，使用前应确认扩展来源可信。
- Edge 与 Chrome 都基于 Chromium，但第三方扩展仍可能存在兼容性问题。
