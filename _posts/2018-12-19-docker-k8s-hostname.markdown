---
layout:     post
title:      "docker和kubernetes中hostname的使用和常见问题"
subtitle:   "Use of hostname and common problems in docker and kubernetes."
date:       2018-12-18 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-2015.jpg"
tags:
    kubernetes
    docker
---


hostname在docker中是使用UTS namespace进行隔离的。docker中主要有两种ns的用法，

- 一种是`docker run --uts="" busybox`。这种会新创建一个新的uts ns。
- 一种是`docekr run --uts="host" busybox`。这种创建的容器将会使用物理机的uts ns。


在k8s中，是这样处理的uts的ns的：

```Go
func modifyHostNetworkOptionForContainer(hostNetwork bool, sandboxID string, hc *dockercontainer.HostConfig) {
	sandboxNSMode := fmt.Sprintf("container:%v", sandboxID)
	hc.NetworkMode = dockercontainer.NetworkMode(sandboxNSMode)
	hc.IpcMode = dockercontainer.IpcMode(sandboxNSMode)
	hc.UTSMode = ""

	if hostNetwork {
		hc.UTSMode = namespaceModeHost
	}
}
```

这里我们可以关注几个事情：

1. pause容器和应用容器都是使用独自的uts namespace。应用容器之间也都是使用独立的namespace，因此任何一个容器启动后修改hostname并不会影响到其他的容器。

2. 如果判断是使用物理机网络就是hostNetwork，则会将uts的mode设置为"host"，也就是使用物理机的uts ns。
因此这时候容器修改hostname，也会影响到物理机。

当然，容器如果想要修改hostname（通过hostname命令），需要privileged权限才可以。

修改后容器直接重启会导致恢复原来的hostname。这个的主要原因是重启会导致重新创建新的uts namespace。

当然，如果容器重建了，比如exit后又被kubelet创建了一个新的容器，则hostname会再次恢复。

一个pod内有两个容器，两个容器修改hostname并不会彼此影响，因为他们的uts namespace是各自独立的。

> 通过修改/etc/hostname的方式动态修改运行容器的hostname无效。


以下是完整实验过程：

```
//容器启动时hostname为busyboxtest
[root@node ~]# docker exec 6f5 hostname
busyboxtest
//修改/etc/hostname文件
[root@node ~]# docker exec 6f5 cat /etc/hostname
busyboxtest123
[root@node ~]# docker exec 6f5 hostname test123
//修改后查看hostname为test123
[root@node ~]# docker exec 6f5 hostname
test123
[root@node ~]# docker inspect 6f5|grep Pid
            "Pid": 15818,
[root@node ~]# ll /proc/15818/ns/
total 0
lrwxrwxrwx 1 root root 0 Jan 10 16:16 ipc -> ipc:[4026535715]
lrwxrwxrwx 1 root root 0 Jan 10 16:16 mnt -> mnt:[4026535789]
lrwxrwxrwx 1 root root 0 Jan 10 16:16 net -> net:[4026535718]
lrwxrwxrwx 1 root root 0 Jan 10 16:16 pid -> pid:[4026535791]
lrwxrwxrwx 1 root root 0 Jan 10 16:18 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Jan 10 16:16 uts -> uts:[4026535790]
//重启容器
[root@node ~]# docker restart 6f5
6f5
[root@node ~]# docker inspect 6f5|grep Pid
            "Pid": 17553,
//可以看到uts的namespace变化了         
[root@node ~]# ll /proc/17553/ns/
total 0
lrwxrwxrwx 1 root root 0 Jan 10 16:19 ipc -> ipc:[4026535715]
lrwxrwxrwx 1 root root 0 Jan 10 16:19 mnt -> mnt:[4026535341]
lrwxrwxrwx 1 root root 0 Jan 10 16:19 net -> net:[4026535718]
lrwxrwxrwx 1 root root 0 Jan 10 16:19 pid -> pid:[4026535714]
lrwxrwxrwx 1 root root 0 Jan 10 16:19 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Jan 10 16:19 uts -> uts:[4026535713]
//hostname恢复
[root@node ~]# docker exec 6f5 hostname
busyboxtest
[root@node ~]# docker exec 6f5 cat /etc/hostname
busyboxtest123
```