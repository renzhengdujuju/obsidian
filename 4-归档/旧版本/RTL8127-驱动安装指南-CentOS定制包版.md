# RTL8127 驱动安装指南（CentOS 定制包版）

> 这是使用 CentOS 定制包的早期版本，已被 [[RTL8127-万兆网卡驱动编译实战|官方原版包方案]] 取代。定制包 `-Werror` 绑得太死，容易卡编译，仅作参考。

## 一、先装编译环境（关键）

源码包（.tar.gz，里面是 C 语言代码）不是现成安装包（如 .rpm），系统必须有编译器（gcc）和内核头文件，才能把代码编译成驱动模块。

如果直接上传后敲 `make`，大概率会报 `gcc: command not found` 或找不到 `kernel-devel`。

在终端依次执行：

```bash
# 1. 更新系统自带的包（如果有外网）
sudo yum makecache

# 2. 安装必备工具和当前内核对应的头文件
sudo yum install -y gcc make kernel-devel-$(uname -r) kernel-headers-$(uname -r) elfutils-libelf-devel
```

> 避坑提醒：执行完后一定要检查 `/lib/modules/$(uname -r)/build` 目录是否存在（`ls -l /lib/modules/$(uname -r)/build`）。该目录必须指向真实存在的内核源码路径（通常在 `/usr/src/kernels/$(uname -r)` 下），否则后面编译必定失败。

> 如果完全断网、也没有本地 YUM 源，`yum` 会报错，需要用 U 盘或其他方式把 gcc、make 和内核头文件的 .rpm 包离线拷过来手动安装。

## 二、上传驱动并编译

环境装好后，把 `r8127-11.016.00-centos-kernel5.10.tar.gz` 传到系统（比如 `/opt/` 目录）：

```bash
cd /opt
tar vxzf r8127-11.016.00-centos-kernel5.10.tar.gz
cd r8127-11.016.00

# 运行官方自带的自动脚本
sudo ./autorun.sh
```

如果中途报错，可以尝试手动编译三连：

```bash
sudo make clean
sudo make modules
sudo make install
```

## 三、屏蔽旧驱动并加载

编译安装完成后，系统旧驱动 r8169 可能会捣乱，需要屏蔽：

```bash
# 屏蔽 r8169
echo "blacklist r8169" | sudo tee /etc/modprobe.d/blacklist-r8169.conf

# 卸载旧驱动（如果提示模块没找到没关系），加载新驱动
sudo rmmod r8169 2>/dev/null
sudo modprobe r8127
```

## 四、检查网口是否出现

```bash
ip a
```

或者：

```bash
lspci -nnk | grep -A2 8127
```

如果 `ip a` 里出现了新的 `enp*` 或 `eth*` 接口，且 `lspci` 里 `Kernel driver in use` 显示为 `r8127`，说明底层的硬件驱动已经搞定。

## 五、用海康的 iptools.sh 配 IP

硬件搞定后，直接敲命令：

```bash
sudo iptools.sh
```

按菜单提示交互式配置网口、IP、网关即可。
