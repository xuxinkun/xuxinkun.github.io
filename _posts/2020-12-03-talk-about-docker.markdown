---
layout:     post
title:      "漫话docker的衰落与kubernetes的兴起"
subtitle:   "Talk about the decline of docker."
date:       2020-12-03 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-2015.jpg"
tags:
    - kubernetes
    - docker
---

## 写在最前

伴随着kubernetes 1.20中对于docker的弃用，关于docker的灭亡与kubernetes的兴起的话题再度热了起来。讨论中关于docker灭亡的观点我不敢苟同。docker还远未到达灭亡的程度。相较而言，我觉得更恰当的说法应该是docker的衰落。本文我也就我个人的角度，聊聊我所经历的docker的衰落与kubernetes的兴起。

## 横空出世——docker的兴起

第一次接触docker，是2014年。当时OpenStack的主要负载还是kvm。而我们也尝试过了更为轻量的lxc，但是以失败而告终了。docker这个集装箱的小图标配上docker container的理念，一下子就吸引住了大家的目光。经历过制作lxc镜像的痛苦，你就会更体会到docker的可贵。繁琐的lxc镜像制作与精简的Dockerfile相比，孰高孰低、孰优孰劣可谓是一目了然。

docker成功的将cgroup、union filesystem、namespace这些较为稳定和成熟的技术结合了起来，辅以docker image的制作工艺，实现了集装箱式的标准交付。这时候的docker，颇有种“举天下之豪杰而莫能与之争”的气势。虽然在生产环节还是或多或少，还有这样那样的问题，但是docker已经跨过了POC（Proof of concept）阶段，进入了pre-product的行列了。在对docker深度定制后，最终我们团队也将OpenStack + docker的组合成功推向了生产。

![docker](https://xuxinkun.github.io/img/talk-about-docker/docker.png)

那两年，知不知道docker、会不会做镜像、懂不懂docker原理成为基础架构领域面试的常见话题。虽然只有少数几个公司敢为天下先，将docker搬上了生产，但是已经没有人可以忽略这颗冉冉升起的新星了。那两年，活跃在各个会议、论坛上的都是docker的话题。大家热衷于讨论生产上docker遇到的坑。大家各出奇招，修修补补，跌跌撞撞，docker总算也是被搬上了生产。而在这时，即使是技术保守、持徘徊观望态度的公司，也都会安排一些人力着手跟进docker的发展与各个公司的实践经验了，这时候的docker真的是风头无两。

## 生来巨人——kubernetes

时间到了2015年，此时我转而负责进行CaaS（Container as a Service）服务的调研。这时候大家都在说CaaS，但是每个人说的都不一样，其实大家都是摸着石头过河。在此期间，以研究OpenStack的magnum为契机，我接触到了swarm和kubernetes。

![swarm](https://xuxinkun.github.io/img/talk-about-docker/swarm.jpg)

swarm是docker公司力推的集群管理方案。docker、swarm和compose组成的三剑客完整覆盖了运行时、集群管理与编排，构成了一个看起来牢不可摧的生态系统。特别是别出心裁的将swarm的api与docker的api进行了拉齐，将集群的管理复杂度与单节点的管理复杂度向用户进行屏蔽，倒是有一种如臂使指的快感。这个设计直到现在我都还觉得立意真的很精巧。

![kubernetes](https://xuxinkun.github.io/img/talk-about-docker/kubernetes.png)

而初出茅庐的kubernetes也来势汹汹。背靠Google的大旗，有Borg的背书，kubernetes在气势上一点不输swarm + compose的组合。伴随着kubernetes 1.0的发布，kubernetes也从幕后走向了台前，开始大规模接受来自全世界的PR提交，在功能、性能和稳定性上快速提升。

到kubernetes 1.2版本，经过我们内部评估，已经具备生产级的品质。而声明式API、简洁的架构、灵活的标签等优秀的设计，在做选型时已经让我们完全没有理由拒绝。而后经过数月紧张的开发，kubernetes + docker的组合被搬上了舞台，并且以极快的速度侵蚀OpenStack + docker的份额。此时的docker已经开始被限制为了容器的运行时和镜像制作工具。捆住了docker的手脚，kubernetes已经没有了可以掰手腕的对手，一统江湖的路上kubernetes再无障碍。

## 美人迟暮——docker的衰落

时至今日，docker的衰落已经成为了不争的事实。而docker的衰落我觉得是多方面的原因。一部分是docker自身的封闭和固执己见。我曾经记得在当时参与了当时社区多个PR的讨论。新增一个feature的超长的周期，已经可以磨掉多数人的耐心。docker社区在多个观点上略显保守的方式，让大家逐渐失去了参与的热情。

另一方面，也是更为重要的一点，是容器技术本身的门槛已经被突破，已经无法形成技术上的护城河。而对于**容器技术之争，已经转化为标准之争**。而对于标准上更有发言权的一方，无疑是具有更多用户、更庞大社区和更强大的平台的一方。在短短两三年的时间内，天平就快速地向kubernetes倾斜，其主导的CRI、CNI、CSI标准已经成为了事实上的通行标准。而相较之下，docker力推的CNM等标准则显得曲高和寡。

![k8s](https://xuxinkun.github.io/img/talk-about-docker/k8s.png)

与此同时，kubernetes并没有放弃主动的进攻。kubernetes的强势和扶植其他容器运行时加速了docker的衰落。在1.6版本弃用docker manager直连docker，转向CRI + dockershim的组合时，就注定了kubernetes会走到完全解耦docker，也就是今天这一步。与此同时，其他容器运行时也开始了瓜分市场份额。如果说gVisor、Kata只是尝试挑战docker在部分场景中的地位，那么红帽的Podman则已经吹响了全面进攻的号角。再结合前两年docker的一些融资和收购传闻，平添了一种英雄末路、美人迟暮的伤感。

## 世间多少英雄戏，每到收场总伤神

六年回望，其实无论是在当时还是现在来看，docker都是一个革命性的产品。**docker的衰落并不是意味着容器运行时不重要了，而是大家越来越习以为常了。**时至今日，容器运行时作为一个大部已经被解决的问题，一个相对成熟的模块，已经成为了整个基础架构体系的一部分。而作为上层的平台和更为上层的用户来说，对此将会给予越来越少的关注。这就像现在大多数用户并不会去关心内核了一样。甚至在未来，我预测运行时都有可能会成为一个内核级别的附属模块，会预装到许多的发行版上，其运行也逐渐对大多数用户变得更加透明(实际红帽已经开始向这个方向做了)。而**越来越多的用户则会将更多的注意力集中在上层的交付、管理、编排上**。至于kubernetes会不会衰落，我觉得在中短期内（五年内）不会。kubernetes已经成为了一个平台级的项目。在这点上，kubernetes作为平台将比工具性质的docker具有更强的生命力。