---
layout:     post
title:      "使用kubeadm进行单master(single master)和高可用(HA)kubernetes集群部署"
subtitle:   "deploy k8s with kubeadm."
date:       2019-6-16 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-build-k8s.jpg"
tags:
    - kubernetes
---

## kubeadm部署k8s

使用kubeadm进行k8s的部署主要分为以下几个步骤：

- 环境预装: 主要安装docker、kubeadm等相关工具。
- 集群部署: 集群部署分为single master(单master，只有一个master节点)和高可用HA集群部署两种模式。主要部署k8s的相关组件。本文将分别做介绍。
- 网络部署: 部署网络环境。本文以flannel为例进行部署。

#### 环境预装

在所有节点上都先要做好环境的准备，这里以debian为例，整理了安装docker和kubeadm的相关命令。这个里面主要需要解决的是国内源的问题。

```
## 准备环境
swapoff -a
systemctl stop firewalld
systemctl disable firewalld

echo "net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1" > /etc/sysctl.d/k8s.conf
sysctl -p /etc/sysctl.d/k8s.conf
modprobe br_netfilter

## 更换apt源
mv /etc/apt/sources.list /etc/apt/sources.list.bak
echo "deb http://mirrors.163.com/debian/ stretch main non-free contrib
deb http://mirrors.163.com/debian/ stretch-updates main non-free contrib
deb http://mirrors.163.com/debian/ stretch-backports main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch-updates main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch-backports main non-free contrib
deb http://mirrors.163.com/debian-security/ stretch/updates main non-free contrib
deb-src http://mirrors.163.com/debian-security/ stretch/updates main non-free contrib" > /etc/apt/sources.list

## 安装docker
apt-get update
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    software-properties-common -y --force-yes
curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -

apt-key fingerprint 0EBFCD88

add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
   
apt-get update
apt-get install docker-ce containerd.io -y

## 使用阿里云安装kubeadm
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

## 关闭swap
echo "vm.swappiness=0" >> /etc/sysctl.d/k8s.conf
sysctl -p /etc/sysctl.d/k8s.conf
```

以上命令在所有节点上都需要执行下。


#### 单master集群部署

单master集群部署是以一个节点作为master节点，其他的节点作为node角色加入到集群中。

首先在master节点上，通过kubeadm进行集群的初始化。

> nodename默认会使用hostname，这里使用ip作为nodename，在查看node节点时会更加直观一些。
> 这里面基本都使用的是国内的azure提供的镜像源，镜像下载速度比较快一些

```
cat << EOF > /root/kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    pod-infra-container-image: gcr.azk8s.cn/google_containers/pause-amd64:3.1
---
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
imageRepository: gcr.azk8s.cn/google_containers
kubernetesVersion: v1.14.2
networking:
  podSubnet: 10.244.0.0/16
EOF

nodename=`ip -4 route get 8.8.8.8|head -n 1|awk -F ' ' '{print $NF}'`
echo $nodename
kubeadm init --config=kubeadm-config.yaml --apiserver-advertise-address=kubemaster.cluster.local --node-name $nodename > kubeadm.log
```

在master中查看kubeadm.log中的最后几行，可以看到其他节点加入集群的命令。

```
root@i-5i2nhchleypb9h6oofb9h5suy:~# tail -n 5 kubeadm.log 
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join kubemaster.cluster.local:6443 --token br1lyz.kgnz5d6apvtcvdlg \
--discovery-token-ca-cert-hash sha256:a8a80875b68ddd8d8e0cd794daa1b81a7893eebceca77c3d5f9074b2dc7e109b 
```

这时切换到其他的node节点，可以直接使用以下命令，加入集群。

```
nodename=`ip -4 route get 8.8.8.8|head -n 1|awk -F ' ' '{print $NF}'`
echo $nodename
kubeadm join kubemaster.cluster.local:6443 --token bbnth6.tyxpf0ec27r5y5b8 \
    --discovery-token-ca-cert-hash sha256:5a9d6d25558c0e5031d4d6a69b555f0db8dd0ac7e798b9f823f7f33352748ae6 --node-name $nodename
```

node节点全部加入后，即完成节点的部署。

#### 高可用(HA)集群部署

高可用集群目前支持两种拓扑结构，一种是etcd在master节点中部署，一种是etcd在另外的节点中进行部署。在小规模集群中，其实采用前者就可以了。本文也是采用这种方式。

高可用集群一般需要三个以上的master节点组成，以及若干个node节点组成。其中master节点需要一个负载均衡器进行流量的分发。但是在小规模集群中，额外再部署一个负载均衡器无疑会增加部署的复杂度。这里我使用了一个偷懒的方式，就是不使用真实的负载均衡器，而是使用域名。这里我有三个节点，我在每个节点的/etc/hosts中做了如下配置。这样`kubemaster.cluster.local`就可以解析到三个节点上。

```
11.62.68.3    kubemaster.cluster.local
11.62.68.4    kubemaster.cluster.local
11.62.68.5    kubemaster.cluster.local
```

> 这里为了防止连接都压到同一台上，可以在不同的节点上调整hosts中的配置顺序。

这里同前面相似，用以下命令进行集群的初始化。这里特别注意下在配置中同上面最大的区别在于加入了`controlPlaneEndpoint: "kubemaster.cluster.local:6443"`配置。实现对于集群控制面的负载均衡器的配置。

```
cat << EOF > /root/kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    pod-infra-container-image: gcr.azk8s.cn/google_containers/pause-amd64:3.1
---
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
imageRepository: gcr.azk8s.cn/google_containers
kubernetesVersion: v1.14.2
controlPlaneEndpoint: "kubemaster.cluster.local:6443"
networking:
  podSubnet: 10.244.0.0/16
EOF

nodename=`ip -4 route get 8.8.8.8|head -n 1|awk -F ' ' '{print $NF}'`
echo $nodename
kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs --node-name $nodename > kubeadm.log
```

在完成初始化后，可以查看相关日志。可以看到同单master部署模式的情况不同，高可用集群中有两个命令。

```
root@i-5i2nhchleypb9h6oofb9h5suy:~# tail -n 15 kubeadm.log 
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join kubemaster.cluster.local:6443 --token woebfo.mwj26odtpe2q0taj \
    --discovery-token-ca-cert-hash sha256:a09c6ac9ff8da73e0d5e19cea1d0df524a7e289ef7b144a6ede2b0052da87edb \
    --experimental-control-plane --certificate-key b73eba61d73c35ca4de43b9dd10a7db88023cbd767c98cc1d19489829ea0fc03

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use 
"kubeadm init phase upload-certs --experimental-upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join kubemaster.cluster.local:6443 --token woebfo.mwj26odtpe2q0taj \
    --discovery-token-ca-cert-hash sha256:a09c6ac9ff8da73e0d5e19cea1d0df524a7e289ef7b144a6ede2b0052da87edb 
```

第一个是加入master的方式。也就是说，通过第一个命令加入后，该节点将会作为master加入到集群中。这里我们使用该命令即可将节点加入到master中，作为master的节点之一。

```
nodename=`ip -4 route get 8.8.8.8|head -n 1|awk -F ' ' '{print $NF}'`
echo $nodename
kubeadm join kubemaster.cluster.local:6443 --token pcumbv.qwa7uwzr7m7cijxy \
    --discovery-token-ca-cert-hash sha256:bae4c6e70cc92cc6cba66bc96d48bf0c5a45fddf83e90b89eea519fc4bad16ac \
    --experimental-control-plane --certificate-key 33f387801d51c0a743353357b138cf4ad70fd3acaa7a6ccec9835627773f1cb7 --node-name $nodename
```

当然，第二个同单master模式就相似了。

```
nodename=`ip -4 route get 8.8.8.8|head -n 1|awk -F ' ' '{print $NF}'`
echo $nodename
kubeadm join kubemaster.cluster.local:6443 --token woebfo.mwj26odtpe2q0taj \
    --discovery-token-ca-cert-hash sha256:a09c6ac9ff8da73e0d5e19cea1d0df524a7e289ef7b144a6ede2b0052da87edb --node-name $nodename
```


#### master节点准备

在master节点上可以做一些相关的准备，方便后面使用kubectl等命令。

```
systemctl enable kubelet.service

## 为kubectl准备config
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

## 去掉master的污点，使得master节点也可以创建容器
## 该命令可选。如果不想让master执行node的角色，也可以不执行
kubectl taint nodes --all node-role.kubernetes.io/master-
```

#### 网络安装

这里使用flannel网络。

```
mkdir -p ~/k8s/
cd ~/k8s
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

## 这里使用quay.azk8s.cn进行进行加速。该命令可选。
sed -i "s/quay.io/quay.azk8s.cn/g" kube-flannel.yml

kubectl apply -f kube-flannel.yml
```


#### 测试

这里我们尝试在每个节点创建一个容器，然后进行集群部署验证。这里给了一个样例。

```
cat << EOF > /root/daemonset.yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: debian
spec:
  template:
    metadata:
      labels:
        debian: debian
    spec:
      containers:
      - name: debian
        image: dockerhub.azk8s.cn/library/debian
        command: ["sleep", "999999d"]
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
EOF

kubectl create -f /root/daemonset.yaml
```

这里我写了一个小工具可以自动进行各个节点的容器之间的网络验证，自动统计哪些节点还没有正常运行。后面我会再开源出来放到博客里。

## 清理命令

有时候安装失败或者清理环境时一些常用的命令，也整理在了下面。

#### 清理环境

该命令主要用于将kubeadm安装的本节点恢复到安装前。需要输入yes确认。该命令适用于master和node节点。

```
kubeadm reset
```

#### 清理cni0网桥

重复安装有时候会导致cni0的配置错误，可以通过删除cni0网桥，然后重新创建实现故障恢复。

```
apt-get install bridge-utils -y
ifconfig cni0 down
brctl delbr cni0
systemctl restart kubelet
```

#### 清理iptables

kubeadm清理时不会清理iptables，里面会有很多冗余的规则，可以使用该命令清理。该命令慎用。

```
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

#### 清理bridge的残余信息

在使用flannel时，有时mac信息过时没能更新，导致无法将消息转发到目标机器的flannel.1。可以在检查后，通过bridge命令清理过期的mac信息。

> flannel的具体原理和排障可以参考我之前的博客，《flannel vxlan工作基本原理及常见排障思路》。其中介绍更为详细。

```
route -n 
arp -e
bridge fdb show
bridge fdb del 76:21:60:e5:ea:0b dev flannel.1
```

## 参考资料

- [Creating a single master cluster with kubeadm https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Options for Highly Available topology https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)
- [Creating Highly Available clusters with kubeadm https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)