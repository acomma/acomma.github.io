---
title: 在 VirtualBox 中安装 Debian 系统（第二版）
date: 2025-04-18 23:41:06
updated: 2025-04-18 23:41:06
tags: [Linux, Debian]
---

在 VirtualBox 上安装 Debian 系统还是比较简单的，让人感到有点困难的是 VirtualBox 的网络配置。以下是 [Virtual Networking](https://www.virtualbox.org/manual/topics/networkingdetails.html) 关于网络模式的一个概览

{% asset_img networking-modes.png %}

比较省事儿的做法是使用桥接模式，这会在宿主机所在的网络加入一台电脑，在有的情况下这不是我们期望的，比如在公司网络中。我们的需求有两点，一是虚拟电脑能够访问外部网络，二是宿主机和虚拟电脑能够相互访问。从上面的网络模式概览中发现 NAT 和 Host-Only 模式的组合正好可以满足我们的需求。

在这篇文章中我们将从零开始在 Windows 上安装 VirtualBox，然后在 VirtualBox 上安装 Debian 系统。安装完成后会创建两个用户

用户 | 密码 | 备注
--- | --- | ---
root | 123456 | 超级用户，权限最大
acomma | 123456 | 普通用户，日常使用

下面让我们开始整个安装与配置过程。

<!-- more -->

## 安装 VirtualBox

从 [VirtualBox](https://www.virtualbox.org/) 网站上下载 [VirtualBox-7.1.8-168469-Win.exe](https://download.virtualbox.org/virtualbox/7.1.8/VirtualBox-7.1.8-168469-Win.exe)，下载完成后双击可执行文件开始安装。安装的过程很简单，一直下一步就行，下面是每一步的截图以供参考

{% asset_img vbox-1.png %}

{% asset_img vbox-2.png %}

{% asset_img vbox-3.png %}

{% asset_img vbox-4.png %}

{% asset_img vbox-5.png %}

{% asset_img vbox-6.png %}

{% asset_img vbox-7.png %}

{% asset_img vbox-8.png %}

{% asset_img vbox-9.png %}

## 配置 VirtualBox

### 切换到欢迎页面

安装完成后首次启动 VirtualBox 显示的是*活动*页面

{% asset_img vbox-10.png %}

点击工具旁边的按钮，在弹出的选项中点击*欢迎*

{% asset_img vbox-11.png %}

现在就可以看到熟悉的*欢迎*页面了

{% asset_img vbox-12.png %}

### 设置虚拟电脑位置

点击*全局设定*按钮

{% asset_img vbox-13.png %}

在弹出的对话框选择 *Basic > 常规*，设置默认虚拟电脑位置为 *D:\VirtualBox VMs*

{% asset_img vbox-14.png %}

## 新建虚拟电脑

从 [Debian](https://www.debian.org/) 网站下载安装镜像 [debian-12.10.0-amd64-DVD-1.iso](https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd/debian-12.10.0-amd64-DVD-1.iso)。

点击*新建*按钮

{% asset_img debian-1.png %}

设置*虚拟电脑名称和系统类型*，注意勾选*跳过自动安装*

{% asset_img debian-2.png %}

设置*硬件*，我们将创建一台 2C4G 的虚拟电脑

{% asset_img debian-3.png %}

设置*虚拟硬盘*

{% asset_img debian-4.png %}

## 检查网络设置

选中创建的虚拟电脑 *debian*，然后点击*设置*按钮

{% asset_img debian-5.png %}

选择 *Expert > 网络 > 网卡 1*，默认已经*启用网络连接*，*连接方式*为*网络地址转换(NAT)*，注意一下 *MAC 地址* 的值 *0800272655FE*

{% asset_img debian-6.png %}

## 安装虚拟电脑

选中虚拟电脑 *debian*，然后点击*启动*按钮

{% asset_img debian-7.png %}

在 *Debian GNU/Linux installer menu (BIOS mode)* 界面使用键盘的*方向键*选择 *Advanced options* 选项

{% asset_img debian-8.png %}

按*回车键*进入 *Advanced options* 界面，选择 *Expert install* 选项

{% asset_img debian-9.png %}

### 选择语言

在 *Debian installer main menu* 选择 *Choose language* 选项

{% asset_img debian-10.png %}

在 *Select a language* 界面选择*中文（简体）*

{% asset_img debian-11.png %}

在*请选择您的位置*界面选择*中国*

{% asset_img debian-12.png %}

在*默认区域设置所属的国家*选择*中国*

{% asset_img debian-13.png %}

在*额外的区域设置*界面中不做任何选择，直接选择*继续*按钮，不设置额外的区域

{% asset_img debian-14.png %}

*Access software for a blind person using a braille display* 没做什么操作，直接跳过

{% asset_img debian-15.png %}

### 配置键盘

在 *Debian 安装程序主菜单*选择*配置键盘*

{% asset_img debian-16.png %}

在*配置键盘*页面选择*汉语*

{% asset_img debian-17.png %}

### 探测并挂载安装介质

在 *Debian 安装程序主菜单*选择*探测并挂载安装介质*

{% asset_img debian-18.png %}

在*探测并挂载安装介质*保持默认并点击*继续*按钮

{% asset_img debian-19.png %}

等待*正在扫描安装介质*完成

{% asset_img debian-20.png %}

在*探测到安装介质*界面点击*继续*按钮

{% asset_img debian-21.png %}

### 从安装介质中加载安装程序的组件

在 *Debian 安装程序主菜单*选择*从安装介质中加载安装程序的组件*

{% asset_img debian-22.png %}

在*从安装介质中加载安装程序的组件*界面把所有的选项都勾选上，然后点击*继续*按钮

{% asset_img debian-23.png %}

等待*正在加载额外组件*完成

{% asset_img debian-24.png %}

### 配置语音合成器的嗓音

在 *Debian 安装程序主菜单*选择*配置语音合成器的嗓音*，这一步没有操作会直接跳过

{% asset_img debian-25.png %}

### 从可移动介质中加载驱动程序

在 *Debian 安装程序主菜单*选择*从可移动介质中加载驱动程序*

{% asset_img debian-26.png %}

在*从可移动介质中加载驱动程序*界面选择*否*

{% asset_img debian-27.png %}

### 探测来自硬件制造商的虚拟驱动盘

在 *Debian 安装程序主菜单*选择*探测来自硬件制造商的虚拟驱动盘*，这一步没有什么操作，会直接跳过

{% asset_img debian-28.png %}

### 探测网络硬件

在 *Debian 安装程序主菜单*选择*探测网络硬件*

{% asset_img debian-29.png %}

这一步很快，没有截到探测过程的图片

### 设置并启动 PPPoE 连接

在 *Debian 安装程序主菜单*选择*设置并启动 PPPoE 连接*

{% asset_img debian-30.png %}

等待*在 {IFACE} 中查找设备*完成

{% asset_img debian-31.png %}

在*没有可用的 PPPoE 设备*界面选择*继续*按钮

{% asset_img debian-32.png %}

在*安装步骤失败*仍然*继续*按钮

{% asset_img debian-33.png %}

### 配置网络

在 *Debian 安装程序主菜单*选择*配置网络*

{% asset_img debian-34.png %}

在*自动配置网络吗？*选择*是*

{% asset_img debian-35.png %}

设置*连接探测时的等待时间*，默认为 3 秒，点击*继续*按钮

{% asset_img debian-36.png %}

等待*正在使用 DHCP 配置网络*完成，这里还有一步*正在尝试 IPv6 自动配置*未能截到图片

{% asset_img debian-37.png %}

填写*主机名*，然后点击*继续*按钮

{% asset_img debian-38.png %}

填写*域名*，然后点击*继续*按钮

{% asset_img debian-39.png %}

### 使用 SSH 继续远程安装

在 *Debian 安装程序主菜单*选择*使用 SSH 继续远程安装*

{% asset_img debian-40.png %}

设置*远程安装密码*，比如 *123456*，然后点击*继续*按钮

{% asset_img debian-41.png %}

再次输入*远程安装密码*，然后点击*继续*按钮

{% asset_img debian-42.png %}

在*启动 SSH* 点击*继续*按钮

{% asset_img debian-43.png %}

### 选择 Debian 镜像仓库

在 *Debian 安装程序主菜单*选择*选择 Debian 镜像仓库*

{% asset_img debian-44.png %}

选择*用于下载文件的协议*，比如 *http*

{% asset_img debian-45.png %}

选择*镜像站点所在的国家*中国

{% asset_img debian-46.png %}

选择 *Debian 仓库镜像站点*

{% asset_img debian-47.png %}

*HTTP 代理信息*留空，点击*继续*按钮

{% asset_img debian-48.png %}

等待*正在检查 Debian 仓库镜像站点*完成

{% asset_img debian-49.png %}

### 设置用户名和密码

在 *Debian 安装程序主菜单*选择*设置用户名和密码*

{% asset_img debian-50.png %}

在*允许以根用户登录吗？*选择*是*

{% asset_img debian-51.png %}

设置 *Root 密码*为 *123456*

{% asset_img debian-52.png %}

再次输入 *Root 密码*

{% asset_img debian-53.png %}

在*现在就创建一个普通用户账户吗？*选择*是*

{% asset_img debian-54.png %}

在*请输入新用户的全名*输入 *acomma*

{% asset_img debian-55.png %}

在*您的账户的用户名*输入 *acomma*

{% asset_img debian-56.png %}

在*请为新用户选择一个密码*输入 *123456*

{% asset_img debian-57.png %}

再次输入新用户的密码

{% asset_img debian-58.png %}

### Free memory (low memory install)

在 *Debian 安装程序主菜单*选择 *Free memory (low memory install)*

{% asset_img debian-59.png %}

这一步没什么操作

### 配置时钟

在 *Debian 安装程序主菜单*选择*配置时钟*

{% asset_img debian-60.png %}

在*使用 NTP 设置时钟吗？*选择*是*

{% asset_img debian-61.png %}

设置*要使用的 NTP 服务器*

{% asset_img debian-62.png %}

等待*正在设置时钟*完成

{% asset_img debian-63.png %}

在*请选择您的时区*选择 *Asia/Shanghai*

{% asset_img debian-64.png %}

### 探测磁盘

在 *Debian 安装程序主菜单*选择*探测磁盘*

{% asset_img debian-65.png %}

这一步比较快，没有截到图。

### 对磁盘进行分区

在 *Debian 安装程序主菜单*选择*对磁盘进行分区*

{% asset_img debian-66.png %}

等待*正在启动分区程序*完成

{% asset_img debian-67.png %}

选择*分区方法*

{% asset_img debian-68.png %}

*请选择要分区的磁盘*

{% asset_img debian-69.png %}

选择*分区方案*

{% asset_img debian-70.png %}

*完成分区操作并将结果写入磁盘*

{% asset_img debian-71.png %}

在*将改动写入磁盘吗？*选择*是*

{% asset_img debian-72.png %}

### 安装基本系统

在 *Debian 安装程序主菜单*选择*安装基本系统*

{% asset_img debian-73.png %}

等待*正在安装基本系统*完成

{% asset_img debian-74.png %}

在安装的过程中需要选择*要安装的内核*

{% asset_img debian-75.png %}

选择*将包含在 initrd 里的驱动程序*

{% asset_img debian-76.png %}

继续等待*正在安装基本系统*完成

{% asset_img debian-77.png %}

### 配置软件包管理器

在 *Debian 安装程序主菜单*选择*配置软件包管理器*

{% asset_img debian-78.png %}

在*扫描额外的安装介质*选择*否*

{% asset_img debian-79.png %}

在*使用网络镜像站点吗？*选择*否*

{% asset_img debian-80.png %}

在*要使用的服务*取消所有选择

{% asset_img debian-81.png %}

### 选择并安装软件

在 *Debian 安装程序主菜单*选择*选择并安装软件*

{% asset_img debian-82.png %}

等待*选择并安装软件*完成

{% asset_img debian-83.png %}

选择*本系统上的更新管理*

{% asset_img debian-84.png %}

在*您要参加软件包流行度调查吗？*选择*否*

{% asset_img debian-85.png %}

在*请选择要安装的软件*选择*标准系统工具*

{% asset_img debian-86.png %}

继续等待*选择并安装软件*完成

{% asset_img debian-87.png %}

### 安装 GRUB 启动引导器

在 *Debian 安装程序主菜单*选择*安装 GRUB 启动引导器*

{% asset_img debian-88.png %}

在*自动运行 os-prober 以检测和引导其他操作系统吗？*选择*否*

{% asset_img debian-89.png %}

在*将 GRUB 启动引导器安装至您的主驱动器？*选择*是*

{% asset_img debian-90.png %}

选择*安装启动引导器的设备*

{% asset_img debian-91.png %}

等待*安装 GRUB 启动引导器*完成

{% asset_img debian-92.png %}

### 结束安装进程

在 *Debian 安装程序主菜单*选择*结束安装进程*

{% asset_img debian-93.png %}

等待*正在结束安装进程*完成

{% asset_img debian-94.png %}

在*系统时钟是否为 UTC？*选择*是*

{% asset_img debian-95.png %}

*请选择 <继续> 以重新启动*

{% asset_img debian-96.png %}

## 配置仅主机网络

等待系统重新启动完成，使用 *root* 用户登录系统，输入 `ip addr` 命令

{% asset_img net-1.png %}

可以发现 *enp0s3* 网卡的 *link/ether* 为 *08:00:27:26:55::fe*，与*检查网络设置*一节中*网卡 1 *的 *MAC 地址*一样。*enp0s3* 网卡的 *inet* 为 *10.0.2.15*，这是*网卡 1* 的 IP 地址。

在命令行输入 `ping www.baidu.com` 命令

{% asset_img net-2.png %}

从结果可以看到虚拟电脑可以正常访问外部网络，但此时我们从*宿主机*还不能连上*虚拟电脑*。现在让我们输入 `shutdown --poweroff` 命令关闭虚拟电脑，配置*仅主机(Host-Only)网络*。

### 检查仅主机网络配置

选中*工具*，然后在弹出菜单中选择*网络*

{% asset_img net-3.png %}

检查*仅主机(Host-Only)网络*的*网卡*配置

{% asset_img net-4.png %}

检查*仅主机(Host-Only)网络*的 *DHCP 服务器*配置

{% asset_img net-5.png %}

这些信息有助于我们配置 *enp0s8* 网卡的信息。

### 启用仅主机网络配置

现在让我们开始配置虚拟电脑的*仅主机网络*，选中虚拟电脑 *debian*，点击*设置*按钮

{% asset_img debian-5.png %}

在弹出的 *Settings* 对话框选择 *Expert > 网络 > 网卡 2*，勾选*启用网络连接*，*连接方式*选择*仅主机(Host-Only)网络*，*名称*选择在检查仅主机网络配置一节中的 *VirtualBox Host-Only Ethernet Adapter*，注意一下 *MAC 地址*的值 *080027B6E8C2*

{% asset_img net-6.png %}

### 配置虚拟电脑仅主机网络网卡

启动虚拟机，使用 *root* 用户登录系统，然后输入 `ip addr` 命令

{% asset_img net-7.png %}

可以看到结果中多了一块网卡 *enp0s8*，网卡的 *link/ether* 为 *08:00:27:b6:e8::c2*，与前一节仅主机网络的 *MAC 地址*一致，但是这块网卡还没有 *inet* 值。

在命令行输入 `vi /etc/network/interfaces` 命令，在文件的末尾增加 *enp0s8* 的配置

```text
# The Host-Only network interface
allow-hotplug enp0s8
auto enp0s8
iface enp0s8 inet dhcp
```

{% asset_img net-8.png %}

保存后在命令行输入 `systemctl restart networking.service` 命令重启网络，重启完成后在命令行输入 `ip addr` 命令查看网络

{% asset_img net-9.png %}

配置完成后可以发现网卡 *enp0s8* 的 *inet* 已经有值 *192.168.56.101*，这个值正是前面 *DHCP 服务器*的*最小地址*。现在我们可以在*宿主机*使用这个 IP 访问*虚拟电脑*，同时虚拟电脑也可以访问外部网络。
