# Kafka介绍

## 1.1 什么是事件流?

事件流是相当于人体中枢神经系统的数字系统。它是“永远在线”世界的技术基础，在这个世界里，企业越来越多地由软件定义和自动化，软件的用户也更多地是软件。

从技术上讲，事件流是指以事件流的形式从数据库、传感器、移动设备、云服务和软件应用等事件源实时捕获数据的实践;持久地存储这些事件流以备以后检索;实时和回顾性地操作、处理和响应事件流;并根据需要将事件流路由到不同的目的地技术。因此，事件流确保了数据的连续流动和解释，从而使正确的信息在正确的地点、正确的时间出现。

## 1.2 我可以使用事件流做什么?

事件流应用于众多行业和组织的各种用例。它的许多例子包括: 1. 实时处理支付和金融交易，如股票交易所、银行和保险。 2. 实时跟踪和监控汽车、卡车、车队和运输，如物流和汽车行业。 3. 持续捕获和分析来自物联网设备或其他设备的传感器数据，如工厂和风电场。 4. 收集并立即响应客户的互动和订单，如零售、酒店和旅游行业，以及移动应用程序。 5. 监测住院病人，预测病情变化，确保在紧急情况下及时治疗。 6. 连接、存储公司不同部门产生的数据并使其可用。 7. 作为数据平台、事件驱动架构和微服务的基础。

## 1.3 Apache Kafka®是一个事件流平台。这是什么意思?

Kafka结合了三个关键的功能，所以你可以用一个单一的战斗测试解决方案来实现端到端事件流的用例: 1. 发布(写)和订阅(读)事件流，包括从其他系统连续导入/导出数据。 2. 持久性和可靠地存储事件流，只要你想。 3. 在事件发生或回顾时处理事件流。

所有这些功能都是以分布式、高度可伸缩、弹性、容错和安全的方式提供的。Kafka可以部署在裸金属硬件、虚拟机和容器上，也可以部署在云上。您可以选择自管理您的Kafka环境和使用由各种供应商提供的完全管理的服务。

## 1.4 简而言之，Kafka是如何工作的?

Kafka是一个分布式系统，由服务器和客户端组成，通过高性能的TCP网络协议进行通信。它可以部署在裸金属硬件、虚拟机和内部环境中的容器上，也可以部署在云环境中。

**服务器:**Kafka作为一个集群运行一个或多个服务器，可以跨越多个数据中心或云区域。其中一些服务器构成存储层，称为代理。其他服务器运行Kafka Connect来持续导入和导出数据作为事件流，将Kafka与您现有的系统集成，如关系数据库以及其他Kafka集群。为了让你实现关键任务的用例，Kafka集群具有高度的可扩展性和容错性:如果它的任何一个服务器发生故障，其他服务器将接管它们的工作，以确保持续的操作而不丢失任何数据。

**客户机:**它们允许您编写分布式应用程序和微服务，这些应用程序和微服务可以并行地、大规模地读取、写入和处理事件流，甚至在出现网络问题或机器故障的情况下也可以容错。Kafka附带了一些这样的客户端，这些客户端被Kafka社区提供的几十个客户端增强了:客户端可以用于Java和Scala，包括更高级别的Kafka Streams库，用于Go、Python、C/ c++和许多其他编程语言以及REST api。

## 1.5 主要概念和术语

事件记录了在世界上或你的企业中“发生了某事”的事实。在文档中也称为记录或消息。当你读或写数据到Kafka时，你以事件的形式做这件事。从概念上讲，事件具有键、值、时间戳和可选的元数据头。下面是一个例子: 1. 活动重点:“爱丽丝” 2. 事件值:“向Bob支付200美元” 3. 事件时间戳:“2020年6月25日下午2:06。”

生产者是那些向Kafka发布(写)事件的客户端应用程序，而消费者是那些订阅(读和处理)这些事件的应用程序。在Kafka中，生产者和消费者是完全解耦的，彼此是不可知的，这是实现Kafka闻名的高可扩展性的一个关键设计元素。例如，生产者从不需要等待消费者。Kafka提供了各种各样的保证，比如精确处理一次事件的能力。

事件被组织并持久地存储在主题中。很简单，一个主题类似于文件系统中的一个文件夹，事件就是该文件夹中的文件。一个示例主题名称可以是“payments”。Kafka中的主题总是多生产者和多订阅者:一个主题可以有0个、1个或多个生产者向它写入事件，也可以有0个、1个或多个消费者订阅这些事件。主题中的事件可以根据需要经常读取——与传统消息传递系统不同，事件在使用后不会删除。相反，你可以通过每个主题的配置设置来定义Kafka应该保留你的事件多长时间，之后旧的事件将被丢弃。Kafka的性能相对于数据大小来说是不变的，所以长时间存储数据是完全可以的。

主题是分区的，这意味着一个主题分散在位于不同Kafka broker上的多个“桶”上。这种数据的分布式位置对于可伸缩性非常重要，因为它允许客户机应用程序同时从/向多个代理读取和写入数据。当一个新事件被发布到一个主题时，它实际上被附加到主题的一个分区中。具有相同事件键(例如，客户或车辆ID)的事件被写入同一个分区，Kafka保证任何给定主题分区的消费者都将始终以写入的完全相同的顺序读取该分区的事件。

<figure data-size="normal">



![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-39059344afb24ff7436bf7fb06bddde4_720w.webp)

让你的数据容错和可用性,每一个主题可以被复制,甚至跨geo-regions或数据中心,这样总有多个经纪人有一份数据以防出错,你想做代理维护,等等。一个常见的生产设置是复制因子3，也就是说，您的数据总是有三个副本。这个复制是在主题分区级别执行的。

## 二.Kafka简介

Kafka是一种消息队列，主要用来处理大量数据状态下的消息队列，一般用来做日志的处理。既然是消息队列，那么Kafka也就拥有消息队列的相应的特性了。

**消息队列的好处** 1. 解耦合 耦合的状态表示当你实现某个功能的时候，是直接接入当前接口，而利用消息队列，可以将相应的消息发送到消息队列，这样的话，如果接口出了问题，将不会影响到当前的功能。

<figure data-size="normal">



![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-e37a18ea7eddc69582d634cc881eb257_720w.webp)

1.  异步处理 异步处理替代了之前的同步处理，异步处理不需要让流程走完就返回结果，可以将消息发送到消息队列中，然后返回结果，剩下让其他业务处理接口从消息队列中拉取消费处理即可。

2.  流量削峰 高流量的时候，使用消息队列作为中间件可以将流量的高峰保存在消息队列中，从而防止了系统的高请求，减轻服务器的请求处理压力。

## 2.1 Kafka消费模式

Kafka的消费模式主要有两种：一种是一对一的消费，也即点对点的通信，即一个发送一个接收。第二种为一对多的消费，即一个消息发送到消息队列，消费者根据消息队列的订阅拉取消息消费。

**一对一**

<figure data-size="normal">



![image-20230525200024084](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230525200024084.png)

</figure>

消息生产者发布消息到Queue队列中，通知消费者从队列中拉取消息进行消费。消息被消费之后则删除，Queue支持多个消费者，但对于一条消息而言，只有一个消费者可以消费，即一条消息只能被一个消费者消费。

**一对多**

<figure data-size="normal">



![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-d97a2898f3bdc417262bc88be616281c_720w.webp)

这种模式也称为发布/订阅模式，即利用Topic存储消息，消息生产者将消息发布到Topic中，同时有多个消费者订阅此topic，消费者可以从中消费消息，注意发布到Topic中的消息会被多个消费者消费，消费者消费数据之后，数据不会被清除，Kafka会默认保留一段时间，然后再删除。

## 2.2 Kafka的基础架构

<figure data-size="normal">



![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-ef94691300117c049301d88c6337c9c2_720w.webp)

Kafka像其他Mq一样，也有自己的基础架构，主要存在生产者Producer、Kafka集群Broker、消费者Consumer、注册中心Zookeeper.

1. Producer：消息生产者，向Kafka中发布消息的角色。

2. Consumer：消息消费者，即从Kafka中拉取消息消费的客户端。

3. Consumer Group：消费者组，消费者组则是一组中存在多个消费者，消费者消费Broker中当前Topic的不同分区中的消息，消费者组之间互不影响，所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。某一个分区中的消息只能够一个消费者组中的一个消费者所消费

4. Broker：经纪人，一台Kafka服务器就是一个Broker，一个集群由多个Broker组成，一个Broker可以容纳多个Topic。

5. Topic：主题，可以理解为一个队列，生产者和消费者都是面向一个Topic

6. Partition：分区，为了实现扩展性，一个非常大的Topic可以分布到多个Broker上，一个Topic可以分为多个Partition，每个Partition是一个有序的队列(分区有序，不能保证全局有序)

7. Replica：副本Replication，为保证集群中某个节点发生故障，节点上的Partition数据不丢失，Kafka可以正常的工作，Kafka提供了副本机制，一个Topic的每个分区有若干个副本，一个Leader和多个Follower

8. Leader：每个分区多个副本的主角色，生产者发送数据的对象，以及消费者消费数据的对象都是Leader。

9. Follower：每个分区多个副本的从角色，实时的从Leader中同步数据，保持和Leader数据的同步，Leader发生故障的时候，某个Follower会成为新的Leader。

上述一个Topic会产生多个分区Partition，分区中分为Leader和Follower，消息一般发送到Leader，Follower通过数据的同步与Leader保持同步，消费的话也是在Leader中发生消费，如果有多个消费者，则分别消费Topic里各个Leader分区中的消息，当Leader发生故障的时候，某个Follower会成为主节点，此时会对齐消息的偏移量。

