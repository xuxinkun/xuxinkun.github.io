---
layout:     post
title:      "详解k8s中的liveness和readiness的原理和区别"
subtitle:   "talk about liveness and readiness."
date:       2019-10-28 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-build-k8s.jpg"
tags:
    - kubernetes
---

## liveness与readiness的探针工作方式源码解析

liveness和readiness作为k8s的探针，可以对应用进行健康探测。
二者支持的探测方式相同。主要的探测方式支持http探测，执行命令探测，以及tcp探测。
探测均是由kubelet执行。

#### 执行命令探测

```go
func (pb *prober) runProbe(p *v1.Probe, pod *v1.Pod, status v1.PodStatus, container v1.Container, containerID kubecontainer.ContainerID) (probe.Result, string, error) {
.....        
        command := kubecontainer.ExpandContainerCommandOnlyStatic(p.Exec.Command, container.Env)
		return pb.exec.Probe(pb.newExecInContainer(container, containerID, command, timeout))
......
        
func (pb *prober) newExecInContainer(container v1.Container, containerID kubecontainer.ContainerID, cmd []string, timeout time.Duration) exec.Cmd {
	return execInContainer{func() ([]byte, error) {
		return pb.runner.RunInContainer(containerID, cmd, timeout)
	}}
}
        
......
func (m *kubeGenericRuntimeManager) RunInContainer(id kubecontainer.ContainerID, cmd []string, timeout time.Duration) ([]byte, error) {
	stdout, stderr, err := m.runtimeService.ExecSync(id.ID, cmd, 0)
	return append(stdout, stderr...), err
}
```

由kubelet，通过CRI接口的ExecSync接口，在对应容器内执行拼装好的cmd命令。获取返回值。

```go
func (pr execProber) Probe(e exec.Cmd) (probe.Result, string, error) {
	data, err := e.CombinedOutput()
	glog.V(4).Infof("Exec probe response: %q", string(data))
	if err != nil {
		exit, ok := err.(exec.ExitError)
		if ok {
			if exit.ExitStatus() == 0 {
				return probe.Success, string(data), nil
			} else {
				return probe.Failure, string(data), nil
			}
		}
		return probe.Unknown, "", err
	}
	return probe.Success, string(data), nil
}
```

kubelet是根据执行命令的退出码来决定是否探测成功。当执行命令的退出码为0时，认为执行成功，否则为执行失败。如果执行超时，则状态为Unknown。

#### http探测

```go
func DoHTTPProbe(url *url.URL, headers http.Header, client HTTPGetInterface) (probe.Result, string, error) {
	req, err := http.NewRequest("GET", url.String(), nil)
	......
    if res.StatusCode >= http.StatusOK && res.StatusCode < http.StatusBadRequest {
		glog.V(4).Infof("Probe succeeded for %s, Response: %v", url.String(), *res)
		return probe.Success, body, nil
	}
	......
```

http探测是通过kubelet请求容器的指定url，并根据response来进行判断。
当返回的状态码在200到400(不含400)之间时，也就是状态码为2xx和3xx，认为探测成功。否则认为失败。


#### tcp探测 

```go
func DoTCPProbe(addr string, timeout time.Duration) (probe.Result, string, error) {
	conn, err := net.DialTimeout("tcp", addr, timeout)
	if err != nil {
		// Convert errors to failures to handle timeouts.
		return probe.Failure, err.Error(), nil
	}
	err = conn.Close()
	if err != nil {
		glog.Errorf("Unexpected error closing TCP probe socket: %v (%#v)", err, err)
	}
	return probe.Success, "", nil
}
```

tcp探测是通过探测指定的端口。如果可以连接，则认为探测成功，否则认为失败。

#### 探测失败的可能原因

执行命令探测失败的原因主要可能是容器未成功启动，或者执行命令失败。当然也可能docker或者docker-shim存在故障。

由于http和tcp都是从kubelet自node节点上发起的，向容器的ip进行探测。
所以探测失败的原因除了应用容器的问题外，还可能是**从node到容器ip的网络不通**。

## liveness与readiness的原理区别

探测方式相同，那么liveness与readiness有什么区别？首先，二者能够起到的作用不同。

```go
func (m *kubeGenericRuntimeManager) computePodContainerChanges(pod *v1.Pod, podStatus *kubecontainer.PodStatus) podContainerSpecChanges {
        ......
        liveness, found := m.livenessManager.Get(containerStatus.ID)
		if !found || liveness == proberesults.Success {
			changes.ContainersToKeep[containerStatus.ID] = index
			continue
		}
        ......
```

liveness主要用来确定何时重启容器。liveness探测的结果会存储在livenessManager中。
kubelet在syncPod时，发现该容器的liveness探针检测失败时，会将其加入待启动的容器列表中，在之后的操作中会重新创建该容器。

readiness主要来确定容器是否已经就绪。只有当Pod中的容器都处于就绪状态，也就是pod的condition里的Ready为true时，kubelet才会认定该Pod处于就绪状态。而pod是否处于就绪状态的作用是控制哪些Pod应该作为service的后端。如果Pod处于非就绪状态，那么它们将会被从service的endpoint中移除。

```go
func (m *manager) SetContainerReadiness(podUID types.UID, containerID kubecontainer.ContainerID, ready bool) {
	    ......
    	containerStatus.Ready = ready
        ......
    	readyCondition := GeneratePodReadyCondition(&pod.Spec, status.ContainerStatuses, status.Phase)
    	......
    	m.updateStatusInternal(pod, status, false)
}
```

readiness检查结果会通过SetContainerReadiness函数，设置到pod的status中，从而更新pod的ready condition。

liveness和readiness除了最终的作用不同，另外一个很大的区别是它们的初始值不同。

```go
    switch probeType {
	case readiness:
		w.spec = container.ReadinessProbe
		w.resultsManager = m.readinessManager
		w.initialValue = results.Failure
	case liveness:
		w.spec = container.LivenessProbe
		w.resultsManager = m.livenessManager
		w.initialValue = results.Success
	}
```

liveness的初始值为成功。这样防止在应用还没有成功启动前，就被误杀。如果在规定时间内还未成功启动，才将其设置为失败，从而触发容器重建。

而readiness的初始值为失败。这样防止应用还没有成功启动前就向应用进行流量的导入。如果在规定时间内启动成功，才将其设置为成功，从而将流量向应用导入。

liveness与readiness二者作用不能相互替代。

例如只配置了liveness，那么在容器启动，应用还没有成功就绪之前，这个时候pod是ready的（因为容器成功启动了）。那么流量就会被引入到容器的应用中，可能会导致请求失败。虽然在liveness检查失败后，重启容器，此时pod的ready的condition会变为false。但是前面会有一些流量因为错误状态导入。

当然只配置了readiness是无法触发容器重启的。

因为二者的作用不同，在实际使用中，可以根据实际的需求将二者进行配合使用。