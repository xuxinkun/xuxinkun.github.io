---
layout:     post
title:      "使用littleTools简化docker/kubectl的命令"
subtitle:   "Simplify the docker / kubectl command with littleTools."
date:       2019-3-20 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-bay.jpg"
tags:
    - docker
    - kubernetes
---

# littleTools

littleTools是我根据日常运维时编写的一个小工具，开源在了[https://github.com/xuxinkun/littleTools](https://github.com/xuxinkun/littleTools)上。

littleTools包含一组简短命令，主要用于简化某些命令的输入。目前littleTools有docker-tools和kube-tools两部分，主要用于简化命令docker和kubectl的输入。例如，如果要进入容器，一般需要输入命令`docker exec -it xxx bash`来完成。但是使用littleTools，只需使用`dt-exec xxx`就可以实现它。

## 开发思路

littleTools主要是为简化命令而做，因此直接采用了最简便直接的shell函数进行编写，因此tab键可以帮助用户自动完成命令。

比如想要实现`dt-exec {containerid}`，则只需要获取参数，填充到`docker exec -it {containerid} bash`的命令中即可。二者效果完全一致。

```
function dt-exec()
{
	docker exec -it $1 bash
}
```

在函数的命名上采用了以下几种方式：

- dt/kt-verb: 执行某个动作
- dt/kt-verb-resource: 显示resource的相关信息
- dt/kt-verb-resourceA-by-resourceB: 根据resourceB获取resourceA

# 命令一览表

## docker-tools

主要用以简化docker的相关命令。

| 命令                | 参数          | 描述                                                     |
| ------------------- | ------------- | -------------------------------------------------------- |
| dt-exec             | {containerid} | 用bash执行到容器中。                                     |
| dt-exec-sh          | {containerid} | 用sh执行到容器中。                                       |
| dt-show-pid         | {containerid} | 显示容器的0号进程在主机上的pid。                         |
| dt-show-pid-all     | {containerid} | 显示容器的所有进程的pids。                               |
| dt-show-flavor      | {containerid} | 显示容器的cpu / memory等资源信息。                       |
| dt-show-flavor-all  | 没有          | 显示所有容器的cpu / memory之类的资源信息。               |
| dt-show-volume      | {containerid} | 显示容器绑定的在主机上的存储路径。                       |
| dt-show-volume-all  | {containerid} | 显示容器绑定的在主机上的存储路径以及在容器中绑定的路径。 |
| dt-lookup-by-pid    | {pid}         | 根据主机上的{pid}查找包含该进程的容器。                  |
| dt-lookup-by-volume | {volume path} | 根据主机上的{volume path}的路径查找绑定该路径的容器。    |

这里特别要说明的是`dt-lookup-by-pid`命令，可以执行根据主机上的某个进程pid号查找对应容器的功能，这个在实际运维中非常实用。
其工作原理是利用了容器中所有进程会使用相同的cgroup path。通过查看该进程的cgroup信息。而后遍历容器的cgroup信息，并进行比对，如果一致，说明该进程属于该容器。

## kube-tools

主要用以简化kubectl的相关命令。

| 命令                 | 参数                                     | 描述                        |
| -------------------- | ---------------------------------------- | --------------------------- |
| kt-exec              | {pod name}或{namespace} {pod name}       | 用bash执行进入pod。         |
| kt-exec-sh           | {pod name}或{namespace} {pod name}       | 用sh执行进入pod。           |
| kt-get-node          | {node name}                              | 描述节点。                  |
| kt-get-node-ready    | 没有                                     | 列出所有ready节点。         |
| kt-get-node-notready | 没有                                     | 列出所有notready的节点。    |
| kt-get-node-all      | 没有                                     | 列出所有节点。              |
| kt-get-pod           | {pod name}或{namespace} {pod name}       | 描述pod。                   |
| kt-get-pod-node      | {pod name}或{namespace} {pod name}       | 使用pod获取pod和节点信息。  |
| kt-get-pod-all       | 没有                                     | 获取所有命名空间的所有pod。 |
| kt-get-pod-by-ns     | {namespace}                              | 获取命名空间中的所有pod。   |
| kt-get-pod-by-rs     | {rs name}或{namespace} {rs name}         | 获取rs的所有pod。           |
| kt-get-pod-by-deploy | {deploy name}或{namespace} {deploy name} | 获取deploy的所有pod。       |
| kt-get-pod-by-svc    | {svc name}或{namespace} {svc name}       | 获取svc的所有pod。          |

样例这里就不重复列举了，可以参考项目的[examples.md](https://github.com/xuxinkun/littleTools/blob/master/examples.md)。