## Kafka重要知识点

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-75cd70ad3052ba44bf706a3ab39e59d5_720w.webp)


Kafka 消费端确保一个 Partition 在一个消费者组内只能被一个消费者消费。这句话改怎么理解呢？

1.  在同一个消费者组内，一个 Partition 只能被一个消费者消费。
2.  在同一个消费者组内，所有消费者组合起来必定可以消费一个 Topic 下的所有 Partition。
3.  在同一个消费组内，一个消费者可以消费多个 Partition 的信息。
4.  在不同消费者组内，同一个分区可以被多个消费者消费。
5.  每个消费者组一定会完整消费一个 Topic 下的所有 Partition。

## 消费组存在的意义

了解了消费者与消费组的关系后，有朋友会比较疑惑消费者组有啥实际存在的意义呢？或者说消费组的作用是什么？

作者对消费组的作用归结了如下两点。

1.  在实际生产中，对于同一个 Topic，可能有 A、B、C 等 N 个消费方想要消费。比如一份用户点击日志，A 消费方想用来做一个用户近 N 天点击过哪些商品；B 消费方想用来做一个用户近 N 天点击过前 TopN 个相似的商品；C 消费方想用来做一个根据用户点击过的商品推荐相关周边的商品需求。对于多应用场景，就可以使用消费组来隔离不同的业务使用场景，从而达到一个 Topic 可以被多个消费组重复消费的目的。
2.  消费组与 Partition 的消费进度绑定。当有新的消费者加入或者有消费者从消费组退出时，会触发消费组的 Repartition 操作（后面会详细介绍 Repartition）；在 Repartition 前，Partition1 被消费组的消费者 A 进行消费，Repartition 后，Partition1 消费组的消费者 B 进行消费，为了避免消息被重复消费，需要从消费组记录的 Partition 消费进度读取当前消费到的位置（即 OffSet 位置），然后在继续消费，从而达到消费者的平滑迁移，同时也提高了系统的可用性。

## Repartition 触发时机

使用过 Kafka 消费者客户端的同学肯定知道，消费者组内偶尔会触发 Repartition 操作，所谓 Repartition 即 Partition 在某些情况下重新被分配给参与消费的消费者。基本可以分为如下几种情况。

1.  消费组内某消费者宕机，触发 Repartition 操作，如下图所示。

<figure data-size="normal">





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-a9ef6a29cb9ba3456a05ad75cb91cb03_720w.webp)

2\. 消费组内新增消费者，触发 Repartition 操作，如下图所示。一般这种情况是为了提高消费端的消费能力，从而加快消费进度。

<figure data-size="normal">





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-8803223d712fdde035b8e7b9170dd3fb_720w.webp)



3.Topic 下的 Partition 增多，触发 Repartition 操作，如下图所示。一般这种调整 Partition 个数的情况也是为了提高消费端消费速度的，因为当消费者个数大于等于 Partition 个数时，在增加消费者个数是没有用的（原因是：在一个消费组内，消费者:Partition = 1:N，当 N 小于 1 时，相当于消费者过剩了），所以一方面增加 Partition 个数同时增加消费者个数可以提高消费端的消费速度。

<figure data-size="normal">





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-8f1a427c6842d9babf139454ce23cfa3_720w.webp)



## 消费者与 ZK 的关系

众所周知，ZK 不仅保存了消费者消费 partition 的进度，同时也保存了消费组的成员列表、partition 的所有者。消费者想要消费 Partition，需要从 ZK 中获取该消费者对应的分区信息及当前分区对应的消费进度，即 OffSert 信息。那么 Partition 应该由那个消费者进行消费，决定因素有哪些呢？从之前的图中不难得出，两个重要因素分别是：消费组中存活的消费者列表和 Topic 对应的 Partition 列表。通过这两个因素结合 Partition 分配算法，即可得出消费者与 Partition 的对应关系，然后将信息存储到 ZK 中。Kafka 有高级 API 和低级 API，如果不需要操作 OffSet 偏移量的提交，可通过高级 API 直接使用，从而降低使用者的难度。对于一些比较特殊的使用场景，比如想要消费特定 Partition 的信息，Kafka 也提供了低级 API 可进行手动操作。

## 消费端工作流程

在介绍消费端工作流程前，先来熟悉一下用到的一些组件。

*   `KakfaConsumer`：消费端，用于启动消费者进程来消费消息。
*   `ConsumerConfig`：消费端配置管理，用于给消费端配置相关参数，比如指定 Kafka 集群，设置自动提交和自动提交时间间隔等等参数，都由其来管理。
*   `ConsumerConnector`：消费者连接器，通过消费者连接器可以获得 Kafka 消息流，然后通过消息流就能获得消息从而使得客户端开始消费消息。

以上三者之间的关系可以概括为：消费端使用消费者配置管理创建出了消费者连接器，通过消费者连接器创建队列（这个队列的作用也是为了缓存数据），其中队列中的消息由专门的拉取线程从服务端拉取然后写入，最后由消费者客户端轮询队列中的消息进行消费。具体操作流程如下图所示。

<figure data-size="normal">





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-122b4a706de39655d257928005a83ff1_720w.webp)



我们在从消费者与 ZK 的角度来看看其工作流程是什么样的？

<figure data-size="normal">





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-4ed25ebb9236986b2084ce8a042f65b9_720w.webp)



## Offset提交流程

从上图可以看出，首先拉取线程每拉取一次消息，同步更新一次拉取状态，其作用是为了下一次拉取消息时能够拉取到最新产生的消息；拉取线程将拉取到的消息写入到队列中等待消费消费线程去真正读取处理。消费线程以轮询的方式持续读取队列中的消息，只要发现队列中有消息就开始消费，消费完消息后更新消费进度，此处需要注意的是，消费线程不是每次都和 ZK 同步消费进度，而是将消费进度暂时写入本地。这样做的目的是为了减少消费者与 ZK 的频繁同步消息，从而降低 ZK 的压力。

> kafka中，消费者会先将消费进度保存到本地，以减少跟zookeeper的交互频次，那么拉取线程是如何每次都能够拉取到最新的消息的呢？
>
> - **拉取线程的工作流程**：
>   1. 消费者根据本地缓存的偏移量向 Broker 发送 `FetchRequest`。
>   2. Broker 返回该偏移量之后的消息（若偏移量无效，根据 `auto.offset.reset` 策略处理）。
>   3. 消费者更新本地偏移量，并定期提交到协调器。
>
> 也就是说跟zookeeper交互的目的是为了获取offset，获取到之后就可以通过api去broker请求对应的消息，后面只需要定期将消费到的offset位置持久化到zookeeper上
>
> 注：Kafka 0.9前版本offset存储在zookeeper，0.9以及之后存储在内部topic __consumer_offsets



Kafka 消费者将 offset 提交到 `__consumer_offsets` 的流程是一个涉及协调器、内部主题和分区机制的复杂过程。以下是详细的流程说明：

------

### 1. 确定协调器（Coordinator）

消费者需要找到负责管理其消费者组的 **协调器（Coordinator）**，协调器是 Kafka 集群中的一个 Broker。具体步骤如下：

1. **计算分区**：消费者根据消费者组 ID（`group.id`）的哈希值，对 `__consumer_offsets` 的分区数取模（默认 50 个分区），确定目标分区。
   - 公式：`partition = hash(group.id) % 50`
2. **找到 Leader Broker**：目标分区的 Leader 所在的 Broker 即为该消费者组的协调器。
3. **连接协调器**：消费者与协调器建立连接，后续的 offset 提交和 Rebalance 均通过此 Broker 完成。

------

### 2. Offset 提交的触发条件

消费者提交 offset 的触发方式有两种：

1. **自动提交**（默认）：
   - 由消费者后台线程周期性触发，间隔由 `auto.commit.interval.ms` 配置（默认 5 秒）。
   - 消费者拉取消息后，若超过时间间隔未提交，则自动提交最后拉取的 offset。
2. **手动提交**：
   - 调用 `commitSync()`（同步提交）或 `commitAsync()`（异步提交）显式提交。
   - 通常在业务逻辑处理完成后手动提交，确保“至少一次”或“精确一次”语义。

------

### 3. Offset 提交的详细流程

#### (1) 消费者准备提交请求

消费者将待提交的 offset 按分区整理，生成 **OffsetCommitRequest**，包含以下信息：

- **消费者组 ID**：标识所属的消费者组。
- **Topic-Partition 列表**：每个分区的当前 offset 值。
- **元数据**（可选）：自定义信息（如处理时间戳）。

#### (2) 发送 OffsetCommitRequest 到协调器

消费者向协调器发送请求，协调器验证请求的合法性：

- 消费者是否有权限提交 offset（例如是否为消费者组的有效成员）。
- 分区是否属于当前消费者组的分配范围。

#### (3) 协调器写入 __consumer_offsets

协调器将 offset 写入 `__consumer_offsets` 主题的对应分区：

- **Key**：由 `group.id`、`topic` 和 `partition` 组成，唯一标识一个分区的 offset。
- **Value**：包含 offset 值、元数据和时间戳。
- **副本同步**：由于 `__consumer_offsets` 的 `acks=all`，需等待所有副本确认写入成功。

#### (4) 响应消费者

- **成功**：协调器返回成功响应，消费者记录提交状态。
- **失败**：若写入失败（如副本不可用），协调器返回错误，消费者根据配置的重试策略（如 `retries`）重新提交。

------

### 4. 数据可靠性保障

Kafka 通过以下机制确保 offset 提交的可靠性：

1. **副本机制**：`__consumer_offsets` 默认配置为多副本（`replication.factor=3`），数据持久化到多个 Broker。
2. **高水位（HW）同步**：所有副本同步完成才确认写入，避免数据丢失。
3. **消费者重试**：手动提交时，若失败可重试；自动提交则由后台线程周期性重试。

------

### 5. 示例场景

假设消费者组 `group-1` 订阅了 Topic `orders`（3 个分区 P0、P1、P2），当前 offset 分别为 100、150、200。

1. **自动提交**：
   - 消费者拉取消息后，每隔 5 秒触发提交。
   - 后台线程将 `orders-0:100`, `orders-1:150`, `orders-2:200` 封装为 OffsetCommitRequest，发送到协调器。
   - 协调器写入 `__consumer_offsets`，返回成功响应。
2. **手动提交**：
   - 消费者处理完 P0 的消息后调用 `commitSync()`，显式提交 `orders-0:100`。
   - 若提交失败（如网络抖动），消费者可选择重试或抛出异常。

------

### 6. 关键配置参数

| **参数**                    | **说明**                                                 |
| :-------------------------- | :------------------------------------------------------- |
| `enable.auto.commit`        | 是否启用自动提交（默认 `true`）。                        |
| `auto.commit.interval.ms`   | 自动提交间隔（默认 5000 毫秒）。                         |
| `offsets.retention.minutes` | `__consumer_offsets` 中 offset 的保留时间（默认 7 天）。 |
| `acks`（Broker 配置）       | `__consumer_offsets` 的写入确认机制（默认 `all`）。      |



## 消费者的三种消费情况

消费者从服务端的 Partition 上拉取到消息，消费消息有三种情况，分别如下：

1.  至少一次。即一条消息至少被消费一次，消息不可能丢失，但是可能会被重复消费。
2.  至多一次。即一条消息最多可以被消费一次，消息不可能被重复消费，但是消息有可能丢失。
3.  正好一次。即一条消息正好被消费一次，消息不可能丢失也不可能被重复消费。

### 1.至少一次

消费者读取消息，先处理消息，在保存消费进度。消费者拉取到消息，先消费消息，然后在保存偏移量，当消费者消费消息后还没来得及保存偏移量，则会造成消息被重复消费。如下图所示：

<figure data-size="normal">





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-1a047ed616ba44daebdb4b6ce786a61a_720w.webp)



### 2.至多一次

消费者读取消息，先保存消费进度，在处理消息。消费者拉取到消息，先保存了偏移量，当保存了偏移量后还没消费完消息，消费者挂了，则会造成未消费的消息丢失。如下图所示：

<figure data-size="normal">





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-1f9f91ae54396c5e5d93ae89251eb1ed_720w.webp)



### 3.正好一次

正好消费一次的办法可以通过将消费者的消费进度和消息处理结果保存在一起。只要能保证两个操作是一个原子操作，就能达到正好消费一次的目的。通常可以将两个操作保存在一起，比如 HDFS 中。正好消费一次流程如下图所示。

<figure data-size="normal">





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-a0bbb114e2ad551227f81c1f26d4bd5d_720w.webp)



## Partition、Replica、Log 和 LogSegment 的关系

假设有一个 Kafka 集群，Broker 个数为 3，Topic 个数为 1，Partition 个数为 3，Replica 个数为 2。Partition 的物理分布如下图所示。





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-f8f21631b138321f25c8821c677c5579_720w.webp)



从上图可以看出，该 Topic 由三个 Partition 构成，并且每个 Partition 由主从两个副本构成。每个 Partition 的主从副本分布在不同的 Broker 上，通过这点也可以看出，当某个 Broker 宕机时，可以将分布在其他 Broker 上的从副本设置为主副本，因为只有主副本对外提供读写请求，当然在最新的 2.x 版本中从副本也可以对外读请求了。将主从副本分布在不同的 Broker 上从而提高系统的可用性。

Partition 的实际物理存储是以 Log 文件的形式展示的，而每个 Log 文件又以多个 LogSegment 组成。Kafka 为什么要这么设计呢？其实原因比较简单，随着消息的不断写入，Log 文件肯定是越来越大，Kafka 为了方便管理，将一个大文件切割成一个一个的 LogSegment 来进行管理；每个 LogSegment 由数据文件和索引文件构成，数据文件是用来存储实际的消息内容，而索引文件是为了加快消息内容的读取。

可能又有朋友会问，Kafka 本身消费是以 Partition 维度顺序消费消息的，磁盘在顺序读的时候效率很高完全没有必要使用索引啊。其实 Kafka 为了满足一些特殊业务需求，比如要随机消费 Partition 中的消息，此时可以先通过索引文件快速定位到消息的实际存储位置，然后进行处理。

总结一下 Partition、Replica、Log 和 LogSegment 之间的关系。消息是以 Partition 维度进行管理的，为了提高系统的可用性，每个 Partition 都可以设置相应的 Replica 副本数，一般在创建 Topic 的时候同时指定 Replica 的个数；Partition 和 Replica 的实际物理存储形式是通过 Log 文件展现的，为了防止消息不断写入，导致 Log 文件大小持续增长，所以将 Log 切割成一个一个的 LogSegment 文件。

**注意：** 在同一时刻，每个主 Partition 中有且只有一个 LogSegment 被标识为可写入状态，当一个 LogSegment 文件大小超过一定大小后（比如当文件大小超过 1G，这个就类似于 HDFS 存储的数据文件，HDFS 中数据文件达到 128M 的时候就会被分出一个新的文件来存储数据），就会新创建一个 LogSegment 来继续接收新写入的消息。

## 写入消息流程分析





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-eb66e4ecf7cf07fcb6b12029bfdd9b71_720w.webp)



流程解析

在第 3 篇文章讲过，生产者客户端对于每个 Partition 一次会发送一批消息到服务端，服务端收到一批消息后写入相应的 Partition 上。上图流程主要分为如下几步：

1.  **客户端消息收集器收集属于同一个分区的消息，并对每条消息设置一个偏移量，且每一批消息总是从 0 开始单调递增。比如第一次发送 3 条消息，则对三条消息依次编号 [0,1,2]，第二次发送 4 条消息，则消息依次编号为 [0,1,2,3]。注意此处设置的消息偏移量是相对偏移量。**
2.  **客户端将消息发送给服务端，服务端拿到下一条消息的绝对偏移量，将传到服务端的这批消息的相对偏移量修改成绝对偏移量。**
3.  **将修改后的消息以追加的方式追加到当前活跃的 LogSegment 后面，然后更新绝对偏移量。**
4.  **将消息集写入到文件通道。**
5.  **文件通道将消息集 flush 到磁盘，完成消息的写入操作。**

了解以上过程后，我们在来看看消息的具体构成情况。

<figure data-size="normal">





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-6e993c95decd5d274b032cd423936504_720w.webp)



一条消息由如下三部分构成：

*   **OffSet：偏移量，消息在客户端发送前将相对偏移量存储到该位置，当消息存储到 LogSegment 前，先将其修改为绝对偏移量在写入磁盘。**
*   **Size：本条 Message 的内容大小**
*   **Message：消息的具体内容，其具体又由 7 部分组成，crc 用于校验消息，Attribute 代表了属性，key-length 和 value-length 分别代表 key 和 value 的长度，key 和 value 分别代表了其对应的内容。**



在 Kafka 中，**同步发送消息**（即调用 `producer.send(record).get()`）和**消息的批处理机制**（消息累积成批次再发送）是两个独立的概念。即使采用同步方式发送消息，Kafka Producer 依然会尽可能累积一批消息再批量发送到 Broker，以提高吞吐量。以下是详细说明：

------

### 1. 同步发送与批处理的关系

#### **(1) 同步发送的定义**

- **同步发送**：调用 `send()` 方法后立即调用 `get()`，阻塞当前线程直到 Broker 返回写入结果的确认（ACK）。

  ```
  // 同步发送示例（Java）
  producer.send(record).get(); // 阻塞直到收到ACK
  ```

- **特点**：

  - **阻塞性**：发送线程必须等待消息写入 Broker 完成才能继续。
  - **高可靠性**：可立即感知发送失败并进行重试。

#### (2) 批处理机制

- **批处理（Batching）**：Producer 会将多条消息累积到一个批次（`RecordBatch`），满足以下条件之一时触发发送：
  - **批次大小达到阈值**：由 `batch.size` 控制（默认 16KB）。
  - **等待时间超时**：由 `linger.ms` 控制（默认 0，即立即发送，不等待）。
- **优势**：
  - **减少网络请求次数**：提升吞吐量。
  - **压缩优化**：批量压缩（如 Snappy、GZIP）效率更高。

#### (3) 同步发送下批处理的行为

- **关键结论**：**同步发送不关闭批处理机制**。
  - Producer 仍会尝试累积消息到批次中，直到满足 `batch.size` 或 `linger.ms` 条件，再将整个批次一次性发送到 Broker。
  - **同步发送仅影响线程阻塞逻辑**，不改变消息累积的底层机制。

### 消息偏移量的计算过程

通过以上流程可以看出，每条消息在被实际存储到磁盘时都会被分配一个绝对偏移量后才能被写入磁盘。在同一个分区内，消息的绝对偏移量都是从 0 开始，且单调递增；在不同分区内，消息的绝对偏移量是没有任何关系的。接下来讨论下消息的绝对偏移量的计算规则。

确定消息偏移量有两种方式，一种是顺序读取每一条消息来确定，此种方式代价比较大，实际上我们并不想知道消息的内容，而只是想知道消息的偏移量；第二种是读取每条消息的 Size 属性，然后计算出下一条消息的起始偏移量。比如第一条消息内容为 “abc”，写入磁盘后的偏移量为：8（OffSet）+ 4（Message 大小）+ 3（Message 内容的长度）= 15。第二条写入的消息内容为“defg”，其起始偏移量为 15，下一条消息的起始偏移量应该是：15+8+4+4=31，以此类推。

## 消费消息及副本同步流程分析

和写入消息流程不同，读取消息流程分为两种情况，分别是消费端消费消息和从副本（备份副本）同步主副本的消息。在开始分析读取流程之前，需要先明白几个用到的变量，不然流程分析可能会看的比较糊涂。

*   **BaseOffSet**：基准偏移量，每个 Partition 由 N 个 LogSegment 组成，每个 LogSegment 都有基准偏移量，大概由如下构成，数组中每个数代表一个 LogSegment 的基准偏移量：[0,200,400,600, ...]。
*   **StartOffSet**：起始偏移量，由消费端发起读取消息请求时，指定从哪个位置开始消费消息。
*   **MaxLength**：拉取大小，由消费端发起读取消息请求时，指定本次最大拉取消息内容的数据大小。该参数可以通过[max.partition.fetch.bytes](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/draft/3020%23)来指定，默认大小为 1M。
*   **MaxOffSet**：最大偏移量，消费端拉取消息时，最高可拉取消息的位置，即俗称的“高水位”。该参数由服务端指定，其作用是为了防止生产端还未写入的消息就被消费端进行消费。此参数对于从副本同步主副本不会用到。
*   **MaxPosition**：LogSegment 的最大位置，确定了起始偏移量在某个 LogSegment 上开始，读取 MaxLength 后，不能超过 MaxPosition。MaxPosition 是一个实际的物理位置，而非偏移量。

假设消费端从 000000621 位置开始消费消息，关于几个变量的关系如下图所示。




![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-cd9c62a71cddccd7bc8a5d810d5af216_720w.webp)

消费端和从副本拉取流程如下：

1.  **客户端确定拉取的位置，即 StartOffSet 的值，找到主副本对应的 LogSegment。**
2.  **LogSegment 由索引文件和数据文件构成，由于索引文件是从小到大排列的，首先从索引文件确定一个小于等于 StartOffSet 最近的索引位置。**
3.  **根据索引位置找到对应的数据文件位置，由于数据文件也是从小到大排列的，从找到的数据文件位置顺序向后遍历，直到找到和 StartOffSet 相等的位置，即为消费或拉取消息的位置。**
4.  **从 StartOffSet 开始向后拉取 MaxLength 大小的数据，返回给消费端或者从副本进行消费或备份操作。**

假设拉取消息起始位置为 00000313，消息拉取流程图如下：

<figure data-size="normal">





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-9417ca60a0c5e9474ec49a77fff18b1b_720w.webp)

**Broker 处理 Fetch 请求**

Broker 收到请求后，需从日志文件中查找消息，流程如下：

**步骤 1：定位日志分段（Log Segment）**

- Kafka 的日志目录按分段存储（Segment），每个分段命名规则为：

  - `<base_offset>.log`（存储消息）
  - `<base_offset>.index`（存储位移索引）
  - `<base_offset>.timeindex`（存储时间戳索引）

  **新段的基准偏移量 = 上一个段的最后一个消息的偏移量 + 1**

  例如：

  ```
  topic-test-0/
      ├── 00000000000000000000.log
      ├── 00000000000000000000.index
      ├── 00000000000000001000.log
      ├── 00000000000000001000.index
  ```

- **查找目标分段**：

  - Broker 根据 `fetch_offset`（如 8）找到包含该 offset 的日志分段：
    - 如果 `fetch_offset` < 1000，则选择 `00000000000000000000.log`。
    - 如果 `fetch_offset` >= 1000，则选择 `00000000000000001000.log`。

**步骤 2：使用索引文件快速定位**

- **位移索引（.index 文件）**：

  - 索引文件存储的是 **offset → 物理位置** 的映射（稀疏索引，非每一条消息都记录）。

  - 例如：

    复制

    ```
    offset: 4 → physical_position: 1024
    offset: 8 → physical_position: 2048
    ```

  - Broker 对索引文件执行 **二分查找**，找到小于等于 `fetch_offset` 的最大索引条目：

    - 例如 `fetch_offset=8`，找到索引条目 `offset=8 → position=2048`。

- **定位到日志文件（.log）**：

  - 根据索引找到的 `physical_position`（如 2048），直接从 `.log` 文件的 2048 字节处开始读取消息。

**步骤 3：读取消息并返回**

- Broker 从 `.log` 文件的指定位置读取消息，并检查：
  - 消息的 `offset` 是否 >= `fetch_offset`。
  - 消息是否在 **高水位（HW）** 以内（避免读取未提交的消息）。
- 返回符合条件的消息集合（`FetchResponse`）。

## kafka 如何保证系统的高可用、数据的可靠性和数据的一致性的？

### kafka 的高可用性：

1.  **Kafka 本身是一个分布式系统，同时采用了 Zookeeper 存储元数据信息，提高了系统的高可用性。**
2.  **Kafka 使用多副本机制，当状态为 Leader 的 Partition 对应的 Broker 宕机或者网络异常时，Kafka 会通过选举机制从对应的 Replica 列表中重新选举出一个 Replica 当做 Leader，从而继续对外提供读写服务（当然，需要注意的一点是，在新版本的 Kafka 中，Replica 也可以对外提供读请求了），利用多副本机制在一定程度上提高了系统的容错性，从而提升了系统的高可用。**

### Kafka 的可靠性：

1.  **从 Producer 端来看，可靠性是指生产的消息能够正常的被存储到 Partition 上且消息不会丢失。Kafka 通过 [request.required.acks](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/edit/49a133ad2b2f2671aa60706b0%23)和[min.insync.replicas](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/edit/49a133ad2b2f2671aa60706b0%23) 两个参数配合，在一定程度上保证消息不会丢失。**
2.  **[request.required.acks](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/edit/49a133ad2b2f2671aa60706b0%23) 可设置为 1、0、-1 三种情况。**




![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-7946f258c85fb8ca3d4aa423269c483a_720w.webp)

设置为 1 时代表当 Leader 状态的 Partition 接收到消息并持久化时就认为消息发送成功，如果 ISR 列表的 Replica 还没来得及同步消息，Leader 状态的 Partition 对应的 Broker 宕机，则消息有可能丢失。

<figure data-size="normal">





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-382c9f37f644feb37dd975c67bc1038f_720w.webp)

设置为 0 时代表 Producer 发送消息后就认为成功，消息有可能丢失。




![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-592996f264baadc64967d6f4b28f4d23_720w.webp)



设置为-1 时，代表 ISR 列表中的所有 Replica 将消息同步完成后才认为消息发送成功；但是如果只存在主 Partition 的时候，Broker 异常时同样会导致消息丢失。所以此时就需要[min.insync.replicas](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/edit/49a133ad2b2f2671aa60706b0%23)参数的配合，该参数需要设定值大于等于 2，当 Partition 的个数小于设定的值时，Producer 发送消息会直接报错。

上面这个过程看似已经很完美了，但是假设如果消息在同步到部分从 Partition 上时，主 Partition 宕机，此时消息会重传，虽然消息不会丢失，但是会造成同一条消息会存储多次。在新版本中 Kafka 提出了幂等性的概念，通过给每条消息设置一个唯一 ID，并且该 ID 可以唯一映射到 Partition 的一个固定位置，从而避免消息重复存储的问题（作者到目前还没有使用过该特性，感兴趣的朋友可以自行在深入研究一下）。

### Kafka 的一致性：

1.  **从 Consumer 端来看，同一条消息在多个 Partition 上读取到的消息是一直的，Kafka 通过引入 HW（High Water）来实现这一特性。**


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-9975539d98bf1a4e1a3038f2eceb2bb9_720w.webp)

从上图可以看出，假设 Consumer 从主 Partition1 上消费消息，由于 Kafka 规定只允许消费 HW 之前的消息，所以最多消费到 Message2。假设当 Partition1 异常后，Partition2 被选举为 Leader，此时依旧可以从 Partition2 上读取到 Message2。其实 HW 的意思利用了木桶效应，始终保持最短板的那个位置。

从上面我们也可以看出，使用 HW 特性后会使得消息只有被所有副本同步后才能被消费，所以在一定程度上降低了消费端的性能，可以通过设置[replica.lag.time.max.ms](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/edit/49a133ad2b2f2671aa60706b0%23)参数来保证消息同步的最大时间。

## kafka 为什么那么快？

kafka 使用了顺序写入和“零拷贝”技术，来达到每秒钟 200w（Apache 官方给出的数据） 的磁盘数据写入量，另外 Kafka 通过压缩数据，降低 I/O 的负担。

1.  **顺序写入**

大家都知道，对于磁盘而已，如果是随机写入数据的话，每次数据在写入时要先进行寻址操作，该操作是通过移动磁头完成的，极其耗费时间，而顺序读写就能够避免该操作。

1.  **“零拷贝”技术**


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-6930901956f341f1ab4a6e5650a0680b_720w.webp)

普通的数据拷贝流程如上图所示，数据由磁盘 copy 到内核态，然后在拷贝到用户态，然后再由用户态拷贝到 socket，然后由 socket 协议引擎，最后由协议引擎将数据发送到网络中。


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-9e44873a63d8addca917e658667f0b61_720w.webp)

采用了“零拷贝”技术后可以看出，数据不在经过用户态传输，而是直接在内核态完成操作，减少了两次 copy 操作。从而大大提高了数据传输速度。

1.  **压缩**

Kafka 官方提供了多种压缩协议，包括 gzip、snappy、lz4 等等，从而降低了数据传输的成本。

## Kafka 中的消息是否会丢失和重复消费？

1.  **Kafka 是否会丢消息，答案相信仔细看过前面两个问题的同学都比较清楚了，这里就不在赘述了。**
2.  **在低版本中，比如作者公司在使用的 Kafka0.8 版本中，还没有幂等性的特性的时候，消息有可能会重复被存储到 Kafka 上（原因见上一个问题的），在这种情况下消息肯定是会被重复消费的。**

**这里给大家一个解决重复消费的思路，作者公司使用了 Redis 记录了被消费的 key，并设置了过期时间，在 key 还没有过期内，对于同一个 key 的消息全部当做重复消息直接抛弃掉。** 在网上看到过另外一种解决方案，使用 HDFS 存储被消费过的消息，是否具有可行性存疑（需要读者朋友自行探索），读者朋友们可以根据自己的实际情况选择相应的策略，如果朋友们还有其他比较好的方案，欢迎留言交流。

## 为什么要使用 kafka，为什么要使用消息队列？

### 先来说说为什么要使用消息队列？

这道题比较主观一些（自认为没有网上其他文章写得话，轻喷），但是都相信大家使用消息队列无非就是为了 **解耦**、**异步**、**消峰**。


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-f7c1bb87ab46ddd03255c58109ce360f_720w.webp)

随着业务的发展，相信有不少朋友公司遇到过如上图所示的情况，系统 A 处理的结构被 B、C、D 系统所依赖，当新增系统 E 时，也需要系统 A 配合进行联调和上线等操作；还有当系统 A 发生变更时同样需要告知 B、C、D、E 系统需要同步升级改造。


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-0f0c8f9531a38f6d79b2cbb2973bfbfc_720w.webp)

引入消息队列后有两个好处：

1.  **各个系统进行了解耦，从上图也可以看出，当系统 A 突然发生热点事件时，同一时间产生大量结果，MQ 充当了消息暂存的效果，防止 B、C、D、E 系统也跟着崩溃。**
2.  **当新系统 E 需要接入系统 A 的数据，只需要和 MQ 对接就可以了，从而避免了与系统 A 的调试上线等操作。**

引入消息队列的坏处：

万事皆具备两面性，看似引入消息队列这件事情很美好，但是同时也增加了系统的复杂度、系统的维护成本提高（如果 MQ 挂了怎么办）、引入了一致性等等问题需要去解决。

## 为什么要使用 Kafka?

作者认为采用 Kafka 的原因有如下几点：

1.  **Kafka 目前在业界被广泛使用，社区活跃度高，版本更新迭代速度也快。**
2.  **Kafka 的生产者和消费者都用 Java 语言进行了重写，在一定程度降低了系统的维护成本（作者的主观意见，因为当下 Java 的使用群体相当庞大）。**
3.  **Kafka 系统的吞吐量高，达到了每秒 10w 级别的处理速度。**
4.  **Kafka 可以和很多当下优秀的大数据组件进行集成，包括 Spark、Flink、Flume、Storm 等等。**

## 为什么 Kafka 不支持读写分离？

Kafka 不支持全面的读写分离（即主写从读）主要源于其设计理念、同步机制及适用场景的特殊性，尽管从 2.4 版本开始支持有限度的读写分离（如允许 Follower 处理只读请求），但在多数场景下仍以主写主读（Leader 处理读写）为主。以下是具体原因分析：

------

### 1. 场景不适用

Kafka 的核心场景是**高吞吐的实时数据流处理**和**日志收集**，其读写负载通常对等且频繁。

- **读写分离的适用条件**：适用于读负载远高于写负载的场景（如传统数据库），但 Kafka 的读写操作频率相近，读写分离无法显著提升性能，反而可能因同步延迟导致数据不一致。
- **负载均衡效果**：Kafka 通过分区机制将 Leader 副本均匀分布在 Broker 上，天然实现读写负载均衡。若强制读写分离，可能因 Follower 节点同时承担其他分区的 Leader 职责，导致资源竞争，整体性能反而下降。

------

### 2. 同步机制与数据一致性

Kafka 的副本同步采用 **PULL 模式**（Follower 主动从 Leader 拉取数据），存在以下问题：

- **复制延迟**：数据从 Leader 写入到同步至 Follower 需经历网络传输、磁盘持久化等步骤，导致 Follower 数据滞后于 Leader。若允许读 Follower，客户端可能读到过时数据，破坏一致性。Leader取ISR Follower fetch请求里最小的LEO作为HW，更新本地HW并将其response给Follower，所以Follower HW的更新会滞后于Leader
- **不一致窗口**：在异步同步过程中，不同 Follower 的同步进度可能不一致，导致多次读取同一分区的不同副本时出现数据跳跃（如第一次读到最新值，第二次读到旧值），无法保证单调读一致性。

------

### 3. 主写主读的优势

Kafka 主写主读模型具有以下优势：

- **简化架构**：无需处理主从同步延迟带来的复杂性，代码实现更简洁，减少潜在错误。
- **高效负载均衡**：通过分区 Leader 的均匀分布，每个 Broker 的读写压力天然均衡。例如，一个 Broker 可能同时是多个分区的 Leader，分散了整体负载。
- **强一致性保障**：所有读写操作均通过 Leader，确保客户端始终访问最新数据，避免因副本同步延迟导致的一致性问题。

------

### 4. 设计初衷与权衡

Kafka 的设计目标是**高吞吐、低延迟的分布式日志系统**，而非传统数据库。

- **日志系统特性**：日志数据需严格保序且快速写入，主写主读模式通过顺序追加写入和批量读取优化性能，更适合 Kafka 的核心场景。
- **权衡收益与成本**：实现全面读写分离需额外处理同步延迟、数据一致性等问题，而 Kafka 通过分区和副本机制已满足多数场景需求，引入读写分离的收益有限。

------

### 5. 有限度的读写分离适用场景

自 Kafka 2.4 起，允许 Follower 处理**只读请求**（如跨机房消费），但其适用性受限：

- **低一致性需求**：仅适用于容忍数据延迟的场景（如离线分析），不适用于实时处理。
- **性能瓶颈**：Follower 的读取效率可能低于 Leader（尤其在同步延迟较高时），无法显著提升吞吐量。

## 总结

本文介绍了几个常见的 Kafka 的面试题

### 常见面试题一览

#### 1.1 Kafka 中的 ISR(InSyncRepli)、 OSR(OutSyncRepli)、 AR(AllRepli)代表什么？

ISR：速率和leader相差低于10s的follower的集合

OSR：速率和leader相差大于10s的follwer

AR：所有分区的副本

#### 1.2 Kafka 中的 HW、 LEO 等分别代表什么？

HW：High Water高水位，根据同一分区中最低的LEO决定（Log End Offset）

LEO：每个分区最大的Offset

#### 1.3 Kafka 中是怎么体现消息顺序性的？

在每个分区内，每条消息都有offset，所以消息在同一分区内有序，无法做到全局有序性

#### 1.4 Kafka 中的分区器、序列化器、拦截器是否了解？它们之间的处理顺序是什么？

分区器Partitioner用来对分区进行处理的，即消息发送到哪一个分区的问题。序列化器，这个是对数据进行序列化和反序列化的工具。拦截器，即对于消息发送进行一个提前处理和收尾处理的类Interceptor，处理顺利首先通过拦截器=>序列化器=>分区器

#### 1.5 Kafka 生产者客户端的整体结构是什么样子的？使用了几个线程来处理？分别是什么？

使用两个线程：main和sender 线程，main线程会一次经过拦截器、序列化器、分区器将数据发送到RecoreAccumulator线程共享变量，再由sender线程从共享变量中拉取数据发送到kafka broker

batch.size达到此规模消息才发送，linger.ms未达到规模，等待当前时长就发送数据。

#### 1.6 消费组中的消费者个数如果超过 topic 的分区，那么就会有消费者消费不到数据”这句 话是否正确？

这句话是对的，超过分区个数的消费者不会在接收数据，主要原因是一个分区的消息只能够被一个消费者组中的一个消费者消费。

#### 1.7 消费者提交消费位移时提交的是当前消费到的最新消息的 offset 还是 offset+1？

在 Kafka 中，消费者提交的消费位移（offset）通常是当前已成功消费的最新消息的 offset + 1，即下一次应该开始消费的消息的 offset

#### 1.8 有哪些情形会造成重复消费？

先消费后提交offset，如果消费完宕机了，则会造成重复消费

#### 1.9 那些情景会造成消息漏消费？

先提交offset，还没消费就宕机了，则会造成漏消费

#### 1.10 当你使用 kafka-topics.sh 创建（删除）了一个 topic 之后， Kafka 背后会执行什么逻辑？

会在 zookeeper 中的/brokers/topics 节点下创建一个新的 topic 节点，如：/brokers/topics/first 触发 Controller 的监听程序 kafka Controller 负责 topic 的创建工作，并更新 metadata cache

#### 1.11 topic 的分区数可不可以增加？如果可以怎么增加？如果不可以，那又是为什么？

可以增加，修改分区个数--alter可以修改分区个数

#### 1.12 topic 的分区数可不可以减少？如果可以怎么减少？如果不可以，那又是为什么？

不可以减少，减少了分区之后，之前的分区中的数据不好处理

#### 1.13 Kafka 有内部的 topic 吗？如果有是什么？有什么所用？

有，__consumer_offsets主要用来在0.9版本以后保存消费者消费的offset

#### 1.14 Kafka 分区分配的概念？

Kafka分区对于Kafka集群来说，分区可以做到负载均衡，对于消费者来说分区可以提高并发度，提高读取效率

#### 1.15 简述 Kafka 的日志目录结构？

每一个分区对应着一个文件夹，命名为topic-0/topic-1…，每个文件夹内有.index和.log文件。
Leader 副本和 Follower 副本的日志目录结构和存储内容是完全相同的，它们都遵循相同的日志分段（Log Segment）存储格式

#### 1.16 如果我指定了一个 offset， Kafka Controller 怎么查找到对应的消息？

offset表示当前消息的编号，首先可以通过二分法定位当前消息属于哪个.index文件中，随后采用seek定位的方法查找到当前offset在.index中的位置，此时可以拿到初始的偏移量。通过初始的偏移量再通过seek定位到.log中的消息即可找到。

#### 1.17 聊一聊 Kafka Controller 的作用？

Kafka集群中有一个broker会被选举为Controller，负责管理集群broker的上下线、所有topic的分区副本分配和leader的选举等工作。Controller的工作管理是依赖于zookeeper的。

#### 1.18 Kafka 中有那些地方需要选举？这些地方的选举策略又有哪些？

在ISR中需要选举出Leader，选择策略为先到先得。在分区中需要选举，需要选举出Leader和follower。

#### 1.19 失效副本是指什么？有那些应对措施？

失效副本为速率比leader相差大于10s的follower，ISR会将这些失效的follower踢出，等速率接近leader的10s内，会重新加入ISR

#### 1.20 Kafka 的哪些设计让它有如此高的性能？

1.  Kafka天生的分布式架构
2.  对log文件进行了分segment，并对segment建立了索引
3.  对于单节点使用了顺序读写，顺序读写是指的文件的顺序追加，减少了磁盘寻址的开销，相比随机写速度提升很多
4.  使用了零拷贝技术，不需要切换到用户态，在内核态即可完成读写操作，且数据的拷贝次数也更少。

