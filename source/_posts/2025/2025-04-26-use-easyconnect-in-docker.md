---
title: 使用 Docker 封印 EasyConnect
date: 2025-04-26 09:00:26
updated: 2025-04-26 09:00:26
tags: [EasyConnect]
---

在这篇文章中只介绍在 Windows 下使用 Docker 封印 EasyConnect 的操作步骤。我们会使用到 4 个软件：[Docker Desktop](https://www.docker.com/products/docker-desktop/)、[VNC Viewer](https://www.realvnc.com/en/connect/download/viewer/)、[Clash for Windows](https://archive.org/download/clash_for_windows_pkg)、[Wintun](https://www.wintun.net/) 和 1 个镜像：[docker-easyconnect](https://github.com/docker-easyconnect/docker-easyconnect)。

<!-- more -->

## 下载 docker-easyconnect 镜像

为了确定要下载的镜像版本请在浏览器地址栏输入 *https://服务端地址/por/ec_pkg.csp?platform=liunx*，请根据实际情况替换*服务端地址*部分

{% asset_img version-1.png %}

根据浏览器输出并结合 [EasyConnect 版本选择](https://github.com/docker-easyconnect/docker-easyconnect/blob/master/doc/usage.md#easyconnect-%E7%89%88%E6%9C%AC%E9%80%89%E6%8B%A9)当前我们应该选择 *7.6.3* 版本

```shell
docker pull hagb/docker-easyconnect:7.6.3
```

## 启动 docker-easyconnect 容器

根据[图形界面版 EasyConnect](https://github.com/docker-easyconnect/docker-easyconnect#%E5%9B%BE%E5%BD%A2%E7%95%8C%E9%9D%A2%E7%89%88-easyconnectx86amd64arm64mips64el-%E6%9E%B6%E6%9E%84) 的说明我们启动容器的命令如下，替换 `$HOME` 变量为 Windows 下的用户目录 *C:\Users\nuc*，`PASSWORD` 设置为 *123456*，其他保持不变

```shell
docker run --rm --device /dev/net/tun --cap-add NET_ADMIN -ti -e PASSWORD=123456 -e URLWIN=1 -v C:\Users\nuc\.ecdata:/root -p 127.0.0.1:5901:5901 -p 127.0.0.1:1080:1080 -p 127.0.0.1:8888:8888 hagb/docker-easyconnect:7.6.3
```

## 使用 VNC Viewer 登录 EasyConnect

打开 VNC Viewer 软件，在菜单栏选择 *File > New connection*

{% asset_img login-1.png %}

在弹出的对话框中，*VNC Server* 项输入在启动容器由参数 `-p 127.0.0.1:5901:5901` 设置地址和端口 *127.0.0.1:5901*；*Name* 项取一个自己容易识别的名字，比如 *Docker EasyConnect*

{% asset_img login-2.png %}

在 VNC Viewer 主界面双击刚创建的 *VNC Server*

{% asset_img login-3.png %}

在弹出的 *Encryption* 确认框中点击 *Continue* 按钮忽略未加密连接警告

{% asset_img login-4.png %}

在弹出的 *Authentication* 对话框中，*Password* 项输入启动容器时通过 `-e PASSWORD=123456` 参数设置的密码 *123456*

{% asset_img login-5.png %}

如果没什么意外将看到 Easy Connect 连接界面，输入*服务器地址*、*用户名*、*密码*进行正常的登录

{% asset_img login-6.png %}

## 使用 Clash for Windows 配置代理

打开 Clash for Windows 软件，选择 *Profilees*，点击 *<>* 按钮开始编辑

{% asset_img proxy-1.png %}

*config.yaml* 配置的 1 ~ 4 行为原始内容，5 ~ 18 行为新增内容。第 8 ~ 9 行是在启动容器时通过 `-p 127.0.0.1:1080:1080` 参数设置地址和端口；第 16 ~ 18 行 *rules* 部分请根据自己实际进行配置

{% asset_img proxy-2.png %}

```yaml
mixed-port: 7890
allow-lan: false
external-controller: 127.0.0.1:58547
secret: e0449ccf-90d3-42d7-b814-3a885827271b
proxies:
  - name: Docker EasyConnect
    type: socks5
    server: 127.0.0.1
    port: 1080
proxy-groups:
  - name: 内部网络
    type: select
    proxies:
      - Docker EasyConnect
rules:
  - DOMAIN,domain1,内部网络
  - DOMAIN,domain2,内部网络
  - IP-CIDR,xxx.yyy.0.0/16,内部网络
```

现在打开 *Proxies*，应该就可以看到前面配置的代理

{% asset_img proxy-3.png %}

为了使代理生效需要在 *General* 中打开 *System Proxy*

{% asset_img proxy-4.png %}

经过以上的配置已经可以在浏览器访问内网应用。

## TUN 模式

选择 *General*，点击 *Open Folder* 打开 *Home Directory*

{% asset_img tun-1.png %}

将 *wintun.dll* 拷贝到 *Home Directory*

{% asset_img tun-2.png %}

拷贝完成后的效果如下图所示

{% asset_img tun-3.png %}

选择 *General*，点击 *Service Mode* 右边的 *Manage* 按钮

{% asset_img tun-4.png %}

在 *Service Management* 弹框可以看到当前状态是 *Inactive*，点击 *Install* 按钮

{% asset_img tun-5.png %}

Clash for Windows 会重启一下，安装完成后的效果

{% asset_img tun-6.png %}

选择 *General*，打开 *TUN Mode*

{% asset_img tun-7.png %}

在*控制面板 > 网络和 Internet > 网络连接*可以看到新加入的 *Clash* 连接

{% asset_img tun-8.png %}

开始愉快地玩耍吧！

## 参考资料

1. [Windows 的 WSL 中运行 EasyConnect](https://blog.csdn.net/zz153417230/article/details/134491639)
2. [Windows下使用Docker封印EasyConnect](https://coast23.github.io/2025/03/01/Windows%E4%B8%8B%E4%BD%BF%E7%94%A8Docker%E5%B0%81%E5%8D%B0EasyConnect/)
3. [封印 Easyconnect](https://blog.ning.moe/posts/fuck-easyconnect/)
4. [新时代的快乐科研：WSL2+Docker+EasyConnect+Clash](https://zhuanlan.zhihu.com/p/485984805)
5. [解决EasyConnect的毒瘤行为](https://vccv.cc/article/docker-easyconnect.html)
6. [Rules 规则](https://a76yyyy.github.io/clash/zh_CN/configuration/rules.html)
7. [记一次Windows下使用docker封印Easyconnect的过程](https://zhuanlan.zhihu.com/p/679435342)
8. [Clash for Windows](https://cfw.tigr.icu/)
9. [参考配置](https://a76yyyy.github.io/clash/zh_CN/configuration/configuration-reference.html)
10. [深入理解Clash配置文件](https://vpsgongyi.com/p/2396/)
11. [解决挂着Clash的时候git操作push失败的问题](https://blog.csdn.net/Naylor_5/article/details/135648311)
12. [Github可以访问，但是git clone报错443端口连接超时](https://blog.csdn.net/caiji00001/article/details/142302401)
13. [开clash后Git依然超时的解决方法](https://www.yoojia.com/article/10074187826780876479.html)
14. [用docker封印EasyConnect并连接远程桌面和数据库](https://zhuanlan.zhihu.com/p/389894063)
15. [解决EasyConnect的毒瘤行为](https://vccv.cc/article/docker-easyconnect.html)
16. [Clash for Windows 使用 TUN 模式实现网卡级代理](https://note.chilfish.top/blogs/2023/CFW_TUN.html)
17. [Clash for Windows 优雅地使用 TUN 模式接管系统流量](https://blog.dejavu.moe/posts/cfw-tun/)
18. [最新 clash for windows 设置TUN模式实现真全局模式(有限真全局)](https://doc.miyun.app/app/clash-for-windows-tun/)
19. [TUN模式 Clash for windows 配置教程](https://dp0d.github.io/clashtun/)
