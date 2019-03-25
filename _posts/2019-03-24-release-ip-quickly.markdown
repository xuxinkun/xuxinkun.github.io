---
layout:     post
title:      "kubernetes容器删除时快速释放ip的方案"
subtitle:   "Quick release ip scheme when kubernetes container is deleted."
date:       2019-3-24 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-bay.jpg"
tags:
    - kubernetes
---

# 问题的来由

在kubernetes集群的生产中，经常遇到这样的一个问题，就是在应用大规模更新时，大量容器删除而后大量容器创建，创建的容器需要很长时间才能就绪。这其中一个可能的原因，就是大量容器删除释放ip过于缓慢，导致新创建的容器无法及时获取ip，从而无法及时启动。

这种情况普遍存在于ip池较小或者应用升级需要ip保持不变的情景下，具有较为普遍的意义。针对这一情况，笔者先前做过一个方案用以解决该问题，在这里与大家分享一下。

# k8s容器删除的流程

首先，我们先来梳理下容器删除的整个流程，看下为什么会释放ip缓慢。kubernetes容器的整个删除流程基本套路如下：

1. 发送删除请求到apiserver，标记容器的deletionTimestamp
2. kubelet watch到该事件，知道pod需要删除，进入删除流程
3. pod执行killPod流程
4. kill app容器
	1. 执行preStopHook
	2. 等待gracePeriod
	3. 停止app容器
5. kill pause容器
	1. 调用cni接口，停止容器网络
	2. 停止pause容器
6. 从apiserver中将pod的信息清除（真正删除掉存储在etcd的pod信息）

那么可以看到，ip的释放其实是发生在调用cni接口的时候。因此，按照常规流程，其需要等待的时间为`执行preStopHook的时间 + gracePeriod + 停止app容器的时间`。这个时间对于希望快速释放ip的情形下，是较为漫长的（一般要几十秒）。

# 加速ip资源释放的方案与利弊

如果想要加速ip资源的释放，那么方式也就是显而易见的，就是在kubernetes的现有流程汇总进行定制开发，将cni的调用提前，将其提前到kill app容器之前。

> 特别注意，这里只是将cni接口调用提前，但是不要将停止pause容器提前，否则先停止pause容器可能会导致app容器停止时会有一些问题。

则修改后的流程如下：

1. 发送删除请求到apiserver，标记容器的deletionTimestamp
2. kubelet watch到该事件，知道pod需要删除，进入删除流程
3. pod执行killPod
4. 调用cni接口，停止容器网络，释放容器ip
5. kill app容器
	1. 执行preStopHook
	2. 等待gracePeriod
	3. 停止app容器
6. kill pause容器
	1. 调用cni接口，停止容器网络（可选）
	2. 停止pause容器
7. 从apiserver中将pod的信息清除（真正删除掉存储在etcd的pod信息）

这样的话，可以在第一时间尽快释放ip，而无需等待过久。

但是这样的方案也存在着很大的弊端，就是首先摘除了容器的网络，而如果preStopHook的执行或者停止app容器时需要依赖容器网络（比如应用需要调用某个接口才能进行下线等），可能会导致流程无法进行，容器被卡住，形成僵尸容器或者形成信息错误。

**因此这个方案最好要做成可选的选项，在annotation中增加一个注解。**而后根据应用的实际情况，选择是否使用快速释放ip的策略。

