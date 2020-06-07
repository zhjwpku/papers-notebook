# 论文笔记

This repo is inspired by [gaocegege/papers-notebook](https://github.com/gaocegege/papers-notebook).

记录阅读论文时的收获。

## 目录(TOC)

* [分布式(Distributed Systems)](#分布式distributed-systems)
    * [一致性(Consensus)](#一致性consensus)
        * [Paxos](#paxos)
        * [Raft](#raft)
    * [分布式跟踪(Tracing)](#tracing)
        * [Dapper](#dapper)
    * [存储(Storage)](#存储storage)
        * [The Google File System](#the-google-file-system)
        * [Bigtable: A Distributed Storage System for Structured Data](#bigtable-a-distributed-storage-system-for-structured-data)
* [数据库(Database)](#数据库database)
    * [主存数据库(MMDB)](#主存数据库mmdb)

## 分布式(Distributed Systems)

### 一致性(Consensus)

一说 Consensus 翻译成`共识`更恰当，以区分 consistency。

一致性是一组参与者就一个结果达成一致的过程，当参与者或它们之前的通信媒介可能经历失败时，这个问题变得困难。常见的分布式一致性协议有 Paxos、Raft。

#### **[Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science))**

*There is only one consensus protocol, and that’s Paxos-all other approaches are just broken versions of Paxos. -- Mike Burrows(Google)*

Paxos 是在不可靠的处理器网络中解决一致性的一系列协议。

#### Raft

### Tracing

#### Dapper

### 存储(Storage)

#### **[The Google File System](https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf)**

GFS 是基于 Google 的业务负载设计的一个分布式文件系统，因此具有一些独特的设计要点。

**与其他DFS相同的设计目标**

- performance
- scalability
- reliability
- availability

**设计假设**

- 由于 GFS 是构建在很多廉价的 commodity hardware 上，因此认为`故障`是正常现象而非异常
- 文件通常都是大文件（>100MB）
- 文件修改大都是 `append` 而非 `overwrite`

**设计要点及架构**

- 一个 master (HDFS namenode), 维护文件系统所有的元数据（文件和chunk的命名空间、文件到chunk的映射、chunk到chunkserver的映射）
- namespaces 和 file-to-chunk mapping 持久化到 operation log, 而 chunk 到 chunkserver 的映射关系是 master 向 chunkserver 索要的
- 多个 chunkserver (HDFS datanode), 文件数据被分为固定大小的 chunk(64MB) 保存在 chunkserver
- 三副本，分布在多个机器上并位于不同 rack
- master 和 chunckserver 之间通过心跳消息来传递指令并收集状态
- client 和 master 之间通信获取元数据，但数据的存取是直接和 chunkserver 进行的
- master 可以通过 operation log 来恢复命名空间，为了减少恢复时间，要求 operation log 不能过大，通过使用 checkpoint（compact B-tree like form） 来达到此目的
- 一致性模型，GFS 通过使用 `atomic record append` 来达到一个比较松弛（relaxed）的一致性模型，record append 使用的是 GFS 选择的 offset 而非应用指定的 offset
- GFS 使用租约（lease）来保证多副本间一致的更改顺序。master 授权其中一个 chunk 为 primary, 由它来确定多个更改的顺序
- 如果 record append 的数据超过了 chunk 的范围，会将每个 replica padding 到结尾。record append 的大小被限制为 16MB，以避免过多的空间浪费。

HDFS 是 GFS 的开源实现。GFS 的一致性模型是我认为该文最难懂的地方，需结合 2.7 3.1 和 3.3 节多看几遍。

Google Filesystem: Architecture + Consistency Model Overview [Part 1](https://www.youtube.com/watch?v=64ioICo0YBo) & [Part 2](https://www.youtube.com/watch?v=kVY_3CNPjhk)

#### **[Bigtable: A Distributed Storage System for Structured Data](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)**

Bigtable 是 Google 用来存储结构化数据的分布式存储系统，底层使用 GFS 存储数据和日志文件。具有可扩展，高性能，高可用性等特点。Apache HBase 是基于 Bigtable 的一个开源实现。

**设计要点**

- Bitable 没有固定的 schema，存储的数据往往是稀疏的，且是多维有序（multidimentional sorted map）的
- (row:string, column:string, time:int64) --> string
- 在逻辑视图层面，包含 table, row, column, version, cell 等概念，类似 RDMS
- 在物理视图层面由于 Column Familiy 概念的存在，数据的存储是按照 K/V 来存储的，且相同 CF 的数据有序排列在一起
- 由于 table 可以很大，且数据是按照 row key 有序存储的，因此可以将 table 划分为多个 tablet 存储在不同的 tablet server
- Column Family 作为最基本的访问控制单元，在将数据存储到对应 CF 之前，CF 必须已经创建
- 一个表的 CF 不能太多（最多几百），但一个 table 的 column 数量可以无限
- 每个 cell 可以包含多个版本的数据，使用 timestamp 作为索引
- Bigtable 只支持单行事务

**SSTable, MemTable, CommitLog**

Bigtable 的使用 SSTable(Sorted Strings Table) 保存持久化数据，包括数据文件和日志文件。

> An SSTable provides a persistent, ordered immutable map from key to values, where both keys and values are arbitray byte strings.

SSTable 文件保存了一些列连续的块，通常为 64KB，在文件的结尾处保存着用于定位块的索引（block index），该索引通常在 open SSTable 的时候被装载进内存，这样查找数据的时候可以先在内存中对 block index 进行二分查找，然后一次读盘就能找到 key 对应的数据。

MemTable 是存储在内存中的一个有序的 buffer，更新首先以 redo log 被记录在 commit log 中，然后插入到 Memtable 中。

commit log 用于失效回放，MemTable 在达到一定阈值之后会转换为 SSTable，并最终与其它 SSTable 合并为一个 SSTable，该过程称为 Compaction，详细介绍见 5.4 节。

如上三种结构保证了数据能够有效地持久化并不丢失。

**架构**

- Bigtable 是一个典型的主从结构，依赖 chubby 提供的分布式锁服务来保证同一时刻只有一次 active master
- chubby 除了提供上述的分布式锁服务，
    - 还保存了 bigtable 的 schema 信息（每个 table 的 column family 信息）
    - 用于发现 tablet server 及终止 tablet server
- 一主多从
    - master 负责将 tablet 分配给 tablet server, 发现 tablet server 的加入或结束，均衡 tablet server 的负载，GC 以及表信息的更改
    - 每一个 tablet server 管理了一系列 tablet，并处理相应 tablet 的读写，另外 tablet server 还负责 tablet 的分裂
    - 每一个 tablet 存储了一段 key 范围内的所有数据，每个 tablet 通常大小为 100-200MB
- 客户端直接与 tablet server 进行读写请求
- 为了减少读盘次数，为 SSTable 指定 Bloom filter 以提高读数据的效率

Bigtable 中 Column Family 是我认为相对难懂一点的概念，读者可以结合 HBase 的介绍及操作来加速理解。下面是几个 HBase 的相关视频：

1. [Apache HBase - Just the Basics](https://www.youtube.com/watch?v=2Ci_QxJ1kiE)
2. [HBase Tutorial For Beginners](https://www.youtube.com/watch?v=V1fXSCASVDc)
3. [Introductio to HBase Command Line](https://www.youtube.com/watch?v=_T9-Hmp1mEY)

## 数据库(Database)

### 主存数据库(MMDB)

#### **[Main Memory Database Systems: An Overview](https://15721.courses.cs.cmu.edu/spring2016/papers/garciamolina-tkde1992.pdf)**


