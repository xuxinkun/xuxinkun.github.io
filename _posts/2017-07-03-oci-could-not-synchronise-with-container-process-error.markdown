---
layout:     post
title:      "docker启动容器报错: could not synchronise with container process: not a directory"
subtitle:   "solve could not synchronise with container process error."
date:       2017-07-03 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-docker-synchronise-process.jpg"
tags:
    - docker
---

# 错误现象

在运行容器时，出现以下错误

```
[root@localhost test]# docker run -it -d -v $PWD/test.txt:/mydir mytest 
fd44cdc550548c0b791d6a7d12d27a2d64855c7c5d498305dd1239d6608b4350
Error response from daemon: Cannot start container fd44cdc550548c0b791d6a7d12d27a2d64855c7c5d498305dd1239d6608b4350: [9] System error: could not synchronise with container process
```

而实际查看容器时，可以看到容器已经成功创建，但是实际无法启动。

```
[root@localhost test]# docker ps -a|grep fd4
fd44cdc55054        mytest              "sh"                     3 minutes ago       Created                                               mad_jones
```

# 问题复现及分析

容器直接启动`docker run -it mytest`是可以正常启动的。

```
[root@localhost site-packages]# docker run -it mytest
/ # stat /mydir
  File: /mydir
  Size: 4096      	Blocks: 8          IO Block: 4096   directory
Device: fd08h/64776d	Inode: 786453      Links: 2
Access: (0755/drwxr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 1970-01-01 00:00:00.000000000
Modify: 2017-07-04 09:39:03.000000000
Change: 2017-07-04 09:39:04.000000000
```


但是加了挂载的参数`-v $PWD/test.txt:/mydir`参数后就无法启动了。所以可以判断问题是出在挂载上。

查看上面那个未能启动的容器，可以看到容器中的挂载。

```
[root@localhost test]# docker inspect fd4
    ...
    "HostConfig": {
        "Binds": [
            "/root/dockerfile/test/test.txt:/mydir"
        ],
    ...
```

通过上面可以看到，容器中，或者说原有的镜像中已经存在了`/mydir`，而且`/mydir`是一个文件夹。
而现在把`test.txt`这个文件映射挂载为文件夹，即会报`could not synchronise with container process: not a directory`的错误。

修正方法只要修改一下挂载点就可以，可以使用`-v $PWD/test.txt:/mydir/test.txt`即可成功启动。