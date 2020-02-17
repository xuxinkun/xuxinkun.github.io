---
layout:     post
title:      "pod删除主要流程源码解析"
subtitle:   "Pod delete main process source code analysis."
date:       2019-11-24 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-2015.jpg"
tags:
    - kubernetes
---

> 本文以v1.12版本进行分析

当一个pod删除时，client端向apiserver发送请求，apiserver将pod的deletionTimestamp打上时间。kubelet watch到该事件，开始处理。

#### syncLoop

kubelet对pod的处理主要都是在syncLoop中处理的。

```
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
for {
...
		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
...
```

与pod删除主要在syncLoopIteration中需要关注的是以下这两个。

```
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
	case u, open := <-configCh:
...
		switch u.Op {
...
		case kubetypes.UPDATE:
			handler.HandlePodUpdates(u.Pods)
...
	case <-housekeepingCh:
		if !kl.sourcesReady.AllReady() {
		} else {
			if err := handler.HandlePodCleanups(); err != nil {
				glog.Errorf("Failed cleaning pods: %v", err)
			}
		}
	}
```

第一个是由于发送给apiserver的DELETE请求触发的，增加了deletionTimestamp的事件。这里对应于kubetypes.UPDATE。也就是会走到HandlePodUpdates函数。

另外一个与delete相关的是每2s执行一次的来自于housekeepingCh的定时事件，用于清理pod，执行的是handler.HandlePodCleanups函数。这两个作用不同，下面分别进行介绍。

#### HandlePodUpdates

先看HandlePodUpdates这个流程。只要打上了deletionTimestamp，就必然走到这个流程里去。

```
func (kl *Kubelet) HandlePodUpdates(pods []*v1.Pod) {
	for _, pod := range pods {
...
		kl.dispatchWork(pod, kubetypes.SyncPodUpdate, mirrorPod, start)
	}
}
```

在HandlePodUpdates中，进而将pod的信息传递到dispatchWork中处理。

```
func (kl *Kubelet) dispatchWork(pod *v1.Pod, syncType kubetypes.SyncPodType, mirrorPod *v1.Pod, start time.Time) {
	if kl.podIsTerminated(pod) {
		if pod.DeletionTimestamp != nil {
			kl.statusManager.TerminatePod(pod)
		}
		return
	}
	// Run the sync in an async worker.
	kl.podWorkers.UpdatePod(&UpdatePodOptions{
		Pod:        pod,
		MirrorPod:  mirrorPod,
		UpdateType: syncType,
		OnCompleteFunc: func(err error) {
...
```

这里首先通过判断了kl.podIsTerminated(pod)判断pod是不是已经处于了Terminated状态。如果是的话，则不进行下面的kl.podWorkers.UpdatePod。

```
func (kl *Kubelet) podIsTerminated(pod *v1.Pod) bool {
	status, ok := kl.statusManager.GetPodStatus(pod.UID)
	if !ok {
		status = pod.Status
	}
	return status.Phase == v1.PodFailed || status.Phase == v1.PodSucceeded || (pod.DeletionTimestamp != nil && notRunning(status.ContainerStatuses))
}
```

这个地方特别值得注意的是，并不是由了DeletionTimestamp就会认为是Terminated状态，而是有DeletionTimestamp且所有的容器不在运行了。也就是说如果是一个正在正常运行的pod，是也会走到kl.podWorkers.UpdatePod中的。UpdatePod通过一系列函数调用，最终会通过异步的方式执行syncPod函数中进入到syncPod函数中。

```
func (kl *Kubelet) syncPod(o syncPodOptions) error {
...
	if !runnable.Admit || pod.DeletionTimestamp != nil || apiPodStatus.Phase == v1.PodFailed {
		var syncErr error
		if err := kl.killPod(pod, nil, podStatus, nil); err != nil {
...
```

在syncPod中，调用killPod从而对pod进行停止操作。

#### killPod

killPod是停止pod的主体。在很多地方都会使用。这里主要介绍下主要的工作流程。停止pod的过程主要发生在killPodWithSyncResult函数中。

```
func (m *kubeGenericRuntimeManager) killPodWithSyncResult(pod *v1.Pod, runningPod kubecontainer.Pod, gracePeriodOverride *int64) (result kubecontainer.PodSyncResult) {
	killContainerResults := m.killContainersWithSyncResult(pod, runningPod, gracePeriodOverride)
...
	for _, podSandbox := range runningPod.Sandboxes {
			if err := m.runtimeService.StopPodSandbox(podSandbox.ID.ID); err != nil {
...
```

killPodWithSyncResult的主要工作分为两个部分。killContainersWithSyncResult负责将pod中的container停止掉，在停止后再执行StopPodSandbox。

```
func (m *kubeGenericRuntimeManager) killContainer(pod *v1.Pod, containerID kubecontainer.ContainerID, containerName string, reason string, gracePeriodOverride *int64) error {
	if err := m.internalLifecycle.PreStopContainer(containerID.ID); err != nil {
		return err
	}
...
	err := m.runtimeService.StopContainer(containerID.ID, gracePeriod)
```

killContainersWithSyncResult的主要工作是在killContainer中完成的，这里可以看到，其中的主要两个步骤是在容器中进行prestop的操作。待其成功后，进行container的stop工作。至此所有的应用容器都已经停止了。下一步是停止pause容器。而StopPodSandbox就是执行这一过程的。将sandbox，也就是pause容器停止掉。StopPodSandbox是在dockershim中执行的。

```
func (ds *dockerService) StopPodSandbox(ctx context.Context, r *runtimeapi.StopPodSandboxRequest) (*runtimeapi.StopPodSandboxResponse, error) {
...
if !hostNetwork && (ready || !ok) {
...
		err := ds.network.TearDownPod(namespace, name, cID, annotations)
...
	}
	if err := ds.client.StopContainer(podSandboxID, defaultSandboxGracePeriod); err != nil {
```

StopPodSandbox中主要的部分是先进行网络卸载，再停止相应的容器。在完成StopPodSandbox后，至此pod的所有容器都已经停止完成。至于volume的卸载，是在volumeManager中进行的。本文不做单独介绍了。停止后的容器在pod彻底清理后，会被gc回收。这里也不展开讲了。

#### HandlePodCleanups

上面这个流程并不是删除流程的全部。一个典型的情况就是，如果container都不是running，那么在UpdatePod的时候都return了，那么又由谁来处理呢？这里我们回到最开始，就是那个每2s执行一次的HandlePodCleanups的流程。也就是说比如container处于crash，container正好不是running等情况，其实是在这个流程里进行处理的。当然HandlePodCleanups的作用不仅仅是清理not running的pod，再比如数据已经在apiserver中强制清理掉了，或者由于其他原因这个节点上还有一些没有完成清理的pod，都是在这个流程中进行处理。

```
func (kl *Kubelet) HandlePodCleanups() error {
...	
	for _, pod := range runningPods {
		if _, found := desiredPods[pod.ID]; !found {
			kl.podKillingCh <- &kubecontainer.PodPair{APIPod: nil, RunningPod: pod}
		}
	}
```

runningPods是从cache中获取节点现有的pod，而desiredPods则是节点上应该存在未被停止的pod。如果存在runningPods中有而desiredPods中没有的pod，那么它应该被停止，所以发送到podKillingCh中。

```
func (kl *Kubelet) podKiller() {
...
	for podPair := range kl.podKillingCh {
...

		if !exists {
			go func(apiPod *v1.Pod, runningPod *kubecontainer.Pod) {
				glog.V(2).Infof("Killing unwanted pod %q", runningPod.Name)
				err := kl.killPod(apiPod, runningPod, nil, nil)
...
			}(apiPod, runningPod)
		}
	}
}
```

在podKiller流程中，会去接收来自podKillingCh的消息，从而执行killPod，上文已经做了该函数的介绍了。

#### statusManager

在最后，statusManager中的syncPod流程，将会进行检测，通过canBeDeleted确认是否所有的容器关闭了，volume卸载了，cgroup清理了等等。如果这些全部完成了，则从apiserver中将pod信息彻底删除。

```
func (m *manager) syncPod(uid types.UID, status versionedPodStatus) {
...
	if m.canBeDeleted(pod, status.status) {
		deleteOptions := metav1.NewDeleteOptions(0)
		deleteOptions.Preconditions = metav1.NewUIDPreconditions(string(pod.UID))
		err = m.kubeClient.CoreV1().Pods(pod.Namespace).Delete(pod.Name, deleteOptions)
...
```