一、概述

Kakfa起初是由LinkedIn公司开发的一个分布式的消息系统，后成为Apache的一部分，它使用Scala编写，以可水平扩展和高吞吐率而被广泛使用。目前越来越多的开源分布式处理系统如Cloudera、Apache Storm、Spark等都支持与Kafka集成。

Kafka凭借着自身的优势，越来越受到互联网企业的青睐，唯品会也采用Kafka作为其内部核心消息引擎之一。Kafka作为一个商业级消息中间件，消息可靠性的重要性可想而知。如何确保消息的精确传输?如何确保消息的准确存储?如何确保消息的正确消费?这些都是需要考虑的问题。本文首先从Kafka的架构着手，先了解下Kafka的基本原理，然后通过对kakfa的存储机制、复制原理、同步原理、可靠性和持久性保证等等一步步对其可靠性进行分析，最后通过benchmark来增强对Kafka高可靠性的认知。

二、Kafka的使用场景

（1）日志收集：一个公司可以用Kafka可以收集各种服务的log，通过kafka以统一接口服务的方式开放给各种consumer，例如Hadoop、Hbase、Solr等；

（2）消息系统：解耦和生产者和消费者、缓存消息等；

（3）用户活动跟踪：Kafka经常被用来记录web用户或者app用户的各种活动，如浏览网页、搜索、点击等活动，这些活动信息被各个服务器发布到kafka的topic中，然后订阅者通过订阅这些topic来做实时的监控分析，或者装载到Hadoop、数据仓库中做离线分析和挖掘；

（4）运营指标：Kafka也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告；

（5）流式处理：比如spark streaming和storm；

（6）事件源；

三、Kafka基本架构



如上图所示，一个典型的Kafka体系架构包括：

若干Producer(可以是服务器日志，业务数据，页面前端产生的page view等等)，

若干broker(Kafka支持水平扩展，一般broker数量越多，集群吞吐率越高)，

若干Consumer (Group)，以及一个Zookeeper集群。

Kafka通过Zookeeper管理集群配置，选举leader，以及在consumer group发生变化时进行rebalance。Producer使用push(推)模式将消息发布到broker，Consumer使用pull(拉)模式从broker订阅并消费消息。

1、Topic & Partition

一个topic可以认为一个一类消息，每个topic将被分成多个partition，每个partition在存储层面是append log文件。任何发布到此partition的消息都会被追加到log文件的尾部，每条消息在文件中的位置称为offset(偏移量)，offset为一个long型的数字，它唯一标记一条消息。每条消息都被append到partition中，是顺序写磁盘，因此效率非常高(经验证，顺序写磁盘效率比随机写内存还要高，这是Kafka高吞吐率的一个很重要的保证)。



每一条消息被发送到broker中，会根据partition规则选择被存储到哪一个partition。partition机制可以通过指定producer的partition.class这一参数来指定，该class必须实现kafka.producer.Partitioner接口。如果partition规则设置的合理，所有消息可以均匀分布到不同的partition里，这样就实现了水平扩展。(如果一个topic对应一个文件，那这个文件所在的机器I/O将会成为这个topic的性能瓶颈，而partition解决了这个问题)。在创建topic时可以在$KAFKA_HOME/config/server.properties中指定这个partition的数量(如下所示)，当然可以在topic创建之后去修改partition的数量。



# The default number of log partitions per topic. More partitions allow greater
# parallelism for consumption, but this will also result in more files across
# the brokers.
#默认partitions数量
num.partitions=1
四、高可靠性存储分析概述

Kafka的高可靠性的保障来源于其健壮的副本(replication)策略。通过调节其副本相关参数，可以使得Kafka在性能和可靠性之间运转的游刃有余。Kafka从0.8.x版本开始提供partition级别的复制,replication的数量可以在$KAFKA_HOME/config/server.properties中配置(default.replication.refactor)。

这里先从Kafka文件存储机制入手，从最底层了解Kafka的存储细节，进而对其的存储有个微观的认知。之后通过Kafka复制原理和同步方式来阐述宏观层面的概念。最后从ISR，HW，leader选举以及数据可靠性和持久性保证等等各个维度来丰富对Kafka相关知识点的认知。

五、Kafka文件存储机制

Kafka中消息是以topic进行分类的，生产者通过topic向Kafka broker发送消息，消费者通过topic读取数据。然而topic在物理层面又能以partition为分组，一个topic可以分成若干个partition，那么topic以及partition又是怎么存储的呢?partition还可以细分为segment，一个partition物理上由多个segment组成，那么这些segment又是什么呢?下面我们来一一揭晓。

为了便于说明问题，假设这里只有一个Kafka集群，且这个集群只有一个Kafka broker，即只有一台物理机。在这个Kafka broker中配置($KAFKA_HOME/config/server.properties中)log.dirs=/tmp/kafka-logs，以此来设置Kafka消息文件存储目录，与此同时创建一个topic：topic_zzh_test，partition的数量为4($KAFKA_HOME/bin/kafka-topics.sh –create –zookeeper localhost:2181 –partitions 4 –topic topic_vms_test –replication-factor 4)。那么我们此时可以在/tmp/kafka-logs目录中可以看到生成了4个目录：



drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-0 
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-1 
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-2 
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-3 

在Kafka文件存储中，同一个topic下有多个不同的partition，每个partiton为一个目录，partition的名称规则为：topic名称+有序序号，第一个序号从0开始计，最大的序号为partition数量减1，partition是实际物理上的概念，而topic是逻辑上的概念。

上面提到partition还可以细分为segment，这个segment又是什么?如果就以partition为最小存储单位，我们可以想象当Kafka producer不断发送消息，必然会引起partition文件的无限扩张，这样对于消息文件的维护以及已经被消费的消息的清理带来严重的影响，所以这里以segment为单位又将partition细分。每个partition(目录)相当于一个巨型文件被平均分配到多个大小相等的segment(段)数据文件中(每个segment 文件中消息数量不一定相等)这种特性也方便old segment的删除，即方便已被消费的消息的清理，提高磁盘的利用率。每个partition只需要支持顺序读写就行，segment的文件生命周期由服务端配置参数(log.segment.bytes，log.roll.{ms,hours}等若干参数)决定。



# 在强制刷新数据到磁盘允许接收消息的数量
log.flush.interval.messages=10000 
# 在强制刷新之前，消息可以在日志中停留的最长时间
log.flush.interval.ms=1000 
# 一个日志的最小存活时间，可以被删除
log.retention.hours=168 
# 一个基于大小的日志保留策略。段将被从日志中删除只要剩下的部分段不低于
log.retention.bytes=1073741824 
#  每一个日志段大小的最大值。当到达这个大小时，会生成一个新的片段。
log.segment.bytes=1073741824 
# 检查日志段的时间间隔，看是否可以根据保留策略删除它们
log.retention.check.interval.ms=300000
segment文件由两部分组成，分别为“.index”文件和“.log”文件，分别表示为segment索引文件和数据文件。这两个文件的命令规则为：partition全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值+1，数值大小为64位，20位数字字符长度，没有数字用0填充，并且index里面的索引是稀疏索引，如下：


00000000000000000000.index 00000000000000000000.log 00000000000000000004.index 00000000000000000004.log 
00000000000000000008.index 00000000000000000008.log 
图中例子是设置了强制 Kafka 在段文件达到 1 KB 时滚动以及禁用基于时间的滚动（仅依赖大小），设置为MAX值

log.segment.bytes: "1024"

kafka_log_roll_ms: "9223372036854775807"
00000000000000000000.index


[appuser@kafka1 test-topic-0]$ kafka-dump-log --files 00000000000000000000.index --print-data-log
Dumping 00000000000000000000.index
offset: 0 position: 0
00000000000000000000.log 


[appuser@kafka1 test-topic-0]$ kafka-dump-log --files 00000000000000000000.log --print-data-log
Dumping 00000000000000000000.log
Log starting offset: 0
baseOffset: 0 lastOffset: 0 count: 1 baseSequence: 0 lastSequence: 0 producerId: 0 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 0 CreateTime: 1746591178220 size: 495 magic: 2 compresscodec: none crc: 1738099099 isvalid: true
| offset: 0 CreateTime: 1746591178220 keySize: -1 valueSize: 425 sequence: 0 headerKeys: [] payload: Apache Kafka is a distributed streaming platform that enables you to build real-time streaming data pipelines and applications. Setting up Kafka can be complex, but Docker Compose simplifies the process by defining and running multi-container Docker applications. This guide provides a step-by-step approach to creating a Kafka topic using Docker Compose, making it accessible for developers and DevOps professionals alike.
​
​
baseOffset: 1 lastOffset: 1 count: 1 baseSequence: 0 lastSequence: 0 producerId: 1 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 495 CreateTime: 1746591185104 size: 154 magic: 2 compresscodec: none crc: 60619224 isvalid: true
| offset: 1 CreateTime: 1746591185104 keySize: -1 valueSize: 84 sequence: 0 headerKeys: [] payload: Docker Compose: A tool for defining and running multi-container Docker applications.
​
​
baseOffset: 2 lastOffset: 2 count: 1 baseSequence: 0 lastSequence: 0 producerId: 2 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 649 CreateTime: 1746591201143 size: 238 magic: 2 compresscodec: none crc: 3698716342 isvalid: true
| offset: 2 CreateTime: 1746591201143 keySize: -1 valueSize: 168 sequence: 0 headerKeys: [] payload: Replace YourTopicName with the desired name for your Kafka topic. The KAFKA_CREATE_TOPICS environment variable format is TopicName:NumberOfPartitions:ReplicationFactor.
​
​
baseOffset: 3 lastOffset: 3 count: 1 baseSequence: 0 lastSequence: 0 producerId: 3 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 887 CreateTime: 1746591208291 size: 123 magic: 2 compresscodec: none crc: 3174751501 isvalid: true
| offset: 3 CreateTime: 1746591208291 keySize: -1 valueSize: 55 sequence: 0 headerKeys: [] payload: You should see YourTopicName listed among the topics.
00000000000000000004.index


[appuser@kafka1 test-topic-0]$ kafka-dump-log --files 00000000000000000004.index --print-data-log
Dumping 00000000000000000004.index
offset: 4 position: 0
00000000000000000004.log 


[appuser@kafka1 test-topic-0]$ kafka-dump-log --files 00000000000000000004.log --print-data-log
Dumping 00000000000000000004.log
Log starting offset: 4
baseOffset: 4 lastOffset: 4 count: 1 baseSequence: 0 lastSequence: 0 producerId: 4 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 0 CreateTime: 1746591213750 size: 175 magic: 2 compresscodec: none crc: 778754623 isvalid: true
| offset: 4 CreateTime: 1746591213750 keySize: -1 valueSize: 105 sequence: 0 headerKeys: [] payload: After executing the command, you can type messages into the console. Press Ctrl+D to send the messages.
​
​
baseOffset: 5 lastOffset: 5 count: 1 baseSequence: 0 lastSequence: 0 producerId: 5 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 175 CreateTime: 1746599240592 size: 145 magic: 2 compresscodec: none crc: 317371178 isvalid: true
| offset: 5 CreateTime: 1746599240592 keySize: -1 valueSize: 75 sequence: 0 headerKeys: [] payload: Open another terminal session, access the Kafka container again, and run:
​
​
baseOffset: 6 lastOffset: 6 count: 1 baseSequence: 0 lastSequence: 0 producerId: 6 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 320 CreateTime: 1746599245393 size: 249 magic: 2 compresscodec: none crc: 715146949 isvalid: true
| offset: 6 CreateTime: 1746599245393 keySize: -1 valueSize: 179 sequence: 0 headerKeys: [] payload: By default the docker creates topics defined with KAFKA_CREATE_TOPICS variable in docker-compose.yaml file. But still you can create new Kafka topics with the following command:
​
​
baseOffset: 7 lastOffset: 7 count: 1 baseSequence: 0 lastSequence: 0 producerId: 7 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 569 CreateTime: 1746599250095 size: 239 magic: 2 compresscodec: none crc: 811968735 isvalid: true
| offset: 7 CreateTime: 1746599250095 keySize: -1 valueSize: 169 sequence: 0 headerKeys: [] payload: NOTE: Due to limitations in metric names, topics with a period (‘.’) or underscore (‘_’) could collide. To avoid issues it is best to use either, but not both.
00000000000000000008.index


[appuser@kafka1 test-topic-0]$ kafka-dump-log --files 00000000000000000008.index --print-data-log
Dumping 00000000000000000008.index
offset: 8 position: 0
00000000000000000008.log 


[appuser@kafka1 test-topic-0]$ kafka-dump-log --files 00000000000000000008.log --print-data-log
Dumping 00000000000000000008.log
Log starting offset: 8
baseOffset: 8 lastOffset: 8 count: 1 baseSequence: 0 lastSequence: 0 producerId: 8 producerEpoch: 0 partitionLeaderEpoch: 0 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 0 CreateTime: 1746599255177 size: 542 magic: 2 compresscodec: none crc: 726286803 isvalid: true
| offset: 8 CreateTime: 1746599255177 keySize: -1 valueSize: 472 sequence: 0 headerKeys: [] payload: You’ve now successfully created a Kafka topic using Docker Compose and verified its functionality by producing and consuming messages. This setup not only simplifies the process of managing Kafka but also provides a scalable and easily reproducible environment for your streaming applications. Whether you’re developing locally or deploying in a production environment, Docker Compose with Kafka offers a powerful toolset to streamline your data streaming pipelines.
日志里面的换行符是文本输入的时候带入的，不是日志格式本身有换行
要怎么知道何时才算读完本条消息，否则就读到下一条消息的内容了?

这个就需要联系到消息的物理结构了，消息都具有固定的物理结构，包括：offset(8 Bytes)、消息体的大小(4 Bytes)、crc32(4 Bytes)、magic(1 Byte)、attributes(1 Byte)、keySize(4 Bytes)、key(K Bytes)、valueSize(4 Bytes)、payload(N Bytes)等等字段，可以确定一条消息的大小，即读取到哪里截止。

六、复制原理和同步方式

Kafka中topic的每个partition有一个预写式的日志文件，虽然partition可以继续细分为若干个segment文件，但是对于上层应用来说可以将partition看成最小的存储单元(一个有多个segment文件拼接的“巨型”文件)，每个partition都由一些列有序的、不可变的消息组成，这些消息被连续的追加到partition中。



上图中有两个新名词：HW和LEO。这里先介绍下LEO，LogEndOffset的缩写，表示每个partition的log最后一条Message的位置。HW是HighWatermark的缩写，是指consumer能够看到的此partition的位置，这个涉及到多副本的概念，这里先提及一下，下节再详表。

言归正传，为了提高消息的可靠性，Kafka每个topic的partition有N个副本(replicas)，其中N(大于等于1)是topic的复制因子(replica fator)的个数。Kafka通过多副本机制实现故障自动转移，当Kafka集群中一个broker失效情况下仍然保证服务可用。在Kafka中发生复制时确保partition的日志能有序地写到其他节点上，N个replicas中，其中一个replica为leader，其他都为follower, leader处理partition的所有读写请求，与此同时，follower会被动定期地去复制leader上的数据。

如下图所示，Kafka集群中有4个broker, 某topic有3个partition,且复制因子即副本个数也为3：



Kafka提供了数据复制算法保证，如果leader发生故障或挂掉，一个新leader被选举并被接受客户端的消息成功写入。Kafka确保从同步副本列表中选举一个副本为leader，或者说follower追赶leader数据。leader负责维护和跟踪ISR(In-Sync Replicas的缩写，表示副本同步队列，具体可参考下节)中所有follower滞后的状态。当producer发送一条消息到broker后，leader写入消息并复制到所有follower。消息提交之后才被成功复制到所有的同步副本。消息复制延迟受最慢的follower限制，重要的是快速检测慢副本，如果follower“落后”太多或者失效，leader将会把它从ISR中删除。

七、零拷贝

Kafka中存在大量的网络数据持久化到磁盘（Producer到Broker）和磁盘文件通过网络发送（Broker到Consumer）的过程。这一过程的性能直接影响Kafka的整体吞吐量。

1、传统模式下的四次拷贝与四次上下文切换

以将磁盘文件通过网络发送为例。传统模式下，一般使用如下伪代码所示的方法先将文件数据读入内存，然后通过Socket将内存中的数据发送出去。


buffer = File.read
Socket.send(buffer)
这一过程实际上发生了四次数据拷贝。首先通过系统调用将文件数据读入到内核态Buffer（DMA拷贝），然后应用程序将内存态Buffer数据读入到用户态Buffer（CPU拷贝），接着用户程序通过Socket发送数据时将用户态Buffer数据拷贝到内核态Buffer（CPU拷贝），最后通过DMA拷贝将数据拷贝到NIC Buffer。同时，还伴随着四次上下文切换，如下图所示。



2、sendfile和transferTo实现零拷贝

Linux 2.4+内核通过sendfile系统调用，提供了零拷贝。数据通过DMA拷贝到内核态Buffer后，直接通过DMA拷贝到NIC Buffer，无需CPU拷贝。这也是零拷贝这一说法的来源。除了减少数据拷贝外，因为整个读文件-网络发送由一个sendfile调用完成，整个过程只有两次上下文切换，因此大大提高了性能。零拷贝过程如下图所示。



从具体实现来看，Kafka的数据传输通过TransportLayer来完成，其子类PlaintextTransportLayer通过Java NIO的FileChannel的transferTo和transferFrom方法实现零拷贝，如下所示。


@Override public long transferFrom(FileChannel fileChannel, long position, long count) throws IOException { 
  return fileChannel.transferTo(position, count, socketChannel);
}
注： transferTo和transferFrom并不保证一定能使用零拷贝。实际上是否能使用零拷贝与操作系统相关，如果操作系统提供sendfile这样的零拷贝系统调用，则这两个方法会通过这样的系统调用充分利用零拷贝的优势，否则并不能通过这两个方法本身实现零拷贝。

八、 ISR（副本同步队列）

上节我们涉及到ISR (In-Sync Replicas)，这个是指副本同步队列。副本数对Kafka的吞吐率是有一定的影响，但极大的增强了可用性。默认情况下Kafka的replica数量为1，即每个partition都有一个唯一的leader，为了确保消息的可靠性，通常应用中将其值(由broker的参数offsets.topic.replication.factor指定)大小设置为大于1，比如3。 所有的副本(replicas)统称为Assigned Replicas，即AR。ISR是AR中的一个子集，由leader维护ISR列表，follower从leader同步数据有一些延迟(包括延迟时间replica.lag.time.max.ms和延迟条数replica.lag.max.messages两个维度, 当前最新的版本0.10.x中只支持replica.lag.time.max.ms这个维度)，任意一个超过阈值都会把follower剔除出ISR, 存入OSR(Outof-Sync Replicas)列表，新加入的follower也会先存放在OSR中。AR=ISR+OSR

Kafka 0.10.x版本后移除了replica.lag.max.messages参数，只保留了replica.lag.time.max.ms作为ISR中副本管理的参数。为什么这样做呢?replica.lag.max.messages表示当前某个副本落后leaeder的消息数量超过了这个参数的值，那么leader就会把follower从ISR中删除。假设设置replica.lag.max.messages=4，那么如果producer一次传送至broker的消息数量都小于4条时，因为在leader接受到producer发送的消息之后而follower副本开始拉取这些消息之前，follower落后leader的消息数不会超过4条消息，故此没有follower移出ISR，所以这时候replica.lag.max.message的设置似乎是合理的。但是producer发起瞬时高峰流量，producer一次发送的消息超过4条时，也就是超过replica.lag.max.messages，此时follower都会被认为是与leader副本不同步了，从而被踢出了ISR。但实际上这些follower都是存活状态的且没有性能问题。那么在之后追上leader,并被重新加入了ISR。于是就会出现它们不断地剔出ISR然后重新回归ISR，这无疑增加了无谓的性能损耗。而且这个参数是broker全局的。设置太大了，影响真正“落后”follower的移除;设置的太小了，导致follower的频繁进出。无法给定一个合适的replica.lag.max.messages的值，故此，新版本的Kafka移除了这个参数。注：ISR中包括：leader和follower。


上面一节还涉及到一个概念，即HW。HW俗称高水位，HighWatermark的缩写，取一个partition对应的ISR中最小的LEO作为HW，consumer最多只能消费到HW所在的位置。另外每个replica都有HW,leader和follower各自负责更新自己的HW的状态。对于leader新写入的消息，consumer不能立刻消费，leader会等待该消息被所有ISR中的replicas同步后更新HW，此时消息才能被consumer消费。这样就保证了如果leader所在的broker失效，该消息仍然可以从新选举的leader中获取。对于来自内部broKer的读取请求，没有HW的限制。

下图详细的说明了当producer生产消息至broker后，ISR以及HW和LEO的流转过程：



由此可见，Kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制。事实上，同步复制要求所有能工作的follower都复制完，这条消息才会被commit，这种复制方式极大的影响了吞吐率。而异步复制方式下，follower异步的从leader复制数据，数据只要被leader写入log就被认为已经commit，这种情况下如果follower都还没有复制完，落后于leader时，突然leader宕机，则会丢失数据。而Kafka的这种使用ISR的方式则很好的均衡了确保数据不丢失以及吞吐率。

Kafka的ISR的管理最终都会反馈到Zookeeper节点上。具体位置为：/brokers/topics/[topic]/partitions/[partition]/state。

目前有两个地方会对这个Zookeeper的节点进行维护：

Controller来维护：Kafka集群中的其中一个Broker会被选举为Controller，主要负责Partition管理和副本状态管理，也会执行类似于重分配partition之类的管理任务。在符合某些特定条件下，Controller下的LeaderSelector会选举新的leader，ISR和新的leader_epoch及controller_epoch写入Zookeeper的相关节点中。同时发起LeaderAndIsrRequest通知所有的replicas。

leader来维护：leader有单独的线程定期检测ISR中follower是否脱离ISR, 如果发现ISR变化，则会将新的ISR的信息返回到Zookeeper的相关节点中。

HW 的更新

Leader 副本的 HW 更新

触发条件：当 Follower 副本向 Leader 发送 Fetch 请求时，携带自身 LEO。

计算逻辑：

Leader 收集所有 ISR 副本的 LEO（包括自己的 LEO）。

取所有 LEO 的最小值作为新 HW。

将新 HW 通过响应返回给 Follower。

示例：

Leader LEO=10，Follower1 LEO=9，Follower2 LEO=8 → HW=8。

若 Follower2 掉出 ISR，则 HW=9（仅 Leader 和 Follower1 参与计算）。

Follower 副本的 HW 更新

Follower 根据 Leader 返回的 HW 值，与自身 LEO 取较小值更新本地 HW。

规则：Follower_HW = MIN(Leader_HW, Follower_LEO)。

HW 持久化与容错

存储位置：每个 Broker 的本地文件（如 replication-offset-checkpoint）记录所有分区的 HW。

恢复机制：Broker 重启时从文件中读取 HW，避免数据丢失或重复消费。

容错性：若 Leader 宕机，新 Leader 从 ISR 中选出，基于持久化的 HW 继续提供服务。

ISR 动态维护对 HW 的影响

ISR 的准入条件：

Follower 的 LEO ≥ Leader 的 HW。

同步延迟不超过 replica.lag.time.max.ms（默认 10 秒）。

淘汰机制：落后于 Leader 的副本会被移出 ISR，不再参与 HW 计算。

示例：若 Follower 长时间未同步数据，Leader 更新 HW 时仅考虑剩余 ISR 副本的 LEO。

异常场景处理

Follower 数据滞后：若 Follower 的 LEO 低于 HW，Leader 会推送缺失数据，直至其重新加入 ISR。

Leader 切换：新 Leader 的 HW 继承自原 Leader 的持久化值，确保消费者可见性不变。

九、数据可靠性和持久性保证

当producer向leader发送数据时，可以通过request.required.acks参数来设置数据可靠性的级别：

1(默认)：这意味着producer在ISR中的leader已成功收到的数据并得到确认后发送下一条message。如果leader宕机了，则会丢失数据。

0：这意味着producer无需等待来自broker的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。

-1：producer需要等待ISR中的所有follower都确认接收到数据后才算一次发送完成，可靠性最高。但是这样也不能保证数据不丢失，比如当ISR中只有leader时(前面ISR那一节讲到，ISR中的成员由于某些情况会增加也会减少，最少就只剩一个leader)，这样就变成了acks=1的情况。

官网配置说明

如果要提高数据的可靠性，在设置request.required.acks=-1的同时，也要min.insync.replicas这个参数(可以在broker或者topic层面进行设置)的配合，这样才能发挥最大的功效。min.insync.replicas这个参数设定ISR中的最小副本数是多少，默认值为1，当且仅当request.required.acks参数设置为-1时，此参数才生效。如果ISR中的副本数少于min.insync.replicas配置的数量时，客户端会返回异常：org.apache.kafka.common.errors.NotEnoughReplicasExceptoin: Messages are rejected since there are fewer in-sync replicas than required。

接下来对acks=1和-1的两种情况进行详细分析：

9.1、request.required.acks=1

producer发送数据到leader，leader写本地日志成功，返回客户端成功;此时ISR中的副本还没有来得及拉取该消息，leader就宕机了，那么此次发送的消息就会丢失。



9.2、request.required.acks=-1

同步(Kafka默认为同步，即producer.type=sync)的发送模式，replication.factor>=2且min.insync.replicas>=2的情况下，不会丢失数据。

有两种典型情况。acks=-1的情况下(如无特殊说明，以下acks都表示为参数request.required.acks)，数据发送到leader, ISR的follower全部完成数据同步后，leader此时挂掉，那么会选举出新的leader，数据不会丢失。



acks=-1的情况下，数据发送到leader后 ，部分ISR的副本同步，leader此时挂掉。比如follower1h和follower2都有可能变成新的leader, producer端会得到返回异常，producer端会重新发送数据，数据可能会重复


当然上图中如果在leader crash的时候，follower2还没有同步到任何数据，而且follower2被选举为新的leader的话，这样消息就不会重复。

注：Kafka只处理fail/recover问题,不处理Byzantine问题。

9.3、关于HW的进一步探讨

考虑上图(即acks=-1,部分ISR副本同步)中的另一种情况，如果在Leader挂掉的时候，follower1同步了消息4,5，follower2同步了消息4，与此同时follower2被选举为leader，那么此时follower1中的多出的消息5该做如何处理呢?

这里就需要HW的协同配合了。如前所述，一个partition中的ISR列表中，leader的HW是所有ISR列表里副本中最小的那个的LEO。类似于木桶原理，水位取决于最低那块短板。



如上图，某个topic的某partition有三个副本，分别为A、B、C。A作为leader肯定是LEO最高，B紧随其后，C机器由于配置比较低，网络比较差，故而同步最慢。这个时候A机器宕机，这时候如果B成为leader，假如没有HW，在A重新恢复之后会做同步(makeFollower)操作，在宕机时log文件之后直接做追加操作，而假如B的LEO已经达到了A的LEO，会产生数据不一致的情况，所以使用HW来避免这种情况。

A在做同步操作的时候，先将log文件截断到之前自己的HW的位置，即3，之后再从B中拉取消息进行同步。

如果失败的follower恢复过来，它首先将自己的log文件截断到上次checkpointed时刻的HW的位置，之后再从leader中同步消息。leader挂掉会重新选举，新的leader会发送“指令”让其余的follower截断至自身的HW的位置然后再拉取新的消息。

当ISR中的个副本的LEO不一致时，如果此时leader挂掉，选举新的leader时并不是按照LEO的高低进行选举，而是按照ISR中的顺序选举。

9.4、Leader选举

一条消息只有被ISR中的所有follower都从leader复制过去才会被认为已提交。这样就避免了部分数据被写进了leader，还没来得及被任何follower复制就宕机了，而造成数据丢失。而对于producer而言，它可以选择是否等待消息commit，这可以通过request.required.acks来设置。这种机制确保了只要ISR中有一个或者以上的follower，一条被commit的消息就不会丢失。

有一个很重要的问题是当leader宕机了，怎样在follower中选举出新的leader，因为follower可能落后很多或者直接crash了，所以必须确保选择“最新”的follower作为新的leader。一个基本的原则就是，如果leader不在了，新的leader必须拥有原来的leader commit的所有消息。这就需要做一个折中，如果leader在表名一个消息被commit前等待更多的follower确认，那么在它挂掉之后就有更多的follower可以成为新的leader，但这也会造成吞吐率的下降。

一种非常常用的选举leader的方式是“少数服从多数”，Kafka并不是采用这种方式。这种模式下，如果我们有2f+1个副本，那么在commit之前必须保证有f+1个replica复制完消息，同时为了保证能正确选举出新的leader，失败的副本数不能超过f个。这种方式有个很大的优势，系统的延迟取决于最快的几台机器，也就是说比如副本数为3，那么延迟就取决于最快的那个follower而不是最慢的那个。“少数服从多数”的方式也有一些劣势，为了保证leader选举的正常进行，它所能容忍的失败的follower数比较少，如果要容忍1个follower挂掉，那么至少要3个以上的副本，如果要容忍2个follower挂掉，必须要有5个以上的副本。也就是说，在生产环境下为了保证较高的容错率，必须要有大量的副本，而大量的副本又会在大数据量下导致性能的急剧下降。这种算法更多用在Zookeeper这种共享集群配置的系统中而很少在需要大量数据的系统中使用的原因。HDFS的HA功能也是基于“少数服从多数”的方式，但是其数据存储并不是采用这样的方式。

实际上，leader选举的算法非常多，比如Zookeeper的Zab、Raft以及Viewstamped Replication。而Kafka所使用的leader选举算法更像是微软的PacificA算法。

Kafka在Zookeeper中为每一个partition动态的维护了一个ISR，这个ISR里的所有replication都跟上了leader，只有ISR里的成员才能有被选为leader的可能(unclean.leader.election.enable=false)。在这种模式下，对于f+1个副本，一个Kafka topic能在保证不丢失已经commit消息的前提下容忍f个副本的失败，在大多数使用场景下，这种模式是十分有利的。事实上，为了容忍f个副本的失败，“少数服从多数”的方式和ISR在commit前需要等待的副本的数量是一样的，但是ISR需要的总的副本的个数几乎是“少数服从多数”的方式的一半。

上文提到，在ISR中至少有一个follower时，Kafka可以确保已经commit的数据不丢失，但如果某一个partition的所有replica都挂了，就无法保证数据不丢失了。这种情况下有两种可行的方案：

等待ISR中任意一个replica“活”过来，并且选它作为leader

选择第一个“活”过来的replica(并不一定是在ISR中)作为leader

这就需要在可用性和一致性当中作出一个简单的抉择。如果一定要等待ISR中的replica“活”过来，那不可用的时间就可能会相对较长。而且如果ISR中所有的replica都无法“活”过来了，或者数据丢失了，这个partition将永远不可用。选择第一个“活”过来的replica作为leader,而这个replica不是ISR中的replica,那即使它并不保障已经包含了所有已commit的消息，它也会成为leader而作为consumer的数据源。默认情况下，Kafka采用第二种策略，即　　 

unclean.leader.election.enable=true，也可以将此参数设置为false来启用第一种策略。

unclean.leader.election.enable这个参数对于leader的选举、系统的可用性以及数据的可靠性都有至关重要的影响。

下面我们来分析下几种典型的场景。



如果上图所示，假设某个partition中的副本数为3，replica-0, replica-1, replica-2分别存放在broker0, broker1和broker2中。AR=(0,1,2)，ISR=(0,1)。

设置request.required.acks=-1, min.insync.replicas=2，unclean.leader.election.enable=false。这里讲broker0中的副本也称之为broker0起初broker0为leader，broker1为follower。

当ISR中的replica-0出现crash的情况时，broker1选举为新的leader[ISR=(1)]，因为受min.insync.replicas=2影响，write不能服务，但是read能继续正常服务。此种情况恢复方案：

尝试恢复(重启)replica-0，如果能起来，系统正常;
如果replica-0不能恢复，需要将min.insync.replicas设置为1，恢复write功能。

当ISR中的replica-0出现crash，紧接着replica-1也出现了crash, 此时[ISR=(1),leader=-1],不能对外提供服务，此种情况恢复方案：

尝试恢复replica-0和replica-1，如果都能起来，则系统恢复正常;
如果replica-0起来，而replica-1不能起来，这时候仍然不能选出leader，因为当设置unclean.leader.election.enable=false时，leader只能从ISR中选举，当ISR中所有副本都失效之后，需要ISR中最后失效的那个副本能恢复之后才能选举leader, 即replica-0先失效，replica-1后失效，需要replica-1恢复后才能选举leader。保守的方案建议把unclean.leader.election.enable设置为true,但是这样会有丢失数据的情况发生，这样可以恢复read服务。同样需要将min.insync.replicas设置为1，恢复write功能;replica-1恢复，replica-0不能恢复，这个情况上面遇到过，read服务可用，需要将min.insync.replicas设置为1，恢复write功能;
replica-0和replica-1都不能恢复，这种情况可以参考情形2.

当ISR中的replica-0, replica-1同时宕机,此时[ISR=(0,1)],不能对外提供服务，此种情况恢复方案：尝试恢复replica-0和replica-1，当其中任意一个副本恢复正常时，对外可以提供read服务。直到2个副本恢复正常，write功能才能恢复，或者将将min.insync.replicas设置为1。

Kafka的Leader选举机制分为两个层次：Kafka Controller的选举和分区Leader的选举。这两个过程共同保障了集群的高可用性和数据一致性。以下是详细的选举流程：

Kafka Controller的选举

Controller是Kafka集群的核心协调者，负责管理分区、副本状态及触发Leader选举。其选举过程如下：

创建临时节点：所有Broker启动时尝试在ZooKeeper的/controller路径下创建临时节点，节点内容为Broker的ID。

竞争机制：仅有一个Broker能成功创建该节点，该Broker成为Controller，其他Broker则监听该节点的变化。

失效处理：若Controller宕机或与ZooKeeper断开连接，临时节点被删除，其他Broker立即重新竞争创建新节点，确保新Controller快速选出。

Session超时恢复：若Controller与ZooKeeper重新建立连接，会触发SessionExpirationListener，关闭旧资源并重新参与选举。

分区Leader的选举

当分区的Leader副本失效或集群配置变更时，由Controller负责触发分区Leader的选举。

触发条件

Leader故障：如Broker宕机、网络分区等。

集群扩缩容：新增或移除Broker导致副本分布变化。

配置变更：如副本集调整或ISR（In-Sync Replicas）列表变化。

选举流程

检测Leader失效
Controller通过心跳机制监控Broker状态，若超过阈值未收到心跳，则判定Leader失效。

检查ISR列表
从该分区的ISR（与Leader同步的副本集合）中选择新Leader。若ISR为空，可能触发Unclean Leader选举（需配置允许）。

选择新Leader的策略

优先级规则：优先选择当前Broker上的副本（减少分区迁移）或同步状态最佳的副本（Log End Offset最大）。

默认策略：通常选择ISR列表中的第一个副本，因其同步状态最接近旧Leader。

更新元数据

Controller将新Leader信息写入ZooKeeper的对应分区节点（早期版本）或直接通知集群（新版本）。

所有Broker监控元数据变化并更新本地缓存。

通知客户端
生产者和消费者通过元数据刷新获取新Leader信息，并重新建立连接。

Unclean Leader选举

触发条件：当ISR列表为空（如所有副本均滞后或失联），且配置unclean.leader.election.enable=true。

风险：可能选择数据滞后的副本作为Leader，导致数据丢失或不一致。

避免方法：设置min.insync.replicas确保足够同步副本，并禁用Unclean选举。

数据一致性与高可用性保障

ISR机制：仅同步副本可参与选举，避免数据不一致。

配置优化：

replica.lag.time.max.ms：定义副本滞后时间阈值，超时则移出ISR。

min.insync.replicas：控制写入时最少同步副本数，防止ISR不足。

快速恢复：新Leader选举通常在毫秒级完成，客户端重平衡短暂，服务中断时间极短。

总结

Kafka的Leader选举通过Controller协调，结合ZooKeeper的临时节点和ISR机制，实现高效故障恢复。正常选举优先选择同步副本，极端情况下允许Unclean选举以维持可用性（需权衡数据一致性）。合理配置参数（如min.insync.replicas）是保障高可用的关键。

9.5、Kafka的发送模式

Kafka的发送模式由producer端的配置参数producer.type来设置，这个参数指定了在后台线程中消息的发送方式是同步的还是异步的，默认是同步的方式，即producer.type=sync。如果设置成异步的模式，即producer.type=async，可以是producer以batch的形式push数据，这样会极大的提高broker的性能，但是这样会增加丢失数据的风险。如果需要确保消息的可靠性，必须要将producer.type设置为sync。

对于异步模式，还有4个配套的参数，如下：



以batch的方式推送数据可以极大的提高处理效率，kafka producer可以将消息在内存中累计到一定数量后作为一个batch发送请求。batch的数量大小可以通过producer的参数(batch.num.messages)控制。通过增加batch的大小，可以减少网络请求和磁盘IO的次数，当然具体参数设置需要在效率和时效性方面做一个权衡。在比较新的版本中还有batch.size这个参数。

十、高可靠性使用分析

10.1、消息传输保障

前面已经介绍了Kafka如何进行有效的存储，以及了解了producer和consumer如何工作。接下来讨论的是Kafka如何确保消息在producer和consumer之间传输。有以下三种可能的传输保障(delivery guarantee):

At most once: 消息可能会丢，但绝不会重复传输

At least once：消息绝不会丢，但可能会重复传输

Exactly once：每条消息肯定会被传输一次且仅传输一次

Kafka的消息传输保障机制非常直观。当producer向broker发送消息时，一旦这条消息被commit，由于副本机制(replication)的存在，它就不会丢失。但是如果producer发送数据给broker后，遇到的网络问题而造成通信中断，那producer就无法判断该条消息是否已经提交(commit)。虽然Kafka无法确定网络故障期间发生了什么，但是producer可以retry多次，确保消息已经正确传输到broker中，所以目前Kafka实现的是at least once。

consumer从broker中读取消息后，可以选择commit，该操作会在Zookeeper中存下该consumer在该partition下读取的消息的offset。该consumer下一次再读该partition时会从下一条开始读取。如未commit，下一次读取的开始位置会跟上一次commit之后的开始位置相同。当然也可以将consumer设置为autocommit，即consumer一旦读取到数据立即自动commit。如果只讨论这一读取消息的过程，那Kafka是确保了exactly once, 但是如果由于前面producer与broker之间的某种原因导致消息的重复，那么这里就是at least once。

考虑这样一种情况，当consumer读完消息之后先commit再处理消息，在这种模式下，如果consumer在commit后还没来得及处理消息就crash了，下次重新开始工作后就无法读到刚刚已提交而未处理的消息，这就对应于at most once了。

读完消息先处理再commit。这种模式下，如果处理完了消息在commit之前consumer crash了，下次重新开始工作时还会处理刚刚未commit的消息，实际上该消息已经被处理过了，这就对应于at least once。

要做到exactly once就需要引入消息去重机制。

10.2、消息去重

Kafka 本身并不提供内置的全局消息去重机制，但在实际应用中可以通过 生产者端幂等性、消费者端去重逻辑

1. 生产者端去重：避免消息重复发送

(1) 幂等生产者（Idempotent Producer）

原理：通过为每条消息分配唯一序列号（Sequence Number），Broker 端会缓存每个生产者（Producer ID）最近的消息序列号，拒绝重复提交。

配置方法：


properties.put("enable.idempotence", "true"); // 开启幂等性
properties.put("acks", "all"); // 必须设置 acks=all
特点：

单分区单会话去重：仅保证同一生产者、同一分区、同一会话（Producer 实例未重启）内的消息不重复。

轻量级：几乎无性能损耗，默认从 Kafka 0.11 开始支持。

(2) 事务（Transactional Producer）

原理：结合事务和消费者偏移量提交，实现跨分区和跨会话的 Exactly-Once 语义。

配置方法：


properties.put("transactional.id", "unique-transaction-id"); // 设置唯一事务ID
producer.initTransactions(); // 初始化事务
使用流程：


producer.beginTransaction();
producer.send(record1);
producer.send(record2);
producer.sendOffsetsToTransaction(offsets, consumerGroup); // 提交消费者偏移量
producer.commitTransaction();
特点：

跨分区原子性：事务内的多条消息要么全部成功，要么全部失败。

Exactly-Once 语义：结合消费者端的 isolation.level=read_committed，确保消息不重复消费。

2. 消费者端去重：避免重复消费

(1) 业务逻辑去重

唯一标识符：为每条消息分配唯一 ID（如 UUID、时间戳+业务字段），消费者将已处理 ID 持久化到数据库或缓存（如 Redis）。

示例：


String messageId = record.headers().lastHeader("Message-ID").value();
if (!redis.exists(messageId)) {
    process(record);
    redis.set(messageId, "processed");
}
缺点：依赖外部存储，可能成为性能瓶颈。

(2) 基于偏移量（Offset）的去重

原理：消费者记录已处理的偏移量，重启时跳过已处理的消息。

实现：

将偏移量与处理结果一起提交到事务型数据库。

消费前检查当前偏移量是否已被处理。

缺点：需手动管理偏移量，复杂度高。

(3) 幂等消费设计

原理：业务逻辑天然幂等（如 UPDATE 操作），重复消费不影响最终结果。

适用场景：

数据库写操作：使用唯一键或覆盖更新。

状态机设计：通过状态判断避免重复执行。

10.3、高可靠性配置

Kafka提供了很高的数据冗余弹性，对于需要数据高可靠性的场景，我们可以增加数据冗余备份数(replication.factor)，调高最小写入副本数的个数(min.insync.replicas)等等，但是这样会影响性能。反之，性能提高而可靠性则降低，用户需要自身业务特性在彼此之间做一些权衡性选择。

要保证数据写入到Kafka是安全的，高可靠的，需要如下的配置：

topic的配置：replication.factor>=3,即副本数至少是3个;2<=min.insync.replicas<=replication.factor

broker的配置：leader的选举条件unclean.leader.election.enable=false

producer的配置：request.required.acks=-1(all)，producer.type=sync

十一、内部网络框架



Broker的内部处理流水线化，分为多个阶段来进行(SEDA)，以提高吞吐量和性能，尽量避免Thead盲等待，以下为过程说明。

Accept Thread负责与客户端建立连接链路，然后把Socket轮转交给Process Thread

Process Thread负责接收请求和响应数据，Process Thread每次基于Selector事件循环，首先从Response Queue读取响应数据，向客户端回复响应，然后接收到客户端请求后，读取数据放入Request Queue。

Work Thread负责业务逻辑、IO磁盘处理等，负责从Request Queue读取请求，并把处理结果放入Response Queue中，待Process Thread发送出去。

十二、rebalance机制



Kafka的Consumer Rebalance（消费者再平衡）机制是确保消费者组内成员动态变化时，分区分配合理且一致的核心协议。其核心逻辑涉及触发条件、协调器选举、分区分配策略及优化方法，具体如下：

一、触发条件

以下情况会触发Rebalance：

消费者组成员数变化：新消费者加入或现有消费者退出（如宕机、主动关闭）。

订阅主题数变化：消费者组使用正则订阅主题时，新增符合条件的主题。

订阅主题的分区数变化：例如通过kafka-topics命令动态增加分区。

二、Rebalance流程

Rebalance分为三个阶段，由协调器（Coordinator）主导：

协调器选举：

每个消费者组通过计算groupId的哈希值与__consumer_offsets分区数的模，确定负责该组的协调器所在的Broker。

例如，groupId哈希值对50取模得到分区号，该分区Leader所在的Broker即为协调器。

消费者加入组（Join Group）：

消费者向协调器发送JoinGroupRequest请求，包含订阅信息。

协调器选举消费者组的Leader（通常是第一个加入的消费者），Leader负责制定分区分配方案。

同步分配方案（Sync Group）：

Leader将分配方案通过SyncGroupRequest发送给协调器。

协调器将方案下发给所有消费者，消费者根据新分配的分区开始消费。

三、分区分配策略

Kafka提供三种分配策略，通过partition.assignment.strategy配置：

Range（范围分配）：

按分区编号排序，平均分配给消费者。若无法整除，前几个消费者多分配一个分区。

示例：10个分区，3个消费者 → 分配为4-3-3。

Round-Robin（轮询）：

所有分区按哈希排序后轮询分配，需保证消费者线程数相同。

示例：10个分区轮询分配给线程C1-0、C2-0、C2-1 → 可能分配为3-3-4。

Sticky（粘性）：

初始分配类似Round-Robin，但Rebalance时尽量保留原有分配，减少分区变动。

示例：若某消费者宕机，其原分区优先分配给剩余消费者，而非完全重新分配。

四、Rebalance的缺陷与优化

1. 主要问题 ：

Herd Effect（羊群效应）：任何Broker或消费者的变动都会触发全组Rebalance。

Split Brain（脑裂）：消费者对集群状态的视图不一致，可能导致错误分配。

性能损耗：Rebalance期间消费者暂停消费，影响吞吐量。

2. 优化方法 ：

调整心跳参数：

session.timeout.ms（默认10秒）：消费者心跳超时时间，建议设为6秒。

heartbeat.interval.ms（默认3秒）：心跳间隔，建议设为2秒，需满足session.timeout.ms ≥ 3 * heartbeat.interval.ms。

session.timeout.ms: 消费者在协调器（Coordinator）中被判定为“离线”的最大无心跳时间
heartbeat.interval.ms: 消费者发送心跳到协调器的间隔时间（需满足 session.timeout.ms ≥ 3 * heartbeat.interval.ms）

为什么是3倍？ 协调器容忍最多 3次心跳失败才会判定消费者离线。

控制消费耗时：

max.poll.interval.ms（默认5分钟）：单次poll()处理的最大时间，需根据业务处理时间调整。

max.poll.records（默认500）：单次拉取消息数，减少单次处理量以避免超时。

避免频繁变动：合理规划消费者数量与分区数的比例，减少扩容/缩容操作。

五、示例与场景

场景1：消费者组初始有2个消费者，订阅含3个分区的Topic。分配为ConsumerA（2分区）、ConsumerB（1分区）。

场景2：新增ConsumerC触发Rebalance，采用Range策略分配为1-1-1；若使用Sticky策略，在保持Consumer原有分配的基础上，将部分分区迁移到新消费者。

总结

Kafka的Rebalance机制通过协调器动态分配分区，核心在于权衡一致性与性能。合理配置参数（如心跳间隔、消费超时）和选择分配策略（如Sticky）可显著减少对系统的影响。未来版本可能进一步优化分配算法，减少"全量重分配"的开销。

十三、BenchMark

Kafka在唯品会有着很深的历史渊源，根据唯品会消息中间件团队(VMS团队)所掌握的资料显示，在VMS团队运转的Kafka集群中所支撑的topic数已接近2000，每天的请求量也已达千亿级。这里就以Kafka的高可靠性为基准点来探究几种不同场景下的行为表现，以此来加深对Kafka的认知，为大家在以后高效的使用Kafka时提供一份依据。

13.1、测试环境

Kafka broker用到了4台机器，分别为broker[0/1/2/3]配置如下：

CPU: 24core/2.6GHZ
Memory: 62G
Network: 4000Mb
OS/kernel: CentOs release 6.6 (Final)
Disk: 1089G
Kafka版本：0.10.1.0
broker端JVM参数设置：


-Xmx8G -Xms8G -server -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled -XX:+CMSScavengeBeforeRemark -XX:+DisableExplicitGC -Djava.awt.headless=true -Xloggc:/apps/service/kafka/bin/../logs/kafkaServer-gc.log -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=9999

客户端机器配置：

CPU: 24core/2.6GHZ

Memory: 3G

Network: 1000Mb

OS/kernel: CentOs release 6.3 (Final)

Disk: 240G

13.2、不同场景测试

场景1：

测试不同的副本数、min.insync.replicas策略以及request.required.acks策略(以下简称acks策略)对于发送速度(TPS)的影响。

具体配置：一个producer;发送方式为sync;消息体大小为1kB;partition数为12。副本数为：1/2/4;min.insync.replicas分别为1/2/4;acks分别为-1(all)/1/0。

具体测试数据如下表(min.insync.replicas只在acks=-1时有效)：



测试结果分析：

客户端的acks策略对发送的TPS有较大的影响，TPS：acks_0 > acks_1 > ack_-1;

副本数越高，TPS越低;副本数一致时，min.insync.replicas不影响TPS;

acks=0/1时，TPS与min.insync.replicas参数以及副本数无关，仅受acks策略的影响。

下面将partition的个数设置为1，来进一步确认下不同的acks策略、不同的min.insync.replicas策略以及不同的副本数对于发送速度的影响，详细请看情景2和情景3。

场景2：

在partition个数固定为1，测试不同的副本数和min.insync.replicas策略对发送速度的影响。

具体配置：一个producer;发送方式为sync;消息体大小为1kB;producer端acks=-1(all)。变换副本数：2/3/4; min.insync.replicas设置为：1/2/4。

测试结果如下：



测试结果分析：

副本数越高，TPS越低(这点与场景1的测试结论吻合)，但是当partition数为1时差距甚微。min.insync.replicas不影响TPS。

场景3：

在partition个数固定为1，测试不同的acks策略和副本数对发送速度的影响。

具体配置：一个producer;发送方式为sync;消息体大小为1kB;min.insync.replicas=1。topic副本数为：1/2/4;acks： 0/1/-1。

测试结果如下：



测试结果分析(与情景1一致)：

副本数越多，TPS越低;

客户端的acks策略对发送的TPS有较大的影响，TPS：acks_0 > acks_1 > ack_-1。

场景4：

测试不同partition数对发送速率的影响

具体配置：一个producer;消息体大小为1KB;发送方式为sync;topic副本数为2;min.insync.replicas=2;acks=-1。partition数量设置为1/2/4/8/12。



测试结果分析：

partition的不同会影响TPS，随着partition的个数的增长TPS会有所增长，但并不是一直成正比关系，到达一定临界值时，partition数量的增加反而会使TPS略微降低。

场景5：

通过将集群中部分broker设置成不可服务状态，测试对客户端以及消息落盘的影响。

具体配置：一个producer;消息体大小1KB;发送方式为sync;topic副本数为4;min.insync.replicas设置为2;acks=-1;retries=0/100000000;partition数为12。



测试结果分析：

kill两台broker后，客户端可以继续发送。broker减少后，partition的leader分布在剩余的两台broker上，造成了TPS的减小;

kill三台broker后，客户端无法继续发送。Kafka的自动重试功能开始起作用，当大于等于min.insync.replicas数量的broker恢复后，可以继续发送;

当retries不为0时，消息有重复落盘;客户端成功返回的消息都成功落盘，异常时部分消息可以落盘。

场景6：

测试单个producer的发送延迟，以及端到端的延迟。

具体配置：一个producer;消息体大小1KB;发送方式为sync;topic副本数为4;min.insync.replicas设置为2;acks=-1;partition数为12。

测试数据及结果(单位为ms)：



各场景测试总结：

当acks=-1时，Kafka发送端的TPS受限于topic的副本数量(ISR中)，副本越多TPS越低;

acks=0时，TPS最高，其次为1，最差为-1，即TPS：acks_0 > acks_1 > ack_-1

min.insync.replicas参数不影响TPS;

partition的不同会影响TPS，随着partition的个数的增长TPS会有所增长，但并不是一直成正比关系，到达一定临界值时，partition数量的增加反而会使TPS略微降低;

Kafka在acks=-1,min.insync.replicas>=1时，具有高可靠性，所有成功返回的消息都可以落盘。 