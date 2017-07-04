---
layout:     post
title:      "docker中执行sed: can't move '/etc/resolv.conf73UqmG' to '/etc/resolv.conf': Device or resource busy错误的处理原因及方式"
subtitle:   "Solve sed error in docker."
date:       2017-07-03 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-docker-sed.jpg"
tags:
    - docker
---

# 错误现象

在docker容器中想要修改`/etc/resolv.conf`中的namesever，使用sed命令进行执行时遇到错误：

```
/ # sed -i 's/192.168.1.1/192.168.1.254/g' /etc/resolv.conf
sed: can't move '/etc/resolv.conf73UqmG' to '/etc/resolv.conf': Device or resource busy
```

但是可以通过vi/vim直接修改这个文件`/etc/resolv.conf`这个文件的内容。

# 问题原因

sed命令的实质并不是修改文件，而是产生一个新的文件替换原有的文件。这里我们做了一个实验。

我先创建了一个`test.txt`的文件，文件内容是`123`。然后我使用`sed`命令对文件内容进行了替换。再次查看`test.txt`。

```
/ # stat test.txt 
  File: test.txt
  Size: 4         	Blocks: 8          IO Block: 4096   regular file
Device: fd28h/64808d	Inode: 265         Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2017-07-04 06:28:35.000000000
Modify: 2017-07-04 06:28:17.000000000
Change: 2017-07-04 06:29:03.000000000

/ # cat test.txt 
123
/ # sed -i 's/123/321/g' test.txt
/ # stat test.txt 
  File: test.txt
  Size: 4         	Blocks: 8          IO Block: 4096   regular file
Device: fd28h/64808d	Inode: 266         Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2017-07-04 06:29:31.000000000
Modify: 2017-07-04 06:29:31.000000000
Change: 2017-07-04 06:29:31.000000000

/ # cat test.txt
321
```

可以看到文件内容被正确修改了，但是同时，文件的inode也修改了。说明了实质上是新生成的文件替换了原有的文件。但是vim/vi是在原文件基础上修改的，所以inode没有变化。

在docker中，`/etc/resolv.conf`是通过挂载入容器的。所以当你想去删除这个挂载文件，也就是挂载点时，自然就会报`Device or resource busy`。

> 这个跟是不是特权privilege没有关系。即使是privilege的容器，也会有这个问题。

```
/ # rm /etc/resolv.conf 
rm: can't remove '/etc/resolv.conf': Device or resource busy
```

其实不仅仅`/etc/resolv.conf`，还有`/etc/hostname`，`/etc/hosts`等文件都是通过挂载方式挂载到容器中来的。所以想要用sed对他们进行修改，都会遇到这样的问题。我们可以通过`df -h`查看容器内的挂载情况。

```
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
/dev/mapper/docker-253:2-807144231-37acfcd86387ddcbc52ef8dac69d919283fc5d9d8ab5f55fd23d1c782e3b1c70
                         10.0G     33.8M     10.0G   0% /
tmpfs                    15.4G         0     15.4G   0% /dev
tmpfs                    15.4G         0     15.4G   0% /sys/fs/cgroup
/dev/mapper/centos-home
                        212.1G    181.8G     30.3G  86% /run/secrets
/dev/mapper/centos-home
                        212.1G    181.8G     30.3G  86% /dev/termination-log
/dev/mapper/centos-home
                        212.1G    181.8G     30.3G  86% /etc/resolv.conf
/dev/mapper/centos-home
                        212.1G    181.8G     30.3G  86% /etc/hostname
/dev/mapper/centos-home
                        212.1G    181.8G     30.3G  86% /etc/hosts
shm                      64.0M         0     64.0M   0% /dev/shm
tmpfs                    15.4G         0     15.4G   0% /proc/kcore
tmpfs                    15.4G         0     15.4G   0% /proc/timer_stats
```

# 如何解决

使用vi固然可以，但是对于批量操作就不是很合适了。可以通过sed和echo的组合命令`echo "$(sed 's/192.168.1.1/192.168.1.254/g' /etc/resolv.conf)" >  /etc/resolv.conf` 即可实现替换。

```
/ # cat /etc/resolv.conf 
search default.svc.games.local svc.games.local games.local
nameserver 192.168.1.1
options ndots:5
/ # echo "$(sed 's/192.168.1.1/192.168.1.254/g' /etc/resolv.conf)" >  /etc/resolv.conf
/ # cat /etc/resolv.conf 
search default.svc.games.local svc.games.local games.local
nameserver 192.168.1.254
options ndots:5
```

> 这里如果使用`sed 's/192.168.1.1/192.168.1.254/g' /etc/resolv.conf > /etc/resolv.conf`是无效的。最终会导致`/etc/resolv.conf`内容为空。