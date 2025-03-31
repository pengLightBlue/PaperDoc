

# Mysql常用存储引擎对比总结

存储引擎是数据库的核心，对于mysql来说，存储引擎是以插件的形式运行的。虽然mysql支持种类繁多的存储引擎，但是常用的就那么几种。这篇文章主要是对其进行一个总结和对比。

## 一、引言

在mysql5之后，支持的存储引擎有十几个，但是常用的就那么几种，而且默认支持的也是[InnoDB](https://zhida.zhihu.com/search?content_id=110830337&content_type=Article&match_order=1&q=InnoDB&zhida_source=entity)，既然要进行一个对比，我们就要从不同的维度来看一下。

我们可以使用命令来看看当前数据库可以支持的存储引擎有哪些。

  

![](https://picx.zhimg.com/v2-389b9ab03103f272058860fcafaff03b_1440w.jpg)

  

在这里我们发现默认支持了9种。还是比较多的，下面我们进行一个对比。

不同的存储引擎都有各自的特点，以适应不同的需求，如表所示。为了做出选择，首先要考虑每一个存储引擎提供了哪些不同的功能。

  

![](https://pic2.zhimg.com/v2-dc3fe4ad61cb8a1f812bc1621b3e5fe7_1440w.jpg)

  

在这里我们列举了一些特点并作出了比较。下面我们来具体分析对比一下。

## 二、存储引擎

### 1、MyISAM

使用这个存储引擎，每个MyISAM在磁盘上存储成三个文件。

（1）frm文件：存储表的定义数据

（2）MYD文件：存放表具体记录的数据

（3）MYI文件：存储索引

frm和MYI可以存放在不同的目录下。MYI文件用来存储索引，但仅保存记录所在页的指针，索引的结构是[B+树](https://zhida.zhihu.com/search?content_id=110830337&content_type=Article&match_order=1&q=B%2B%E6%A0%91&zhida_source=entity)结构。下面这张图就是MYI文件保存的机制：

  

![](https://pica.zhimg.com/v2-da3450b2f34b2d8f77af3df998b5ea2c_1440w.jpg)

  

从这张图可以发现，这个存储引擎通过MYI的B+树结构来查找记录页，再根据记录页查找记录。并且支持全文索引、B树索引和数据压缩。

支持数据的类型也有三种：

（1）静态固定长度表

这种方式的优点在于存储速度非常快，容易发生缓存，而且表发生损坏后也容易修复。缺点是占空间。这也是默认的存储格式。

（2）动态可变长表

优点是节省空间，但是一旦出错恢复起来比较麻烦。

（3）压缩表

上面说到支持数据压缩，说明肯定也支持这个格式。在数据文件发生错误时候，可以使用check table工具来检查，而且还可以使用repair table工具来恢复。

有一个重要的特点那就是不支持事务，但是这也意味着他的存储速度更快，如果你的读写操作允许有错误数据的话，只是追求速度，可以选择这个存储引擎。

### 2、InnoDB

InnoDB是默认的数据库存储引擎，他的主要特点有：

（1）可以通过自动增长列，方法是auto\_increment。

（2）支持事务。默认的事务隔离级别为可重复度，通过[MVCC](https://zhida.zhihu.com/search?content_id=110830337&content_type=Article&match_order=1&q=MVCC&zhida_source=entity)（并发版本控制）来实现的。

（3）使用的锁粒度为[行级锁](https://zhida.zhihu.com/search?content_id=110830337&content_type=Article&match_order=1&q=%E8%A1%8C%E7%BA%A7%E9%94%81&zhida_source=entity)，可以支持更高的并发；

（4）支持[外键约束](https://zhida.zhihu.com/search?content_id=110830337&content_type=Article&match_order=1&q=%E5%A4%96%E9%94%AE%E7%BA%A6%E6%9D%9F&zhida_source=entity)；外键约束其实降低了表的查询速度，但是增加了表之间的耦合度。

（5）配合一些热备工具可以支持在线热备份；

（6）在InnoDB中存在着缓冲管理，通过[缓冲池](https://zhida.zhihu.com/search?content_id=110830337&content_type=Article&match_order=1&q=%E7%BC%93%E5%86%B2%E6%B1%A0&zhida_source=entity)，将索引和数据全部缓存起来，加快查询的速度；

（7）对于InnoDB类型的表，其数据的物理组织形式是[聚簇表](https://zhida.zhihu.com/search?content_id=110830337&content_type=Article&match_order=1&q=%E8%81%9A%E7%B0%87%E8%A1%A8&zhida_source=entity)。所有的数据按照主键来组织。数据和索引放在一块，都位于B+数的叶子节点上；

当然InnoDB的存储表和索引也有下面两种形式：

（1）使用共享表空间存储：所有的表和索引存放在同一个表空间中。

（2）使用多表空间存储：表结构放在frm文件，数据和索引放在IBD文件中。分区表的话，每个分区对应单独的IBD文件，分区表的定义可以查看我的其他文章。使用分区表的好处在于提升查询效率。

对于InnoDB来说，最大的特点在于支持事务。但是这是以损失效率来换取的。

### 3、Memory

将数据存在内存，为了提高数据的访问速度，每一个表实际上和一个磁盘文件关联。文件是frm。

（1）支持的数据类型有限制，比如：不支持TEXT和BLOB类型，对于字符串类型的数据，只支持固定长度的行，VARCHAR会被自动存储为CHAR类型；

（2）支持的锁粒度为表级锁。所以，在访问量比较大时，表级锁会成为MEMORY存储引擎的瓶颈；

（3）由于数据是存放在内存中，一旦服务器出现故障，数据都会丢失；

（4）查询的时候，如果有用到临时表，而且临时表中有BLOB，TEXT类型的字段，那么这个临时表就会转化为MyISAM类型的表，性能会急剧降低；

（5）默认使用hash索引。

（6）如果一个内部表很大，会转化为磁盘表。

在这里只是给出3个常见的存储引擎。使用哪一种引擎需要灵活选择，一个数据库中多个表可以使用不同引擎以满足各种性能和实际需求，使用合适的存储引擎，将会提高整个数据库的性能