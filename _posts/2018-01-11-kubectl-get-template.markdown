---
layout:     post
title:      "使用go-template自定义kubectl get输出"
subtitle:   "Use go-template to customize kubectl get output."
date:       2018-1-11 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-kubectl-get-template.jpg"
tags:
    - kubernetes
---


# kubectl get

`kubectl get`相关资源，默认输出为kubectl内置，一般我们也可以使用`-o json`或者`-o yaml`查看其完整的资源信息。但是很多时候，我们需要关心的信息并不全面，因此我们需要自定义输出的列，那么可以使用`go-template`来进行实现。

`go-template`是golang的一种模板，可以参考[template的相关说明](https://golang.org/pkg/text/template/)。

比如仅仅想要查看获取的pods中的各个pod的uid，则可以使用以下命令：

```
[root@node root]# kubectl get pods --all-namespaces -o go-template --template='{{range .items}}{{.metadata.uid}}
{{end}}'
0313ffff-f1f4-11e7-9cda-40f2e9b98448
ee49bdcd-f1f2-11e7-9cda-40f2e9b98448
f1e0eb80-f1f2-11e7-9cda-40f2e9b98448
```


```
[root@node-106 xuxinkun]# kubectl get pods -o yaml 
apiVersion: v1
items:
- apiVersion: v1
  kind: Pod
  metadata:
    name: nginx-deployment-1751389443-26gbm
    namespace: default
    uid: a911e34b-f445-11e7-9cda-40f2e9b98448
  ...
- apiVersion: v1
  kind: Pod
  metadata:
    name: nginx-deployment-1751389443-rsbkc
    namespace: default
    uid: a911d2d2-f445-11e7-9cda-40f2e9b98448
  ...
- apiVersion: v1
  kind: Pod
  metadata:
    name: nginx-deployment-1751389443-sdbkx
    namespace: default
    uid: a911da1a-f445-11e7-9cda-40f2e9b98448
  	...
kind: List
metadata: {}
resourceVersion: ""
```

因为get pods的返回结果是List类型，获取的pods都在items这个的value中，因此需要遍历items，也就有了`{{range .items}}`。而后通过模板选定需要展示的内容，就是items中的每个{{.metadata.uid}}

这里特别注意，要做一个特别的处理，就是要把`{{end}}`前进行换行，以便在模板中插入换行符。

当然，如果觉得这样处理不优雅的话，也可以使用`printf`函数，在其中使用`\n`即可实现换行符的插入

```
[root@node root]# kubectl get pods --all-namespaces -o go-template --template='{{range .items}}{{printf "%s\n" .metadata.uid}}{{end}}'
```

其实有了`printf`，就可以很容易的实现对应字段的输出，且样式可以进行自己控制。比如可以这样

```
[root@node root]# kubectl get pods --all-namespaces -o go-template --template='{{range .items}}{{printf "|%-20s|%-50s|%-30s|\n" .metadata.namespace .metadata.name .metadata.uid}}{{end}}'
|console             |console-4d2d7eab-1377218307-7lg5v                 |0313ffff-f1f4-11e7-9cda-40f2e9b98448|
|console             |console-4d2d7eab-1377218307-q3bjd                 |ee49bdcd-f1f2-11e7-9cda-40f2e9b98448|
|cxftest             |ipreserve-f15257ec-3788284754-nmp3x               |f1e0eb80-f1f2-11e7-9cda-40f2e9b98448|
```


除了使用`go-template`之外，还可以使用`go-template-file`。则模板不用通过参数传进去，而是写成一个文件，然后需要指定`template`指向该文件即可。

```
[root@node root]#  kubectl get pods --all-namespaces -o go-template-file --template=/home/template_file
[root@node root]# cat /home/template_file
{{range .items}}{{printf "%s\n" .metadata.uid}}{{end}}
```