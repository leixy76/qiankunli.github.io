---

layout: post
title: 《Apache Kafka源码分析》——server
category: 技术
tags: Scala
keywords: Scala  akka

---

## 前言

* TOC
{:toc}

建议先阅读下[《Apache Kafka源码分析》——Producer与Consumer](http://qiankunli.github.io/2017/12/08/kafka_learn_1.html)

![](/public/upload/scala/kafka_server_framework.jpg)

对kafka好奇的几个问题：

1. 数据在磁盘上是如何存储的？

## 网络层

kafka client (producer/consumer) 与kafka server通信时使用自定义的协议，一个线程 一个selector 裸调java NIO 进行网络通信。 

面对高并发、低延迟的需求，kafka 服务端使用了多线程+多selector ，参见[java nio的多线程扩展](http://qiankunli.github.io/2015/06/19/java_nio_2.html)

![](/public/upload/netty/kafka_server_nio.jpg)

这是一款常见的服务端 实现

1. 分层实现，网络io 部分负责读写数据，并将数据序列化/反序列化为协议请求。
2. 协议请求交给 上层处理， API层就好比 tomcat 中的servlet


## 单机——基本实现

### 初始化过程

![](/public/upload/scala/kafka_server_init.png)

### 整体思路

kafka 服务端核心是 KafkaServer，KafkaServer 没什么特别的，聚合和启动各个能力对象即可（kafka 称之为subsystem）。各能力对象都用到了统一的线程池，各自干各自的活儿。

1. LogManager。 The entry point to the kafka log management subsystem. The log manager is responsible for log creation, retrieval, and cleaning. All read and write operations are delegated to the individual log instances. LogManager 干了日志文件的维护，单纯的日志写入交给了Log 对象
2. ReplicaManager的主要功能是管理一个Broker 范围内的Partition 信息。代码上，Partition 对象为ReplicaManager 分担一部分职能
3. KafkaController，在Kafka集群的多个Broker中， 有一个Broker会被推举为Controller Leader，负责管理整个集群中分区和副本的状态。

![](/public/upload/scala/kafka_server_object.png)

从下文可以看到，broker 的主要逻辑就是接收各种请求并处理。除了使用了自定义网络协议导致网络层不一样，在api层/业务层，broker 与webserver 的开发逻辑是类似的。作为一个“webserver”，KafkaServer 的一个很重要组件是请求分发——KafkaApis。KafkaServer 将各个组件startup后，KafkaApis 聚合各个能力组件，将请求分发到 各个能力组件具体的方法上。

要注意两个层次的概念

1. broker 层次的leader 和 follower
2. replica 层次的leader 和 follower

![](/public/upload/scala/kafka_framework_3.png)

从单机角度看，自定义协议 + 主流程 + 旁路subsystem，与mysql 有点神似。

### 写日志过程

![](/public/upload/scala/kafka_server_write_log.png)

## 日志存储

### 整体思想

接收到客户端的请求之后，不同的系统（都要读写请求数据）有不同的反应

1. redis 读取/写入内存
2. mysql 读取/写入本地磁盘
3. web 系统，转手将请求数据处理后写入数据；或者查询数据库并返回响应信息，其本身就是一个中转站。

读写本地磁盘的系统 一般有考虑几个问题

1. 磁盘只能存储二进制数据或文本数据，文本数据你可以一行一行读取/写入。二进制数据则要求制定一个文件格式，一次性读写特定长度的数据。
2. 如果文件较大，为加快读写速度，还要考虑读写索引文件
3. 内存是否需要缓存热点磁盘数据

建议和mysql 对比学习下  [《mysql技术内幕》笔记1](http://qiankunli.github.io/2017/10/31/inside_mysql1.html)

|逻辑概念|对应的物理概念|备注|
|---|---|---|
|Log|目录|目录的命名规则`<topic_name>_<partition_id>`|
|LogSegment|一个日志文件、一个索引文件|命名规则`[baseOffset].log`和`[baseOffset].index` <br> baseOffset 是日志文件中第一条消息的offset|
|offset|消息在日志文件中的偏移|类似于数据库表中的主键<br> 由索引文件记录映射关系|

![](/public/upload/scala/kafka_index_file.jpg)

## 多机——基于zk协作的两种方式

在kafka中，broker、分区、副本状态等 作为集群状态信息，一旦发生改变，都会需要集群的broker作出反应，那么broker 之间如何协同呢？

在Kafka 早期版本中，每个broker 都会向zookeeper 上注册watcher，当分区或副本状态变化时会唤醒很多不必要的watcher， 导致羊群效应及zookeeper 集群过载。

在新版的设计中，只有Controller Leader 在zookeeper上注册wather，其它的broker 几乎不用再监听zookeeper 中的数据变化。 每个Broker 启动时都会创建一个KafkaController 对象，但是集群中只能存在一个Controller Leader来对外提供服务。在集群启动时，多个Broker上的KafkaController 会在指定路径下竞争创建节点，只有第一个成功创建节点的KafkaController 才能成为Leader（其余的成为Follower）。当Leader出现故障后，所有的Follower会收到通知，再次竞争新的Leader。KafkaController 与Broker 交互，Broker 处理来自KafkaController 的LeaderAndIsrRequest、StopReplicaRequest、UpdateMetadataRequest 等请求

简单说，老版本Broker 之间的数据传递依赖于Zookeeper，每个Broker 对zookeeper 的所有数据数据变化 相机做出反应 并更新zookeeper，比较低效。新版本Broker 选举出Controller Leader 后， 由Controller Leader 相机向各个Broker 发出指令。有人告诉你做什么，总比你拿到数据后自己分析判断再行动要容易些。

作为对比，hadoop 之类框架有明确的master/slave 之分，但为了高可用，master 往往要多个副本。除此之外，分布式框之间的协同 应该是相通的

1. 每个组件启动后，向zk 注册自己的信息
2. 接收master/leader 的各种请求（http协议或自定义协议） 并处理即可，处理完相机更新zk的数据

从这个角度看，每个slave 组件的逻辑与业务程序猿常写的web server 也别无二致

## 小结

面向对象的源码分析，一般先宏观（比如如何启动，比如业务逻辑的实现路径等）后细节，就是类图和主流程序列图

1. 类图表达依赖关系，反映了代码的组织和业务抽象
2. 主流程展示主要执行路径，反应了业务的逻辑

[声明式编程范式初探](http://www.nowamagic.net/academy/detail/1220525)命令式编程中的变量本质上是抽象化的内存，变量值是该内存的储存内容。JVM 线程通信 靠共享内存，反映在代码上 就是共享对象。

[源码分析体会](http://qiankunli.github.io/2019/01/24/source_parse.html)任何一个系统的设计都有功能和性能（泛化一下就是功能性和非功能性） 两个部分，识别系统模块属于哪个部分，有助于简化对系统的认识。通常，一个系统的最早版本只专注于功能，后续除非大的变动，后来的演化大部分都是为了性能上的追求。在kafka 这块，zk的协作方式等方面的变化 有很充分的体现。





