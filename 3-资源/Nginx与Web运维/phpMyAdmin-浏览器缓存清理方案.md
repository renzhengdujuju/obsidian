**方案一：只清理 php.test.com，最推荐**

保持页面打开，按 `F12`，进入：

```
应用程序 / Application
  -> 存储 / Storage
```

找到：

```
清除网站数据 / Clear site data
```

勾选：

```
Cookie 和其他网站数据
缓存的图片和文件
服务工作者
```

然后点击：

```
清除网站数据
```

也可以进入：

```
应用程序 -> 存储 -> Cookie -> http://php.test.com
```

删除这些 Cookie：

```
phpMyAdmin
pmaAuth-1
pmaUser-1
pma_lang
```

关闭 Edge，再重新打开访问。

**方案二：Edge 设置中只删除该站点**

在地址栏打开：

```
edge://settings/content/all
```

搜索：

```
php.test.com
```

点击 `php.test.com`，再点击删除或垃圾桶图标。

删除后按 `Ctrl + F5` 强制刷新。

**方案三：完整清理一次缓存和 Cookie**

按：

```
Ctrl + Shift + Delete
```

时间范围选择：

```
所有时间
```

至少勾选：

```
Cookie 及其他站点数据
缓存的图片和文件
```

点击“立即清除”。

这不是必须的，但可以排除所有旧缓存和认证 Cookie。

**方案四：关闭安全 DNS**

本地 `.test.com` 域名实验很容易受安全 DNS 影响。

打开：

```
edge://settings/privacy
```

找到：

```
使用安全 DNS
```

选择：

```
关闭
```

然后刷新 DNS 缓存：

```
ipconfig /flushdns
```

**方案五：禁用扩展检查**

你的启动器使用了：

```
--disable-extensions
```

说明干净配置下可能没有扩展影响。可以在 Edge 打开：

```
edge://extensions
```

先把所有扩展关闭，再测试 `php.test.com`。如果正常，说明是某个扩展修改了代理或网络请求。

**方案六：检查代理**

打开：

```
edge://settings/system
```

点击：

```
打开计算机的代理设置
```

确保没有启用 HTTP 代理、SOCKS 代理或 PAC 脚本。

当前系统层面的代理已经是直连，但 Edge 扩展仍然可以单独修改请求。

**推荐执行顺序**

```
1. DevTools 清除 php.test.com 站点数据
2. Ctrl + F5 强制刷新
3. 仍然失败则关闭安全 DNS
4. 执行 ipconfig /flushdns
5. 禁用所有 Edge 扩展
6. 再失败再使用 Ctrl+Shift+Delete 全量清理
```

你现有的 `open-phpmyadmin.cmd` 之所以正常，是因为它：

```
使用独立 Edge 配置
禁用扩展
强制直连
强制域名解析到 192.168.75.10
使用随机参数绕过缓存
```

所以默认 Edge 的根本原因很可能在它自己的配置文件里，不在三台虚拟机。先做站点级清理，再关闭安全 DNS 和扩展，通常就能恢复。

```
  Worked for 20s • 17:56
```