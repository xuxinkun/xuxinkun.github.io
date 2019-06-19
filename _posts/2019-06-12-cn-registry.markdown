---
layout:     post
title:      "docker/kubernetes国内源/镜像源解决方式"
subtitle:   "docker/kubernetes source in cn."
date:       2019-6-11 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-build-k8s.jpg"
tags:
    - kubernetes
    - docker
---


最近在使用kubeadm时，被各种连接不上搞到崩溃。费了很多力气，基本都解决了。这里统一整理了国内的一些镜像源，apt源，kubeadm源等，以便查阅。

## 国内镜像源

Azure China提供了目前用过的质量最好的镜像源，涵盖了docker.io，gcr.io，quay.io。无论是速度还是覆盖范围，体验都极佳。而且都支持匿名拉取，也就是不需要登录。这点特别友好。azk8s.cn支持的镜像代理转换如下列表。
 
| global | proxy in China | format | example |
| ---- | ---- | ---- | ---- |
| [dockerhub](hub.docker.com) (docker.io) | [dockerhub.azk8s.cn](http://mirror.azk8s.cn/help/docker-registry-proxy-cache.html) | `dockerhub.azk8s.cn/<repo-name>/<image-name>:<version>` | `dockerhub.azk8s.cn/microsoft/azure-cli:2.0.61` `dockerhub.azk8s.cn/library/nginx:1.15` |
| gcr.io | [gcr.azk8s.cn](http://mirror.azk8s.cn/help/gcr-proxy-cache.html) | `gcr.azk8s.cn/<repo-name>/<image-name>:<version>` | `gcr.azk8s.cn/google_containers/hyperkube-amd64:v1.13.5` |
| quay.io | [quay.azk8s.cn](http://mirror.azk8s.cn/help/quay-proxy-cache.html) | `quay.azk8s.cn/<repo-name>/<image-name>:<version>` | `quay.azk8s.cn/deis/go-dev:v1.10.0` |

> Note:
`k8s.gcr.io` would redirect to `gcr.io/google-containers`, following image urls are identical:

```
k8s.gcr.io/pause-amd64:3.1
gcr.io/google_containers/pause-amd64:3.1
```

 
这里，我开发了一个小的脚本azk8spull，这个脚本可以自动根据镜像名称进行解析，转换为azure的mirror镜像源域名。并进行拉取。拉取完成后会自动进行tag重命名为原本的镜像名。该脚本已经开源在 [https://github.com/xuxinkun/littleTools#azk8spull](https://github.com/xuxinkun/littleTools#azk8spull) 上。以下是使用样例。

```
[root@node-64-216 ~]# azk8spull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1
## azk8spull try to pull image from mirror quay.azk8s.cn/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1.
0.24.1: Pulling from kubernetes-ingress-controller/nginx-ingress-controller
Digest: sha256:76861d167e4e3db18f2672fd3435396aaa898ddf4d1128375d7c93b91c59f87f
Status: Image is up to date for quay.azk8s.cn/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1
## azk8spull try to tag quay.azk8s.cn/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1 to quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1.
## azk8spull finish pulling.

[root@node-64-216 ~]# azk8spull k8s.gcr.io/pause-amd64:3.1
## azk8spull try to pull image from mirror gcr.azk8s.cn/google_containers/pause-amd64:3.1.
3.1: Pulling from google_containers/pause-amd64
Digest: sha256:59eec8837a4d942cc19a52b8c09ea75121acc38114a2c68b98983ce9356b8610
Status: Image is up to date for gcr.azk8s.cn/google_containers/pause-amd64:3.1
## azk8spull try to tag gcr.azk8s.cn/google_containers/pause-amd64:3.1 to k8s.gcr.io/pause-amd64:3.1.
## azk8spull finish pulling.
```


## debian apt源

debian的apt的国内源更换为163的源，可以显著加速。

```
mv /etc/apt/sources.list /etc/apt/sources.list.bak
# 换下源
echo "deb http://mirrors.163.com/debian/ stretch main non-free contrib
deb http://mirrors.163.com/debian/ stretch-updates main non-free contrib
deb http://mirrors.163.com/debian/ stretch-backports main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch-updates main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch-backports main non-free contrib
deb http://mirrors.163.com/debian-security/ stretch/updates main non-free contrib
deb-src http://mirrors.163.com/debian-security/ stretch/updates main non-free contrib" > /etc/apt/sources.list
```

## kubeadm源

kubeadm直接使用阿里云的源即可。速度也比较快。

```
# Debian/Ubuntu
apt-get update && apt-get install -y apt-transport-https
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl

# CentOS/RHEL/Fedora
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
```

## 参考链接

- [Docker Registry Proxy Cache 帮助 http://mirror.azure.cn/help/docker-registry-proxy-cache.html](http://mirror.azure.cn/help/docker-registry-proxy-cache.html) 
- [Azure China container registry proxy https://github.com/Azure/container-service-for-azure-china/tree/master/aks#22-container-registry-proxy](https://github.com/Azure/container-service-for-azure-china/tree/master/aks#22-container-registry-proxy)
- [k8s for China https://github.com/maguowei/kubernetes-for-china](https://github.com/maguowei/kubernetes-for-china)
- [http://mirror.azure.cn/](http://mirror.azure.cn/)