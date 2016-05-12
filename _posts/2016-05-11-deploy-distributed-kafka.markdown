---
layout:     post
title:      "Kafka分布式集群搭建"
subtitle:   "Deploy Distributed kafka cluster"
date:       2016-05-10 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-dis-kafka.jpg"
tags:
    - kafka
    - confluent
---

# 环境说明

kafka自0.9之后增加了connector的特性。本文主要是搭建一个分布式的kafka connector和broker。

本文用了三台机器进行部署，使用centos 6.6。

| hostname | ip         | role                                     |
| -------- | ---------- | ---------------------------------------- |
| node1    | 10.8.65.63 | zk + kafak broker + schema-registry + kafka connector |
| node2    | 10.8.65.60 | kafak broker + kafka connector           |
| node3    | 10.8.65.62 | kafak broker + kafka connector           |


# 安装

先安装jdk/mysql/confluent等组件

```shell
yum install -y wget curl mysql mysql-server lokkit

cd /root

## 安装jdk
wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie"  http://download.oracle.com/otn-pub/java/jdk/8u91-b14/jdk-8u91-linux-x64.rpm
rpm -ivh jdk-8u91-linux-x64.rpm
## 安装jdbc-mysql-driver
wget http://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.39.tar.gz
tar xzvf mysql-connector-java-5.1.39.tar.gz
sed -i '$a export CLASSPATH=/root/mysql-connector-java-5.1.39/mysql-connector-java-5.1.39-bin.jar:$CLASSPATH' /etc/profile
source /etc/profile
## 安装confluent
cd /usr/local
wget http://packages.confluent.io/archive/2.0/confluent-2.0.1-2.11.7.tar.gz 
tar xzvf confluent-2.0.1-2.11.7.tar.gz
```

# 部署

## 准备

### 声明变量
```sh
## node1
export role_num=1
## node2上
export role_num=2
## node3上
export role_num=3
```

### 设置hostname和hosts

```sh
##　把hostname设置为对应的名字
hostname node$role_num

## 在/etc/hosts中配上node1~3的IP。
sed -i '$a 10.8.65.63 node1' /etc/hosts
sed -i '$a 10.8.65.62 node3' /etc/hosts
sed -i '$a 10.8.65.60 node2' /etc/hosts
```

## 数据源准备

数据源使用mysql

```sh
/etc/init.d/mysqld restart
```

```sql
mysql
>
CREATE USER 'ct'@'%' IDENTIFIED BY '123456';
CREATE USER 'ct'@'localhost' IDENTIFIED BY '123456';
CREATE USER 'ct'@'node1' IDENTIFIED BY '123456';
GRANT ALL ON *.* TO 'ct'@'%';
GRANT ALL ON *.* TO 'ct'@'localhost';
GRANT ALL ON *.* TO 'ct'@'node1';
CREATE DATABASE mytest;
use mytest;
CREATE TABLE accounts(id INTEGER PRIMARY KEY AUTO_INCREMENT NOT NULL, name VARCHAR(255));
INSERT INTO accounts(name) VALUES('alice');
INSERT INTO accounts(name) VALUES('bob');
```

## zookeeper

只有node1上有zk，只在node1上执行

```sh
cd /usr/local/confluent-2.0.1/
./bin/zookeeper-server-start ./etc/kafka/zookeeper.properties &
lokkit -p 2181:tcp
```

## broker

在node1~3上执行

```sh
sed -i 's/zookeeper.connect=localhost:2181/zookeeper.connect=node1:2181/' ./etc/kafka/server.properties
sed -i 's/broker.id=0/broker.id='$role_num'/' ./etc/kafka/server.properties
./bin/kafka-server-start ./etc/kafka/server.properties &
lokkit -p 9092:tcp
```

## scheme-registry

只在node1上执行

```sh
./bin/schema-registry-start ./etc/schema-registry/schema-registry.properties
lokkit -p 8081:tcp
```

## connector

在node1~3上执行

```sh
sed -i 's|localhost:8081|node1:8081|' ./etc/schema-registry/connect-avro-distributed.properties
bin/connect-distributed etc/schema-registry/connect-avro-distributed.properties &
lokkit -p 8083:tcp
```

# 测试

分布式的connector需要使用[api](http://docs.confluent.io/2.0.1/connect/userguide.html#rest-interface)来进行查询

## 创建connector

```sh
curl -X POST -H "Content-Type: application/json" --data '{"name": "test-mysql-jdbc-autoincrement", "config": {"connector.class":"io.confluent.connect.jdbc.JdbcSourceConnector", "tasks.max":"3", "connection.url":"jdbc:mysql://node1:3306/mytest?user=ct&password=123456","topic":"connect-test","mode":"incrementing","incrementing.column.name":"id","topic.prefix":"test-mysql-jdbc-" }}' http://localhost:8083/connectors
```

## 获取connector信息

```sh
## 获取所有的connectors
curl -X GET -H "Content-Type: application/json" http://localhost:8083/connectors 
## 获取connector的tasks
curl -X GET -H "Content-Type: application/json" http://localhost:8083/connectors/test-mysql-jdbc-autoincrement/tasks
## 获取connector的config
curl -X GET -H "Content-Type: application/json" http://localhost:8083/connectors/test-mysql-jdbc-autoincrement/config
```

## 消费

```sh
./bin/kafka-avro-console-consumer --new-consumer --bootstrap-server localhost:9092 --topic test-mysql-jdbc-accounts --from-beginning
```

## 删除connector

```sh
curl -X DELETE -H "Content-Type: application/json" http://localhost:8083/connectors/test-mysql-jdbc-autoincrement
```
