---
title: kubernetes部署高可用 
date: 2022-12-07 12:03:10
tags:
categories:
- devops
---
本文将从零开始，在干净的机器上安装 Docker、Kubernetes (使用 kubeadm)、Calico，搭建一个高可用生产级的 Kubernetes。

## 环境准备

开始部署之前，请先确定当前满足如下条件，本次集群搭建，所有机器处于同一内网网段，并且可以互相通信。请部署前一定要满足以下条件：

- 两台以上主机
- 每台主机的主机名、Mac 地址、UUID 不相同
- CentOS 7
- 每台机器最好有 2G 内存或以上
- Control-plane/Master至少 2U 或以上
- 各个主机之间网络相通
- Control-plane/Master和Worker节点分别开放端口，例如6443等

### 机器配置

本次使用找到5台centos7机器，具体如下：

|  hostname   |  ip  |    说明    |
| :---------: | :--: | :--------: |
| k8s-master1 |      | master节点 |
| k8s-master2 |      | master节点 |
| k8s-master3 |      | master节点 |
|  k8s-work1  |      |  work节点  |
|  k8s-work2  |      |  work节点  |

### 软件版本

| 软件       | 版本号                        | 说明     |
| ---------- | ----------------------------- | -------- |
| 内核       | 3.10.0-514.el7.x86_64         |          |
| centos     | CentOS Linux release 7.9.2009 |          |
| docker-ce  | 20.10.19                      | 社区版本 |
| kubernetes | 1.23.9                        | 稳定版本 |
| calico     | 2.23                          |          |

### 安装初始化

#### 系统升级

使用1.23版本的kubernetes，升级软件包和centos7系统。

```bash
# 安装wget
$ yum install wget -y

# 备份
$ mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
# 更换CentOS YUM源为阿里云yum源
$ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
# 获取阿里云epel源
$ wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
# 清理缓存并创建新的缓存
$ yum clean all && yum makecache
# 系统更新
$ yum update -y

# 查看linux内核
$ uname -r
3.10.0-514.el7.x86_64

#查看redhat版本
$ cat /etc/redhat-release
CentOS Linux release 7.9.2009 (Core)
```

#### 时间同步

centos7后使用timedatectl命令来完成时间设置。

```Bash
$ timedatectl

#设置时区
$ timedatectl set-timezone Asia/Shanghai

#设置同步ntp时间
$ timedatectl set-ntp true  
```

#### 关闭防火墙、SeLinux、swap

```bash
$ systemctl stop firewalld
$ systemctl disable firewalld

# 关闭 SeLinux
$ sed -i 's/enforcing/disabled/' /etc/selinux/config 
$ setenforce 0

#关闭swap
$ swapoff -a  
$ sed -ri 's/.*swap.*/#&/' /etc/fstab
```

## docker安装

### 配置宿主机网卡转发

```bash
$ modprobe br_netfilter

$ cat <<EOF >  /etc/sysctl.d/docker.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward=1
EOF
$ sysctl -p /etc/sysctl.d/docker.conf
```

### 安装

```bash
# 替换Docker仓库，换成阿里云的源。
$ wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#查看可安装docker版本
$ yum list docker-ce.x86_64 --showduplicates | sort -r

#安装20.10.19版本
$ yum install -y docker-ce-20.10.19 docker-ce-cli-20.10.19 containerd.io-1.6.10
```

### 配置

```bash
# 初始化docker配置文件
# exec-opts：需要将Docker 的 Cgroup Driver 修改为 systemd，不然在为Kubernetes 集群添加节点时会报如下错误
# registry-mirrors： 镜像加速地址
# insecure-registries：私有镜像库非https地址
# graph：修改默认地址
$ cat > /etc/docker/daemon.json <<EOF
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
    	"max-size": "100m"
    },
    "data-root": "/opt/docker",
    "registry-mirrors": [
    		"https://duzqzdq4.mirror.aliyuncs.com",
        "http://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.docker-cn.com"
    ],
    "insecure-registries": ["172.23.131.24:8222"]
}
EOF
```

### 启动

```bash
## 设置开机自启
$ systemctl enable docker  
# 配置文件
$ systemctl daemon-reload

## 启动docker
$ systemctl start docker 

# 重启
$ systemctl restart docker 

#查看状态
$ systemctl status docker 
```

## kubernetes部署

### 配置宿主机网卡转发

```bash
$ cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward=1
EOF
$ sysctl -p /etc/sysctl.d/k8s.conf
```

### 配置源

```bash
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 安装

```bash
#查看版本号
$ yum list kubectl --showduplicates | sort -r

#安装指定版本号
$ yum install -y kubectl-1.23.9 kubelet-1.23.9 kubeadm-1.23.9

#默认启动
$ systemctl enable kubelet
```

#### 配置

```bash
#查看k8s1.23.9 所需要的镜像
$ kubeadm config images list --kubernetes-version=v1.23.9

k8s.gcr.io/kube-apiserver:v1.23.9
k8s.gcr.io/kube-controller-manager:v1.23.9
k8s.gcr.io/kube-scheduler:v1.23.9
k8s.gcr.io/kube-proxy:v1.23.9
k8s.gcr.io/pause:3.6
k8s.gcr.io/etcd:3.5.1-0
k8s.gcr.io/coredns/coredns:v1.8.6
```

## 集群初始化

### master1节点

使用kubeadm init命令初始化

```bash
# 初始化 Control-plane/Master 节点
kubeadm init \
    --apiserver-advertise-address 0.0.0.0 \
    # API 服务器所公布的其正在监听的 IP 地址,指定“0.0.0.0”以使用默认网络接口的地址
    # 切记只可以是内网IP，不能是外网IP，如果有多网卡，可以使用此选项指定某个网卡
    --apiserver-bind-port 6443 \
    # API 服务器绑定的端口,默认 6443
    --cert-dir /etc/kubernetes/pki \
    # 保存和存储证书的路径，默认值："/etc/kubernetes/pki"
    --control-plane-endpoint kuber4s.api \
    # 为控制平面指定一个稳定的 IP 地址或 DNS 名称,
    # 这里指定的 kuber4s.api 已经在 /etc/hosts 配置解析为本机IP
    --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
    # 选择用于拉取Control-plane的镜像的容器仓库，默认值："k8s.gcr.io"
    # 因 Google被墙，这里选择国内仓库
    --kubernetes-version 1.23.9 \
    # 为Control-plane选择一个特定的 Kubernetes 版本， 默认值："stable-1"
    --node-name master01 \
    #  指定节点的名称,不指定的话为主机hostname，默认可以不指定
    --pod-network-cidr=10.244.0.0/16 \
    # 指定pod的IP地址范围
    --service-cidr=10.1.0.0/16  \
    # 指定Service的VIP地址范围
    --service-dns-domain cluster.local \
    # 为Service另外指定域名，默认"cluster.local"
    --upload-certs
    # 将 Control-plane 证书上传到 kubeadm-certs Secret
```

#### 高可用

使用 kube-vip 来实现集群的高可用，首先在 master1 节点上生成基本的 Kubernetes 静态 Pod 资源清单文件。

文档：https://kube-vip.io/docs/installation/static/

```bash
# 设置VIP地址
$ export VIP=172.23.131.100
$ export INTERFACE=eth0
$ docker pull plndr/kube-vip:v0.5.7

$ mkdir -pv /etc/kubernetes/manifests/
$ alias kube-vip="docker run --network host --rm docker.io/plndr/kube-vip:v0.5.7"
$ kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --services \
    --arp \
    --leaderElection | tee /etc/kubernetes/manifests/kube-vip.yaml
    
# 自动生成 /etc/kubernetes/manifests/kube-vip.yaml文件，kubeadm在初始化时会生成kube-vip的静态pod。
```

#### 集群初始化

```bash
$ kubeadm init \
 --apiserver-advertise-address 172.23.131.28 \
 --control-plane-endpoint 172.23.131.28:6443 \
 --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
 --kubernetes-version 1.23.9 \
 --pod-network-cidr=10.244.0.0/16 \
 --service-cidr=10.1.0.0/16 \
 --upload-certs
```

成功后显示，如果出现任何错误，可以根据提示找出错误原因。

```bash
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

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 172.23.131.28:6443 --token zi4l69.b9cf99ut3yihcc4w \
	--discovery-token-ca-cert-hash sha256:ce3c955b27a1743b5a5a48c0a010d10c8553663d74f6ca3107702715bffb04e6 \
	--control-plane --certificate-key 16edd3ee4550b63d1719b4a55ffdd3e31f74e1687e1982387adecefc1b131b3f

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.23.131.28:6443 --token zi4l69.b9cf99ut3yihcc4w \
--discovery-token-ca-cert-hash sha256:ce3c955b27a1743b5a5a48c0a010d10c8553663d74f6ca3107702715bffb04e6 
```

根据上面的提示执行命令

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 安装ipvs(放到k8s)

```bash
$ yum -y install ipvsadm ipset
```

修改kube-proxy模式

```bash
$ kubectl edit configmap -n kube-system kube-proxy

#在yaml文件找到 mode: ""，保存后退出
mode: "ipvs"

#删除kube-proxy pod
$ kubectl delete pod kube-proxy-xxx -n kube-system

#查看是否启动成功
$ kubectl get pods --all-namespaces

#ipvs是否正常
$ ipvsadm -L -n
```

### master2节点

#### 高可用

```bash
# 设置VIP地址
$ export VIP=172.23.131.100
$ export INTERFACE=eth0
$ docker pull plndr/kube-vip:v0.5.7

$ mkdir -pv /etc/kubernetes/manifests/
$ alias kube-vip="docker run --network host --rm docker.io/plndr/kube-vip:v0.5.7"
$ kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --services \
    --arp \
    --leaderElection | tee /etc/kubernetes/manifests/kube-vip.yaml
    
# 自动生成 /etc/kubernetes/manifests/kube-vip.yaml文件，kubeadm在初始化时会生成kube-vip的静态pod。
```

#### 加入master1

```bash
  $ kubeadm join 172.23.131.100:6443 --token nqxel9.ymptu99i8b585gnq \
  --discovery-token-ca-cert-hash sha256:1756048aecb4bd7c9d43e9afe604b662e2b2a0dd067e05eb616670a73cfb1d5c \
  --control-plane --certificate-key fd6dc5187fe46ab99d8af144927d6952af4ec6926b93bb3e45339aec9a050241
```

成功后可以看到以下内容

```bash
This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```

### 安装网络插件

关于 Kubernetes 网络，建议读完这篇 [文章](https://yuerblog.cc/2019/02/25/flannel-and-calico/)，以及文末的其他链接，如[这个](https://juejin.im/entry/599d33ad6fb9a0247804d430)。

集群必须安装Pod网络插件，以使Pod可以相互通信，只需要在Master节点操作，其他新加入的节点会自动创建相关pod。必须在任何应用程序之前部署网络组件。另外，在安装网络之前，CoreDNS将不会启动（你可以通过命令 `kubectl get pods --all-namespaces|grep coredns` 查看 CoreDNS 的状态）。kubeadm 支持多种网络插件，我们选择 Calico 网络插件（kubeadm 仅支持基于容器网络接口（CNI）的网络（不支持kubenet）。）

Calico最重要的是要选对版本号：https://projectcalico.docs.tigera.io/getting-started/kubernetes/requirements

安装参考：https://projectcalico.docs.tigera.io/archive/v3.23/getting-started/kubernetes/quickstart

```bash
$ kubectl create -f https://projectcalico.docs.tigera.io/archive/v3.23/manifests/tigera-operator.yaml

# 修改pod地址
$ wget https://projectcalico.docs.tigera.io/archive/v3.23/manifests/custom-resources.yaml
$ vim custom-resources.yaml
cidr: 10.244.0.0/16

#安装
$ kubectl apply -f custom-resources.yaml

#查看安装进度
$ watch kubectl get pods -n calico-system
```

### 验证

```bash
#查看所有pod是否已经running
$ kubectl get pods --all-namespaces

NAMESPACE          NAME                                         READY   STATUS    RESTARTS      AGE
calico-apiserver   calico-apiserver-78fff95c54-l9vh6            1/1     Running   0             2m17s
calico-apiserver   calico-apiserver-78fff95c54-m4tcl            1/1     Running   0             2m17s
calico-system      calico-kube-controllers-55f554875b-qlmjw     1/1     Running   0             11m
calico-system      calico-node-c8wxt                            1/1     Running   0             11m
calico-system      calico-node-hnhp6                            1/1     Running   0             11m
calico-system      calico-typha-59f95fc874-46k7m                1/1     Running   0             11m
kube-system        coredns-65c54cc984-ghfzk                     1/1     Running   0             77m
kube-system        coredns-65c54cc984-m7wbr                     1/1     Running   0             77m
kube-system        etcd-host-172-23-131-27                      1/1     Running   0             47m
kube-system        etcd-host-172-23-131-28                      1/1     Running   6 (66m ago)   77m
kube-system        kube-apiserver-host-172-23-131-27            1/1     Running   0             48m
kube-system        kube-apiserver-host-172-23-131-28            1/1     Running   6 (65m ago)   77m
kube-system        kube-controller-manager-host-172-23-131-27   1/1     Running   0             48m
kube-system        kube-controller-manager-host-172-23-131-28   1/1     Running   7 (47m ago)   77m
kube-system        kube-proxy-mk89n                             1/1     Running   0             39m
kube-system        kube-proxy-szdn4                             1/1     Running   0             39m
kube-system        kube-scheduler-host-172-23-131-27            1/1     Running   0             48m
kube-system        kube-scheduler-host-172-23-131-28            1/1     Running   8 (47m ago)   77m
tigera-operator    tigera-operator-7885599d97-vdf6z             1/1     Running   0             27m
```

### node1节点

```bash
$ kubeadm join 172.23.131.100:6443 --token 9vi77p.nkla7mjdekzwngbx \
--discovery-token-ca-cert-hash sha256:c13e578e5d18f098c74b0777bbd43f46566dea9a72950a77534ff757653bd05e 

kubeadm join 172.23.131.28:6443 --token ks62mn.ixb479nvk4yyuw1n \
--discovery-token-ca-cert-hash sha256:02f1b6fb7962f9b841017c21b846635e738504ed57a895f321e02de1a0dee8f9 

#验证是否成功
$ kubectl get nodes
```

### node2节点

```bash
$ kubeadm join 172.23.131.100:6443 --token 9vi77p.nkla7mjdekzwngbx \
--discovery-token-ca-cert-hash sha256:c13e578e5d18f098c74b0777bbd43f46566dea9a72950a77534ff757653bd05e 

#验证是否成功
$ kubectl get nodes
```



## 集群重置

### 重置

```Bash
#重置
$ kubeadm reset -f  
```

### 删除多余文件

```bash
#docker所有容器删除
$ docker rm $(docker ps -a -q)

#重置ipvs
$ ipvsadm --clear
#iptables重置
$ iptables -F

#删除原有集群文件
$ rm -rf /etc/cni /opt/cni /var/lib/cni 
$ rm -rf /var/etcd /var/lib/etcd
$ rm -rf /etc/kubernetes /var/lib/dockershim /var/lib/kubelet /var/run/kubernetes /var/run/calico ~/.kube/*

#重启docker
$ systemctl restart docker
```

