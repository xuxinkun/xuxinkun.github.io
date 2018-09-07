---
layout:     post
title:      "graph easy绘制ascii简易流程图"
subtitle:   "graph easy."
date:       2018-9-3 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-graph-easy.jpg"
tags:

---

# graph-easy

日常我们经常需要画一些简易流程图，但是如果使用visio等工具来作图，一则略显大材小用，二则图片导出后再要粘贴。相比下，如果可以简单的用一些text的图来表达，则会简单的多。比如这种：

```Bash
[root@host /]# echo '[kubectl],[kube-proxy],[kube-scheduler],[kube-controller],[kubelet]->[kube-api]->[etcd]' |graph-easy
                        +------------+
                        |  kubectl   |
                        +------------+
                          |
                          |
                          v
+-----------------+     +------------+     +---------+
| kube-controller | --> |            | --> |  etcd   |
+-----------------+     |  kube-api  |     +---------+
+-----------------+     |            |     +---------+
| kube-scheduler  | --> |            | <-- | kubelet |
+-----------------+     +------------+     +---------+
                          ^
                          |
                          |
                        +------------+
                        | kube-proxy |
                        +------------+
```

这种流程图纯用ascii的符合组合而成，因此称为ascii流程图。本文推荐的graph-easy，就是ascii流程图作图的佼佼者。

# graph-easy安装

这里以centos 7为例进行安装。可以从graph-easy[官网](https://metacpan.org/pod/Graph::Easy)进行下载包。

```Bash
//下载安装包
wget https://cpan.metacpan.org/authors/id/S/SH/SHLOMIF/Graph-Easy-0.76.tar.gz
//解决依赖与编译安装
yum install perl perl-ExtUtils-MakeMaker graphviz
Makefile.PL
make test
make install
```

# graph-easy的使用

graph-easy的使用比较简单，官方提供了完整的[操作文档](http://bloodgate.com/perl/graph/manual/)。可以参考。

这里我举一些常用的例子，方便大家学习。

#### hello world

先来一个入门的hello world。

```Bash
[root@host /]# echo '[hello]->[world]' | graph-easy
+-------+     +-------+
| hello | --> | world |
+-------+     +-------+
```

graph-easy的语法相对来说比较宽松，`[hello]->[world]`，`[hello]-->[world]`,`[ hello ]-->[ world ]`都是可以的。这里可以根据个人的风格。我比较喜欢紧凑的风格。所以后面都是使用紧凑的方式来做。

#### 线上加个上标

有时候要在连接线上加一个标志说明，比如我想要表明从上海坐车到北京，则可以使用下面的方式：

```Bash
[root@host /]# echo "[shanghai]-- car -->[beijing]" | graph-easy
+----------+  car   +---------+
| shanghai | -----> | beijing |
+----------+        +---------+
```

#### 画一个环

```Bash
[root@host /]# echo "[a]->[b]->[a]" | graph-easy

  +---------+
  v         |
+---+     +---+
| a | --> | b |
+---+     +---+
[root@host /]# echo "[a]->[a]" | graph-easy

  +--+
  v  |
+------+
|  a   |
+------+
```

#### 多个目标或者多个源

```Bash
[root@host /]# echo "[a],[b]->[c]" | graph-easy
+---+     +---+     +---+
| a | --> | c | <-- | b |
+---+     +---+     +---+
[root@host /]# echo "[a]->[b],[c]" | graph-easy
+---+     +---+
| a | --> | b |
+---+     +---+
  |
  |
  v
+---+
| c |
+---+
```

#### 多个流程在一个图内

```Bash
[root@host /]# echo "[a]->[b]  [c]->[d]" | graph-easy
+---+     +---+
| a | --> | b |
+---+     +---+
+---+     +---+
| c | --> | d |
+---+     +---+
```

#### 改变图方向

默认图方向是从左到右的。有时候想要从上向下的流程图。可以用标签来调整

```Bash
[root@host /]# echo "graph{flow:south} [a]->[b]" | graph-easy
+---+
| a |
+---+
  |
  |
  v
+---+
| b |
+---+
```

其他还有诸如改变线型等，就不一一介绍了，可以参考官方文档来学习。