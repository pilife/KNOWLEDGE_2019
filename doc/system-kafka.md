---
typora-copy-images-to: ../image/system-kafka
---

# Kafka

[TOC]

### 概述

#### Definition:

Apache Kafka® is a **distributed streaming platform**

一个流式平台包括以下**特性**：

1. 发布和订阅流式记录，类似消息队列/企业消息系统
2. 处理流式记录
3. 以容错的持久方式存储记录流

因此通常被用于一下两种应用：

- Building real-time streaming data pipelines that reliably get data between systems or applications，即构建可在系统或应用程序之间可靠获取数据的实时流数据管道
- Building real-time streaming applications that transform or react to the streams of data，即构建转换或响应数据流的实时流应用程序




> #### Pre Concepts：
> - Kafka作为一个集群运行在一个或多个可跨多个数据中心的服务器上
> - Topic: The Kafka cluster stores streams of records in categories called topics
> - Record: Each record consists of a key, a value, and a timestamp


#### Core APIs:

- The [Producer API](https://kafka.apache.org/documentation.html#producerapi) allows an application to publish a stream of records to one or more Kafka topics.（by特性1）
- The [Consumer API](https://kafka.apache.org/documentation.html#consumerapi) allows an application to subscribe to one or more topics and process the stream of records produced to them.（by特性1）
- The [Streams API](https://kafka.apache.org/documentation/streams) allows an application to act as a *stream processor*, consuming an input stream from one or more topics and producing an output stream to one or more output topics, effectively transforming the input streams to output streams.（by特性2）
- The [Connector API](https://kafka.apache.org/documentation.html#connect) allows building and running reusable producers or consumers that connect Kafka topics to existing applications or data systems. For example, a connector to a relational database might capture every change to a table.（by特性2）

![img](../image/system-kafka/kafka-apis.png)



#### 通信协议：

在Kafka中，客户端和服务器之间的通信是通过简单，高性能，语言无关的TCP协议完成的。 此协议已版本化并保持与旧版本的向后兼容性。 我们为Kafka提供Java客户端，但客户端有多种语言版本（第三方提供）。

---

### 基本概念：

#### Topic && Log: 

- topic：
  - 概念：record的集合
- partitioned log：
  - 概念：topic的分割
  - 目的：为了拓展性+并行化。
  - 实现：每个分区都是一个有序的，不可变的记录序列，不断附加到结构化的提交日志中。分区中的记录每个都分配了一个称为偏移的顺序ID号，它唯一地标识分区中的每个记录。
- 问题：
  - partition的数量怎么确定？// todo
  - record属于该topic下的哪个partitioned log怎么确定？见[生产者部分-具体实现](####生产者)

![img](../image/system-kafka/log_anatomy.png)

- 每个消费者保留的唯一元数据是该消费者在日志中的偏移或位置。所以Kafka消费者非常轻量。消费者的上下线对集群或者其他消费者没有太大影响。

![img](../image/system-kafka/log_consumer.png)



#### 分布式

- topic log：被partitioned分布在Kafka集群中的服务器上
- each partitioned log
  - 可以复制在多台机器上（类似多副本），以实现容错。
  - 每个分区都有一个服务器充当“领导者”，零个或多个服务器充当“追随者”。（类似raft，但是读写都是leader，因此可以达到N-1容错） 领导者处理分区的所有读取和写入请求，而关注者被动地复制领导者。 如果领导者失败，其中一个粉丝将自动成为新的领导者。 每个服务器都充当其某些分区的领导者和其他服务器的追随者，因此负载在群集中得到很好的平衡。

#### 地域复制

Kafka MirrorMaker为您的群集提供地理复制支持。 使用MirrorMaker，消息在多个数据中心或云区域复制。

用途：

- 进行备份和恢复;

- 将数据放置在离用户较近的位置，或支持数据本地化需求。

#### 生产者

- 概念：将数据发布到他们选择的主题。

- 具体实现：生产者负责选择把记录分配给主题中哪个分区。
  - 这可以以循环方式完成，仅仅是为了平衡负载
  - 或者可以根据一些语义分区功能（例如基于记录中的某些键）来完成。 

#### 消费者

- 概念：消费者以group组织，即kafka只以group为单位进行消费（即每个group是逻辑上的订阅者）
- 实现：Kafka中实现消费的方式是通过在消费者实例上划分日志中的分区。即F(分区)=Instance，F满射函数（group中的instance数量必须小于partition数量，否则必定有instance浪费）

![img](../image/system-kafka/consumer-groups.png)



#### 多租户

// todo

---

### 用途（优势）

#### Kafka as a Messaging System

消息系统传统上有两种模型：[queuing](http://en.wikipedia.org/wiki/Message_queue) and [publish-subscribe](http://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)

- queue: 每个消息被一个消费者消费
  - Strength: 把数据的处理分布在多个消费者上，具备消费的拓展性
  - Weakness: 一个消息不支持多个消费者消费，一旦被消费完就弹出队列了
- publish-subscribe: 每个消息广播给所有消费者
  - Strength: 将消息广播给所有订阅者
  - Weakness: 无法拓展消费，因为每个消息传播给所有订阅者

##### 优势1: Kafka利用group的概念，完成了两者的结合。

- 消息对group内的消费者而言是queue模型
- 对group而言是publish-subscribe模型

##### 优势2: 负载均衡和有序

​	通过把partition分配给组内的消费者（满射），每个分区被组内一个消费者消费。由于分区是有序的，所以保证了消费的有序性。而分区可以被均衡的分配在不同的消费者实例上，从而实现消费的负载均衡。*注意，消费者组中的消费者实例不能超过分区数（满射）。*

#### Kafka as a Storage System

##### 优势1: 多副本容错

##### 优势2: 可拓展性好

#### Kafka for Stream Processing

优势：Kafka提供了完全集成的Streams API



---



### Reference

[kafka官方文档-Introduction](https://kafka.apache.org/intro.html#intro_multi-tenancy)


