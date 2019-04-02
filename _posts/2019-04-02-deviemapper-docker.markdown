---
layout:     post
title:      "详解docker中容器devicemapper设备的挂载流程"
subtitle:   "Explain the mounting process of devicemapper in docker."
date:       2019-4-2 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-build-k8s.jpg"
tags:
    - docker
---

## 事故起因

> 版本说明：本文中docker版本主要基于1.10版本，操作系统为centos7。devicemapper在文中缩写为dm。

某个用户的容器启动不起来，启动时候一直报错。通过docker log查看日志，可以看到报错信息如下

```
Timestamp: 2019-04-01 16:19:26.33690413 +0800 CST
Code: System error

Message: can't create pivot_root dir , error mkdir /export/docker/devicemapper/mnt/7a35b08cc8815ef439dfcc74fd375e2fbb618208674d6eff1ea5d0b61cc7ef1c/rootfs/.pivot_root492082001: no space left on device
```

对照docker源码，可以看到报错位置，就是在容器启动时候，docker会在容器中创建一个临时文件夹。而由于空间不足，因此导致报错，容器无法启动。

```
pivotDir, err := ioutil.TempDir(tmpDir, ".pivot_root")
if err != nil {
	return fmt.Errorf("can't create pivot_root dir %s, error %v", pivotDir, err)
}
```

初步可以判断是由于用户将dm的根目录磁盘写满了导致。那么要解决这个问题，其实思路也就很简单，就是将容器的dm设备手动挂载起来，然后清理冗余文件后，再恢复。那么在docker中，是如何将容器的dm盘进行创建挂载的，笔者对此进行了一下梳理。

## docker中容器的dm盘挂载原理

#### 获取dm设备信息

这里我们假设要模拟的是`f34cee76e36a`容器的挂载。
首先进入到docker的存储目录，可以从image的层文件夹中根据容器的id，获取mount-id。
而后从`devicemapper/metadata/`文件夹中，根据mount-id，获取device_id。这里，该容器的device_id为34709。

```
[root@localhost docker]# docker ps -a
CONTAINER ID        IMAGE                                            COMMAND                  CREATED             STATUS                       PORTS                                                                                              NAMES
f34cee76e36a        xuxinkun/kubesql                                 "kubesql-server"         3 weeks ago         Exited (137) 3 hours ago                                                                                                        kubesql-server
[root@localhost docker]# cat image/devicemapper/layerdb/mounts/f34cee76e36af13dbda34f78406ebe1c9a426d16e4eaae02055ace6b557b7539/mount-id 
dad0ce9e1adcfe76df319443f429698eafa52dd6c6cb4fb8192f0846656f8824
[root@localhost docker]# cat devicemapper/metadata/dad0ce9e1adcfe76df319443f429698eafa52dd6c6cb4fb8192f0846656f8824
{"device_id":34709,"size":10737418240,"transaction_id":844170,"initialized":false,"deleted":false}
```

先看一下已经创建的dm设备列表和table。


```
[root@localhost docker]# dmsetup ls
cl-swap	(253:1)
cl-root	(253:0)
docker-253:2-3693860-e85dd79353abe6967cbc4f968934ae078d4d9da02010033b73a3df9659e4d547	(253:4)
cl-home	(253:2)
docker-253:2-3693860-pool	(253:3)
[root@localhost docker]# dmsetup table
cl-swap: 0 32899072 linear 8:2 2048
cl-root: 0 104857600 linear 8:2 1647560704
docker-253:2-3693860-e85dd79353abe6967cbc4f968934ae078d4d9da02010033b73a3df9659e4d547: 0 20971520 thin 253:3 34719
cl-home: 0 1614659584 linear 8:2 32901120
docker-253:2-3693860-pool: 0 167772160 thin-pool 7:1 7:0 128 32768 1 skip_block_zeroing 
```

这其中主要要关注的是`docker-253:2-3693860-pool`，也就是`253:3`这个设备。后面的所有的docker容器的dm设备都是从这个pool中分出去的。
从table中可以看到一个已经创建好的，它的table是`0 20971520 thin 253:3 34719`。table的格式如下

```
logical_start_sector num_sectors target_type target_args
开始扇区             扇区数      设备类型    设备参数
```

> 20971520的扇区数应与docker启动时为容器分配的容量相关。

其中，设备参数的34719即为容器对应的device_id。

#### 创建dm设备

那么现在模拟容器挂载，首先创建一个dm设备。

```
[root@localhost docker]# dmsetup create docker-253:2-3693860-dad0ce9e1adcfe76df319443f429698eafa52dd6c6cb4fb8192f0846656f8824 --table "0 20971520 thin 253:3 34709"
```

然后可以查看下dm设备。

```
[root@localhost docker]# dmsetup ls
cl-swap	(253:1)
cl-root	(253:0)
docker-253:2-3693860-dad0ce9e1adcfe76df319443f429698eafa52dd6c6cb4fb8192f0846656f8824	(253:5)
docker-253:2-3693860-e85dd79353abe6967cbc4f968934ae078d4d9da02010033b73a3df9659e4d547	(253:4)
cl-home	(253:2)

[root@localhost tmp]# ls /dev/dm-*
/dev/dm-0  /dev/dm-1  /dev/dm-2  /dev/dm-3  /dev/dm-4  /dev/dm-5 
```

可以看到，新增了一个dm-5的设备。

> 当然，也可以在/dev/mapper下查看

#### 挂载dm设备

现在将/dev/dm-5设备挂载到自己创建了/tmp/test目录下。

```
[root@localhost docker]# mount /dev/dm-5 /tmp/test
[root@localhost docker]# ll /tmp/test/
total 4
-rw-------  1 root root  64 Mar 11 15:40 id
drwxr-xr-x 16 root root 274 Apr  2 09:54 rootfs
[root@localhost docker]# ll /tmp/test/rootfs/
total 16
-rw-r--r--  1 root root 12076 Dec  5 09:37 anaconda-post.log
lrwxrwxrwx  1 root root     7 Dec  5 09:36 bin -> usr/bin
drwxr-xr-x  4 root root    43 Mar 11 17:50 dev
drwxr-xr-x 48 root root  4096 Mar 11 17:50 etc
drwxr-xr-x  3 root root    21 Mar 11 17:38 home
lrwxrwxrwx  1 root root     7 Dec  5 09:36 lib -> usr/lib
lrwxrwxrwx  1 root root     9 Dec  5 09:36 lib64 -> usr/lib64
drwxr-xr-x  2 root root     6 Apr 11  2018 media
drwxr-xr-x  2 root root     6 Apr 11  2018 mnt
drwxr-xr-x  2 root root     6 Apr 11  2018 opt
drwxr-xr-x  2 root root     6 Dec  5 09:36 proc
dr-xr-x---  4 root root   140 Mar 11 17:37 root
drwxr-xr-x 12 root root   163 Mar 11 17:50 run
lrwxrwxrwx  1 root root     8 Dec  5 09:36 sbin -> usr/sbin
drwxr-xr-x  2 root root     6 Apr 11  2018 srv
drwxr-xr-x  2 root root     6 Dec  5 09:36 sys
drwxrwxrwt  7 root root   132 Mar 11 17:38 tmp
drwxr-xr-x 13 root root   155 Dec  5 09:36 usr
drwxr-xr-x 18 root root   238 Dec  5 09:36 var
```

可以在目录下看到rootfs文件夹，该文件夹下即为容器的所有文件。而后找出冗余文件删除即可。

#### 卸载移除设备

最后清理恢复，卸载目录，移除dm设备。

```
[root@localhost docker]# umount /tmp/test
[root@localhost docker]# dmsetup remove docker-253:2-3693860-e85dd79353abe6967cbc4f968934ae078d4d9da02010033b73a3df9659e4d547
```

重启容器后，恢复正常。至此，该问题得以修复。

以上是手动进行容器dm设备的创建挂载流程。其实在docker中，也基本上是这样来进行操作的。具体代码可以参考graphdriver下的devmapper中的相关流程。代码这里就不再一一走读了。

## docker中mnt挂载点不可见的问题

在这里顺带还解释下docker挂载的一个疑惑，就是docker中dm设备最终都会mount到devicemapper/mnt/{mount-id}/目录下。

但是在`/proc/mounts`下并不可见。这是因为docker daemon单独创建了一个mount的namespace。

```
[root@localhost tmp]# ll /proc/1/ns/mnt 
lrwxrwxrwx 1 root root 0 Mar 19 19:03 /proc/1/ns/mnt -> mnt:[4026531840]
[root@localhost tmp]# ps -fe|grep docker
root       845 23449  0 17:04 pts/1    00:00:00 grep --color=auto docker
root     23121     1  0 09:09 ?        00:00:38 /usr/bin/docker-current daemon --selinux-enabled --log-driver=journald -H tcp://127.0.0.1:5050 -H unix:///var/run/docker.sock -g /home/docker --storage-opt dm.blkdiscard=false --storage-opt dm.mountopt=nodiscard --storage-opt dm.loopdatasize=80G --storage-opt dm.loopmetadatasize=4G
[root@localhost tmp]# ll /proc/23121/ns/mnt 
lrwxrwxrwx 1 root root 0 Apr  2 10:25 /proc/23121/ns/mnt -> mnt:[4026532450]
```

23121为docker daemon的进程id。可以看到docker daemon使用了一个单独的mount namespace。因此可以通过查看该进程的mounts即可看到相应的挂载信息。

```
[root@localhost tmp]# cat /proc/23121/mounts|grep mnt
/dev/mapper/docker-253:2-3693860-e85dd79353abe6967cbc4f968934ae078d4d9da02010033b73a3df9659e4d547 /home/docker/devicemapper/mnt/e85dd79353abe6967cbc4f968934ae078d4d9da02010033b73a3df9659e4d547 xfs rw,relatime,nouuid,attr2,inode64,logbsize=64k,sunit=128,swidth=128,noquota 0 0
```

当然，也可以通过nsenter切入到docker daemon进程的mount ns下进行查看。

```
[root@localhost tmp]# nsenter -t 23121 -m mount -l|grep mnt
/dev/mapper/docker-253:2-3693860-e85dd79353abe6967cbc4f968934ae078d4d9da02010033b73a3df9659e4d547 on /home/docker/devicemapper/mnt/e85dd79353abe6967cbc4f968934ae078d4d9da02010033b73a3df9659e4d547 type xfs (rw,relatime,nouuid,attr2,inode64,logbsize=64k,sunit=128,swidth=128,noquota)
```

