---
title: 一种在虚拟机安装奇安信产品访问内网的方法
date: 2025-08-30 13:03:02
updated: 2025-08-30 13:03:02
tags: [3proxy, Clash, VirtualBox, Qianxin]
---

在我们的场景中访问内网资源需要通过*奇安信天信*，一个零信任网络访问系统，功能类似 *EasyConnect*，同时也需要安装*奇安信天擎*和*微步*两个软件。我们都知道*奇安信天擎*等产品一旦安装想要卸载非常麻烦，而*奇安信天信*目前还没有类似 *docker-easyconnect* 的软件，经过实践把这三个软件安装在虚拟机中，然后在虚拟机中安装 *[3proxy](https://github.com/3proxy/3proxy)* 代理服务器，在主机中通过 *Clash.for.Windows* 连接代理服务器从而避免在主机中安装*奇安信天擎*等产品

{% asset_img architecture.drawio.svg %}

<!-- more -->

虚拟机和主机之间的网络选择*桥接模式*或者其他等价设置

{% asset_img network-setting.png %}

在配置安装配置 *3proxy* 时要在它的 *bin* 目录创建一个 *3proxy.cfg* 文件，使用 *socks* 方式，文件内容如下所示

```
nserver 8.8.8.8
nserver 8.8.4.4
nscache 65536
auth none
socks -p1080
```

注意配置的 *socks* 端口在 *Clash.for.Windows* 中会用到，后者的配置和使用方法参考[使用 Docker 封印 EasyConnect](https://acomma.github.io/2025/04/26/use-easyconnect-in-docker/) 一文，下面的例子在前文的基础上增加了 *VirtualBox 3proxy*，第 12 行是虚拟机的 IP 地址，第 13 行是 *3proxy* 配置的端口

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
  - name: VirtualBox 3proxy
    type: socks5
    server: 192.168.1.5
    port: 1080
proxy-groups:
  - name: 内部网络
    type: select
    proxies:
      - Docker EasyConnect
      - VirtualBox 3proxy
rules:
  - DOMAIN,domain1,内部网络
  - DOMAIN,domain2,内部网络
  - IP-CIDR,xxx.yyy.0.0/16,内部网络
```

完~
