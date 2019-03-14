---
layout:     post
title:      "解析docker中的环境变量使用和常见问题解决"
subtitle:   "Enviroment in docker."
date:       2019-03-12 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-env-docker.jpg"
tags:
    - docker
---

# docker容器中的环境变量

docker可以为容器配置环境变量。配置的途径有两种：

1. 在制作镜像时，通过`ENV`命令为镜像增加环境变量。在容器启动时使用该环境变量。
2. 在容器启动时候，通过参数配置环境变量，如果与镜像中有重复的环境变量，会覆盖镜像的环境变量。

使用`docker exec {containerID} env`即可查看容器中生效的环境变量。

```
[root@localhost ~]# docker exec 984 env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/java/default/bin
TERM=xterm
AUTHORIZED_KEYS=**None**
JAVA_HOME=/usr/java/default
HOME=/root
...
```

容器启动的进程，也就是ENTRYPOINT+CMD中，可以通过相应的系统库获取容器的环境变量。

进入到容器中，查看进程的环境变量，可以通过/proc下进行查看。

```
cat /proc/{pid}/environ
```

因此，容器中的环境变量也可以通过在容器中查看1号进程的环境变量来获取。可以通过执行`cat /proc/1/environ |tr '\0' '\n'`命令进行查看。

```
[root@localhost ~]# docker exec -it 984 cat /proc/1/environ |tr '\0' '\n'
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/java/default/bin
TERM=xterm
AUTHORIZED_KEYS=**None**
JAVA_HOME=/usr/java/default
HOME=/root
...
```

一般来说，从父进程产生出来的子进程都会默认继承父进程的环境变量。因此容器中的各个进程的环境变量应该是大致相同的。当然，在一些特殊的情况下，环境变量也会被重置，导致产生一些误解和问题。下面就对容器中一些常见的情况进行相关讲解。

# 常见问题及解决

## 切换不同用户后环境变量消失

在容器中，启动后切换不同用户，比如使用`su - admin`切换admin用户后，发现配置的容器环境变量丢失了。

这是因为切换用户会导致环境变量重置。因此要使用`su -p admin`这样的方式，才可以继承先前的环境变量。


我们可以通过help来看下su的相关参数描述。

```
[root@adworderp-03a38d62-4103555841-m81qk /]# su --help
Usage: su [OPTION]... [-] [USER [ARG]...]
Change the effective user id and group id to that of USER.

...
  -m, --preserve-environment   do not reset HOME, SHELL, USER, LOGNAME
                               environment variables
  -p                           same as -m
...
```

## 容器中的乱码问题

一些业务在迁移到容器中时，常常报告打印日志乱码。一般的原因是locale没有配置正确导致。

可以通过`locale`查看当前容器的语言环境。如果没设置，一般会是POSIX。我们可以通过`locale -a`查看当前容器支持的语言环境，而后根据需要进行设置。
要想一劳永逸，最好的方式还是在容器启动或者镜像的环境变量中添加`LANG={xxx}`，选择合适的语言，从而避免因此导致的乱码问题。

## ssh的环境变量问题

容器中启用sshd，可以方便连接和排障，以及进行一些日常的运维操作。
但是很多用户进入到容器中却发现，在docker启动时候配置的环境变量通过`env`命令并不能够正常显示。
这个的主要原因还是ssh为用户建立连接的时候会导致环境变量被重置。
这样导致的最大问题就是通过ssh启动的容器进程将无法获取到容器启动时候配置的环境变量。

了解了原理后，这个问题有个简单的方法解决。就是可以通过将容器的环境变量重新设置到ssh连接后的session中。
具体的实现方式是，ssh连接后，会自动执行`source /etc/profile`。
那么我们其实只要在`/etc/profile`追加几行代码，从1号进程获取容器本身的环境变量，然后循环将环境变量export一下即可。

以下是一个简单的for循环实现。

```
for item in `cat /proc/1/environ |tr '\0' '\n'`
do
 export $item
done
```

当然，有更简洁的命令，就是`export $(cat /proc/1/environ |tr '\0' '\n' | xargs)`，可以实现同样的效果。

