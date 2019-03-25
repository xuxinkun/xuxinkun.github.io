---
layout:     post
title:      "我的开源项目与社区提交"
subtitle:   "My open source projects and merge requests."
date:       2019-1-1 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-2015.jpg"
tags:
---


# 开源项目

### kubesql

- 项目地址：[https://github.com/xuxinkun/kubesql](https://github.com/xuxinkun/kubesql)
- 项目介绍：kubesql是一个使用sql查询kubernetes资源的工具。诸如node，pod等kubernetes的资源被处理为table，而后可以使用sql语句对其进行查询。

### littleTools

- 项目地址：[https://github.com/xuxinkun/littleTools](https://github.com/xuxinkun/littleTools)
- 项目介绍：根据日常运维时编写的一个小工具，主要用于简化命令docker和kubectl的输入，使用shell函数编写，支持自动补全。

### kubernetes-ansible

- 项目地址：[https://github.com/xuxinkun/kubernetes-ansible](https://github.com/xuxinkun/kubernetes-ansible)
- 项目介绍：使用ansible部署k8s各个组件。

### docker-rpm-centos6

- 项目地址：[https://github.com/xuxinkun/docker-rpm-centos6](https://github.com/xuxinkun/docker-rpm-centos6)
- 项目介绍：主要用于将docker编译打包为rpm包

# 社区提交

### ansible

[connection plugin kubectl for kubernetes. https://github.com/ansible/ansible/pull/26668](https://github.com/ansible/ansible/pull/26668)

[use docker exec instead of docker cp.  https://github.com/ansible/ansible/pull/26571](https://github.com/ansible/ansible/pull/26571)

### kubernetes

#### kubernetes

[fix kube::log::error in start_rsyncd_container. https://github.com/kubernetes/kubernetes/pull/37920](https://github.com/kubernetes/kubernetes/pull/37920)

[fix systemd service file for custom args. https://github.com/kubernetes/kubernetes/pull/47894](https://github.com/kubernetes/kubernetes/pull/47894)

#### website

[Add more options for self-registration. https://github.com/kubernetes/website/pull/2567](https://github.com/kubernetes/website/pull/2567)

[Add link for case studies. https://github.com/kubernetes/website/pull/4269](https://github.com/kubernetes/website/pull/4269)

### docker

#### compose 

[Add cpuset config. https://github.com/docker/compose/pull/1331](https://github.com/docker/compose/pull/1331)

#### runc

[fix cpu.cfs_quota_us changed when systemd daemon-reload using systemd.  https://github.com/opencontainers/runc/pull/1344](https://github.com/opencontainers/runc/pull/1344)

### swagger-codegen

[[Java][okhttp] fix handleResponse to not leak okhttp connections https://github.com/swagger-api/swagger-codegen/pull/4997](https://github.com/swagger-api/swagger-codegen/pull/4997)

### helm-charts

[move datasources to init container. https://github.com/helm/charts/pull/9842](https://github.com/helm/charts/pull/9842)

### k8s-sidecar

[Added list feature for sidecar. https://github.com/kiwigrid/k8s-sidecar/pull/12](https://github.com/kiwigrid/k8s-sidecar/pull/12)

### harbor

[fix responses for /repositories/tags https://github.com/goharbor/harbor/pull/1029](https://github.com/goharbor/harbor/pull/1029)

### zh-google-styleguide

[update the Google Style Guide home page. https://github.com/zh-google-styleguide/zh-google-styleguide/pull/24](https://github.com/zh-google-styleguide/zh-google-styleguide/pull/24)

[update to v2.59. https://github.com/zh-google-styleguide/zh-google-styleguide/pull/5](https://github.com/zh-google-styleguide/zh-google-styleguide/pull/5)