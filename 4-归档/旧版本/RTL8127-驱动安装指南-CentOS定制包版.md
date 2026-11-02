必须先装编译环境，不能直接开干！  
  
老板发给你的是 .tar.gz 源码包（里面是 C 语言代码），不是现成的安装包（如 .rpm）。系统必须有“翻译官”（编译器）和“图纸”（内核头文件），才能把这些代码翻译成系统能用的驱动模块。  
  
如果你直接传上去敲 make，大概率会报错 gcc: command not found 或者找不到 kernel-devel。  
  
以下是真实落地的最稳妥执行顺序，建议直接照着敲：  
  
第一步：先装编译环境（非常关键）  
  
在终端依次执行：  
  
```bash  
# 1. 更新一下系统自带的包（如果有外网）  
sudo yum makecache  
  
# 2. 安装必备工具和当前内核对应的头文件  
sudo yum install -y gcc make kernel-devel-$(uname -r) kernel-headers-$(uname -r) elfutils-libelf-devel  
```  
  
💡 避坑提醒：执行完后，一定要检查一下 /lib/modules/$(uname -r)/build 这个目录存不存在（ls -l /lib/modules/$(uname -r)/build）。这个目录必须指向真实存在的内核源码路径（通常在 /usr/src/kernels/$(uname -r) 下），否则后面编译绝对会失败。  
  
第二步：上传驱动并编译（开干）  
  
环境装好后，把老板发的 r8127-11.016.00-centos-kernel5.10.tar.gz 传到系统（比如 /opt/ 目录）：  
  
```bash  
cd /opt  
tar vxzf r8127-11.016.00-centos-kernel5.10.tar.gz  
cd r8127-11.016.00  
  
# 运行官方自带的自动脚本  
sudo ./autorun.sh  
```  
  
(如果中途报错，你可以尝试手动编译三连：sudo make clean -> sudo make modules -> sudo make install)  
  
第三步：屏蔽旧驱动并加载  
  
编译安装完成后，系统旧驱动 r8169 可能会捣乱，需要把它屏蔽掉：  
  
```bash  
# 屏蔽 r8169  
echo "blacklist r8169" | sudo tee /etc/modprobe.d/blacklist-r8169.conf  
  
# 卸载旧驱动（如果提示模块没找到没关系），加载新驱动  
sudo rmmod r8169 2>/dev/null  
sudo modprobe r8127  
```  
  
第四步：检查网口是否出现  
  
```bash  
ip a  
```  
  
或者  
  
```bash  
lspci -nnk | grep -A2 8127  
```  
  
如果 ip a 里出现了新的 enp* 或 eth* 接口，且 lspci 里 Kernel driver in use 显示为 r8127，恭喜你，底层的硬件驱动已经搞定了！  
  
第五步：用海康的 iptools.sh 配 IP  
  
硬件搞定后，直接敲命令：  
  
```bash  
sudo iptools.sh  
```  
  
按菜单提示交互式配置你的网口、IP、网关即可。  
  
---  
  
准备开干前，请先确认一件事：你这台 HikvisionOS 系统现在能连外网吗？（或者公司内网有配置好的本地 YUM 源吗？）  
  
· 如果能连网，直接执行第一步的 yum install 就能成功。  
· 如果完全断网，也没有本地源，那 yum 会报错，你需要用 U 盘或其他方式把 gcc、make 和内核头文件的 .rpm 包离线拷过来手动安装。