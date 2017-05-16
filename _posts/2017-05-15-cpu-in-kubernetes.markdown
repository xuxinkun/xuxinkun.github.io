---
layout:     post
title:      "docker对cpu使用及在kubernetes中的应用"
subtitle:   "The way to use cpu in docker and kubernetes."
date:       2017-05-15 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-cpu-kubernetes.jpg"
tags:
    - docker
    - kubernetes
    - cpu
---

# docker对CPU的使用


docker对于CPU的可配置的主要几个参数如下：

```
  --cpu-shares                    CPU shares (relative weight)
  --cpu-period                    Limit CPU CFS (Completely Fair Scheduler) period
  --cpu-quota                     Limit CPU CFS (Completely Fair Scheduler) quota
  --cpuset-cpus                   CPUs in which to allow execution (0-3, 0,1)
```  

这些参数主要是通过配置在容器对应cgroup中，由cgroup进行实际的CPU管控。其对应的路径可以从cgroup中查看到

## cpuset-cpus

```
[root@node-156 ~]# cat /sys/fs/cgroup/cpuset/docker/12c35c978d926902c3e5f1235b89a07e69d484402ff8890f06d0944cc17f8a71/cpuset.cpus
0-31
```

`cpuset`主要用于指定容器运行的CPU编号，也就是我们所谓的绑核。

## cpushare

```
[root@node-156 ~]# cat /sys/fs/cgroup/cpu/docker/12c35c978d926902c3e5f1235b89a07e69d484402ff8890f06d0944cc17f8a71/cpu.shares
1024
```

`cpushare`主要用于cfs中调度的权重。一般来说，在条件相同的情况下，cpushare值越高的，将会分得更多的时间片。

> 两个容器的CPU时间片比重并不是严格权重的比值，因为两个容器可能对CPU时间片的需求不同。

## cpu-period和cpu-quota 

```
[root@node-156 ~]# cat /sys/fs/cgroup/cpu/docker/12c35c978d926902c3e5f1235b89a07e69d484402ff8890f06d0944cc17f8a71/cpu.cfs_quota_us 
-1
```
```
[root@node-156 ~]# cat /sys/fs/cgroup/cpu/docker/12c35c978d926902c3e5f1235b89a07e69d484402ff8890f06d0944cc17f8a71/cpu.cfs_period_us 
100000
```

`cfs_quota_us`和`cfs_period_us`两个值是联合使用的，两者的比值，即`cfs_quota_us`/`cfs_period_us`代表了该容器实际可用的做多的CPU核数。

比如`cfs_quota_us`=50000，`cfs_period_us`=100000，那么二者的比值是0.5，也就是说该容器可以使用0.5个cpu。这样的管控粒度更细，在cgroup使用systemd时最低可以到0.01核。

> `cfs_quota_us`如果为`-1`，则表示容器使用CPU不受限制。

## 绑核方式的益处和弊端

在我们先前的1.0中，主要使用的是`cpuset`，也就是通过绑核的方式。这一方式严格的保证了容器可以使用的CPU的真正的核数。并通过调度使得其他容器不绑定这几个CPU，使得容器可以独享这些cpu。这也就意味着容器的最多使用的CPU个数和最小消耗的CPU的数目都是这些核数。

这样的方式安全性高，保证容器的效率，但是弊端也很多：

- 不够灵活
- 资源利用率低，因为容器可能声明使用了多个CPU，但是实际利用率很低
- 在NUMA架构下，未考虑CPU亲和性的话，可能会导致性能下降

# kubernetes中的CPU使用

kubernetes对容器可以设置两个值：

```
spec.containers[].resources.limits.cpu
spec.containers[].resources.requests.cpu
```

limits主要用以声明使用的最大的CPU核数。通过设置`cfs_quota_us`和`cfs_period_us`。比如limits.cpu=3，则`cfs_quota_us`=300000。

>`cfs_period_us`值一般都使用默认的100000

request则主要用以声明最小的CPU核数。一方面则体现在设置`cpushare`上。比如request.cpu=3，则`cpushare`=1024*3=3072。

另一方面是提供调度时候使用。

当创建一个Pod时，Kubernetes调度程序将为Pod选择一个节点。每个节点具有每种资源类型的最大容量：可为Pods提供的CPU和内存量。调度程序确保对于每种资源类型，调度的容器的资源请求的总和小于节点的容量。尽管节点上的实际内存或CPU资源使用量非常低，但如果容量检查失败，则调度程序仍然拒绝在节点上放置Pod。

而计算节点CPU的已经分配的量就是通过计算所有容器的request的和得到的。

可以参考[Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)

## kubernetes对CPU使用的益处

- 更加灵活，更细粒度的控制。CPU的限制不仅仅在CPU核这个级别，甚至可以到0.01核。
- CPU复用。绑核之后容器既无法使用其他的CPU，容器自己本身绑定的CPU也无法被其他容器使用。最小最大资源使用量都是这几个核。而kubernetes的方式可以实现所有的CPU成为一个CPU池，提供给CPU使用。
- 可控和可靠的“超卖”
- best-effort任务支持。可以充分利用闲置的CPU资源，使得best-effort任务得到最大限度的资源支持。同时当资源紧张时，又可以优先杀死best-effort，保证Guaranteed的容器的资源使用。可以参考[Resource Quality of Service in Kubernetes](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-qos.md)。

## 可能的问题

linux对NUMA下的CPU调度是有一些优化的。可以参考[Linux 的 NUMA 技术](https://www.ibm.com/developerworks/cn/linux/l-numa/)。

numa架构下，可能还会一些其他问题需要关注，比如[NUMA架构的CPU](http://cenalulu.github.io/linux/numa/)中提的，可能会有些影响。