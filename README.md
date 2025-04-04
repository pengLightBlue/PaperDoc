# Paper Doc

> Personal doc project.

<!-- _sidebar.md -->

* Golang
  * [Java跟Go对比](ProjectDocs/golang/Java跟Go对比.md) 
  * [Go包介绍与初始化：搞清Go程序的执行次序](ProjectDocs/golang/Go包介绍与初始化：搞清Go程序的执行次序.md)  
  * [Go数组与切片](ProjectDocs/golang/Go数组与切片.md)  
  * [深入解析Go中Slice底层实现](ProjectDocs/golang/深入解析Go中Slice底层实现.md)  
  * [Map的实现原理](ProjectDocs/golang/map的实现原理.md)  
  * [Defer关键字](ProjectDocs/golang/defer关键字.md)  
  * [深入了解Go的interface{}底层原理](ProjectDocs/golang/深入了解Go的interface{}底层原理.md)  
  * [上下文context](ProjectDocs/golang/上下文context.md)  
  * [Go深入理解深拷贝和浅拷贝](ProjectDocs/golang/Go深入理解深拷贝和浅拷贝.md)  
  * [互斥锁的实现原理](ProjectDocs/golang/互斥锁的实现原理.md)  
  * [深入理解sync.RWMutex实现原理](ProjectDocs/golang/深入理解sync.RWMutex实现原理.md)  
  * [sync.WaitGroup原理深度剖析](ProjectDocs/golang/sync.WaitGroup原理深度剖析.md)  
  * [sync.Once原理](ProjectDocs/golang/sync.Once原理.md)  
  * [sync.Cond条件变量源码分析](ProjectDocs/golang/sync.Cond条件变量源码分析.md)  
  * [深度分析sync.Pool底层原理](ProjectDocs/golang/深度分析sync.Pool底层原理.md)  
  * [sync.Map实现原理](ProjectDocs/golang/sync.Map实现原理.md)  
  * [golang中channel的详细使用、使用注意事项及死锁分析](/ProjectDocs/golang/golang中channel的详细使用、使用注意事项及死锁分析.md)  
  * [channel设计原理](/ProjectDocs/golang/channel设计原理.md)  
  * [select使用与底层原理](ProjectDocs/golang/select使用与底层原理.md)    
  * [Golang调度器GMP原理与调度全分析](ProjectDocs/golang/Golang调度器GMP原理与调度全分析.md)  
  * [Go内存分配](ProjectDocs/golang/Go内存分配.md)  
  * [Golang三色标记混合写屏障GC模式全分析](ProjectDocs/golang/Golang三色标记混合写屏障GC模式全分析.md)  
  * [Go逃逸分析](ProjectDocs/golang/go逃逸分析.md)  
* Mysql
  * [执行一条select语句，期间发生了什么](/ProjectDocs/mysql/执行一条select语句，期间发生了什么.md)
  * [MySQL一行记录是怎么存储的](/ProjectDocs/mysql/MySQL一行记录是怎么存储的.md)
  * [索引常见面试题](/ProjectDocs/mysql/索引常见面试题.md)
  * [从数据页的角度看B+树](/ProjectDocs/mysql/从数据页的角度看B+树.md)
  * [为什么MySQL采用B+树作为索引](/ProjectDocs/mysql/为什么MySQL采用B+树作为索引.md)
  * [MySQL单表不能超过2000W行吗](/ProjectDocs/mysql/MySQL单表不能超过2000W行吗.md)
  * [索引失效有哪些](ProjectDocs/mysql/索引失效有哪些.md)
  * [MySQL使用like "%x"，索引一定会失效吗？](ProjectDocs/mysql/mysql使用like，索引一定会失效吗.md)
  * [count(*)和count(1)有什么区别？哪个性能最好？](ProjectDocs/mysql/count(*)和count(1)有什么区别？哪个性能最好.md)
  * [事务隔离级别是怎么实现的](ProjectDocs/mysql/事务隔离级别是怎么实现的.md)
  * [MySQL可重复读隔离级别，完全解决幻读了吗](ProjectDocs/mysql/MySQL可重复读隔离级别，完全解决幻读了吗.md)
  * [MySQL有哪些锁](ProjectDocs/mysql/MySQL有哪些锁.md)
  * [MySQL是怎么加锁的](ProjectDocs/mysql/mysql是怎么加锁的.md)
  * [update没加索引会锁全表](ProjectDocs/mysql/update没加索引会锁全表.md)
  * [MySQL记录锁+间隙锁可以防止删除操作而导致的幻读吗？](ProjectDocs/mysql/MySQL记录锁+间隙锁可以防止删除操作而导致的幻读吗？.md)
  * [MySQL死锁了，怎么办？](ProjectDocs/mysql/MySQL死锁了，怎么办.md)
  * [加了什么锁，导致的死锁？](ProjectDocs/mysql/加了什么锁，导致的死锁.md)
  * [MySQL日志：undo log、redo log、binlog有什么用](ProjectDocs/mysql/MySQL日志：undo-log、redo-log、binlog有什么用.md)
  * [MySQL分库分表策略](ProjectDocs/mysql/MySQL分库分表策略.md)
  * [揭开Buffer Pool的面纱](ProjectDocs/mysql/MySQL缓冲池.md)
  * [Mysql常用存储引擎对比总结](ProjectDocs/mysql/Mysql常用存储引擎对比总结.md)
* Redis
  * [Redis常见面试题](/ProjectDocs/redis/redis常见面试题.md)
  * 数据结构
    * [Redis常见数据类型和应用场景](/ProjectDocs/redis/Redis常见数据类型和应用场景.md)
    * [Redis数据结构](/ProjectDocs/redis/Redis数据结构.md)
    * [从ziplist到quicklist，再到listpack的启发](/ProjectDocs/redis/从ziplist到quicklist，再到listpack的启发.md)
  * 事件框架   
    * [为什么单线程的Redis如何做到每秒数万QPS](/ProjectDocs/redis/为什么单线程的Redis如何做到每秒数万QPS.md)
    * [Redis事件驱动框架（上）：何时使用select、poll、epoll？](/ProjectDocs/redis/Redis事件驱动框架（上）：何时使用select、poll、epoll？.md)
    * [Redis事件驱动框架（中）：Redis实现了Reactor模型吗？](/ProjectDocs/redis/Redis事件驱动框架（中）：Redis实现了Reactor模型吗？.md)
    * [Redis事件驱动框架（下）：Redis有哪些事件？](/ProjectDocs/redis/Redis事件驱动框架（下）：Redis有哪些事件？.md)
  * 持久化
    * [AOF持久化是怎么实现的](/ProjectDocs/redis/AOF持久化是怎么实现的.md)
    * [RDB快照是怎么实现的](/ProjectDocs/redis/RDB快照是怎么实现的.md)
    * [Redis大Key对持久化有什么影响](/ProjectDocs/redis/Redis大Key对持久化有什么影响.md)
  * [Redis过期删除策略和内存淘汰策略有什么区别](/ProjectDocs/redis/Redis过期删除策略和内存淘汰策略有什么区别.md)        
  * 高可用
    * [主从复制是怎么实现的](/ProjectDocs/redis/主从复制是怎么实现的.md)
    * [Redis主从同步与故障切换，有哪些坑](/ProjectDocs/redis/Redis主从同步与故障切换，有哪些坑.md)
    * [为什么要有哨兵](/ProjectDocs/redis/为什么要有哨兵.md)
    * [切片集群：数据增多了，是该加内存还是加实例](/ProjectDocs/redis/切片集群：数据增多了，是该加内存还是加实例.md)
    * [Redis集群方案选择](/ProjectDocs/redis/Redis集群方案选择.md)
    * [通信开销：限制Redis Cluster规模的关键因素](/ProjectDocs/redis/通信开销：限制Redis-Cluster规模的关键因素.md)
  * 缓存    
    * [什么是缓存雪崩、击穿、穿透](/ProjectDocs/redis/什么是缓存雪崩、击穿、穿透.md)
    * [数据库和缓存如何保证一致性](/ProjectDocs/redis/数据库和缓存如何保证一致性.md)
  * [Redis分布式锁](/ProjectDocs/redis/Redis分布式锁.md)        
* Kafka
  * [Kafka是什么](ProjectDocs/Kafka/Kafka是什么.md)
  * [Kafka常见问题总结](ProjectDocs/Kafka/Kafka常见问题总结.md)
  * [kafka原理介绍](ProjectDocs/Kafka/kafka原理介绍.md)
  * [kafka架构介绍](ProjectDocs/Kafka/kafka架构介绍.md)
  * [kafka集群工作原理](ProjectDocs/Kafka/kafka集群工作原理.md)
  * [kafka重要知识点](ProjectDocs/Kafka/kafka重要知识点.md)
  * [kafka实现延迟队列、死信队列、重试队列](ProjectDocs/Kafka/kafka实现延迟队列、死信队列、重试队列.md)
  * [如何提升kafka消费者服务性能](ProjectDocs/Kafka/如何提升kafka消费者服务性能.md)
* Zookeeper
  * [Zookeeper知识点](ProjectDocs/zookeeper/zookeeper知识点.md)  
* ES
  * [elasticSearch是什么？工作原理是怎么样的？](ProjectDocs/es/elasticSearch是什么？工作原理是怎么样的.md)
  * [es知识点总结](ProjectDocs/es/es知识点总结.md)  
* Network
  * Auth
    * [cookie、session、token、jwt的区别](ProjectDocs/network/auth/cookie、session、token、jwt的区别.md) 
  * 基础篇
    * [TCP/IP 网络模型有哪几层？](ProjectDocs/network/TCP-IP网络模型有哪几层？.md) 
    * [键入网址到网页显示，期间发生了什么？](ProjectDocs/network/键入网址到网页显示，期间发生了什么？.md) 
    * [Linux系统是如何收发网络包的？](ProjectDocs/network/Linux系统是如何收发网络包的？.md) 
  * HTTP
    * [HTTP常见面试题](/ProjectDocs/network/HTTP常见面试题.md) 
    * [HTTP1.1如何优化](/ProjectDocs/network/HTTP1.1如何优化.md) 
    * [HTTPS RSA握手解析](/ProjectDocs/network/HTTPS-RSA握手解析.md) 
    * [HTTPS ECDHE握手解析](/ProjectDocs/network/HTTPS-ECDHE握手解析.md) 
    * [HTTP2牛逼在哪](/ProjectDocs/network/HTTP2牛逼在哪.md) 
    * [HTTP3强势来袭](/ProjectDocs/network/HTTP3强势来袭.md) 
    * [既然有HTTP协议，为什么还要有RPC](/ProjectDocs/network/既然有HTTP协议，为什么还要有RPC.md) 
    * [既然有HTTP协议，为什么还要有WebSocket](/ProjectDocs/network/既然有HTTP协议，为什么还要有WebSocket.md)  
  * TCP
    * [TCP三次握手与四次挥手](/ProjectDocs/network/TCP三次握手与四次挥手.md)     
    * [TCP重传、滑动窗口、流量控制、拥塞控制](/ProjectDocs/network/TCP重传、滑动窗口、流量控制、拥塞控制.md) 
  * 底层原理
    * [深入操作系统，一文搞懂Socket到底是什么](/ProjectDocs/network/深入操作系统，一文搞懂Socket到底是什么.md)
    * [什么是零拷贝？](/ProjectDocs/network/什么是零拷贝？.md)
    * [IO多路复用：select&poll&epoll](/ProjectDocs/network/IO多路复用：select&poll&epoll.md)
    * [高性能网络模式：Reactor和Proactor](/ProjectDocs/network/高性能网络模式：Reactor和Proactor.md)
    * [什么是一致性哈希？](/ProjectDocs/network/什么是一致性哈希？.md)    
* System
  * [操作系统常见面试题总结(上)](/ProjectDocs/system/操作系统常见面试题总结(上).md)
  * [操作系统常见面试题总结(下)](/ProjectDocs/system/操作系统常见面试题总结(下).md)
  * [进程间通信IPC](ProjectDocs/system/进程间通信IPC.md)
  * [操作系统知识点](ProjectDocs/system/操作系统知识点.md)
* Linux
  * [Linux基础知识总结](ProjectDocs/linux/Linux基础知识总结.md)
  * [Linux常见命令](ProjectDocs/linux/Linux常见命令.md)      
* Chatbot
  * [美团智能客服核心技术与实践](/ProjectDocs/chatbot/美团智能客服核心技术与实践.md)
* Saas
  * [多租户架构设计](/ProjectDocs/saas/多租户架构设计.md)
  * [智能会话机器人SaaS平台的设计与思考](/ProjectDocs/saas/智能会话机器人SaaS平台的设计与思考.md)
* Algorithm
  * [数据结构与算法](ProjectDocs/algorithm/数据结构与算法.md)  
  * [算法就像搭乐高：手撸LRU算法](ProjectDocs/algorithm/算法就像搭乐高：手撸LRU算法.md)  
  * [字符串匹配的KMP算法](/ProjectDocs/algorithm/字符串匹配的KMP算法.md)  
  * [Trie树:如何实现搜索引擎的搜索关键词提示功能？](/ProjectDocs/algorithm/Trie树：如何实现搜索引擎的搜索关键词提示功能？.md)  
* Prometheus
  * [彻底理解Prometheus查询语法](/ProjectDocs/prometheus/彻底理解Prometheus查询语法.md)
* Ai
  * [一文搞懂rag](ProjectDocs/ai/一文搞懂rag.md)  
  * [Agent的原理介绍与应用发展思考](ProjectDocs/ai/Agent的原理介绍与应用发展思考.md)
  * [Transformer模型详解](ProjectDocs/ai/Transformer模型详解.md)
* Design pattern
  * [常用设计模式](ProjectDocs/designpattern/常用设计模式.md)  
* Distributed system
  * [分布式知识点](ProjectDocs/distributed-system/分布式知识点.md)  
* Domain Driven Design
  * [DDD领域驱动设计](ProjectDocs/ddd/DDD领域驱动设计.md)  
* System design
  * [系统设计](ProjectDocs/system-design/系统设计.md)  
  * [后台服务器高性能设计思想](ProjectDocs/system-design/后台服务器高性能设计思想.md)
  * [秒杀系统如何设计](ProjectDocs/system-design/秒杀系统如何设计.md)
  * [千万级、亿级数据，如何性能优化](ProjectDocs/system-design/千万级、亿级数据，如何性能优化.md)
  * [评论系统架构设计](ProjectDocs/system-design/评论系统架构设计.md)