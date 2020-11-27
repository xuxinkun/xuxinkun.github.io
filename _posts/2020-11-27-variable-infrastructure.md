---
layout:     post
title:      "基于Kubernetes和OpenKruise的可变基础设施实践"
subtitle:   "Variable infrastructure practice based on Kubernetes and OpenKruise."
date:       2020-11-26 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-kube-ep-disappear.jpg"
tags:
    kubernetes
---

## 对于可变基础设施的思考

#### kubernetes中的可变与不可变基础设施

在云原生逐渐盛行的现在，不可变基础设施的理念已经逐渐深入人心。不可变基础设施最早是由Chad Fowler于2013年提出的，其核心思想为任何基础设施的实例一旦创建之后变成为只读状态，如需要修改和升级，则使用新的实例进行替换。这一理念的指导下，实现了运行实例的一致，因此在提升发布效率、弹性伸缩、升级回滚方面体现出了无与伦比的优势。

kubernetes是不可变基础设施理念的一个极佳实践平台。Pod作为k8s的最小单元，承担了应用实例这一角色。通过ReplicaSet从而对Pod的副本数进行控制，从而实现Pod的弹性伸缩。而进行更新时，Deployment通过控制两个ReplicaSet的副本数此消彼长，从而进行实例的整体替换，实现升级和回滚操作。

我们进一步思考，我们是否需要将Pod作为一个完全不可变的基础设施实例呢？其实在kubernetes本身，已经提供了一个替换image的功能，来实现Pod不变的情况下，通过更换image字段，实现Container的替换。这样的优势在于无需重新创建Pod，即可实现升级，直接的优势在于免去了重新调度等的时间，使得容器可以快速启动。

从这个思路延伸开来，那么我们其实可以将Pod和Container分为两层来看**。将Container作为不可变的基础设施，确保应用实例的完整替换；而将Pod看为可变的基础设施，可以进行动态的改变，亦即可变层。**

#### 关于升级变化的分析

对于应用的升级变化种类，我们来进行一下分类讨论，将其分为以下几类：

| 升级变化类型   | 说明                                    |
| -------------- | --------------------------------------- |
| 规格的变化     | cpu、内存等资源使用量的修改             |
| 配置的变化     | 环境变量、配置文件等的修改              |
| 镜像的变化     | 代码修改后镜像更新                      |
| 健康检查的变化 | readinessProbe、livenessProbe配置的修改 |
| 其他变化       | 调度域、标签修改等其他修改              |

针对不同的变化类型，我们做过一次抽样调查统计，可以看到下图的一个统计结果。

![image-20201116110018301](https://xuxinkun.github.io/img/2020/image-20201116110018301.png)

> 在一次升级变化中如果含有多个变化，则统计为多次。

可以看到支持镜像的替换可以覆盖一半左右的的升级变化，但是仍然有相当多的情况下导致不得不重新创建Pod。这点来说，不是特别友好。所以我们做了一个设计，将对于**Pod的变化分为了三种Dynamic，Rebuild，Static**三种。

| 修改类型         | 修改类型说明              | 修改举例                         | 对应变化类型           |
| ---------------- | ------------------------- | -------------------------------- | ---------------------- |
| Dynamic 动态修改 | Pod不变，容器无需重建     | 修改了健康检查端口               | 健康检查的变化         |
| Rebuild 原地更新 | Pod不变，容器需要重新创建 | 更新了镜像、配置文件或者环境变量 | 镜像的变化，配置的变化 |
| Static 静态修改  | Pod需要重新创建           | 修改了容器规格                   | 规格的变化             |

这样动态修改和原地更新的方式可以**覆盖90%以上的升级变化**。在Pod不变的情况下带来的收益也是显而易见的。

1. 减少了调度、网络创建等的时间。
2. 由于同一个应用的镜像大部分层都是复用的，大大缩短了镜像拉取的时间。
3. 资源锁定，防止在集群资源紧缺时由于出让资源重新创建进入调度后，导致资源被其他业务抢占而无法运行。
4. IP不变，对于很多有状态的服务十分友好。

## Kubernetes与OpenKruise的定制

#### kubernetes的定制

那么如何来实现Dynamic和Rebuild更新呢？这里需要对kubernetes进行一下定制。

###### 动态修改定制

liveness和readiness的动态修改支持相对来说较为简单，主要修改点在与prober_manager中增加了UpdatePod函数，用以判断当liveness或者readiness的配置改变时，停止原先的worker，重新启动新的worker。而后将UpdatePod嵌入到kubelet的HandlePodUpdates的流程中即可。

```
func (m *manager) UpdatePod(pod *v1.Pod) {
	m.workerLock.Lock()
	defer m.workerLock.Unlock()

	key := probeKey{podUID: pod.UID}
	for _, c := range pod.Spec.Containers {
		key.containerName = c.Name
		{
			key.probeType = readiness
			worker, ok := m.workers[key]
			if ok {
				if c.ReadinessProbe == nil {
					//readiness置空了，原worker停止
					worker.stop()
				} else if !reflect.DeepEqual(*worker.spec, *c.ReadinessProbe) {
					//readiness配置改变了，原worker停止
					worker.stop()
				}
			}
			if c.ReadinessProbe != nil {
				if !ok || (ok && !reflect.DeepEqual(*worker.spec, *c.ReadinessProbe)) {
					//readiness配置改变了，启动新的worker
					w := newWorker(m, readiness, pod, c)
					m.workers[key] = w
					go w.run()
				}
			}
		}
		{
			//liveness与readiness相似
			......
		}
	}
}
```

###### 原地更新定制

kubernetes原生支持了image的修改，对于env和volume的修改是未做支持的。因此我们对env和volume也支持了修改功能，以便其可以进行环境变量和配置文件的替换。这里利用了一个小技巧，就是我们在增加了一个ExcludedHash，用于计算Container内，包含env，volume在内的各项配置。

```
func HashContainerExcluded(container *v1.Container) uint64 {
	copyContainer := container.DeepCopy()
	copyContainer.Resources = v1.ResourceRequirements{}
	copyContainer.LivenessProbe = &v1.Probe{}
	copyContainer.ReadinessProbe = &v1.Probe{}
	hash := fnv.New32a()
	hashutil.DeepHashObject(hash, copyContainer)
	return uint64(hash.Sum32())
}
```

这样当env，volume或者image发生变化时，就可以直接感知到。在SyncPod时，用于在计算computePodActions时，发现容器的相关配置发生了变化，则将该容器进行Rebuild。

```
func (m *kubeGenericRuntimeManager) computePodActions(pod *v1.Pod, podStatus *kubecontainer.PodStatus) podActions {
	......
	for idx, container := range pod.Spec.Containers {
		......
		if expectedHash, actualHash, changed := containerExcludedChanged(&container, containerStatus); changed {
		    // 当env，volume或者image更换时，则重建该容器。
			reason = fmt.Sprintf("Container spec exclude resources hash changed (%d vs %d).", actualHash, expectedHash)			
			restart = true
		}
		......
		message := reason
		if restart {
			//将该容器加入到重建的列表中
			message = fmt.Sprintf("%s. Container will be killed and recreated.", message)
			changes.ContainersToStart = append(changes.ContainersToStart, idx)
		}
......
	return changes
}
```

###### Pod的生命周期

在Pod从调度完成到创建Running中，会有一个ContaienrCreating的状态用以标识容器在创建中。而原生中当image替换时，先前的一个容器销毁，后一个容器创建过程中，Pod状态会一直处于Running，容易有错误流量导入，用户也无法识别此时容器的状态。

因此我们为原地更新，在ContainerStatus里增加了ContaienrRebuilding的状态，同时在容器创建成功前Pod的Ready Condition置为False，以便表达容器整在重建中，应用在此期间不可用。利用此标识，可以在此期间方便识别Pod状态、隔断流量。

#### OpenKruise的定制

OpenKruise(https://openkruise.io/)是阿里开源的一个项目，提供了一套在Kubernetes核心控制器之外的扩展 workload 管理和实现。其中Advanced StatefulSet，基于原生 StatefulSet 之上的增强版本，默认行为与原生完全一致，在此之外提供了原地升级、并行发布（最大不可用）、发布暂停等功能。

Advanced StatefulSet中的原地升级即与本文中的Redbuild一致，但是原生只支持替换镜像。因此我们在OpenKruise的基础上进行了定制，使其不仅可以支持image的原地更新，也可以支持当env、volume的原地更新以及livenessProbe、readinessProbe的动态更新。这个主要在`shouldDoInPlaceUpdate`函数中进行一下判断即可。这里就不再做代码展示了。

还在生产运行中还发现了一个基础库的小bug，我们也顺带向社区做了提交修复。https://github.com/openkruise/kruise/pull/154。

另外，还有个小坑，就是在pod里为了标识不同的版本，加入了controller-revision-hash值。

```
[root@xxx ~]# kubectl get pod -n predictor  -o yaml predictor-0 
apiVersion: v1
kind: Pod
metadata:
  labels:
    controller-revision-hash: predictor-85f9455f6
...
```

一般来说，该值应该只使用hash值作为value就可以了，但是OpenKruise中采用了{sts-name}+{hash}的方式，这带来的一个小问题就是`sts-name`就要因为label value的长度受到限制了。

## 写在最后

定制后的OpenKruise和kubernetes已经大规模在各个集群上上线，广泛应用在多个业务的后端运行服务中。经统计，通过原地更新覆盖了87%左右的升级部署需求，基本达到预期。

特别鸣谢阿里贡献的开源项目OpenKruise。
