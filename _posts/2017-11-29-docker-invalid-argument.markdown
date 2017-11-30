---
layout:     post
title:      "启动docker容器时的Error response from daemon: devmapper: Error mounting: invalid argument. 错误解决"
subtitle:   "Error response from daemon: devmapper: Error mounting: invalid argument."
date:       2017-11-27 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-docker-invalid-argument.jpg"
tags:
    - docker
---


# 错误出现

在一台物理机重启后，以前创建的容器无法启动了。一启动，则会报出错误。

```
[root@217TN1V ~]# docker start e7e
Error response from daemon: devmapper: Error mounting '/dev/mapper/docker-253:4-11534337-ee772425c4996ca581e5c234806adf41aede9424a83ce1402596105a9f66434d' on '/export/docker/devicemapper/mnt/ee772425c4996ca581e5c234806adf41aede9424a83ce1402596105a9f66434d': invalid argument
```

# 错误原因

这个错误的主要原因是因为selinux enable的时候，创建了该容器。而后修改了`/etc/selinux/config`，修改成selinux为disabled。

物理机重启后，selinux处于关闭状态，则原先在selinux enable时候创建的容器就会无法启动，报出这种错误。

可以参考[github上的相关问题介绍](https://github.com/moby/moby/issues/29622)。

# 修复方法

修复方法主要有两种：

1. 可以将selinux重新置为enable，然后重启物理机，即可修复。
2. 修改容器的配置。比如我的容器的配置是`/var.lib/docker/containers/e7ef71494940ba293be4b3f74198bf34835c35537810053b051d9a6c33adbd32/config.v2.json`文件。将其中的`"MountLabel": "system_u:object_r:svirt_sandbox_file_t:s0:c12,c257", "ProcessLabel": "system_u:system_r:svirt_lxc_net_t:s0:c12,c257"`重修修改为`"MountLabel": "", "ProcessLabel": ""`，然后重新启动docker daemon，容器即可修复。