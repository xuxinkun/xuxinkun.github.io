---
layout:     post
title:      "使用ansible kubectl插件连接kubernetes pod以及实现原理"
subtitle:   "Use ansible to connect to pods of kubernetes."
date:       2019-3-14 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-curl-ca.jpg"
tags:
    - ansible
    - kubernetes
---


# ansible kubectl connection plugin

ansible是目前业界非常火热的自动化运维工具。ansible可以通过ssh连接到目标机器上，从而完成指定的命令或者操作。
在kubernetes集群中，因为并不是所有的服务都是那么容器化。有时候也会用到ansible进行一些批量运维的工作。
一种方式是可以在容器中启动ssh，然后再去连接执行。但是并不是所有的容器都会启动ssh。

针对于这种情况，我想到了直接用kubectl进行连接操作，因此开发了kubectl的connection插件，并贡献给了社区。
该功能无需容器中启动ssh服务即可使用，已经合入ansible的主干，自ansible 2.5版本后随ansible发布。
详细操作文档可以参考[https://docs.ansible.com/ansible/latest/plugins/connection/kubectl.html](https://docs.ansible.com/ansible/latest/plugins/connection/kubectl.html)。本文将对其主要功能和设计的思路原理进行一下介绍。

## 安装使用

ansible 2.5后内置了该connection plugin，所以之后的版本都可以自动支持。如果是之前的版本，需要自行合并该PR。

PR参考:https://github.com/ansible/ansible/commit/ca4eb07f46e7e3112757cb1edc7bd71fdd6dacad#diff-12d9b364560fd0bec5a1cee5bd0b3c24。

该connection plugin**需要使用kubectl的二进制文件，务必将kubectl先进行安装**，一般可以放置在/usr/bin下。

## 操作样例

以下是一个inventory的配置。

```
[root@f34cee76e36a kubesql]# cat inventory 
[kube-test:vars]
ansible_connection=kubectl
ansible_kubectl_kubeconfig=/etc/kubeconfig
[kube-test]
hcnmore-32385abb-9f753dda-nxr8h ansible_kubectl_namespace=nevermore
```

对于kube-test组，`ansible_connection=kubectl`用于标识使用kubectl的连接插件。

`ansible_kubectl_kubeconfig`用于标识使用的kubeconfig的位置。当然，也支持使用无认证、使用用户名密码认证等等方式进行连接。这些配置可以参考官网的说明: [https://docs.ansible.com/ansible/latest/plugins/connection/kubectl.html#parameters](https://docs.ansible.com/ansible/latest/plugins/connection/kubectl.html#parameters)

在inventory配置完成后，即可进行ansible的操作了。这里我们查看一下当前的工作目录。

```
[root@f34cee76e36a kubesql]# ansible -i inventory kube-test -m shell -a "pwd"     
hcnmore-32385abb-9f753dda-nxr8h | CHANGED | rc=0 >>
/
```

## playbook样例使用

该connection plugin同样支持使用playbook对批量任务进行执行。以下是一个playbook的样例。

```
- hosts: kube-test
  gather_facts: False
  tasks:
    - shell: pwd
```

我们复用了先前样例的inventory进行执行，同时打开`-v`进行观察。

```
[root@f34cee76e36a kubesql]# ansible-playbook -i inventory playbook -v 
Using /etc/ansible/ansible.cfg as config file

PLAY [kube-test] *************************************************************************************************************************************************************************************************

TASK [shell] *****************************************************************************************************************************************************************************************************
changed: [hcnmore-32385abb-9f753dda-nxr8h] => {"changed": true, "cmd": "pwd", "delta": "0:00:00.217860", "end": "2019-03-18 20:55:39.479859", "rc": 0, "start": "2019-03-18 20:55:39.261999", "stderr": "", "stderr_lines": [], "stdout": "/", "stdout_lines": ["/"]}

PLAY RECAP *******************************************************************************************************************************************************************************************************
hcnmore-32385abb-9f753dda-nxr8h : ok=1    changed=1    unreachable=0    failed=0  
```

可以看到在目标机器上成功进行了命令执行。

## 优势与弊端

kubectl connection plugin不仅支持命令的执行，其他如cp文件、fetch文件等基本操作都可以执行。当然如执行shell、script脚本、密钥管理等复杂操作也都可以通过这个插件进行执行。但是这些复杂操作有的需要目标机器上安装python等环境进行支持，这个要根据不同模块的需求具体来看。

使用openshift的插件oc的使用方式与kubectl类似，这里不重复介绍了。
 
可以说这个kubectl的插件可以解决大部分的日常运维操作，但是也有一些弊端，就是使用kubectl的二进制进行连接。每次每个连接都需要启动一个进程进行kubectl执行，而且连接速度不快。因此不太适合大批量的容器高频操作。

我个人的建议是可以用这个插件进行注入密钥、低频启停服务的运维操作。大批量的操作仍然最好使用ssh进行。


# 实现原理

## ansible connection plugin

ansible的connection plugin有很多，目前支持的有ssh、docker、kubectl等等。要实现一个ansible的连接插件，其实主要实现的是三个接口:

- exec_command: 在目标机执行一个命令，并返回执行结果。
- put_file: 将本地的一个文件传送到目标机上。
- fetch_file: 将目标机的一个文件拉回到本地。

这三个接口对应三个原生的模块，依次是raw，cp和fetch。而ansible的几乎所有复杂功能，都是通过这三个接口来进行实现的。比如shell或者script，就是通过将脚本或者命令形成的脚本cp到目标机上，而后进行命令执行完成的。

## kubectl exec

kubectl exec天然就是执行命令的。docker client 有cp的功能，但是kubectl并不具有(k8s 1.5版本后已支持了cp)。这里用了一个小的技巧，就是使用dd命令配合kubectl exec进行文件的传输和拉取。

> 因此要求目标容器中也要有dd命令，否则该插件也无法工作。

dd主要用于读取、转换和输出数据。我们将一个src文件拷贝到另外一个地方dest，可以使用`dd if=/tmp/src of=/tmp/dest`。dd也支持从标准输入中获取数据或者输出到标准输出中。因此拷贝也就可以使用`dd if=/tmp/src | dd of=/tmp/dest`的方法。

理解了这些，我们就可以使用kubectl来进行文件的传输了。那么向容器中传输文件的方式就可以使用 `dd if=/tmp/src | kubectl exec -i podname dd of=/tmp/dest`。反过来，从容器中拉取文件就对应可以使用`kubectl exec -i podname dd of=/tmp/src | dd of=/tmp/src`。

> 特别注意，向容器中传输文件的`-i`参数是必须的。因为需要用-i参数开启支持kubectl从标准输入中获取数据。

这样就通过kubectl exec，实现了执行命令、文件传输以及拉取的效果。对应在ansible的connection plugin中进行实现，即可使得kubernetes的pod容器也支持ansible的运维控制。具体实现代码就不再重复讲述，详细可以参考[https://github.com/ansible/ansible/blob/devel/lib/ansible/plugins/connection/kubectl.py](https://github.com/ansible/ansible/blob/devel/lib/ansible/plugins/connection/kubectl.py)。

