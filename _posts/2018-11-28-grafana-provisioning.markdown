---
layout:     post
title:      "使用grafana provisioning通过配置方式添加datasource和dashboard"
subtitle:   "Use grafana provisioning to add datasources and dashboards."
date:       2018-11-27 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-grafana-provisioning.jpg"
tags:
    grafana
---

# grafana provisioning

grafana provisioning (http://docs.grafana.org/administration/provisioning/#provisioning-grafana)是grafana 5.0后引入的功能，用以支持通过配置的方式进行datasource和dashboard的配置。

要开启该功能，首先要在grafana的配置中增加provisioning的选项(http://docs.grafana.org/installation/configuration/#provisioning)。
即在grafana.ini中增加

```
[paths]
# folder that contains provisioning config files that grafana will apply on startup and while running.
;provisioning = /etc/grafana/provisioning
```

而后在/etc/grafana/provisioning中增加`dashboards`和`datasources`文件夹。

```
[root@local provisioning]# ll
total 0
drwxr-xr-x 2 root grafana 25 Nov 28 03:09 dashboards
drwxr-xr-x 2 root grafana 25 Nov 28 03:09 datasources
```

# datasources

datasource只支持静态配置，即，在datasources中配置好后，grafana启动时候将会进行加载。在grafana启动后在加入该文件夹，需要重启才能生效。

datasoures文件夹下需要放置对应的datasource的yaml文件，这里以`sample.yaml`为例：

```
[root@local provisioning]# cat datasources/sample.yaml 
apiVersion: 1
deleteDatasources:
 - name: influxdb
   orgId: 1
datasources:
 - id: 17
   orgId: 1
   name: influxdb
   type: influxdb
   typeLogoUrl: ''
   access: proxy
   url: http://localhost:8086
   password: root
   user: root
   database: clustersch
   basicAuth: false
   basicAuthUser: ''
   basicAuthPassword: ''
   withCredentials: false
   isDefault: false
   jsonData:
     keepCookies: []
   secureJsonFields: {}
   version: 4
   readOnly: false
```

可以看到yaml分为三部分，`apiVersion`是固定的。`deleteDatasources`是启动时候将会首先从数据库中删除的datasource的名称。通过provisioning加载datasource无法从页面进行删除，只能在`deleteDatasources`中进行删除。
再一部分就是`datasources`，是一个列表，用以表示不同的datasource。这里以influxdb为例。其他的也类似，具体可以参考其他datasource的参数说明。

# dashboards

不同于datasource，dashboards是支持动态加载的。这里介绍一个标准样例。

```
[root@local provisioning]# cat dashboards/sample.yaml 
apiVersion: 1
providers:
 - name: 'default'
   orgId: 1
   folder: ''
   type: file
   updateIntervalSeconds: 10
   options:
     path: /tmp/grafana
```

`apiVersion`是固定字段。providers是一个列表，用来存储不同的dashboard源。这里主要介绍从本机某个路径加载dashboard。`updateIntervalSeconds`是指动态加载的刷新频率，也就是10s进行一次刷新，从`/tmp/grafana`中读取所有的dashboard配置，然后将其添加或者更新到grafana中。

在`/tmp/grafana`中，只需要将dashboard的json文件丢到里面去就可以了。grafana会自动加载。json文件就是从grafana的dashboard中导出的文件即可。注意一下相关`datasource`的配置。

```
[root@local provisioning]# ll /tmp/grafana/test.json 
-rw-r--r-- 1 root root 24126 Nov 28 03:10 /tmp/grafana/test.json
```
