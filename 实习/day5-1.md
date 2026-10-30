收到！结合你提供的 history 截图，整个过程充满了“强行突破兼容性”的技术含金量。  
  
这份为你量身定制的《HikvisionOS 适配 Realtek RTL8127 万兆网卡及网络/SSH配置全记录》已经整理完毕。你可以直接复制到 Word 或 Markdown 笔记软件中作为项目资产保存。  
  
---  
  
📝 项目笔记：HikvisionOS 适配 RTL8127 万兆网卡实战  
  
一、 项目背景与硬件环境  
  
· 设备：海康威视微型服务器（HIKOS）  
· 系统：HikvisionOS（基于 openEuler，内核版本 5.10.0-216.0.0.115.oe2203sp4.x86_64）  
· 硬件：  
  · 有线网卡：Realtek RTL8127 10GbE (10ec:8127) × 2  
  · 无线网卡：Intel Wi-Fi 7 BE200 (8086:272b) × 1  
· 问题现象：ip a 命令仅显示 lo 回环接口，物理网卡未被系统识别。  
· 原因分析：5.10 内核发布较早，未原生包含 RTL8127 驱动；且 Intel BE200 在 5.10 内核下缺乏核心代码支持（仅有固件无用）。  
  
---  
  
二、 准备工作与文件传输  
  
1. 确认编译环境已具备  
  
由于系统无法连外网 YUM 源，首先盲测系统是否自带环境：  
  
```bash  
gcc -v       # 确认 gcc 已安装 (版本 10.3.1)  
make -v      # 确认 make 已安装  
ls /lib/modules/$(uname -r)/build  # 确认内核头文件目录存在  
```  
  
(若缺失，必须通过离线 RPM 包或手机 USB 共享网络解决)  
  
2. 挂载 U 盘并拷贝源码  
  
将官方驱动包 r8127-11.016.00-centos-kernel5.10.tar.gz 拷入系统。  
  
```bash  
# 查看磁盘设备，确认 U 盘分区（通常为 sda1 或 sdb1）  
lsblk  
  
# 创建挂载点并挂载（注意 Ventoy 盘的数据分区）  
mkdir -p /mnt/usb  
mount /dev/sda2 /mnt/usb  
  
# 拷贝源码到 /opt 目录并解压  
cp /mnt/usb/test/r8127-*.tar.gz /opt/  
cd /opt  
tar vxzf r8127-11.016.00-centos-kernel5.10.tar.gz  
cd /opt/r8127-11.016.00  
```  
  
---  
  
三、 核心排错与驱动编译（踩坑实录）  
  
在 5.10 老内核上编译新网卡，遇到了 3 个致命报错，以下是解决记录：  
  
❌ 报错 1：函数名冲突 (dev_page_is_reusable)  
  
原因：驱动源码中私有定义的函数与 5.10 内核头文件中的标准函数同名但类型不同。  
解决：全局替换所有源码文件中的该函数名。  
  
```bash  
find src -type f -exec sed -i 's/dev_page_is_reusable/rtl8127_page_is_reusable/g' {} +  
```  
  
❌ 报错 2：严格编译模式中断 (-Werror=incompatible-pointer-types)  
  
原因：GCC 高版本将指针类型不兼容和 C90 语法规范视为致命错误。  
解决：彻底清理 Makefile 中的 -Werror。  
  
```bash  
# 暴力替换所有严格报错选项为空  
sed -i 's/-Werror[^ ]*//g' src/Makefile  
```  
  
❌ 报错 3：DASH 远程管理模块结构体缺失  
  
原因：老内核缺少 DASH 相关的结构体成员变量。  
解决：在编译时直接传入 KCFLAGS 强行降级警告，绕过严格检查，并不编译 DASH 模块。  
  
✅ 最终编译与安装  
  
```bash  
sudo make clean  
# 核心命令：通过 KCFLAGS 传入忽略指针和声明顺序的警告  
sudo make modules KCFLAGS="-Wno-error=incompatible-pointer-types -Wno-error=declaration-after-statement"  
  
# 编译成功后，安装并加载  
sudo make install  
sudo depmod -a  
sudo modprobe r8127  
  
# 验证是否出现 enp172s0 和 enp173s0  
ip a  
```  
  
---  
  
四、 网络配置：iptools.sh 使用指南  
  
海康系统自带了专用网络配置工具，交互式配置非常直观。  
  
1. 运行工具  
  
```bash  
sudo iptools.sh  
```  
  
2. 交互式配置详解（按实际需求填写）  
  
1. 选择网卡：  
   · 屏幕提示 enp172s0 对应 0，enp173s0 对应 1。  
   · 输入：0 （代表选中第一个万兆网口）。  
2. 输入网段与掩码：  
   · 提示：Please enter the network address and mask  
   · 输入：172.30.240.0/24 （注意必须是网段 .0/24，不是网关 IP）。  
   · 作用：工具会自动扫描该网段，防止 IP 冲突。  
3. 输入目标 IP 地址：  
   · 提示：Please enter the IP address:  
   · 输入：172.30.240.150 （你为设备规划的静态 IP）。系统提示 is available 代表可用。  
4. 输入子网掩码：  
   · 提示：Please enter the mask  
   · 输入：255.255.255.0  
5. 配置网关：  
   · 提示：Whether you need to configure the gateway... [y/n]  
   · 输入：y，然后输入网关 IP，例如 172.30.240.1。  
6. 配置 DNS：  
   · 提示：Whether you need to configure the DNS... [y/n]  
   · 输入：y，然后输入公共 DNS，例如 114.114.114.114。  
7. 配置主机名 (HOSTNAME)：  
   · 提示：Whether you need to configure the HOSTNAME... [y/n]  
   · 输入：n （通常不需要修改）。  
8. 确认并重启网卡。  
  
3. 设置开机自动加载驱动（必做）  
  
避免重启后网卡丢失：  
  
```bash  
echo "r8127" | sudo tee /etc/modules-load.d/r8127.conf  
```  
  
---  
  
五、 SSH 远程管理工具的使用与避坑  
  
海康系统为了安全，对 SSH 做了定制化修改。  
  
1. 连接参数  
  
· 端口：默认端口从 22 改为了 55555。  
· 账号限制：默认禁止直接用 root 登录，只允许使用出厂默认账户 hik。  
  
2. 正确连接方法  
  
```bash  
# 在局域网其他电脑上连接  
ssh -p 55555 hik@172.30.240.150  
```  
  
3. 登入后的操作  
  
· 使用 hik 账户登录后，提示符为 $。  
· 需要提权执行命令时，输入：  
  ```bash  
  sudo su -  
  ```  
· 输入 hik 账户的密码，提示符变为 #，即获得 Root 权限。  
  
4. 退出终端  
  
· 不要使用 quit，会报错。  
· 正确退出命令：  
  ```bash  
  exit  
  ```  
  
5. 可选：开放 Root 直接远程登录  
  
如果你觉得每次用 hik 麻烦，修改配置文件：  
  
```bash  
# 编辑配置文件  
vi /etc/ssh/sshd_config  
# 找到 PermitRootLogin 改为 yes  
# 重启服务  
systemctl restart sshd  
```  
  
---  
  
💡 项目复盘总结  
  
1. 驱动移植的暴力美学：面对跨版本内核编译，千万别被满屏红字吓到。核心逻辑就是：找冲突函数（全局改名） -> 找严格限制（去掉 -Werror） -> 强制传参（KCFLAGS）。  
2. 善用官方工具：海康的 iptools.sh 非常好用，比手动改 ifcfg-enp... 文件安全且高效。  
3. 安全机制：定制化系统通常会在 SSH 端口和默认账号上留坑，需要仔细查看提示符（$ 与 #）并善用 sudo。