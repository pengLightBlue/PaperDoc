# Paper Doc

> Personal doc project.

<!-- _sidebar.md -->

* Mysql
  * [执行一条select语句，期间发生了什么](/ProjectDocs/mysql/执行一条select语句，期间发生了什么.md)
  * [MySQL一行记录是怎么存储的](/ProjectDocs/mysql/MySQL一行记录是怎么存储的.md)
  * [索引常见面试题](/ProjectDocs/mysql/索引常见面试题.md)
  * [从数据页的角度看B+树](/ProjectDocs/mysql/从数据页的角度看B+树.md)
  * [为什么MySQL采用B+树作为索引](/ProjectDocs/mysql/为什么MySQL采用B+树作为索引.md)
  * [MySQL单表不能超过2000W行吗](/ProjectDocs/mysql/MySQL单表不能超过2000W行吗.md)
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
* Network
  * Auth
    * [cookie、session、token、jwt的区别](ProjectDocs/network/auth/cookie、session、token、jwt的区别.md) 
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
* Golang
  * [golang中channel的详细使用、使用注意事项及死锁分析](/ProjectDocs/golang/golang中channel的详细使用、使用注意事项及死锁分析.md)  
  * [channel设计原理](/ProjectDocs/golang/channel设计原理.md)  
  * [Golang调度器GMP原理与调度全分析](ProjectDocs/golang/Golang调度器GMP原理与调度全分析.md)  
  * [Golang三色标记混合写屏障GC模式全分析](ProjectDocs/golang/Golang三色标记混合写屏障GC模式全分析.md)  
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
* System
  * [操作系统常见面试题总结(上)](/ProjectDocs/system/操作系统常见面试题总结(上).md)
  * [操作系统常见面试题总结(下)](/ProjectDocs/system/操作系统常见面试题总结(下).md)
* Ai
  * [一文搞懂rag](ProjectDocs/ai/一文搞懂rag.md)  
  * [Agent的原理介绍与应用发展思考](ProjectDocs/ai/Agent的原理介绍与应用发展思考.md)