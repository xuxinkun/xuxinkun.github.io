---
layout:     post
title:      "使用docker部署ambari的若干要点"
subtitle:   "Using docker to deploy ambari."
date:       2017-05-14 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-ambari.jpg"
tags:
    - docker
    - ambari
---

# ambari部署各个组件

使用ambari进行部署时主要需要的组件包括：

- ambari-server: 主要部署的控制节点，负责控制agent进行部署。
- mysql: server存储的数据库。也支持postgresql等数据库。
- ambari-agent: 主要执行部署的节点，根据控制节点，部署相应的服务的相应组件(compoment)。
- repo: 可以是公网的库，也可以是本地源。主要提供各个服务安装的rpm包等。ambari主要使用的是HDP(hortonworks data platform)的库。
- consul: 用于DNS解析。因为各个节点之间需要通过域名来相互进行访问。用consul来提供DNS解析服务，无需在每个节点上配置hosts。对应的，各个容器也需要将DNS(即resolve.conf)指定为部署consul的ip。

# 部署流程

每个组件都可以单独做成镜像。其中repo可以使用公网的库，也可以使用自己搭建的本地源。

HDP的版本要和ambari的版本对应。对应关系可以查看[hdp官网](https://zh.hortonworks.com/products/data-center/hdp/)。

在实验中我使用的是ambari 2.2.1-v20的镜像和HDP 2.4.3。

- [agent镜像地址](https://hub.docker.com/r/hortonworks/ambari-agent/tags/)
- [server镜像地址](https://hub.docker.com/r/hortonworks/ambari-server/tags/)
- [HDP](http://public-repo-1.hortonworks.com/HDP/centos7/2.x/updates/2.4.3.0/HDP-2.4.3.0-centos7-rpm.tar.gz)
- [HDP-UTILS](http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.20/repos/centos7/HDP-UTILS-1.1.0.20-centos7.tar.gz)

## 搭建repo库

主要是安装httpd并把HDP和HDP-UTILS的tar包解压到指定目录。这个不详述了。

## 创建server和agent容器

使用[docker-ambari](https://github.com/sequenceiq/docker-ambari)的`ambari-functions`来创建集群。

1. 修改`ambari-functions`中的server和agent镜像名称
2. `source ambari-functions`
3. 运行`amb-setttings`，查看配置是否有问题
4. 运行`amb-start-cluster 3`。启动server/agent/consul容器。
5. 此时ambari-server就正常启动了。
6. 进入ambari-server容器，`ssh-keygen -t rsa -P ''`生成密钥。
7. 进入ambari-agent，`yum install -y sudo`，`mkdir -p /var/log/ambari-agent`, `mkdir -p /var/lib/ambari-agent`。将ambari-server的公钥拷贝到`/root/.ssh/authorized_keys`文件中。
8. 从页面访问ambari-server。即可按步骤添加多个agent到集群中，并安装对应的service。

我在虚拟机上单机安装了HDFS+YARN+MAPREDUCE+SPARK服务。spark可用。我再装storm时，虚拟机配置太差，撑不住，服务无法启动。

ambari的好处是集成了监控等功能，组件很全面。

# 一些问题和待解决的点

- 集群编排问题。比如需要创建几个容器，每个容器应该是什么角色，安装什么组件，要事先规划好，再去创建。
- ambari-agent容器挂掉重启后，默认不会重新加入回集群。需要配置适当的脚本，使得
- 官方ambari-agent没有sudo，而且对应的ambari-agent的log目录等都没有创建。因此需要在官方镜像基础上再进行改造。
- 密钥的生成以及分发。
- ambari-agent的规划问题。比如agent作为datanode时，需要使用VOLUME的外挂盘来对数据进行保存，而不是使用容器本身的存储(容器本身存储仅10G，也不够用)。当然，这也可以做到容器的镜像中或者生成容器时动态挂载。
- 自动创建集群。这里主要的难点是使用ambari的api创建cluster，添加service等。还需要深入研究下。[参考api](https://cwiki.apache.org/confluence/display/AMBARI/Adding+a+New+Service+to+an+Existing+Cluster)。
- ambari-agent镜像细化的问题。现在ambari-agent中实际是一个空的镜像，没有安装service。那么我们是否可以根据service的不同，分别制作出hadoop-ambari-agent镜像，spark-ambari-agent镜像等，省去一部分服务安装的时间。