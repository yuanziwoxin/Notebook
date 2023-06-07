# TiDB体系架构

- 水平扩容或缩容
- 金融级高可用
  - 通过Raft实现三副本；
- 实时HTAP
- 云原生的分布式数据库
- 兼容MySQL 5.7协议

![image-20230421232425914](1-TiDB数据库核心和架构.assets/image-20230421232425914.png)

PD：

- 保存数据Region的元数据信息（如Region和TiKV的对应关系）；
- 为TiDB进行授时服务（如分配TSO，即分配时间戳）；

## TiDB Server

-  处理客户端的连接
- SQL语句的解析和编译
- 关系型数据与KV的转化
  - 将行记录转换为KV对，存入TiKV；
- SQL语句的执行
- 执行 online DDL
- 垃圾回收（GC）
  - 定时回收MVCC中过期的历史版本数据；
- 热点小表缓存（V6.0）
  - cache table：不经常修改，但是访问频繁，放到缓存中，提高吞吐量；

![image-20230421233309153](1-TiDB数据库核心和架构.assets/image-20230421233309153.png)

## TiKV

- 数据持久化
  - TiKV节点中运行着单机的存储引擎，内部是通过rocksdb来保证数据持久化的；
    - rocksdb kv：将表数据转换为KV对，存储在单机的rocksdb kv实例中；
    - rocksdb raft：存储指令的，对表的一些增删改记录都存在rocksdb raft实例中，然后由TiKV将这些修改记录应用到rocksdb kv实例中；
- 副本的强一致性和高可用性
  - raft协议实现三副本：
    - leader角色的Region可以读写，其他两个Region由leader角色的Region复制而来；
    - 修改集群中的大多数Region才算成功（如三个Region，必须修改成功两个才算成功）；
- MVCC（多版本并发控制）
  - 读取的是快照；
- 分布式事务支持
  - 两阶段提交
- Coprocessor （算子下推）
  - 在靠近数据存储的地方过滤好数据;
  - 在每个TiKV节点进行计算，充分利用分布式计算的优势；

![image-20230421233907677](1-TiDB数据库核心和架构.assets/image-20230421233907677.png)

## PD

PD，即 Placement Driver；

- 整个集群TiKV的元数据存储；
- 分配全局ID和事务ID；
- 生成全局时间戳TSO；
  - 事务开始时间戳
  - 事务结束时间戳
- 收集集群信息进行调度；
  - 根据集群信息（比如热节点，迁移到冷节点）进行负载均衡
- 提供TiDB Dashboard服务；
  - 热力图
  - TOP SQL
  - SQL分析

![image-20230422000013400](1-TiDB数据库核心和架构.assets/image-20230422000013400.png)

## TiFlash

- 异步复制
- 一致性
  - 相当于TiKV的列存副本，TiKV做的修改会同步到TiFlash中；
- 列式存储提供统计分析查询效率
- 业务隔离
  - 交易型业务走TiKV
  - 分析型业务走TiFlash
- 智能选择
  - 通过智能选择（TiDB优化器）判断是走TiKV还是走TiFlash；

![image-20230421235943852](1-TiDB数据库核心和架构.assets/image-20230421235943852.png)



# TiDB Server



![image-20230422095454402](1-TiDB数据库核心和架构.assets/image-20230422095454402.png)

SQL语句经过Parse、Compile生成具体的执行计划；

Executor、DistSQL和KV主要是负责SQL语句的执行，Executor对SQL进行分类，范围扫描使用DistSQL，点查使用KV；

Online DDL（不阻塞读写）使用schema load、worker、start job这三个模块；

memBuffer：缓存元数据、统计信息等；

## SQL语句的解析和编译

SQL解析成抽象语法树（AST，Abstract Syntax  Tree）

![image-20230422100539180](1-TiDB数据库核心和架构.assets/image-20230422100539180.png)



![image-20230422100745201](1-TiDB数据库核心和架构.assets/image-20230422100745201.png)



## 行记录转化为KV对

TiDB 自动将 SQL 结构映射为 KV 结构。具体的可以参考 [《三篇文章了解 TiDB 技术内幕 - 说计算》 ](https://pingcap.com/blog-cn/tidb-internal-2)这篇文档。简单来说，TiDB 做了两件事：

- 一行数据映射为一个 KV，Key 以 `TableID`构造前缀，以行 ID 为后缀
- 一条索引映射为一个 KV，Key 以 `TableID+IndexID`构造前缀，以索引值构造后缀

可以看到，对于一个表中的数据或者索引，会具有相同的前缀，这样在 TiKV 的 Key 空间内，这些 Key-Value 会在相邻的位置。那么当写入量很大，并且集中在一个表上面时，就会造成写入的热点，特别是连续写入的数据中某些索引值也是连续的(比如 update time 这种按时间递增的字段)，会在很少的几个 Region 上形成写入热点，成为整个系统的瓶颈。同样，如果所有的数据读取操作也都集中在很小的一个范围内 (比如在连续的几万或者十几万行数据上)，那么可能造成数据的访问热点。

### 聚簇表

主键是整型单字段；

表编号和主键作为Key，行记录除主键之外的字段作为Value；

![image-20230422101616438](1-TiDB数据库核心和架构.assets/image-20230422101616438.png)



![image-20230422101757050](1-TiDB数据库核心和架构.assets/image-20230422101757050.png)

![image-20230422101915248](1-TiDB数据库核心和架构.assets/image-20230422101915248.png)

### 非聚簇表

没有主键或者非整形单字段主键或者是联合主键





## SQL读写相关模块



![image-20230422102040662](1-TiDB数据库核心和架构.assets/image-20230422102040662.png)

- DistSQL将复杂SQL执行计划（join、子查询）转化为单个表操作的组合发送给TiKV；

- 点查直接走KV模块；

## 在线DDL

在线DDL请求通过TiDB start job模块转化为对应的job，然后放入到TiKV的 job队列中，然后Owner角色的TiDB Server的worker负责从job队列中按顺序拿出第一个job进行执行，并放入到history队列中；因此在同一时间只能有一个在线DDL的job在执行；

> Owner是通过选举出来的，会不断地进行切换；

![image-20230422103150968](1-TiDB数据库核心和架构.assets/image-20230422103150968.png)



## GC机制和相关模块

GC：定期清理MVCC历史版本的快照数据；

每次GC会保留当前时间到safe point时间点之间的数据；

GC Life Time：一般10min，即保留10分钟之内的数据，10分钟以前的数据会被清掉;

先找到过期数据，其次检查是否有锁信息，清理锁，然后清理drop数据，最后是delete数据...（？）

![image-20230422103528769](1-TiDB数据库核心和架构.assets/image-20230422103528769.png)

## TiDB Server的缓存

- TiDB Server缓存组成
  - SQL结果
    - 表连接、子查询等
      - 多张大表的join是在缓存中执行的，缓存占用还是比较大的；
  - 线程缓存
  - 元数据，统计信息
- TiDB Server缓存管理
  - tidb_mem_quota_query
    - 单个SQL语句占用缓存的上限；
  - oom-action
    - SQL语句占用缓存超过tidb_mem_quota_query时，如何处理？报错还是记录日志...

## 热点小表缓存

- 表的数据量不大（须小于64M）
- 只读表或修改不频繁的表
- 表的访问很频繁

难点：如果保证修改之后数据的一致性，是先修改TiKV还是先修改cache table？

![image-20230422105827848](1-TiDB数据库核心和架构.assets/image-20230422105827848.png)

### 热点小表缓存 - 原理

tidb_table_cache_lease：租约时间，默认是5s；

- 租约时间内，从缓存中读数据，但无法进行写操作；
- 租约时间到期之后，才可以写数据，
- 租约到期后读写操作执行到TiKV节点上执行
  - 因此在从缓存中读数据转换到从TiKV中执行数据的时候会有一个数据抖动；

![image-20230422111523325](1-TiDB数据库核心和架构.assets/image-20230422111523325.png)

- 租约时间内，无法进行写操作；
  - 租约时间为5s，当前还剩3s到期，写操作被阻塞；

![image-20230422111741359](1-TiDB数据库核心和架构.assets/image-20230422111741359.png)

- 租约到期，数据过期
- 写操作不在被阻塞
- 读写操作直接到TiKV节点上执行

![image-20230422111930657](1-TiDB数据库核心和架构.assets/image-20230422111930657.png)

- 数据更新完毕，租约继续开启

![image-20230422112309666](1-TiDB数据库核心和架构.assets/image-20230422112309666.png)

### 热点小表缓存 - 应用

![image-20230422112454744](1-TiDB数据库核心和架构.assets/image-20230422112454744.png)



# TiKV

## TiKV 架构和作用

![image-20230422113438383](1-TiDB数据库核心和架构.assets/image-20230422113438383.png)

- rocksdb raft：存储raft日志；
- rocksdb kv：存储具体的KV数据对；

## RocksDB



![image-20230422114025945](1-TiDB数据库核心和架构.assets/image-20230422114025945.png)

### RocksDB写入

先写入Disk的WAL文件（即预写日志，Write Ahead Log），然后在写入MemTable，当MemTable满了之后，直接将其转化为immutable MemTable（固定MemTable，即不再有写入了），然后通过独立的后台线程将immutable MemTable刷入到Disk的SST文件中；

注意：

- （1）MemTable转化为immutable MemTable之后就可以创建一个新的MemTable，继续进行写入；
- （2）先写入Disk的WAL文件，可以避免内存出现故障之后，可以通过WAL文件恢复数据到MemTable中，避免数据的丢失；
- （3）如果MemTable写入速度比较快，导致累积的immutable MemTable达到5个默认就会触发RocksDB的一个流量控制（Write Stall）; 
  - 客户端写入的速度快于磁盘刷入速度；

![image-20230422114223224](1-TiDB数据库核心和架构.assets/image-20230422114223224.png)

- Disk的WAL文件一般是有限制的，当immutable MemTable刷入磁盘之后，就会被删除；
  - 不断被覆写；
- Level 0 ：和inmutable MemTable基本一样，客户端怎么写，Level 0 就怎么存；
- 当 Level 0 文件达到4个之后，就开始启动向下一层的compaction操作（压缩），4个压缩成1个Level 1的SST文件，当Level 1 文件达到 256M，则继续往下一层进行compaction操作；下面的操作也是如此，只是触发的条件不一样；
  - 注意每次compaction操作得到的都是经过重新按Key合并排序的SST文件（存储的KV对）；

![image-20230422132259159](1-TiDB数据库核心和架构.assets/image-20230422132259159.png)

### RocksDB查询

![image-20230422125609069](1-TiDB数据库核心和架构.assets/image-20230422125609069.png)

### RocksDB：Column Families

Column Family：列簇（CF）

RocksDB通过Column Family实现分片技术，即将KV对按照不同的属性分配给不同的Column Family；

如：

write(cf1，id，name，age)

write(cf2，id，tel，address)

不指定写入的Column Family，则默认写入default的Column Family；

![image-20230422125914839](1-TiDB数据库核心和架构.assets/image-20230422125914839.png)

## 分布式事务

数据库有多种并发控制方法，这里只介绍以下两种：

- 乐观并发控制（OCC）：在事务提交阶段检测冲突
- 悲观并发控制（PCC）：在事务执行阶段检测冲突

乐观并发控制期望事务间数据冲突不多，只在提交阶段检测冲突能够获取更高的性能。悲观并发控制更适合数据冲突较多的场景，能够避免乐观事务在这类场景下事务因冲突而回滚的问题，但相比乐观并发控制，在没有数据冲突的场景下，性能相对要差。



TiDB 基于 Google [Percolator](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36726.pdf) 实现了支持完整 ACID、基于快照隔离级别（Snapshot Isolation）的分布式乐观事务。TiDB 乐观事务需要将事务的所有修改都保存在内存中，直到提交时才会写入 TiKV 并检测冲突。



在事务开始的时候，会先获得一个事务开始时间戳，然后往TiKV的Default CF和Lock CF分别预写入修改数据和锁信息，等提交时再获取一个事务提交时间戳，并将数据的指针写入Write CF，并写入删除锁的记录到Lock CF，这样一个事务就完成了；

> 注意：
>
> （1）如果数据小于255字节，则会直接存储在Write CF中；否则会存在Default CF中；
>
> （2）Default列存储的是超过255字节的数据；

![image-20230422131042155](1-TiDB数据库核心和架构.assets/image-20230422131042155.png)

![image-20230426084829182](1-TiDB数据库核心和架构.assets/image-20230426084829182.png)

![image-20230426084759375](1-TiDB数据库核心和架构.assets/image-20230426084759375.png)

## MVCC

MVCC：Multi-Version Concurrency Control，多版本并发控制；

![image-20230426085421121](1-TiDB数据库核心和架构.assets/image-20230426085421121.png)



![image-20230426085051181](1-TiDB数据库核心和架构.assets/image-20230426085051181.png)



![image-20230426085126171](1-TiDB数据库核心和架构.assets/image-20230426085126171.png)



![image-20230426085218225](1-TiDB数据库核心和架构.assets/image-20230426085218225.png)



![image-20230426085247944](1-TiDB数据库核心和架构.assets/image-20230426085247944.png)

## Raft 与 Multi Raft

![image-20230422221553309](1-TiDB数据库核心和架构.assets/image-20230422221553309.png)

### Raft 日志复制

- Propose
  - 写入的是一个raft日志；
- Append：leader节点的Append操作
  - 在leader节点上接收到一个raft日志，然后持久化到rocksdb raft实例中；
- Replicate：leader节点分发raft日志给其他节点，其他节点也要把raft日志持久到本地的rocksdb raft实例中；
  - Append：其他节点的Append操作
- Committed
  - 其他节点在将leader节点发送的raft日志持久化之后，就给leader节点一个反馈；
- Apply
  - 将rocksdb实例的raft log写入到rocksdb kv中，然后region中的数据就可以被访问了；

> Raft 日志复制中的Committed是指follower节点收到leader节点分发的raft日志，并且已经持久化到本地的rocksdb raft中后给leader节点发送反馈的意思；
>
> 而应用的Committed是指事务已提交成功，数据写入到数据库中，业务可以访问该数据了； 

![image-20230422222212501](1-TiDB数据库核心和架构.assets/image-20230422222212501.png)



![image-20230422222843530](1-TiDB数据库核心和架构.assets/image-20230422222843530.png)

![image-20230422223934077](1-TiDB数据库核心和架构.assets/image-20230422223934077.png)

![image-20230422224030419](1-TiDB数据库核心和架构.assets/image-20230422224030419.png)

![image-20230422224215723](1-TiDB数据库核心和架构.assets/image-20230422224215723.png)

![image-20230422224719508](1-TiDB数据库核心和架构.assets/image-20230422224719508.png)



## Raft 选举

在无leader状态下，最早达到election_timeout的follower节点发起选举；

比如节点2最早到达election_timeout时间，则节点2率先发起选举自己为leader节点进入term2的投票，节点1收到发起节点2发起的投票之后，发现term2比自己的term1更大，则同意投票并反馈给节点2，节点3也是如此；因此最后节点2收到两个同意票，再加自己的同意票超过半数，所以选举成功，节点2成为新的leader节点；

![image-20230423223946448](1-TiDB数据库核心和架构.assets/image-20230423223946448.png)

如果follower节点超过heartbeat_time_interval时间没有收到来自leader节点的心跳（如leader节点宕机了），最早超过heartbeat_time_interval的follower节点就会发起投票进入下一个term，另一个follower节点发现投票的term比自己的term大，就会同意投票，这样就有两个节点同意了投票（满足大多数票），所以发起投票的follower节点就变成了leader节点，原本宕机的节点（旧leader节点）在恢复之后就会以follower节点的形式加入进来；

![image-20230423224437550](1-TiDB数据库核心和架构.assets/image-20230423224437550.png)

同时发起投票，导致每个节点只有一票，选举不成功，则继续选举，直到选举成功；

>  为了避免选举一直不成功，采用random_election_timeout，即每个节点的election_timeout时间是随机的，从而尽可能降低同时发起投票的概率；

![image-20230423225115964](1-TiDB数据库核心和架构.assets/image-20230423225115964.png)

![image-20230423225606190](1-TiDB数据库核心和架构.assets/image-20230423225606190.png)

## 读写与Coprocessor

### 数据写入

用户提交事务之后，从PD获取TSO和Region信息，然后通过解析编译形成物理执行计划，通过Propose操作生成具体的Raft日志发送到leader节点的raftstore pool，然后leader节点raftstore pool中的raft日志持久化到rocksdb raft实例中（Leader节点的Append操作），然后learder节点分发raft日志给follower节点的raftstore pool，follower节点也将自身raftstore pool中的raft日志持久化到rocksdb raft实例中，之后follower节点再向leader节点进行反馈（raft的committed操作），最后再中各自将rocksdb raft实例中的日志转换为具体的操作，将数据以KV对的形式写入到各自的rocksdb kv实例，这个时候region中的数据才是可以访问的，即这个时候才是用户事务committed结束的时候；

![image-20230423230353967](1-TiDB数据库核心和架构.assets/image-20230423230353967.png)



### 数据读取

#### ReadIndex Read

![image-20230423231256060](1-TiDB数据库核心和架构.assets/image-20230423231256060.png)

![image-20230423231615894](1-TiDB数据库核心和架构.assets/image-20230423231615894.png)

#### Lease Read

![image-20230423232351893](1-TiDB数据库核心和架构.assets/image-20230423232351893.png)





#### Follower Read

![image-20230423232855995](1-TiDB数据库核心和架构.assets/image-20230423232855995.png)

![image-20230423233013536](1-TiDB数据库核心和架构.assets/image-20230423233013536.png)

![image-20230423233114355](1-TiDB数据库核心和架构.assets/image-20230423233114355.png)

## Coprocessor

![image-20230423234047641](1-TiDB数据库核心和架构.assets/image-20230423234047641.png)

![image-20230423234121568](1-TiDB数据库核心和架构.assets/image-20230423234121568.png)

![image-20230423234222366](1-TiDB数据库核心和架构.assets/image-20230423234222366.png)

## 事务大小限制

由于分布式事务要做两阶段提交，并且底层还需要做 Raft 复制，如果一个事务非常大，会使得提交过程非常慢，并且会卡住下面的 Raft 复制流程。为了避免系统出现被卡住的情况，我们对事务的大小做了限制：

- 单个事务包含的 SQL 语句不超过 5000 条（默认）
- 单条 KV entry 不超过 6MB
- KV entry 的总条数不超过 30W
- KV entry 的总大小不超过 100MB

## 总结

![image-20230423234335163](1-TiDB数据库核心和架构.assets/image-20230423234335163.png)

# PD

PD，即 Placement Driver；

![image-20230425082554697](1-TiDB数据库核心和架构.assets/image-20230425082554697.png)

- Store实际就是指TiKV实例，一个TiKV实例就是一个Store；
- Peer

![image-20230425083053001](1-TiDB数据库核心和架构.assets/image-20230425083053001.png)



![image-20230425083215201](1-TiDB数据库核心和架构.assets/image-20230425083215201.png)

## TSO

TSO=physical time logical time

int64位，logical time 中 1ms可以分配264212个TSO；

### TSO分配

![image-20230425083754579](1-TiDB数据库核心和架构.assets/image-20230425083754579.png)



![image-20230425084239010](1-TiDB数据库核心和架构.assets/image-20230425084239010.png)



## 调度

![image-20230425084828744](1-TiDB数据库核心和架构.assets/image-20230425084828744.png)



- Store heartbeat：TiKV实例的心跳；
- Region heartbeat：Region的心跳；

![image-20230425084854240](1-TiDB数据库核心和架构.assets/image-20230425084854240.png)



![image-20230425085019515](1-TiDB数据库核心和架构.assets/image-20230425085019515.png)



![image-20230425085206129](1-TiDB数据库核心和架构.assets/image-20230425085206129.png)

## Label 与 高可用

![image-20230425085714080](1-TiDB数据库核心和架构.assets/image-20230425085714080.png)

Lable需要在PD和TiKV两个组件上进行配置；

zone：对应数据中心编号；

rack：对应机柜编号；

host：对应服务器；

location-labels用来告诉PD，配置了哪些Label参数；

isolation-level：表示隔离级别，如isolation-level="zone"，表示同一个DC中不能有两个相同的Region，即相同的Region必须分布在不同的DC上；

![image-20230425090044151](1-TiDB数据库核心和架构.assets/image-20230425090044151.png)

# TiDB数据库SQL执行流程

## DML

![image-20230425120945833](1-TiDB数据库核心和架构.assets/image-20230425120945833.png)

![image-20230425121047992](1-TiDB数据库核心和架构.assets/image-20230425121047992.png)

## DDL

![image-20230425121347967](1-TiDB数据库核心和架构.assets/image-20230425121347967.png)

## SQL的Parse与Compile

![image-20230425121620142](1-TiDB数据库核心和架构.assets/image-20230425121620142.png)

- preprocess：SQL语法是否正确；查询的数据是否只读取一行，如果是则直接点查；如果不是，则进行优化流程（逻辑优化和物理优化）；
  - 物理优化是结合统计信息；

## 读取的执行

![image-20230425122103545](1-TiDB数据库核心和架构.assets/image-20230425122103545.png)

- 第一次获取Region信息从PD节点上，后面会region信息会缓存到TiDB中的TiKV Client中的region Cache，下次获取会从region Cache获取Region信息，如果leader节点发现变化、Region已经分裂或者合并，所以访问时反馈错误信息（即backoff），则会PD节点会通知TiDB进行变化；

![image-20230425122429321](1-TiDB数据库核心和架构.assets/image-20230425122429321.png)

![image-20230425122647951](1-TiDB数据库核心和架构.assets/image-20230425122647951.png)



## 写入的执行

![image-20230425122754907](1-TiDB数据库核心和架构.assets/image-20230425122754907.png)

![image-20230425123021279](1-TiDB数据库核心和架构.assets/image-20230425123021279.png)

## DDL的执行

![image-20230425123245210](1-TiDB数据库核心和架构.assets/image-20230425123245210.png)

# HTAP

![image-20230425203518015](1-TiDB数据库核心和架构.assets/image-20230425203518015.png)



![image-20230425203934428](1-TiDB数据库核心和架构.assets/image-20230425203934428.png)



![image-20230425204020087](1-TiDB数据库核心和架构.assets/image-20230425204020087.png)



![image-20230425204128417](1-TiDB数据库核心和架构.assets/image-20230425204128417.png)

## MPP

![image-20230425204342786](1-TiDB数据库核心和架构.assets/image-20230425204342786.png)





![image-20230425204655953](1-TiDB数据库核心和架构.assets/image-20230425204655953.png)



![image-20230425204746009](1-TiDB数据库核心和架构.assets/image-20230425204746009.png)



![image-20230425204800480](1-TiDB数据库核心和架构.assets/image-20230425204800480.png)



![image-20230425204834908](1-TiDB数据库核心和架构.assets/image-20230425204834908.png)



![image-20230425205043692](1-TiDB数据库核心和架构.assets/image-20230425205043692.png)



![image-20230425205134422](1-TiDB数据库核心和架构.assets/image-20230425205134422.png)

![image-20230425205150668](1-TiDB数据库核心和架构.assets/image-20230425205150668.png)

## 混合工作负载场景

![image-20230425205226582](1-TiDB数据库核心和架构.assets/image-20230425205226582.png)

## 流式计算场景

![image-20230425205317074](1-TiDB数据库核心和架构.assets/image-20230425205317074.png)

![image-20230425205450378](1-TiDB数据库核心和架构.assets/image-20230425205450378.png)

# TiFlash

## TiFlash架构

![image-20230425210142971](1-TiDB数据库核心和架构.assets/image-20230425210142971.png)



TiFlash主要功能：

- 异步复制
- 一致性读取
- 引擎智能选择
- 计算加速

> 一般使用TiFlash的QPS应小于50；

## 异步复制

![image-20230425210634039](1-TiDB数据库核心和架构.assets/image-20230425210634039.png)

## 一致性读取

就是说在TiDB Server通过访问TiFlash副本查询数据的时候，TiFlash会向TiKV发送一个非常轻量级的请求查询TiKV最新写的数据位移（idx）是多少，默认然后等TiFlash异步复制到之前查询的TiKV写的最新位移数据时，然后根据查询TiFlash副本数据的时间戳确定查询距离该时间戳最近的一个TiFlash副本快照数据；从而保证TiKV和TiFlash数据读取的一致性。

![image-20230425210756851](1-TiDB数据库核心和架构.assets/image-20230425210756851.png)



![image-20230425212123063](1-TiDB数据库核心和架构.assets/image-20230425212123063.png)



![image-20230425212216514](1-TiDB数据库核心和架构.assets/image-20230425212216514.png)



![image-20230425212235472](1-TiDB数据库核心和架构.assets/image-20230425212235472.png)



![image-20230425212334146](1-TiDB数据库核心和架构.assets/image-20230425212334146.png)



![image-20230425212450244](1-TiDB数据库核心和架构.assets/image-20230425212450244.png)

## 智能选择

![image-20230425211534237](1-TiDB数据库核心和架构.assets/image-20230425211534237.png)



# TiDB 6.0新特性

## Placement Rules in SQL

![image-20230425214112763](1-TiDB数据库核心和架构.assets/image-20230425214112763.png)

![image-20230425214433091](1-TiDB数据库核心和架构.assets/image-20230425214433091.png)

### Placement Rules in SQL的使用步骤

![image-20230425214639640](1-TiDB数据库核心和架构.assets/image-20230425214639640.png)



![image-20230425214807704](1-TiDB数据库核心和架构.assets/image-20230425214807704.png)

![image-20230425214944084](1-TiDB数据库核心和架构.assets/image-20230425214944084.png)

### Placement Rules in SQL 的应用

![image-20230425215048904](1-TiDB数据库核心和架构.assets/image-20230425215048904.png)

## 热点小表缓存



![image-20230425215334079](1-TiDB数据库核心和架构.assets/image-20230425215334079.png)



## 内存悲观锁

悲观锁：在写入事务提交前，将锁信息写入到TiKV存储中（lock CF）；



![image-20230425215853797](1-TiDB数据库核心和架构.assets/image-20230425215853797.png)



- 锁信息**只**写入到Leader角色Region所在的TiKV节点上的缓存中；

![image-20230425220323137](1-TiDB数据库核心和架构.assets/image-20230425220323137.png)



- 锁信息无副本，只存在于leader角色Region所在的TiKV节点的缓存中，锁信息丢失后，TiDB为了保证事务的一致性，只能失败回滚；

![image-20230425220543199](1-TiDB数据库核心和架构.assets/image-20230425220543199.png)



## Top SQL

![image-20230425221017148](1-TiDB数据库核心和架构.assets/image-20230425221017148.png)



![image-20230425221213861](1-TiDB数据库核心和架构.assets/image-20230425221213861.png)



![image-20230425221257728](1-TiDB数据库核心和架构.assets/image-20230425221257728.png)

![image-20230425221325156](1-TiDB数据库核心和架构.assets/image-20230425221325156.png)



![image-20230425221342393](1-TiDB数据库核心和架构.assets/image-20230425221342393.png)

## TiDB Enterprise Manager（TiEM）

![image-20230425222005934](1-TiDB数据库核心和架构.assets/image-20230425222005934.png)

![image-20230425222048865](1-TiDB数据库核心和架构.assets/image-20230425222048865.png)

![image-20230425221708819](1-TiDB数据库核心和架构.assets/image-20230425221708819.png)

# TiDB Cloud



![image-20230425224900096](1-TiDB数据库核心和架构.assets/image-20230425224900096.png)



## 为什么选择TiDB

![image-20230425224925578](1-TiDB数据库核心和架构.assets/image-20230425224925578.png)



![image-20230425225136318](1-TiDB数据库核心和架构.assets/image-20230425225136318.png)



![image-20230425225506585](1-TiDB数据库核心和架构.assets/image-20230425225506585.png)

## 什么是TiDB Cloud

DBaaS：Database as a Service，数据库即服务；

IaaS：Infrastructure as a Service，基础设施即服务；

- 基础设施服务，如硬件

PaaS：Platform as a Service，平台即服务；

- 基础软件服务，如操作系统；

SaaS：Software as a Service，软件即服务；

- 软件以服务的方式提供给你，只需运营即可

![image-20230425225556434](1-TiDB数据库核心和架构.assets/image-20230425225556434.png)



![image-20230425230059484](1-TiDB数据库核心和架构.assets/image-20230425230059484.png)



![image-20230425230326037](1-TiDB数据库核心和架构.assets/image-20230425230326037.png)



![image-20230425230928163](1-TiDB数据库核心和架构.assets/image-20230425230928163.png)



![image-20230425231019353](1-TiDB数据库核心和架构.assets/image-20230425231019353.png)



![image-20230425231122695](1-TiDB数据库核心和架构.assets/image-20230425231122695.png)

![image-20230425231940085](1-TiDB数据库核心和架构.assets/image-20230425231940085.png)



![image-20230425231959696](1-TiDB数据库核心和架构.assets/image-20230425231959696.png)