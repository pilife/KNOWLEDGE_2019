---
typora-copy-images-to: ../image
---

# ZooKeeper

### 服务端:
#### 功能概述：
维护和监控(**观察者模式**)一个目录节点树中存储的数据的状态。
并对外提供操作目录树接口以及监控目录节点接口。

#### 数据模型：
数据模型如图所示：

![zookeeper](../image/zookeeper.gif)

具有以下特点：

1. 每个子目录项如 NameService 都被称作为 znode，这个 znode 是被它所在的路径**唯一标识**，如 Server1 这个 znode 的标识为 /NameService/Server1
2. znode 可以有子节点目录，并且每个 znode 可以**存储数据**（<u> 这里可以想象下Java实现，每个Znode是一个对象，存储的数据就是属性，且会保存对子节点的引用</u> ），注意 EPHEMERAL 类型的目录节点不能有子节点目录
3. znode 是有版本的，每个 znode 中存储的数据可以有多个版本，也就是一个访问路径中可以存储多份数据
4. znode 可以是临时节点，一旦创建这个 znode 的客户端与服务器失去联系，这个 znode 也将自动删除，Zookeeper 的客户端和服务器通信采用长连接方式，每个客户端和服务器通过心跳来保持连接，这个连接状态称为 session，如果 znode 是临时节点，这个 session 失效，znode 也就删除了
5. znode 的目录名可以自动编号，如 App1 已经存在，再创建的话，将会自动命名为 App2
6. znode 可以被监控，包括这个目录节点中存储的数据的修改，子节点目录的变化等，一旦变化可以通知设置监控的客户端，这个是 Zookeeper 的核心特性，Zookeeper 的很多功能都是基于这个特性实现的，后面在典型的应用场景中会有实例介绍

### 客户端
#### 使用概述：
客户端要连接 Zookeeper 服务器可以通过创建 org.apache.zookeeper. ZooKeeper 的一个实例对象，然后调用这个类提供的接口来和服务器交互。

#### 服务端提供给客户端的接口：
对ZooKeeper对象的操作和操作目录节点树大体一样。
如：创建一个目录节点\给某个目录节点设置数据\获取某个目录节点的所有子目录节点\给某个目录节点设置权限\监控这个目录节点的状态变化。
具体的API见(ZooKeeper API文档)[https://zookeeper.apache.org/doc/r3.4.6/api/org/apache/zookeeper/ZooKeeper.html]

### 常用应用场景：
统一命名服务（Name Service）
配置管理（Configuration Management）
集群管理（Group Membership）
	1. 集群机器状态的订阅发布
	2. 集群Master的选举
共享锁（Locks）
队列管理


### 协议：ZAB（类似Raft）
//todo

### QA:
Q：是不是请求任意客户端都可以？
A：所有的**读请求**由Zk Server本地响应，所有的**更新请求**将转发给Leader，由Leader实施。




Reference:
[分布式服务框架 Zookeeper](https://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/index.html)
[ZooKeeper一致性原理](https://www.cnblogs.com/sunddenly/p/4138580.html)