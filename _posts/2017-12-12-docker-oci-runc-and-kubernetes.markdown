---
layout:     post
title:      "docker、oci、runc以及kubernetes梳理"
subtitle:   "docker, oci runc and kubernetes."
date:       2017-12-12 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-docker-oci-runc-k8s.jpg"
tags:
    - docker
    - oci
    - runc
    - kubernetes
---


容器无疑是近年来云计算中最火热的关键词。随着docker的大热，docker、oci、runc、containerd等等名词也逐渐传播开来。这么多的名词，也容易让人混淆。本文对相关名词和其之间的联系进行一下梳理和总结，方便大家更好地理解。

## container

首先说的是**container**容器。随着docker的大热，docker的经典图标，一条鲸鱼拖着若干个集装箱的经典形象已经深入人心。docker中container的翻译是译为容器还是集装箱，中文社区做过一次小小的讨论。[讨论参见http://dockone.io/question/408](http://dockone.io/question/408)。在这次讨论中，笔者的意见是container并不是docker出现了才有的，而在之前，linux container就已经翻译为linux容器并被大家接受。而从含义来看，一开始选定把“容器”作为container的翻译，也应该是准确的。而随着docker出现，container的概念深入人心，而其与原来的linux container中的container，含义应该说是一致的。所以沿用容器的翻译，笔者认为是比较合适的。

那么何为容器。容器本质上是受到资源限制，彼此间相互隔离的若干个linux进程的集合。这是有别于基于模拟的虚拟机的。对于容器和虚拟机的区别的理解，大家可以参考《京东基础架构建设之路》中的阐释，这里不再赘述。一般来说，容器技术主要指代用于资源限制的cgroup，用于隔离的namespace，以及基础的linux kernel等。

## OCI

**Open Container Initiative**，也就是常说的**OCI**，是由多家公司共同成立的项目，并由linux基金会进行管理，致力于container runtime的标准的制定和runc的开发等工作。

所谓**container runtime**，主要负责的是容器的生命周期的管理。oci的runtime spec标准中对于容器的状态描述，以及对于容器的创建、删除、查看等操作进行了定义。

**runc**，是对于OCI标准的一个参考实现，是一个可以用于创建和运行容器的CLI(command-line interface)工具。runc直接与容器所依赖的cgroup/linux kernel等进行交互，负责为容器配置cgroup/namespace等启动容器所需的环境，创建启动容器的相关进程。

为了兼容oci标准，docker也做了架构调整。将容器运行时相关的程序从docker daemon剥离出来，形成了**containerd**。Containerd向docker提供运行容器的API，二者通过grpc进行交互。containerd最后会通过runc来实际运行容器。

![containerd](http://xuxinkun.github.io/img/docker-oci-runc-k8s/containerd.png)

## 容器引擎

容器引擎，或者说容器平台，不仅包含对于容器的生命周期的管理，还包括了对于容器生态的管理，比如对于镜像等。现在的docker、rkt以及阿里推出的pouch均可属于此范畴。

docker，笔者认为可以分为两个阶段来理解。在笔者接触docker之初，docker版本为1.2，当时的docker的主要作用是容器的生命周期管理和镜像管理，当时的docker在功能上更趋近于现在的container runtime。而后来，随着docker的发展，docker就不再局限于容器的管理，还囊括了存储(volume)、网络(net)等的管理，因此后来的docker更多的是一个容器及容器生态的管理平台。


## kubernetes与容器

kubernetes在初期版本里，就对多个容器引擎做了兼容，因此可以使用docker、rkt对容器进行管理。以docker为例，kubelet中会启动一个docker manager，通过直接调用docker的api进行容器的创建等操作。

在k8s 1.5版本之后，kubernetes推出了自己的运行时接口api--**CRI**(container runtime interface)。cri接口的推出，隔离了各个容器引擎之间的差异，而通过统一的接口与各个容器引擎之间进行互动。

与oci不同，cri与kubernetes的概念更加贴合，并紧密绑定。cri不仅定义了容器的生命周期的管理，还引入了k8s中pod的概念，并定义了管理pod的生命周期。在kubernetes中，pod是由一组进行了资源限制的，在隔离环境中的容器组成。而这个隔离环境，称之为**PodSandbox**。在cri开始之初，主要是支持docker和rkt两种。其中kubelet是通过cri接口，调用docker-shim，并进一步调用docker api实现的。

如上文所述，docker独立出来了containerd。kubernetes也顺应潮流，孵化了**cri-containerd**项目，用以将containerd接入到cri的标准中。

![cri-containerd](http://xuxinkun.github.io/img/docker-oci-runc-k8s/cri-containerd.png)

为了进一步与oci进行兼容，kubernetes还孵化了**cri-o**，成为了架设在cri和oci之间的一座桥梁。通过这种方式，可以方便更多符合oci标准的容器运行时，接入kubernetes进行集成使用。可以预见到，通过cri-o，kubernetes在使用的兼容性和广泛性上将会得到进一步加强。

![kubelet](http://xuxinkun.github.io/img/docker-oci-runc-k8s/kubelet.png)



## 参考资料

- opencontainers: https://www.opencontainers.org
- oci runtime spec: https://github.com/opencontainers/runtime-spec
- runc: https://github.com/opencontainers/runc
- cri: https://github.com/kubernetes/community/blob/master/contributors/devel/container-runtime-interface.md
- container-runtime-interface-cri-in-kubernetes: http://blog.kubernetes.io/2016/12/container-runtime-interface-cri-in-kubernetes.html
- cri-containerd: https://github.com/kubernetes-incubator/cri-containerd
- cri-o https://github.com/kubernetes-incubator/cri-o

