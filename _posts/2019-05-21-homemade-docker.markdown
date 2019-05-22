---
layout:     post
title:      "土法搞docker系列之自制docker的graph driver vdisk"
subtitle:   "Homemade docker."
date:       2019-5-20 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-build-k8s.jpg"
tags:
    - docker
---

## 写在最前

偶然整理，翻出来14年刚开始学docker的时候的好多资料。当时docker刚刚进入国内，还有很多的问题。当时我们的思考方式很简单，docker确实是个好的工具，虽然还不成熟。但是不能因为短时间内造桥不行，就不过河了。我们的方式很简单，先造个小船划过去。由于各种条件的局限，所以很多方法真的是因陋就简，土法上马，一切就是为了抓紧落地。时代更迭、版本变迁，这其中的很多技术方案本身可能已经无法为现有的方案提供有力的帮助了。但是解决问题的思路和原理可能还能为大家提供一点参考吧。这于我自己，也是一个整理回顾。所以我计划写成一个小的系列文章，这个系列直接取名为土法搞docker。

当时遇到的第一个问题，就是docker的底层graph driver，在centos 6下的devicemapper不稳定，有很大的概率会造成内核崩溃。但是如果不解决这个问题，是绝对无法将docker上到生产环境中的。以我贫瘠的内核知识和存储知识，完全无力解决。那怎么办，那就用土办法，自己写一个graph driver。之所以叫自制而不叫自研，因为真的没有多少可以称之为研究的东西，完全是拼凑而成。自制的这个driver本身没有多少技术含量，但是需要深入了解docker的运行原理和底层的存储方式，然后寻找一种恰当的方式来解决它。

## graph driver原理

graph driver的原理和接口从1.3到现在的最新版本，基本没有什么变化。这也有赖于docker当时优秀的设计。首先说graph driver是干什么的。我们都知道docker的镜像/容器是由多层组成。graph driver其实就是负责了层文件的管理工作。

这里是driver接口的一些方法：

```
// ProtoDriver defines the basic capabilities of a driver.
// This interface exists solely to be a minimum set of methods
// for client code which choose not to implement the entire Driver
// interface and use the NaiveDiffDriver wrapper constructor.
//
// Use of ProtoDriver directly by client code is not recommended.
type ProtoDriver interface {
	// String returns a string representation of this driver.
	String() string
	// Create creates a new, empty, filesystem layer with the
	// specified id and parent. Parent may be "".
	Create(id, parent string) error
	// Remove attempts to remove the filesystem layer with this id.
	Remove(id string) error
	// Get returns the mountpoint for the layered filesystem referred
	// to by this id. You can optionally specify a mountLabel or "".
	// Returns the absolute path to the mounted layered filesystem.
	Get(id, mountLabel string) (dir string, err error)
	// Put releases the system resources for the specified id,
	// e.g, unmounting layered filesystem.
	Put(id string)
	// Exists returns whether a filesystem layer with the specified
	// ID exists on this driver.
	Exists(id string) bool
	// Status returns a set of key-value pairs which give low
	// level diagnostic status about this driver.
	Status() [][2]string
	// Cleanup performs necessary tasks to release resources
	// held by the driver, e.g., unmounting all layered filesystems
	// known to this driver.
	Cleanup() error
}
```

为了方便，我们举一个例子。假设某个镜像由两层组成，layer1(lower)和layer2(upper)。layer1中有一个文件A，layer2中有一个文件B。那么对单一的layer2来说，其实只有一个文件也就是B。但是通过联合文件系统将layer1和layer2联合起来时，得到layer1+2，将layer1+2挂载起来，那么获得的挂载点文件夹下应该包含了A和B两个文件(本文中将这种挂载点称为联合挂载点)。

![graph](https://xuxinkun.github.io/img/vdisk/graph.png)

这里结合这个例子分别对这些中比较重要的方法进行一下介绍：

- Create: 创建一层，比如创建layer2
- Remove: 移除某一层，比如移除layer2
- Get: 将id及其下层的所有层通过联合文件系统联合起来的layer1+2，将layer1+2挂载起来，返回挂载点
- Put: 将id及其下层的所有层通过联合文件系统联合起来的layer1+2的挂载点umount掉
- Exists: 判断id层是否存在
- Cleanup: 将所有的挂载起来的全部卸载掉

其实docker对于层文件的操作，都是通过这些接口组合而成的。比如docker创建容器时，最终需要用Create为容器创建一个新的可读写的层。而docker运行容器时，需要通过Get接口获取容器及其镜像所有层联合起来的文件从而形成容器的rootfs。

在分析这些接口时，我们其实可以发现一个问题，其实接口中没有获取单层，比如只获取layer2的接口。比如docker save镜像时，因为要导出每一层的单独的文件，这又是如何实现的呢？其基本原理其实算是Get(layer1)以及Get(layer2)，然后将两层的挂载的文件夹进行diff，从而得到只归属于layer2层的文件。

```
func (gdw *NaiveDiffDriver) Diff(id, parent string) (arch archive.Archive, err error) {
	driver := gdw.ProtoDriver
	//获取id的联合挂载点
	layerFs, err := driver.Get(id, "")
	...
	//获取parent的联合挂载点
	parentFs, err := driver.Get(parent, "")
	...
	//遍历两个挂载点内的所有文件并进行比较，得到二者的差异文件
	//则差异文件就是只属于id层的文件列表
	changes, err := archive.ChangesDirs(layerFs, parentFs)
	...
```

这里我们可以特别想下，接口没有获取单层，也就是获取layer2这种层的接口，那么其实就意味着docker其实并不真的需要一个联合文件系统。这也就是我们能够自制vdisk的基础。

那么大家可能会有个小疑问，既然不一定真的需要联合文件系统，那么使用或者不使用联合文件系统有什么差别呢？差别并不在Get接口上，而是在Create接口上。使用联合文件系统时，创建一个新的单层可以非常快速，因为新的层的内容为空。而不使用联合文件系统呢，则需要将所有父层的文件全部拷贝到新的层中，以便在Get接口调用时可以快速挂载。这样二者的创建效率就一目了然了。

> docker自身也支持一个默认的非联合文件系统的graph driver，也就是vfs。
> vfs这个驱动简单明了。我当年就是从这里开始graph driver的理解和学习的。


## vdisk原理

我们的实际需求其实是要在centos下用一个非联合文件系统的方式来取代devicemapper，实现一个稳定可靠的底层存储。那么如何实现，其实有几种路线选择。vfs足够简单稳定，但是无法限制用户对于磁盘的使用量。使用不同的lvm盘来存储每层，由于需要预分配足够的磁盘空间，又会导致磁盘空间的浪费。最终，我们选择了一个折中的方案。就是使用稀疏文件来存储每一层，然后通过loop设备挂载，来表达联合文件系统的挂载效果。

那么同上一个例子，对于layer1的所在层，我们其实可以创建稀疏文件file1，并在其中存储文件A。而对于layer2的所在层如何处理呢？因为接口中没有获取单层文件的接口，我们因此可以创建file2，并在其中存储文件A和B，也就是layer1+2，来实现layer1和layer2的联合。而对于只导出layer2时，只需要将file1和file2的文件进行diff就可以处理了(同上文所说)。

明白了这个原理后，其实代码就好写了。这也是我当时刚学golang后写的第一个docker功能。代码原理上我参考了vfs的实现，也参考了dm驱动的deviceset进行loop设备的管理。其实完全是东拼西凑来的，这里就不献丑了，回头我传到github (https://github.com/xuxinkun) 上去，有兴趣的再来围观吧。

## vdisk的弊端

这个驱动因为使用的是稀疏文件和loop设备，因此我命名为loopfile，后来被改名为vdisk。这个驱动原是想应急使用。但是因为足够简单，所以足够稳定。在线上几乎是零故障。虽然后来修复了devicemapper的bug，但是在JDOS 1.0的集群上仍然大规模使用的是这个。当然这其中的一个重要原因其实是因为1.0(基于openstack，采用nova+docker方式管理)还是将容器当做虚拟机来使用，实际创建完容器，仍然需要用户通过部署平台来部署脚本。因此对于容器创建时间不是那么敏感。同时由于镜像预分发，所以创建时间并不是太大的问题。

但是如果镜像层数过多，因为每层的文件中要包含全部父层的文件，存在很大的冗余空间占用。为了解决Dockerfile或者多次commit导致的镜像多层问题，我还为docker增加了compress功能，用以将多层压缩为一层。这个的实现方式我将在后续文章中讲述。

后来，进入到JDOS 2.0时代，这种方式就完全无法应付快速启动容器的需求了。dm的问题也由团队后来的内核专家进行了解决。从此我们就跨入了dm的时代。当然这些就是后话了。