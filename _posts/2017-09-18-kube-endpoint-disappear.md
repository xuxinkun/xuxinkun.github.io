---
layout:     post
title:      "kubernetes endpoint一会消失一会出现的问题剖析"
subtitle:   "Endpoints of kubernetes disappear for a while."
date:       2017-09-18 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-kube-ep-disappear.jpg"
tags:
    - kubernetes
---


# 问题现象

发现某个service的后端endpoint一会显示有后端，一会显示没有。显示没有后端，意味着后端的address被判定为notready。

endpoint不正常的时候：

```
[root@localhost /]# kubectl get ep --namespace cxqt npth-price  -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  ...
  uid: 9ed3abd1-8eff-11e7-b345-f8758831889c
subsets:
- notReadyAddresses:
  - ip: 10.1.3.70
    nodeName: 11.2.3.10
...
```    

endpoint正常的时候：

```
[root@localhost /]# kubectl get ep --namespace cxqt npth-price  -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  ...
  uid: 9ed3abd1-8eff-11e7-b345-f8758831889c
subsets:
- addresses:
  - ip: 10.1.3.70
    nodeName: 11.2.3.10
...
```

# 问题分析

查看源码，可以看到endpoint是根据pod的status中的conditions中type是Ready的字典中的status是否为True进行判断。

```
// IsPodReady returns true if a pod is ready; false otherwise.
func IsPodReady(pod *Pod) bool {
	return IsPodReadyConditionTrue(pod.Status)
}

// IsPodReady retruns true if a pod is ready; false otherwise.
func IsPodReadyConditionTrue(status PodStatus) bool {
	condition := GetPodReadyCondition(status)
	return condition != nil && condition.Status == ConditionTrue
}
```

```
apiVersion: v1
kind: Pod
metadata:
	...
    name: e9ebca20-0f3e-4974-8178-715cbbf5c627
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: 2017-09-08T02:58:41Z
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: 2017-09-08T02:59:11Z
    status: "False"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: 2017-09-08T02:58:41Z
    status: "True"
    type: PodScheduled
...
```

再进行日志查看，发现这个status字段是在由kube-controller-manager进行的更新为False。

查看日志，发现kube-controller-manager更新的原因是因为controller-manager判断node上报心跳超时了。

```
I0919 16:05:35.383806   20248 nodecontroller.go:1007] node 11.2.3.10 hasn't been updated for 40.032883982s. Last ready condition is: {Type:Ready Status:True LastHeartbeatTime:2017-09-19 16:04:46 +0800 CST LastTransitionTime:2017-09-19 16:04:46 +0800 CST Reason:KubeletReady Message:kubelet is posting ready status}
...
I0919 16:05:35.387629   20248 controller_utils.go:320] Recording status change NodeNotReady event message for node 11.2.3.10
I0919 16:05:35.387679   20248 controller_utils.go:238] Update ready status of pods on node [11.2.3.10]
```

而反过来查看11.2.3.10节点上的kubelet，上面因为有许多容器、镜像等。kubelet在准备上报信息时，需要收集容器、镜像等的信息。虽然kubelet默认是10秒上报一次，但是实际的上报周期约为20~50秒。而kube-controller-manager判断node上报心跳超时的时间为40秒。所以会有一定概率超时。一旦超时，kube-controller会将该node上的所有pod的conditions中type是Ready的字典中的status置为False。

# 解决方案

目前一个较为简单的方案是在kube-controller上配置这个超时时间`node-monitor-grace-period `长一些。建议配置为60~120s。

