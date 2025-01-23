---
title: 在 VirtualBox 中安装 Debian 系统
date: 2025-01-01 17:13:13
updated: 2025-01-01 17:13:13
tags: [Linux, Debian]
---

本文是在 Windows 的 VirtualBox 上安装 Debian 11.4.0 系统。安装完成后会创建两个用户

| 用户 | 密码 | 备注 |
| --- | --- | --- |
| root | 123456 | 超级用户，权限最大 |
| acomma | 123456 | 普通用户，日常使用 |

<!-- more -->

## 创建虚拟机

双击桌面上的 *Oracle VM VirtualBox* 图标打开 *Oracle VM VirtualBox 管理器*

{% asset_img 1.png %}

点击*新建*按钮打开*新建虚拟电脑*对话框

{% asset_img 2.png %}

点击*专家模式*按钮，打开专家模式对话框

{% asset_img 3.png %}

依次填写*名称*、*文件夹*、*类型*、*版本*、*内存大小*、*虚拟硬盘*，然后点击*创建*按钮打开*创建虚拟硬盘*对话框

{% asset_img 4.png %}

依次填写*文件大小*、*虚拟硬盘文件类型*、*存储在物理硬盘上*，然后点击*创建*按钮完成虚拟机的创建过程。

## 设置虚拟机

选择刚刚创建的虚拟机 *debian*，然后点击*设置*按钮

{% asset_img 5.png %}

设置*处理器*个数

{% asset_img 6.png %}

选择*存储*

{% asset_img 7.png %}

点击*分配光驱*右边的的光盘图标，在弹出的下拉框选择*选择虚拟盘*选项

{% asset_img 8.png %}

在弹出的*请选择一个虚拟光盘文件*对话框中选择下载好的 *debian-11.4.0-amd64-netinst.iso* 文件，然后点击*打开*按钮完成选择

{% asset_img 9.png %}

设置网络，这里选择*桥接网卡*的方式，然后点击 *OK* 按钮完成虚拟机的设置过程

{% asset_img 10.png %}

## 安装虚拟机

在 *Oracle VM VirtualBox 管理器*选中刚刚设置的虚拟机，然后点击*启动*按钮进入安装过程

{% asset_img 11.png %}

在 *Debian GNU/Linux installer menu* 界面使用键盘的*上下选择键*选择 *Advanced options* 选项

{% asset_img 12.png %}

敲击*回车键*进入进入 *Advanced options* 界面，选择 *Expert install* 选项

{% asset_img 13.png %}

### 选择语言

在 *Debian installer main menu* 选择 *Choose language* 选项

{% asset_img 14.png %}

在 *Select a language* 界面选择*中文（简体）*

{% asset_img 15.png %}

在*请选择你的区域*界面选择*中国*

{% asset_img 16.png %}

在*配置区域*界面选择*中国*，设置默认区域

{% asset_img 17.png %}

在*配置区域*界面中不做任何选择，直接选择*继续*按钮，不设置附加区域

{% asset_img 18.png %}

Access software for a blind person using a braille display 没做什么操作，直接跳过

{% asset_img 19.png %}

### 配置键盘

在 *Debian 安装程序主菜单*选择*配置键盘*

{% asset_img 20.png %}

在*配置键盘*页面选择*汉语*

{% asset_img 21.png %}

### 检测并挂载安装介质

在 *Debian 安装程序主菜单*选择*检测并挂载安装介质*

{% asset_img 22.png %}

在*检测并挂载安装介质*保持默认并点击*继续*按钮

{% asset_img 23.png %}

在*探测到安装介质*界面点击*继续*按钮

{% asset_img 24.png %}

### 从安装介质中加载安装程序的组件

在 *Debian 安装程序主菜单*选择*从安装介质中加载安装程序的组件*

{% asset_img 25.png %}

在*从安装介质中加载安装程序的组件*界面把所有的选项都勾选上，然后点击*继续*按钮

{% asset_img 26.png %}

等待*正在加载额外组件*完成

{% asset_img 27.png %}

### 配置语音合成器的嗓音

在 *Debian 安装程序主菜单*选择*配置语音合成器的嗓音*，这一步没有操作会直接跳过

{% asset_img 28.png %}

### 从可移动介质中加载驱动程序

在 *Debian 安装程序主菜单*选择*从可移动介质中加载驱动程序*

{% asset_img 29.png %}

在*从可移动介质中加载驱动程序*界面选择*否*

{% asset_img 30.png %}

### 探测来自硬件制造上的虚拟驱动盘

在 *Debian 安装程序主菜单*选择*探测来自硬件制造上的虚拟驱动盘*，这一步没有什么操作，会直接跳过

{% asset_img 31.png %}

### 探测网络硬件

在 *Debian 安装程序主菜单*选择*探测网络硬件*

{% asset_img 32.png %}

这一步很快，没有截到探测过程的图片

### 设置并启动 PPPoE 连接

在 *Debian 安装程序主菜单*选择*设置并启动 PPPoE 连接*

{% asset_img 33.png %}

等待*在 {IFACE} 中查找设备*完成

{% asset_img 34.png %}

在*设置并启动 PPPoE 连接*界面选择*继续*按钮

{% asset_img 35.png %}

仍然选择*继续*按钮

{% asset_img 36.png %}

### 配置网络

在 *Debian 安装程序主菜单*选择*配置网络*

{% asset_img 37.png %}

选择*是*

{% asset_img 38.png %}

时间默认为 3 秒，点击*继续*按钮

{% asset_img 39.png %}

等待配置完成

{% asset_img 40.png %}

填写主机名，然后点击*继续*按钮

{% asset_img 41.png %}

填写域名，然后点击*继续*按钮

{% asset_img 42.png %}

### 使用 SSH 继续远程安装

在 *Debian 安装程序主菜单*选择*使用 SSH 继续远程安装*

{% asset_img 43.png %}

设置远程安装密码，然后点击*继续*按钮

{% asset_img 44.png %}

再次输入远程安装密码，然后点击*继续*按钮

{% asset_img 45.png %}

点击*继续*按钮

{% asset_img 46.png %}

### 选择 Debian 镜像仓库

在 *Debian 安装程序主菜单*选择*选择 Debian 镜像仓库*

{% asset_img 47.png %}

选择 *http* 协议

{% asset_img 48.png %}

选择镜像所在的国家*中国*

{% asset_img 49.png %}

选择中科大镜像源

{% asset_img 50.png %}

HTTP 代理信息留空，点击*继续*按钮

{% asset_img 51.png %}

### 设置用户名和密码

在 *Debian 安装程序主菜单*选择*设置用户名和密码*

{% asset_img 52.png %}

启用影子口令

{% asset_img 53.png %}

允许根用户登录

{% asset_img 54.png %}

设置根用户密码

{% asset_img 55.png %}

再次输入根用户密码

{% asset_img 56.png %}

继续创建普通用户账号

{% asset_img 57.png %}

设置新用户全名

{% asset_img 58.png %}

设置账号的用户名

{% asset_img 59.png %}

设置账号的密码

{% asset_img 60.png %}

再次输入账号的密码

{% asset_img 61.png %}

### Free memory (low memory install)

在 *Debian 安装程序主菜单*选择 *Free memory (low memory install)*

{% asset_img 62.png %}

这一步没什么操作

### 配置时钟

在 *Debian 安装程序主菜单*选择*配置时钟*

{% asset_img 63.png %}

使用 NTP 设置时钟

{% asset_img 64.png %}

选择 NTP 服务器

{% asset_img 65.png %}

选择时区

{% asset_img 66.png %}

### 探测磁盘

在 *Debian 安装程序主菜单*选择*探测磁盘*

{% asset_img 67.png %}

这一步比较快，没有截到图

### 磁盘分区

在 *Debian 安装程序主菜单*选择*磁盘分区*

{% asset_img 68.png %}

选择分区方法

{% asset_img 69.png %}

选择要分区的磁盘

{% asset_img 70.png %}

选择分区方案

{% asset_img 71.png %}

确认分区

{% asset_img 72.png %}

开始分区

{% asset_img 73.png %}

### 安装基本系统

在 *Debian 安装程序主菜单*选择*安装基本系统*

{% asset_img 74.png %}

等待安装完成

{% asset_img 75.png %}

在安装的过程中需要选择内核

{% asset_img 76.png %}

选择将包含在 initrd 里的驱动程序

{% asset_img 77.png %}

### 配置软件包管理器

在 *Debian 安装程序主菜单*选择*配置软件包管理器*

{% asset_img 78.png %}

不需要扫描额外的安装介质

{% asset_img 79.png %}

使用网络镜像

{% asset_img 80.png %}

设置文件下载协议

{% asset_img 81.png %}

选择仓库镜像所在的国家

{% asset_img 82.png %}

选择仓库镜像地址

{% asset_img 83.png %}

不设置 HTTP 代理信息

{% asset_img 84.png %}

使用 no-free 软件

{% asset_img 85.png %}

启用 APT 源代码软件源

{% asset_img 86.png %}

去掉要使用的服务中的所有的勾选，即不使用

{% asset_img 87.png %}

### 选择并安装软件

在 *Debian 安装程序主菜单*选择*选择并安装软件*

{% asset_img 88.png %}

选择系统的更新管理方式

{% asset_img 89.png %}

不参加软件包流行度调查

{% asset_img 90.png %}

选择要安装的软件，只选择*标准系统工具*

{% asset_img 91.png %}

{% asset_img 92.png %}

### 安装 GRUB 启动引导

在 *Debian 安装程序主菜单*选择*安装 GRUB 启动引导*

{% asset_img 93.png %}

将 GRUB 启动器安装到主驱动器

{% asset_img 94.png %}

选择安装启动引导器的设备

{% asset_img 95.png %}

不强制将 GRUB 安装到 EFI 可移动介质

{% asset_img 96.png %}

### 结束安装进程

在 *Debian 安装程序主菜单*选择*结束安装进程*

{% asset_img 97.png %}

等待结束安装进程

{% asset_img 98.png %}

设置系统时钟为 UTC

{% asset_img 99.png %}

重启系统完成安装

{% asset_img 100.png %}
