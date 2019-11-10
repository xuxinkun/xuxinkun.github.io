---
layout:     post
title:      "如何进行kubernetes问题的排障"
subtitle:   "how to debug problems of kubernetes."
date:       2019-11-06 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-2015.jpg"
tags:
    - kubernetes
---

## 排障的前置条件

k8s的成熟度很高，伴随着整个项目的扩增，以及新功能和新流程的不断引入，也伴随这产生了一些问题。虽然自动化测试可以排除掉大部分，但是一些复杂流程以及极端情况却很难做到bug的完全覆盖。因此在实际的工作过程中，需要对运行的集群进行故障定位和解决。

当然，进行排障的前提是对于k8s的流程和概念进行掌握，对于源码有一定的掌握能力，才可以更好的进行。待排障的环境和版本和源代码的版本需要进行匹配。版本号可以通过version命令获取，然后从源码进行对照。而且kubectl version还可以展示更为git的commit id。这样更为精准一些。本文以一次排障过程为例，介绍进行kubernetes问题排障的一般思路和方法。

## 故障背景

在某个压测的集群(集群版本为v1.12.10)内，为了测试极端性能，于是kubelet上配置了单节点可以创建的容器数从110调整为了600。并且进行反复大批量的容器创建和删除。在压测后一段时间，陆续多个节点变为NotReady，直到整个节点全部变为了NotReady。在节点上看到有大量的容器待删除。kubelet虽然仍在运行，但是已经不进行任何的pod生命周期的管理了，已经呆住了。其他组件大都正常。此时停了压测工具，kubelet仍然不能够恢复正常。尝试将一个节点的kubelet重启后，节点恢复正常。

## 故障分析

#### 日志分析

首先从日志上进行分析。日志是日常排障的最主要的工具。从长期经验来看，我们的主要方式是将日志写入到文件，并配合glogrotate进行日志的回滚。不使用journal的主要原因一个是习惯，另外就是使用效率上也没有文件来的快速。关于日志级别，日志级别太高，日志量会很大；而级别太低，日志信息量又不足。日志级别按照经验我们一般定位4级。

从日志上进行分析，可以看到这样一条日志。

```
I1105 09:50:27.583544  548093 kubelet.go:1829] skipping pod synchronization - [PLEG is not healthy: pleg was last seen active 58h29m3.043779855s ago; threshold is 3m0s]
```

也就是PLEG不再健康了。那么这一行是怎么报出来的呢？对照代码，我们可以找到这样的信息。

```
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
......
	for {
		if rs := kl.runtimeState.runtimeErrors(); len(rs) != 0 {
			glog.Infof("skipping pod synchronization - %v", rs)
			// exponential backoff
			time.Sleep(duration)
			duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
			continue
		}
		// reset backoff if we have a success
		duration = base

		kl.syncLoopMonitor.Store(kl.clock.Now())
		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
......
```

PLEG不再健康，这里就会直接continue，也就不会走到下面的syncLoopIteration函数。而这个函数通过逐层调用，最终会到syncPod上。这也就解释了为什么节点上kubelet不再处理pod生命周期的原因了。但是为什么PLEG不再健康。那么它的判断标准又是什么呢？继续看代码。

```
func (g *GenericPLEG) Healthy() (bool, error) {
	relistTime := g.getRelistTime()
	elapsed := g.clock.Since(relistTime)
	if elapsed > relistThreshold {
		return false, fmt.Errorf("pleg was last seen active %v ago; threshold is %v", elapsed, relistT
hreshold)
	}
	return true, nil
}
```
同时，这里可以看到PLEG的健康状态是以上次relist的时间来确定的。那么relist的时间又是在哪更新的呢？这个可以通过代码找到`func (g *GenericPLEG) relist(){...}`函数，也就是在这个函数里进行relist的时间更新的。那么可以初步判定应该可能在这个函数的某个流程里卡住了，导致的整个问题。但是这个函数有上百行，我们怎么定位呢？好像日志分析能够提供的帮助已经很有限了。那么我们需要一些其他的工具来辅助定位。

#### pprof

pprof就是这样的一个工具。pprof是什么，怎么用，这里不展开讲，可以去搜具体的资料。kubelet里有一个配置`enable-debugging-handlers`，通过配置为true进行开启（默认为true）。开启后，借助于这个工具，我们进行进一步的定位。kubelet的工作默认端口为10250。因为我们的集群中kubelet开启了认证，所以我们这里用curl命令直接把需要的堆栈信息拉取下来。

```
curl -k --cert admin.pem --key admin-key.pem "https://127.0.0.1:10250/debug/pprof/goroutine?debug=1" > stack.txt
```

因为卡住了，所以有非常多的goroutine信息。堆栈信息太长，这里就不全部拉取了，这里只截取我们关心的，就是relist卡在了哪里。我们搜索relist，可以看到这样的信息。

```
goroutine profile: total 2814
...
1 @ 0x42fcaa 0x42fd5e 0x4066cb 0x406465 0x30d4f3e 0x30d682a 0xad7474 0xad6a0d 0xad693d 0x45d2f1
#	0x30d4f3d	k8s.io/kubernetes/pkg/kubelet/pleg.(*GenericPLEG).relist+0x74d						/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/pkg/kubelet/pleg/generic.go:261
#	0x30d6829	k8s.io/kubernetes/pkg/kubelet/pleg.(*GenericPLEG).(k8s.io/kubernetes/pkg/kubelet/pleg.relist)-fm+0x29	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/pkg/kubelet/pleg/generic.go:130
#	0xad7473	k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait.JitterUntil.func1+0x53			/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:133
#	0xad6a0c	k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait.JitterUntil+0xbc				/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:134
#	0xad693c	k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait.Until+0x4c					/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:88
```

而且我们反复对照了多个节点，可以看到出现故障的节点，都卡在了这个261行的地方。这时候去找代码。

```
func (g *GenericPLEG) relist() {
......
		for i := range events {
			// Filter out events that are not reliable and no other components use yet.
			if events[i].Type == ContainerChanged {
				continue
			}
			g.eventChannel <- events[i]	//261行
		}
```

位于261行的地方是`g.eventChannel <- events[i]`这个代码。很奇怪，这个就发送一个event到channel里，为什么会卡住？难道channel close掉了？这个eventChannel又是个什么。追溯eventChannel的使用可以看到channel是在syncLoopIteration中被消费。同时还发现，eventChannel是一个有固定容量，1000的一个channel。那么问题逐渐变得清晰了起来。

#### 故障推断分析

反复大量的创建和删除，导致产生了非常多的event。这些event被发送进了eventChannel里，进而在syncLoopIteration的流程中进行消费。但是当syncLoopIteration消费的速度过慢，而events产生过快的时候，就会在`g.eventChannel <- events[i]`地方卡住。

由于上面event发送卡住了，relist流程无法正常进行。那么这里会导致PLEG变为not healthy。PLEG not healthy之后，rs := kl.runtimeState.runtimeErrors(); 将会生成错误。生成的错误就是PLEG is not healthy。更加糟糕的事情就这样发生了，PLEG不健康会导致调过syncLoopIteration的流程。而PLEG里面的eventChannel的消费是在syncLoopIteration里完成的。也就是说eventChannel没有人去消费，会一直保持满的状态。导致整体死锁了。新数据进不来，已有数据也无人消费。

#### 故障模拟

分析完成后是不是就可以完全断定了呢？最为安全保险的做法是在进行一下故障的模拟验证，以便验证自己的猜想。

那么对于本次的问题，那么我们如何验证呢？其实也很简单，减少eventChannel的容量，增加几行日志，打印channel的len，在进行一下压测，很快就能复现了。这里就不重复叙述了。

#### 故障解决

知道了问题的定位之后，想办法解决就可以了。对于这个问题我们如何解决呢？第一反应是可以增加eventChannel的容量。这个会额外消耗一些资源，但是整体问题不大。但是有没有更加优雅的方式呢？我试着去社区找一下，发现在今年已经有人发现并解决了这一问题了。[https://github.com/kubernetes/kubernetes/pull/72709](https://github.com/kubernetes/kubernetes/pull/72709)

解决方式很简洁，就是直接往eventChannel里塞，如果塞满了，直接记录一个错误日志，就不管了。

```
		for i := range events {
			// Filter out events that are not reliable and no other components use yet.
			if events[i].Type == ContainerChanged {
				continue
			}
			select {
			case g.eventChannel <- events[i]:
			default:
				metrics.PLEGDiscardEvents.WithLabelValues().Inc()
				klog.Error("event channel is full, discard this relist() cycle event")
			}
		}
```

这样的方式合理么？会丢弃event么？当然不会。因为relist是定时执行的。本次虽然丢弃了event，但是在下次relist的时候，会重新产生，并尝试再次塞入eventChannel。因此这种方式还是很简洁和高效的。这个bug fix已经合入了1.14以后的版本，但是1.12的版本未能覆盖。这里我们就将这个代码手动合入了我们自己的版本了。

## 写在最后

在生产和测试环境中，由于种种原因，可能是k8s的问题，可能是运维的问题，也可能是用户使用的问题，可能会出现种种状况。排查问题还是务求谨慎细致。我的一般原则是生产环境优先保存日志和堆栈后，立即恢复环境，保障用户使用。而在测试环境，优先把问题定位出来，尽量把错误前置，在测试环境发现并解决，防止其扩散到生产环境。

