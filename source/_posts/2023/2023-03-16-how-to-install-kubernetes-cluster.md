---
title: 如何安装 Kubernetes 集群？
date: 2023-03-16 21:45:09
updated: 2023-03-16 21:45:09
tags:
---

## 集群规划

这里搭建的集群由一个控制平面节点和两个工作节点组成

服务器 IP | 角色 | 备注
--- | --- | ---
192.168.59.116 | control plane | master
192.168.59.117 | worker node | node1
192.168.59.118 | worker node | node2

选用的软件及其版本清单如下所示

软件名称 | 版本号
--- | ---
CentOS | 7.9.2009
containerd | 1.6.19
runc | 1.1.4
CNI Plugins | 1.2.0
Kubernetes | 1.26.2

<!-- more -->

## 安装和配置先决条件

### 设置服务器主机名

将 IP 为 192.168.59.116 的服务器的主机名设置为 master

```shell
hostnamectl set-hostname master
```

将 IP 为 192.168.59.117 的服务器的主机名设置为 node1

```shell
hostnamectl set-hostname node1
```

将 IP 为 192.168.59.118 的服务器的主机名设置为 node2

```shell
hostnamectl set-hostname node2
```

设置完成后重新登录服务器即可看到 CLI 的提示已经变更，比如 `[root@master ~]# `

### 设置 IP 地址与主机名关系

在集群的每一台服务器上使用下面的指令设置 IP 地址与主机名的关系

```shell
cat >> /etc/hosts << EOF
192.168.59.116 master
192.168.59.117 node1
192.168.59.118 node2
EOF
```

这样在任意一台服务器上既可以使用 IP 地址也可以使用主机名访问其他服务器

### 关闭并禁用防火墙

在集群的每一台服务器上使用下面的指令检查防火墙是否处于关闭状态

```shell
systemctl status firewalld
```

当输出结果包含 `Active: active (running)` 信息时表示防火墙处于开启状态，这是需要使用如下两条指令关闭防火墙

```shell
systemctl stop firewalld
```

防火墙关闭后再次检查防火墙的状态，此时的输出结果中包含 `Active: inactive (dead)` 信息，然后可以使用如下指令禁用防火墙

```shell
systemctl disable firewalld
```

### 关闭 Swap 分区

在集群中的每一台服务器上执行如下指令检查 Swap 分区的状态

```shell
free -m
```

这条指令会输出如下的结果

```
              total        used        free      shared  buff/cache   available
Mem:           3789         252        3274           8         262        3311
Swap:          3275           0        3275
```

可以看到此时 Swap 行的 total 和 free 两列的数据不为 0，这时可以使用如下的指令临时关闭

```shell
swapoff -a
```

关闭 Swap 后再次查看 Swap 分区的状态发现 Swap 行的 total 和 free 两列的数据均为 0，如果想要永久关闭 Swap 分区需要执行下面的指令

```shell
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

也就是把 `/etc/fstab` 文件的 `/dev/mapper/centos-swap swap                    swap    defaults        0 0` 内容注释掉

### 转发 IPv4 并让 iptables 看到桥接流量

这部分内容全部来自官方文档的[转发 IPv4 并让 iptables 看到桥接流量](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#install-and-configure-prerequisites)一节，这些指令需要在集群的所有节点上执行。

执行下述指令：

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

通过运行以下指令确认 br_netfilter 和 overlay 模块被加载：

```shell
lsmod | grep br_netfilter
lsmod | grep overlay
```

通过运行以下指令确认 net.bridge.bridge-nf-call-iptables、net.bridge.bridge-nf-call-ip6tables 和 net.ipv4.ip_forward 系统变量在你的 sysctl 配置中被设置为 1：

```shell
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

## 安装 kubeadm

这部分的内容大部分来自官方文档[安装 kubeadm](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) 一节

### 确保每个节点上 MAC 地址和 product_uuid 的唯一性

使用命令 `ip link` 或 `ifconfig -a` 来获取网络接口的 MAC 地址，下面是在三台机器上执行 `ip link` 的结果

```
# master
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:f1:88:b5 brd ff:ff:ff:ff:ff:ff
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:de:74:d5 brd ff:ff:ff:ff:ff:ff
    
# node1
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:49:a2:32 brd ff:ff:ff:ff:ff:ff
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:d0:92:d9 brd ff:ff:ff:ff:ff:ff

# node2
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:39:5d:6b brd ff:ff:ff:ff:ff:ff
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:8b:ff:fd brd ff:ff:ff:ff:ff:ff
```

可以看到三台机器的 MAC 地址均不一样


使用 `sudo cat /sys/class/dmi/id/product_uuid` 命令对 product_uuid 校验，下面是在三台机器上执行该名令后的结果

```
# master
87D9BBEC-F979-6547-97EB-D7DEE69ED38B
    
# node1
5A1DD514-90A5-914B-A1A0-6DFDE4856400

# node2
0274E21F-4A64-FE48-86FF-CC64B1F6F902
```

可以看到三台机器的 product_uuid 均不一样

### 检查网络适配器

目前集群中每台服务器均满足要求

### 检查所需端口

目前集群中每台服务器都在一个局域网内，同时防火墙已经关闭

### 安装容器运行时

集群中的每一台服务器均需要安装容器运行时，并且选择 [containerd](https://containerd.io) 作为 Kubernetes 的容器运行时，同时选择使用官方二进制包进行安装，安装参考文档在[这里](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#option-1-from-the-official-binaries)

#### Step 1: Installing containerd

```shell
wget https://github.com/containerd/containerd/releases/download/v1.6.19/containerd-1.6.19-linux-amd64.tar.gz

tar Cxzvf /usr/local containerd-1.6.19-linux-amd64.tar.gz
```

##### systemd

```shell
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

mkdir -p /usr/local/lib/systemd/system
cp containerd.service /usr/local/lib/systemd/system

systemctl daemon-reload
systemctl enable --now containerd
```

#### Step 2: Installing runc

```shell
wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64

install -m 755 runc.amd64 /usr/local/sbin/runc
```

#### Step 3: Installing CNI plugins

```shell
wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz

mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.2.0.tgz
```

#### Step 4: Customizing containerd

安装完成后生成默认配置文件

```shell
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

将 `registry.k8s.io` 替换为 `registry.cn-hangzhou.aliyuncs.com/google_containers`

```shell
sed -i 's/registry.k8s.io/registry.cn-hangzhou.aliyuncs.com\/google_containers/' /etc/containerd/config.toml
```

替换完成后使用如下指令重启 containerd

```shell
systemctl restart containerd
```

### 安装 kubeadm、kubelet 和 kubectl

在集群的每一台服务器上执行下面的指令

```shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet-1.26.2 kubeadm-1.26.2 kubectl-1.26.2 --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

### 配置 cgroup 驱动程序

参考文档在[这里](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver)，当前版本没有配置

## 使用 kubeadm 创建集群

这部分的内容大多参考官方文档[使用 kubeadm 创建集群](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm)一节的内容

### 初始化控制平面节点

下面初始化控制平面节点的指令只需要在角色为控制平面的服务器上执行

```shell
kubeadm init \
    --kubernetes-version 1.26.2 \
    --apiserver-advertise-address 192.168.59.116 \
    --control-plane-endpoint master \
    --service-cidr 10.96.0.0/12 \
    --pod-network-cidr 192.178.0.0/16 \
    --image-repository registry.aliyuncs.com/google_containers
```

`--apiserver-advertise-address` 指定 API 服务器所公布的其正在监听的 IP 地址，当前设置为当前节点的 IP 地址。

因为当前主机的 IP 地址为 `192.168.0.0/16`，为了避免和主机的 IP 地址冲突，`--pod-network-cidr` 的值设置为 `192.178.0.0/16`。因为选择的 Kubernetes 网络组件是 Calico，而它的 `cidr` 的默认值是 `192.168.0.0/16`，因此在安装 Calico 时需要修改它的 `cidr` 值为 `--pod-network-cidr` 指定的值。

`--image-repository` 指定用于拉取控制平面镜像的容器仓库，默认值是 registry.k8s.io，国内无法访问，需要替换为阿里云的镜像地址。

上面的指令执行完成后将得到类型下面的结果

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join master:6443 --token 1hb9ko.m5s9ribly9sm82t5 \
        --discovery-token-ca-cert-hash sha256:92b6b62997f6b7b6c3e5dcc5037db146da0b6bcf287f91ad9c5cb4f6745a463a \
        --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join master:6443 --token 1hb9ko.m5s9ribly9sm82t5 \
        --discovery-token-ca-cert-hash sha256:92b6b62997f6b7b6c3e5dcc5037db146da0b6bcf287f91ad9c5cb4f6745a463a
```

要使非 root 用户可以运行 kubectl，请运行以下命令， 它们也是 kubeadm init 输出的一部分：

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

或者，如果你是 root 用户，则可以运行：

```shell
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### 安装 Pod 网络附加组件

网络组建只需要在角色为控制平面的服务器上进行安装即可。

选择网络组件为 [Calico](https://www.tigera.io/project-calico)，参考官方文档 [Quickstart for Calico on Kubernetes](https://docs.tigera.io/calico/3.25/getting-started/kubernetes/quickstart) 进行安装

1. Install the Tigera Calico operator and custom resource definitions.
    ```shell
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
    ```
1. Install Calico by creating the necessary custom resource.
    因为需要修改 Calico 的 `cidr` 的值，因此不能直接执行上面的指令，需要单独下载 `custom-resources.yaml`
    ```shell
    wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml
    ```
    文件下载完成后使用下面的指令将 `cidr` 的值修改为由前面 `kubeadm init` 指令的 `--pod-network-cidr` 参数指定的值 `192.178.0.0/16`
    ```shell
    sed -i 's/192.168.0.0\/16/192.178.0.0\/16/' custom-resources.yaml
    ```
    修改完成后使用下面的指令进行安装
    ```shell
    kubectl create -f custom-resources.yaml
    ```
1. Confirm that all of the pods are running with the following command.
    ```shell
    watch kubectl get pods -n calico-system
    ```
    Wait until each pod has the STATUS of Running.
1. Remove the taints on the master so that you can schedule pods on it. (如果你希望能够在控制平面节点上调度 Pod 才需要执行这一步，参考[控制平面节点隔离](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation))
    ```shell
    kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    kubectl taint nodes --all node-role.kubernetes.io/master-
    ```
    It should return the following.
    ```shell
    node/<your-hostname> untainted
    ```
1. Confirm that you now have a node in your cluster with the following command.
    ```shell
    kubectl get nodes -o wide
    ```
    It should return something like the following.
    ```
    NAME     STATUS   ROLES           AGE    VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
    master   Ready    control-plane   9m3s   v1.26.2   192.168.59.116   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   containerd://1.6.19
    ```

### 加入节点

在集群中角色为工作节点的服务器上使用 root 用户执行下面的指令，指令的内容在执行 kubeadm init 指令的输出中

```shell
kubeadm join master:6443 --token 1hb9ko.m5s9ribly9sm82t5 \
        --discovery-token-ca-cert-hash sha256:92b6b62997f6b7b6c3e5dcc5037db146da0b6bcf287f91ad9c5cb4f6745a463a
```

执行结束后再角色为控制平面的服务器执行 `kubectl get nodes` 指令将会看到刚刚加入的两个节点

```
NAME     STATUS   ROLES           AGE     VERSION
master   Ready    control-plane   14m     v1.26.2
node1    Ready    <none>          3m26s   v1.26.2
node2    Ready    <none>          3m17s   v1.26.2
```

## 体验 Kubernetes

在集群中角色为控制平面的节点上执行下面的指令部署 Nginx 服务进行验证

```shell
kubectl create deployment nginx --image=nginx

kubectl expose deployment nginx --port=80 --type=NodePort
```

部署完成后使用下面的指令查看 Nginx 服务的状态

```shell
kubectl get pods,service
```

上面的指令将会输出类似如下的结果

```
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-748c667d99-cl9xk   1/1     Running   0          3m31s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        10h
service/nginx        NodePort    10.111.41.204   <none>        80:31772/TCP   3m22s
```

第 6 行 PORT 列显示的 `31772` 即为 Nginx 服务暴露的端口，使用集群中任一主机的 IP 加上这个端口即可访问 Nginx 服务，比如 `http://192.168.59.116:31772`。
