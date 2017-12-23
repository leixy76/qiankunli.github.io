---

layout: post
title: 《mysql技术内幕》笔记1
category: 技术
tags: Architecture
keywords: mysql innodb

---

## 简介

《mysql技术内幕》 主要讲存储引擎部分。


## 内存分配

我们一般做的web系统，即或是用到了缓存数据，缓存也是分散在各个类中。那么内存集中管理的例子，或者说谈得上内存模型的例子：

1. jvm，分为不同用途的数据，堆栈等，不同区域使用不同的分配与回收逻辑
2. netty的arena，多种粒度划分现有内存，手动触发分配与回收
3. mysql的缓冲池，以页为单位，LRU（有改动）为分配回收算法

innodb引擎内存占用

1. 缓冲池
2. 重做日志缓冲
3. 额外内存池

## 批量读取数据

与操作系统操作磁盘块的逻辑基本一致，操作时以块/页为单位，而不是直接从磁盘上读取记录（曾经真的这么以为）。


||读取过程|写回磁盘||
|---|---|---|---|
|操作系统读取磁盘块|判断目标数据所在的磁盘块是否在内存（对应的缓冲块），若在则修改数据，不在则读取磁盘块到内存。|操作系统负责将缓冲块同步到磁盘上。修改的缓冲块称为脏块，操作系统会根据脏块的占比决定同步到磁盘期间，是否继续响应用户操作。|磁盘块有超级块和一般数据块的不同，不准确的说，超级块的数据加载到磁盘，刚好是一个superblock list、inode list|
|innodb引擎读取页|类似|mysql有专门线程负责将脏页（有改动的页）写入到磁盘。因为redo 日志的存在，msyql会根据redo log的剩余空间占比决定在master thread中同步还是异步 将脏页写入到磁盘。|数据页组织在一起刚好是一个B+Tree|


1. 有些文件格式（二进制文件），必须整体读取才能解析和展示，有些（主要是文本文件）则可以一部分一部分的解析和展示，比如txt。
2. 有些文件格式，在内存的数据结构与磁盘一致。有些则需要转换后写入到磁盘上。

读取记录，一次读取一页，还带来一个好处，一个页的数据结构是一个整体，支持复杂的数据结构。假设一个记录10byte，无需页offset 0~9就是第一条记录，offset10~19是第二条记录（以前我就是这么想的），即用链表而不是位置维持数据的有序性。

这其实解决了笔者一直以来的一个困惑。假设一个数据库表的主键按1~1亿分布，我先插入一条主键=1的记录，再插入主键=1000的记录，再插入主键从2~100的记录。无论采用何种索引方式，每次插入，都意味着数据文件或者索引文件中数据记录的移动，操作起磁盘来就不太高效。


1. 书中写到：**聚集索引的存储并不是物理上连续的，而是逻辑上连续的。**主要有两点：页按照主键顺序排序，通过双向链表链接；页中的每个记录也按照双向链表进行维护。
2. **设若按页的方式进行内存和磁盘的交互，并且几个页组织在一起刚好是一个完整的数据结构（B+Tree）**，在内存中改变B+Tree的操作负担不大，然后有一个周期性的机制将页刷新回磁盘即可。

![](/public/upload/architecture/inner_mysql_3.png)

假设一个表，一个表空间文件，表名test，对应文件test.ibd，ibd就是一个文件格式，有专门的工具解析，跟rm、mp3性质上一样一样的。


## 日志点分析

[MySQL checkpoint深入分析](http://www.cnblogs.com/geaozhang/p/7341333.html)

[MySQL · 引擎特性 · InnoDB redo log漫游](http://mysql.taobao.org/monthly/2015/05/01/)



![](/public/upload/architecture/inner_mysql_1.png)

为了防止数据丢失，采用WAL，事务（具体应该是数据增删改操作）提交时，先写重做日志，再修改页。LSN(log sequence number) 用于记录日志序号，它是一个不断递增的 unsigned long类型整数。**因为写redo log是第一个要做的事儿，因此可以用lsn来做一些标记。**在 InnoDB 的日志系统中，LSN 无处不在，它既用于表示修改脏页时的日志序号，也用于记录checkpoint，通过LSN，可以具体的定位到其在redo log文件中的位置。

为了管理脏页，在 Buffer Pool 的每个instance上都维持了一个flush list，flush list 上的 page 按照修改这些 page 的LSN号进行排序。猜测：脏页刷新到磁盘时，应该也是按lsn顺序来的，不会存在较大lsn已经刷盘，而较小lsn未刷盘的情况。


|编号|lsn的某个状态值|说明|本阶段的lsn redo log所在位置|本阶段的lsn对应页的内存和硬盘一致性状态|备注|
|---|---|---|---|---|---|
|1|Log sequence number|最新日志号|||
|2|Log flushed up to |日志刷盘量|2~1:内存|2~1:不一致||
|3|Pages flushed up to|脏页刷盘量|3~2:硬盘|3~2:不一致|没找到地方显式存在|
|4|Last checkpoint at |上一次检查点的位置|4~3:硬盘|4~3:一致，此时5~3对应的redo日志已失效，可以被覆盖||
|5|0|起始lsn|5~4:硬盘|5~4:一致||

我们来回顾一下：

1. 为了保证宕机时数据不丢失，采用WAL，为了减少恢复的时间，使用了checkpoint，为了加快日志的写入速度使用了redo log buffer。磁盘上的redo log容量有限，在两个checkpoint之间，发现redo log快不够时，则刷新一定量的脏页，其对应范围的lsn redo log可以被覆盖（释放）。

2. 为了加快增删改查数据的速度，使用了缓冲池。缓冲池的容量有限，所以使用了lru。lru决定将某页从缓冲池中移除，该页恰好是脏页时，需要将数据同步到内存，连带更新Pages flushed up to。

各个环节环环相扣，像艺术品。

## 索引

[MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html) 基本要点：

1. 每种查找算法都只能应用于特定的数据结构之上，例如二分查找要求被检索数据有序，而二叉树查找只能应用于二叉查找树上，但是数据本身的组织结构不可能完全满足各种数据结构（例如，理论上不可能同时将两列都按顺序进行组织），所以，在数据之外，数据库系统还维护着满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据，这样就可以在这些数据结构上实现高级查找算法。这种数据结构，就是索引。**就好像一本汉字字典，汉字整体上按照拼音组织的，但也新增了部首索引。** 提高查找效率，就要往树、B+树那个方向想。

2. 为什么使用B+Tree作为索引数据结构？一般来说，索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。这样的话，索引查找过程中就要产生磁盘I/O消耗，所以评价一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘I/O操作次数的渐进复杂度。在mysql中，**将一个B+Tree节点的大小设为等于一个内存页（磁盘块），这样每个节点只需要一次I/O就可以完全载入。（类比一下操作系统inode块）**B-Tree中一次检索最多需要h（树的高度）-1次I/O（根节点常驻内存）。

	![](/public/upload/architecture/inner_mysql_2.png)

3. 对比MyISAM和InnoDB两种存储引擎的索引实现，了解在数据文件和索引文件的关系、辅助索引（一个表可能根据多个字段查询，除了按主键索引外，一般有多个辅助索引）的构建方式上有多重选择。

InnoDB存储B+Tree节点的方式确实非常精巧，MyISAM主要是记录了主键与对应记录地址（偏移）的映射关系。InnoDB引擎在页范围内查找一个记录，也“记录了主键与对应记录地址（偏移）的映射关系”，但还有些不一样，参见innodb page directory。[InnoDB备忘录 - 数据页结构](http://zhongmingmao.me/2017/05/09/innodb-table-page-structure/)

## 磁盘文件

内存管理系统将内存条编址，对每个进程看到的都是0~n。文件系统相当于将离散的磁盘存储空间编址，对每个文件看到的都是0~n。当然，进程根据进程号查找就可以，文件要根据文件名查找，因此多了一些结构。

我们常说逻辑结构、物理/存储结构

1. 存储结构是扁平的，一个文件/对象/实体的数据或连续或分散，能从offset 0到结尾找到就行。存在磁盘上通常有一定的文件格式rm、mp3，一般分为文件header和body两个部分。存在内存时，真应该也定一个数据格式。
2. 逻辑结构，逻辑结构通常不是扁平的，能够承载一定的抽象概念。比如此处的innodb存储引擎文件，物理上就是一个个页连续构成，offset 0~16kb是第一个页（假设一个页大小16kb），接着第二个页等。但页有系统页、数据页，页上有共同的segment id，那么就有了段的概念，段的功能有又不同，最终组成了一个复杂的结构。

### 物理结构

首先test.ibd 被划分为一个个页，每个页有不同的功能

每个页从offset 0到结束，有一定的格式约定。页有一个重要组成部分是行记录

行记录从offset 0到结束，有一定的格式约定。


### 逻辑结构

数据部分，将一个B+Tree存在一个文件里一个个连续摆放的页上。

表空间 ==> 段 ==> 区 ==> 页。

体现在文件上，就是一个个页（看不出来段和区）。页按大小划分，这样根据页号*大小就知道页的地址。区也固定大小，分为多个页，区大小/页大小=区内页的数量。页大小可调，区大小不可调，通过两个大小维度实现固定与灵活有机统一吧。段则界定了页数据的性质，有点类似内存管理的段页机制。


## 小结

换个角度想想，我们说数据库主要做增删改查，为了增删改、安全的增删改，有wal和checkpoint等一系列机制。那么为了查询，有索引等一些列机制。