---
layout:     post
title:      "kubernetes版本融合解决方案"
subtitle:   "kubernetes version fusion solution."
date:       2018-9-3 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-k8s-version-fusion.jpg"
tags:
    - kubernetes
---

# kubernetes版本融合背景

在kubernetes 1.6版本的基础上进行了深度的定制。而且该版本已经相当稳定。但是随着kubernetes版本迭代，后期使用的如service mesh/kubeflow项目依赖于高版本的kubernetes，比如1.8或者1.10以上的版本。这样就产生了一定的矛盾。直接将1.10的k8s合并到1.6上，成本很高，难度也很大。因此需要其他方案进行版本融合。

# 融合方案思路与设计

k8s的资源可以分为两类，一类是核心资源，如pod/svc/node/ep/rc等。这些资源是k8s的安身立命之本，也是最初的k8s就有的基础资源。节点上的服务如kubelet、kubeproxy主要关心的也是这些资源。另一类是高级资源，如statefulset/deployment等等，这些资源并不会直接影响到节点，而是通过controller影响核心资源如pod的变化，进而反映到节点上的容器等的变化。基础资源的api自1.5之后就比较稳定了，较少有改动。融合方案设计的主要思路也是基于此。

方案设计如下图所示：

```
echo "[kubeflow]- crd ->[k8s 1.10 api]->[etcd] [kubelet],[kube-controller],[kube-scheduler]->[k8s 1.6 api]->[etcd]" | graph-easy
```

```Shell
                        +----------------+
                        | kube-scheduler |
                        +----------------+
                          |
                          |
                          v
+-----------------+     +----------------+     +------+
| kube-controller | --> |  k8s 1.6 api   | --> | etcd |
+-----------------+     +----------------+     +------+
                          ^                      ^
                          |                      |
                          |                      |
+-----------------+     +----------------+       |
|    kubeflow     |     |    kubelet     |       |
+-----------------+     +----------------+       |
  |                                              |
  | crd                                          |
  v                                              |
+-----------------+                              |
|  k8s 1.10 api   | -----------------------------+
+-----------------+
```

部署k8s 1.6和1.10两个版本的api，二者共用同一个etcd。kubelet、controller、scheduler等接入k8s 1.6 api不变。

以kubeflow为例。kubeflow主要使用的是cdr、pod、svc资源。kubeflow通过1.10 api创建相关资源。

该方案在实际部署中会遇到一些问题。比如通过1.10创建了额外的其他资源，如rs、deployment等。由于其版本信息与1.6的不同，会导致controller无法正常工作。因此需要屏蔽从1.10创建除crd之外的资源。这里我采用了ABAC的授权模式进行了屏蔽，这样就无法使用1.10创建rs等资源。

```
--authorization-mode=ABAC  --authorization-policy-file=/etc/kubernetes/policy
```

其中授权文件样例如下：

```
[root@host-1 ~]# cat /etc/kubernetes/policy 
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "namespace": "*", "resource": "*","apiGroup": ""}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "namespace": "*", "resource": "ingresses","apiGroup": "extensions"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "namespace": "*", "resource": "*","apiGroup": "apiextensions.k8s.io"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "namespace": "*", "resource": "*","apiGroup": "config.istio.io"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "namespace": "*", "resource": "*","apiGroup": "kubeflow.org"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "namespace": "*", "resource": "*","apiGroup": "networking.istio.io"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "namespace": "*", "resource": "*","apiGroup": "rbac.istio.io"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "namespace": "*", "resource": "*","apiGroup": "authentication.istio.io"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "nonResourcePath": "*", "readonly": true}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:unauthenticated", "nonResourcePath": "*", "readonly": true}}
```

# 后记

这里kubeflow具有一定的特殊性，就是只用到了crd和pod等基础资源。假定某个新的服务service不仅用到了这些，还用到了1.10版本的rs、deployment、crontab等高级资源。则上述方案就不能使用了。但是依然应该有办法可以解决。


```
echo "[service]->[k8s 1.10 api]- rs/deployment ->[etcd 1.10] [k8s 1.10 api]- pod/svc/node ->[etcd 1.6]  [kube-controller 1.10]->[k8s 1.10 api] [kubelet],[kube-controller 1.6],[kube-scheduler]->[k8s 1.6 api]->[etcd 1.6]" | graph-easy
```

```Shell
                                                      +----------------------------------------------+
                                                      |                                              |
                       +----------------------+     +--------------+  rs/deployment   +-----------+  |
                       | kube-controller 1.10 | --> | k8s 1.10 api | ---------------> | etcd 1.10 |  |
                       +----------------------+     +--------------+                  +-----------+  |
                                                      ^                                              |
                                                      |                                              |
                                                      |                                              |
                       +----------------------+     +--------------+                                 |
                       |       kubelet        |     |   service    |                                 |
                       +----------------------+     +--------------+                                 |
                         |                                                                           |
                         |                                                                           |
                         v                                                                           |
+----------------+     +----------------------+     +--------------+  pod/svc/node                   |
| kube-scheduler | --> |     k8s 1.6 api      | --> |   etcd 1.6   | <-------------------------------+
+----------------+     +----------------------+     +--------------+
                         ^
                         |
                         |
                       +----------------------+
                       | kube-controller 1.6  |
                       +----------------------+
```

k8s依然部署1.10/1.6两套api，同时也部署etcd 1.10和1.6两套。
其中k8s 1.10的api中将资源分隔，配置时将pod/svc/node等基础资源指向etcd 1.6，将rs/deployment指向etcd 1.10。这样可以达到基础资源的共用。
同时，部署两套kube-controller 1.10和1.6。在1.10中，禁用svc、node等指向基础资源的controller，防止冲突。另外，其在svc上注册的kube-controller，也需要更名。

service依然可以通过k8s 1.10的api创建相关资源。rs等资源也将由kube-controller 1.10进行管控。


这个方案我并没有去部署调试过，但是可以预见的是理论上是可行的。这里面可能会有一些坑，目前我可以预见的坑主要是两者管理时，容器的标签要做正确规划，以免出现冲突。另外pod上会使用ownerReference标明其所属的rs资源等。因此要在controller中相关的地方进行一下定制。