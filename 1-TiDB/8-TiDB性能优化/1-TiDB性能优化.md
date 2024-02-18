# 值得一读的性能优化好文

- [TiDB 性能分析和优化](https://docs.pingcap.com/zh/tidb/v6.1/performance-tuning-methods)

- [OLTP 负载性能优化实践](https://docs.pingcap.com/zh/tidb/v6.1/performance-tuning-practices)

- [Performance Overview 面板重要监控指标详解](https://tidb.net/blog/90492e9b)

- [掌握这两个调优技巧，让TiDB性能提速千倍！](https://developer.aliyun.com/article/879825)
  - 解释了为什么不建议使用自增主键？

## 故障诊断相关

- [定位慢查询](https://docs.pingcap.com/zh/tidb/v6.1/identify-slow-queries)

- [分析慢查询](https://docs.pingcap.com/zh/tidb/v6.1/analyze-slow-queries)

![image-20230611152758538](1-TiDB性能优化.assets/image-20230611152758538.png)



# 热点问题

## 为什么不建议使用自增主键？

在TiDB的整个架构中，分布式数据存储引擎TiKV Server负责存储数据。在存储数据时，TiKV采用范围切分（range）的方式对数据进行切分，切分的最小存储单位是region。每个region有大小限制（默认上限为96M），会有多个副本，每一组副本，成为一个raft group。每个raft group中由leader负责执行这个块数据的读&写。leader会自动地被PD组件（Placement Driver，简称“PD”，是整个集群的管理模块）均匀调度在不同的物理节点上，用以均分读写压力，实现负载均衡。

![img](1-TiDB性能优化.assets/b0078dd696cf44a5aa13f136dcda162c.png)

TiDB会为每个表分配一个TableID，为每一个索引分配一个IndexID，为每一行分配一个RowID（默认情况下，如果表使用整数型的Primary Key，那么会用Primary Key的值当做RowID）。同一个表的数据会存储在以表ID开头为前缀的一个range中，数据会按照RowID的值顺序排列。在插入（insert）表的过程中，如果RowID的值是递增的，则插入的行只能在末端追加。

数据在TiKV中是KV的形式存储，Key为一条数据的标识，Value为实际的数据；

```sql
-- 1、表数据与Key—Value的映射关系
Key:   tablePrefix{TableID}_recordPrefixSep{RowID}
Value: [col1, col2, col3, col4]
-- 2、索引数据与Key-Value的映射关系
-- （1）唯一索引
Key:   tablePrefix{tableID}_indexPrefixSep{indexID}_indexedColumnsValue
Value: RowID
-- （2）非唯一索引：tablePrefix{TableID}_indexPrefixSep{IndexID}_indexedColumnsValue 这个键值可能对应多行
Key:   tablePrefix{TableID}_indexPrefixSep{IndexID}_indexedColumnsValue_{RowID}
Value: null
```

> `tablePrefix`、`recordPrefixSep` 和 `indexPrefixSep` 都是字符串常量，用于在 Key 空间内区分其他数据

当Region达到一定的大小之后会进行分裂，**分裂之后还是只能在当前range范围的末端追加，并永远仅能在同一个Region上进行insert操作，由此形成热点（即单个节点的过高负载）**，陷入TiDB使用的“反模式”。

常见的increment类型自增主键就是按顺序递增的，默认情况下，在主键为整数型时，会将主键值作为RowID ，此时RowID也为顺序递增，在大量insert时就会形成表的写入热点。同时，TiDB中RowID默认也按照自增的方式顺序递增，主键不为整数类型时，同样会遇到写入热点的问题。

在使用MySQL数据库时，为了方便，我们都习惯使用自增ID来作为表的主键。**因此，将数据从MySQL迁移到TiDB之后，原来的表结构都保持不变，仍然是以自增ID作为表的主键。**这样就造成了批量导入数据时出现TiDB写入热点的问题，导致Region分裂不断进行，消耗大量资源。

对此，在进行TiDB优化时，我们从表结构入手，对以自增ID作为主键的表进行重建，删除自增ID，使用TiDB隐式的_tidb_rowid列作为主键，将

```sql
create table t (a int primary key auto_increment, b int)；
```

改为：

```sql
create table t (a int, b int)SHARD_ROW_ID_BITS=4 PRE_SPLIT_REGIONS=2
```

通过设置SHARD_ROW_ID_BITS，将RowID打散写入多个不同的Region，从而缓解写入热点问题。

此处需要注意，SHARD_ROW_ID_BITS值决定分片数量：

- SHARD_ROW_ID_BITS = 0 表示 1 个分片
- SHARD_ROW_ID_BITS = 4 表示 16 个分片
- SHARD_ROW_ID_BITS = 6 表示 64 个分片

**SHARD_ROW_ID_BITS值设置的过大会造成RPC请求数放大，增加CPU和网络开销**，这里我们将SHARD_ROW_ID_BITS设置为4。

PRE_SPLIT_REGIONS指的是建表成功后的预均匀切分，我们通过设置PRE_SPLIT_REGIONS=2，实现建表成功后预均匀切分2^(PRE_SPLIT_REGIONS)个Region。

#### 经验总结

- 以后新建表禁止使用自增主键，

- 考虑使用业务主键

- 加上参数SHARD_ROW_ID_BITS = 4  PRE_SPLIT_REGIONS=2；

# SQL优化

## 原则和方法

参考：[简单谈谈MySQL索引失效问题](https://juejin.cn/post/7032198003968442405)

### 是否应该建索引

#### 哪些情况需要创建

- **频繁作为查询条件的字段**
- 与其他表关联，作为查询条件的**外键字段**
- 单键索引/复合索引？高并发下推荐复合索引
- **查询中排序的字段**，排序字段通过索引访问将提高排序速度
- **查询中统计或分组的字段**，group by也要排序，所以也推荐建立索引

#### 哪些情况不需要创建

- **表记录太少**，建了也没效果，可能引起索引失效
- **频繁更新的字段**，更新字段的同时也会更新索引，消耗性能、降低效率
- **where条件中不使用的字段**
- **经常增删改的表**
- 包含许多**重复内容的数据列**，建立索引没有太大的效果

> 假如一个表有10万行记录，有一个字段A只有T和F两种值，且每个值的分布概率大约为50%，那么对这种表A字段建索引一般不会提高数据库的查询速度。

> 索引的选择性是指索引列中不同值的数目与表中记录数的比。如果一个表中有2000条记录，表索引列有1980个不同的值，那么这个索引的选择性就是1980/2000=0.99
>
> 一个索引的选择性越接近于1，这个索引的效率就越高

### 一些原则

- 越早过滤越好，越早limit越好
  - 业务逻辑能提前过滤，则先提前过滤；
  - 关注SQL语句的执行顺序：如where优先级高于having，能写在where中的条件就不要写在having中了；
- 避免笛卡尔积
  - 笛卡尔积很容易造成oom问题；
- 避免关联字段类型和过滤字段类型不一致，而导致不能走索引；
  - varchar的查询条件为数字时，会变成字符串截取数字来查询
  - int的查询条件为字符串时，会隐式地将数字转换为字符串来查询，所以会走索引
- 按需查询字段，不要select *
- 小表驱动大表

即：先操作小的数据集，再操作大的数据集

**in和exists怎么选用？**

in先查内表，exists先查外表

所以in里的子集小时，优先用in，反之用exists

- exists：
  - 将主查询的数据，放到子查询中做条件验证，根据验证结果(TRUE或FALSE)来决定主查询的数据结果是否得以保留
  - 因为只返回结果TRUE/FALSE，所以子查询中不管是select *还是select 1或其他都可以，执行时会忽略字段的
- in：
  - in中的查询只执行一次，它将查询出的所有的数据缓存起来，然后检查外表中查询出的字段在缓存中是否存在，如果存在则将外表的查询数据加入到结果集中，直到遍历完外表中所有的结果集为止

```sql
sql复制代码# 先查B表，再放入A表查询
select * from A where id in (select id from B)

# 先查A表，再联系B表查询
select * from A where exists (select 1 from B where B.id = A.id)
```

我们的宗旨是先操作小的，

那么当B数据量比A小的时候，我们应该先操作B表

而in先查内表，即先查B表，所以：B小用in

当A数据量比B小的时候，我们应该先操作A表

而exists先查外表，即先查A表，所以：A小用exists；

### 索引失效

- ORDER BY a DESC , b ASC , c DESC (排序不一致)

```sql
CREATE index idx_abcd on abcd(a,b,c,d);

-- 可以使用到索引idx_abcd中的a,b,c,d;
select * from abcd where a='a' and b='b' and c='c' and d='d'

-- 可以使用到索引idx_abcd中的a,b,c,d; 条件中的顺序不会影响到索引的使用
select * from abcd where d='d' and c='c' and b='b' and a='a'

-- 可以使用到索引idx_abcd中的a,b,c,d
select * from abcd where a='a' and b='b' and d>='d' and c='c'

-- 只能使用到索引idx_abcd中的a,b,c？因为c为范围查询，所以范围查询后面的d不走索引？
select * from abcd where a='a' and b='b' and c>='c' and d='d'

-- 可以使用到索引idx_abcd中的a,b,c; a和b用来过滤，而c用来排序
select * from abcd where a='a' and b='b' and d='d' ORDER BY c

-- 只能使用到索引idx_abcd中的a,b
select * from abcd where a='a' and b='b' ORDER BY d

-- 可以使用到索引idx_abcd中的a,b,c
select * from abcd where a='a'  ORDER BY b,c;

-- 只使用到了索引idx_abcd中的a；b，c没有使用到，因为不符合顺序，在order by中须严格遵守顺序；
select * from abcd where a='a' ORDER BY c,b;
```



我们先说一下`where a='a'  ORDER BY b,c`用到了几个索引？

3个，a、b、c都用到了

根据顺序来的，a用于查找，b、c用于排序，无Using filesort，用到了3个索引，没毛病把

进入正题，下列sql用到了几个索引？

1个，只有a用到了，c和b没有用到，因为不符合顺序，在order by中严格遵守顺序



索引失效口诀：**模型数空运最快**

- 模：模糊查询LIKE以%开头

- 型：数据类型错误

  - 关联类型或过滤类型不一致

- 数：对索引字段使用内部函数

- 空：索引列是NULL

  - 只要列中包含有NULL值都将不会被包含在索引中
  - 复合索引中只要有一列含有NULL值，那么这一列对于此复合索引就是无效的

- 运：索引列进行四则运算

- 最：复合索引不按索引列最左开始查找

- 快：全表查找预计比索引更快

  

### 关联查询太多join：从表索引

可能是开发人员编写sql时设计的缺陷

也可能是业务中不得已的需求导致的

以left join左连接为例

在左连接中，左表会将数据全部访问，所以我们应该为右表建立索引

例如`select * from a left join b on a.id = b.id`时，我们为b表的id建立索引

right join右连接同理给左表条件字段建立索引

多表查询中也是如此

- 尽可能减少Join语句中的NestedLoop的循环总次数：永远用小结果集驱动大的结果集
- 优先优化NestedLoop的内层循环
- 保证Join语句中被驱动表上Join条件字段已经被索引
- 当无法保证被驱动表的Join条件字段被索引且内存资源充足的前提下，不要太吝惜JoinBuffer的设置

所以得出结论：**从表建立索引**



## join

```sql
select a.name,b.account from user a left join user_info b on a.id=b.user_id
```

其次我们来看看这种 join 方式的原理：

1. 从 user 表扫描一条数据，然后去 user_info 表中匹配
2. 在连接键 user_id 有索引的情况下，可以利用索引快速匹配
3. 然后把 user 表中的 name 和 user_info 表中的 account 作为结果集的一部分返回回去
4. 重复 1-3 步骤，直至 user 表扫描完毕，数据全部返回。

其中第三步骤，每次组合一条数据的时候，并不是立马返回给客户端，这样效率太低，其实是有缓冲区的，也就是先把数据放在缓冲区中，等缓冲区满了，一次性响应给客户端可以大大提升效率。

从原理来看和上面的遍历查询差不多，主要不同的是，客户端不需要和服务端多次通信。

### join buffer (Block Nested Loop)

以上说的还是连接键有索引的，我们来看看连接键没有索引的情况，这时候你通过 explain 来看 MySQL 的执行计划，你会发现其中 user_info 的 extra 字段中会提示这个：

```vbnet
Using where; Using join buffer (Block Nested Loop)
```

这是什么意思呢？

因为没有索引，所以每次去 user 表得到一条数据的时候，肯定是要再到 user_info 表做全表扫描，这个扫描的成本我们上面也提到了，就是 n+n*m=n(1+m)，因此这个时间复杂度是和 n 成正比的，这也是为什么我们一般推荐**小表驱动大表**的方式。

但是如果我们按照这个方式来做 join，未免开销太大了，太耗时了，于是还是沿用老套路，也就是用个临时存储区，也就是 extra 中的 `join buffer`，有了这个 join buffer 后，首先会把 user 表的数据放进去，然后扫描 user_info 表，每扫描一行数据，就和 join buffer 中的每一行 user 数据匹配，如果匹配上了，也就是我们要的结果，因为 user_info 表有 m 条数据，因此需要判断 n*m 次，咦！这个也没减少呀，还是和上面的一样。其实不一样，这里的 m 条数据其实每次都是和内存中的 n 条数据做匹配的，并非磁盘，内存的速度不用多说。

聪明的读者可能会发现，如果 user 表的数据很多，join buffer 能放得下吗？

```diff
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| join_buffer_size | 262144 |
+------------------+--------+
```

buffer 默认是 256K，多的话确实放不下，放不下的话，怎么办？其实也很简单，分段放即可，当读 user 表的数据占满 buffer 的时候，就不放了，然后直接和 user_info 做匹配，逻辑还是同上，在 buffer 的数据处理完之后，就清空它，接着上次的位置继续读入数据，再次重复同样的逻辑，直至数据读完。

虽说连接键没有索引的时候，会通过 join buffer 来优化速度，但是现实中，还是建议大家尽量要保证连接键有索引。

### join查询无法走索引的情况

- 连接键两边的字段类型不一样，比如一边是int类型，一边是varchar；

- 连接键两边的字符编码不一致也可能无法索引；

  ```sql
  -- A表utf8mb4, B表utf8
  -- A表为主表，可以走索引，因为utf8mb4可以兼容utf8
  A join B on A.name = B.name
  -- B表为主表，不可以走索引,因为utf8兼容不了utf8mb4
  B join A on B.name = A.name
  ```

### 过滤条件放在on和where后面的区别

- 在inner join中，过滤条件放在on和where后面，返回结果没有差别；
- 在left join中，过滤条件放在on后面，只对右表有效；过滤条件放在where后，表示在left join关联之后的结果集进行过滤，对左表和右表都有效；
- 在right join中，过滤条件放在on后面，只对左表有效；过滤条件放在where后，表示在right join关联之后的结果集进行过滤，对左表和右表都有效；

### 多表关联

```sql
CREATE TABLE `t1` (
 `id` int(11) NOT NULL,
 `a` int(11) DEFAULT NULL,
 `b` int(11) DEFAULT NULL,
 `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

create table t2 like t1;
create table t3 like t2;
insert into ... //初始化三张表的数据
```

三个表的join

```sql
select * from t1 join t2 on(t1.a=t2.a) join t3 on (t2.b=t3.b) where t1.c>=X and t2.c>=Y and t3.c>=Z;
```

如果改写成 straight_join，要怎么指定连接顺序，以及怎么给三个表创建索引。
第一原则是要尽量使用 BKA （Block Key Access）算法。
需要注意的是，使用 BKA 算法的时候，并不是“先计算两个表 join 的结果，再跟第三个表 join”，而是直接嵌套查询的。
具体实现是：

- 在 t1.c>=X、t2.c>=Y、t3.c>=Z 这三个条件里，选择一个经过过滤以后，数据最少的那个表，作为第一个驱动表。

此时，可能会出现如下两种情况。

- 第一种情况，如果选出来是表 t1 或者 t3，那剩下的部分就固定了。

  - 如果驱动表是 t1，则连接顺序是 t1->t2->t3，要在被驱动表字段创建上索引，也就是 t2.a 和 t3.b 上创建索引；
  - 如果驱动表是 t3，则连接顺序是 t3->t2->t1，需要在 t2.b 和 t1.a 上创建索引。同时，我们还需要在第一个驱动表的字段 c 上创建索引。

- 第二种情况是，如果选出来的第一个驱动表是表 t2 的话，则需要评估另外两个条件的过滤效果。

  - 假如以t2表为驱动，首先t2.c加上索引，然后t2表关联t1，给t1加上t1.a索引扫描最少行，在join_buffer中判断t1.c＞x所以t1.c不用加索引，t2和t1联表的结果与t3关联，需要给t3加上t3.b索引。
    结论：t2表作为驱动表，加t2.c，t1.a，t3.b索引

    > 第二种情况待验证

总之，整体的思路就是，尽量让每一次参与 join 的驱动表的数据集，越小越好，因为这样我们的驱动表就会越小。



### 总结

a join b：假设a表为小表

- 一般使用a表驱动b表，从a表每扫描一条数据然后去匹配b表匹配一次；
- 一般会把驱动表（a表）放入join buffer（默认256KB）, 当驱动表大小超过 join buffer时，会分段放入，上一段数据处理完就清空，然后再放入下一段；
- 推荐小表驱动大表；
  - 将行数少的表作为主表，然后去join行数多的表，这样索引扫描可以减少扫描的行数，并且如果使用到大表的索引，会最大程度的发挥索引的价值；
- 尽量保证连接键有索引；

Join优化的思路：

- 1. 尽量让小表驱动大表，表的大小不单单是看表的总数据量，需要结合join 的条件过滤来看；
- 2. 尽量让被驱动表走索引，让被驱动表的条件判断能走索引；
- 3. 考虑join buffer； 

## in 和 exists

索引是个前提，其实和选择与否 还是要看表的大小。

选择的标准，你可以理解为： 小表驱动大表。这种方式下效率是最高的。

比如 ：

```sql
SELECT * FROM A WHERE cc IN (SELECT cc FROM B) 
SELECT * FROM A WHERE EXIST (SELECT cc FROM B WHERE B.cc=A.cc) 
```

当A小于B时，用EXIST。

因为EXIST的实现，相当于外表循环，实现的逻辑类似于：

```java
for i in A    
	for j in B        
		if j.cc == i.cc then ... 
```

当B小于A时，用IN，因为实现的逻辑类似于：

```java
for i in B    
    for j in A        
        if j.cc == i.cc then ...
```

 所以哪个表小就用哪个表来驱动，A表小 就用EXIST，B表小 就用IN。

实际上在查询过程中，在我们对 cc 列建立索引的情况下，我们还需要判断表 A 和表 B 的大小。

- 如果表 A 比表 B 大，那么 IN 子查询的效率要比 EXIST 子查询效率高，因为这时 B 表中如果对 cc 列进行了索引，那么 IN 子查询的效率就会比较高。

- 如果表 A 比表 B 小，那么使用 EXISTS 子查询效率会更高，因为我们可以使用到 A 表中对 cc 列的索引，而不用从 B 中进行 cc 列的查询。

#### in和exists的执行过程

**exists的执行原理：**

对外表做loop循环，每次loop循环再对内表（子查询）进行查询，那么因为对内表的查询使用的索引（内表效率高，故可用大表），而外表有多大都需要遍历，不可避免（尽量用小表），故内表大的使用exists，可加快效率；

**in的执行原理：**

是把外表和内表做连接，先查询内表，再把内表结果与外表匹配，对外表使用索引（外表效率高，可用大表），而内表多大都需要查询，不可避免，故外表大的使用in，可加快效率。

参考：[MySQL中的IN与EXISTS](https://zhuanlan.zhihu.com/p/147044422)

## semi-join



```sql
EXPLAIN SELECT * FROM t1 WHERE id IN (SELECT t1_id FROM t2 WHERE t1_id != t1.int_col);
```

```sql
+-----------------------------+-----------+-----------+------------------------------+--------------------------------------------------------------------------------------------------------+
| id                          | estRows   | task      | access object                | operator info                                                                                          |
+-----------------------------+-----------+-----------+------------------------------+--------------------------------------------------------------------------------------------------------+
| MergeJoin_9                 | 45446.40  | root      |                              | semi join, left key:test.t1.id, right key:test.t2.t1_id, other cond:ne(test.t2.t1_id, test.t1.int_col) |
| ├─IndexReader_24(Build)     | 180000.00 | root      |                              | index:IndexFullScan_23                                                                                 |
| │ └─IndexFullScan_23        | 180000.00 | cop[tikv] | table:t2, index:t1_id(t1_id) | keep order:true                                                                                        |
| └─TableReader_22(Probe)     | 56808.00  | root      |                              | data:Selection_21                                                                                      |
|   └─Selection_21            | 56808.00  | cop[tikv] |                              | ne(test.t1.id, test.t1.int_col)                                                                        |
|     └─TableFullScan_20      | 71010.00  | cop[tikv] | table:t1                     | keep order:true                                                                                        |
+-----------------------------+-----------+-----------+------------------------------+--------------------------------------------------------------------------------------------------------+
6 rows in set (0.00 sec)
```

由上述查询结果可知，TiDB 执行了 `Semi Join`。不同于 `Inner Join`，`Semi Join` 仅允许右键 (`t2.t1_id`) 上的第一个值，也就是该操作将去除 `Join` 算子任务中的重复数据。`Join` 算法也包含 `Merge Join`，会按照排序顺序同时从左侧和右侧读取数据，这是一种高效的 `Zipper Merge`。

可以将原语句视为*关联子查询*，因为它引入了子查询外的 `t1.int_col` 列。然而，`EXPLAIN` 语句的返回结果显示的是[关联子查询去关联](https://docs.pingcap.com/zh/tidb/v6.1/correlated-subquery-optimization)后的执行计划。条件 `t1_id != t1.int_col` 会被重写为 `t1.id != t1.int_col`。TiDB 可以从表 `t1` 中读取数据并且在 `└─Selection_21` 中执行此操作，因此这种去关联和重写操作会极大提高执行效率。

> Semi Join 就是
>
> 所谓的semi-join是指semi-join子查询。 当一张表在另一张表找到匹配的记录之后，半连接（semi-jion）返回第一张表中的记录。
>
> 与条件连接相反，即使在右表中找到几条匹配的记录，左表也只会返回一条记录。另外，右节点的表一条记录也不会返回。半连接通常使用IN  或 EXISTS 作为连接条件。（说白了会对右表的数据去重！！！）
>
> 换句话说，最后的结果集是在outer_tables中的，而semi-join的作用只是对outer_tables中的记录进行筛选。这也是我们进行 semi-join优化的基础，即我们只需要从semi-join中获取到最少量的足以对outer_tables记录进行筛选的信息就足够了。



# TiDB数据库架构概述

## TiDB体系架构

![image-20240124122049949](1-TiDB性能优化.assets/image-20240124122049949.png)



## TiDB Server

![image-20240124122154668](1-TiDB性能优化.assets/image-20240124122154668.png)

## TiKV

![image-20240124122317047](1-TiDB性能优化.assets/image-20240124122317047.png)

## PD

![image-20240124122553942](1-TiDB性能优化.assets/image-20240124122553942.png)

## TiFlash

![image-20240124122656856](1-TiDB性能优化.assets/image-20240124122656856.png)

# TiDB Server

## TiDB Server架构

![image-20240124123412338](1-TiDB性能优化.assets/image-20240124123412338.png)

## TiDB Server主要功能

![image-20240124123642840](1-TiDB性能优化.assets/image-20240124123642840.png)

## SQL语句的解析和编译

### Parse

![image-20240124123848035](1-TiDB性能优化.assets/image-20240124123848035.png)

### Compile

![image-20240124124027978](1-TiDB性能优化.assets/image-20240124124027978.png)

## SQL读写相关模块

![image-20240124214036941](1-TiDB性能优化.assets/image-20240124214036941.png)

## 在线DDL相关模块

![image-20240124214443203](1-TiDB性能优化.assets/image-20240124214443203.png)

## GC机制与相关模块

![image-20240124214811770](1-TiDB性能优化.assets/image-20240124214811770.png)

## TiDB Server的缓存

![image-20240124215212281](1-TiDB性能优化.assets/image-20240124215212281.png)



# TiKV

![image-20240124215917171](1-TiDB性能优化.assets/image-20240124215917171.png)

## TiKV架构和作用

![image-20240124215930949](1-TiDB性能优化.assets/image-20240124215930949.png)



## 分布式事务



![image-20240125084142390](1-TiDB性能优化.assets/image-20240125084142390.png)



![image-20240125085532683](1-TiDB性能优化.assets/image-20240125085532683.png)



![image-20240125085644078](1-TiDB性能优化.assets/image-20240125085644078.png)

同一个事务里只有一个主锁，即修改的第一行为主锁，后面的行的锁均指向于主锁。

- 获取commit时间戳；
- 写入提交信息到Write CF；
- 删除锁信息（注意：是put一条锁删除的信息）



两阶段提交事务中，如果一行在一个TiKV节点修改成功了，而另外一行在另外一个TiKV节点没有写入成功，怎么办？

当发现第二条数据没有成功提交到TiKV Node2，但是TiKV Node2的Lock CF中还有锁信息，并指向TiKV Node1中的第一条修改记录的主锁，到TiKV Node1发现该主锁已经删除，说明主锁已经释放，事务已经提交，所以此时TiKV Node2的第二条数据进行补提交（即补充提交信息和TiKV Node2的Lock CF中补充删除锁信息），这样保证分布式事务的原子性。

![image-20240125090647308](1-TiDB性能优化.assets/image-20240125090647308.png)



## MVCC

![image-20240126083855175](1-TiDB性能优化.assets/image-20240126083855175.png)

![image-20240126084312290](1-TiDB性能优化.assets/image-20240126084312290.png)

![image-20240126084616241](1-TiDB性能优化.assets/image-20240126084616241.png)

![image-20240126084833555](1-TiDB性能优化.assets/image-20240126084833555.png)



![image-20240126084946655](1-TiDB性能优化.assets/image-20240126084946655.png)

读：先读Write CF，然后再读Default CF；

写：先读Write CF，然后再读Lock CF，最后读Default CF；



## Raft

### Raft 与 Multi Raft

![image-20240126085227799](1-TiDB性能优化.assets/image-20240126085227799.png)



### Raft - 日志复制![image-20240126085834331](1-TiDB性能优化.assets/image-20240126085834331.png)

4_1,log{PUT key=1,name=tom}：4表示Region编号，1表示Raft日志编号；

- Propose：把数据变成Raft日志；
- Append：表示把Raft日志写入到本地的Rocksdb raft实例；
- Committed：

![image-20240126121435400](1-TiDB性能优化.assets/image-20240126121435400.png)

![image-20240126121619184](1-TiDB性能优化.assets/image-20240126121619184.png)



![image-20240126121637795](1-TiDB性能优化.assets/image-20240126121637795.png)





![image-20240126121751473](1-TiDB性能优化.assets/image-20240126121751473.png)



![image-20240126121937522](1-TiDB性能优化.assets/image-20240126121937522.png)

用户的Committed是Apply之后，用户已经可以查询提交的用户的数据，和Raft的Committed不是一个概念，Raft的Committed是指多数节点收到了Raft日志。

![image-20240126122220930](1-TiDB性能优化.assets/image-20240126122220930.png)

Apply是指Raft日志不仅写入到Rocksdb Raft中了，而且已经写入到Rocksdb kv中，已经可以供用户查询提交的数据了，即这个时候才是用户的Committed。

### Raft - Leader选举

![image-20240126122503473](1-TiDB性能优化.assets/image-20240126122503473.png)

- election timeout：长时间没有leader（heartbeat time interval没有收到leader的心跳），没有发起选举，最新发现没有leader的发起选举，term自动加1，其他节点收到发现term编号比自己的term更大，则投票同意该选举。

![image-20240126123257763](1-TiDB性能优化.assets/image-20240126123257763.png)

- election timeout
- random election timeout：随机的，避免多个节点同时发起选举；

![image-20240126123458107](1-TiDB性能优化.assets/image-20240126123458107.png)

raft-base-tick-interval：默认是1s。

![image-20240127135336157](1-TiDB性能优化.assets/image-20240127135336157.png)

## 读取：Coprocessor

### ReadIndex Read

![image-20240127140612684](1-TiDB性能优化.assets/image-20240127140612684.png)

![image-20240127140921021](1-TiDB性能优化.assets/image-20240127140921021.png)

### Lease Read

![image-20240127141834606](1-TiDB性能优化.assets/image-20240127141834606.png)



![image-20240127141922593](1-TiDB性能优化.assets/image-20240127141922593.png)

### Follower Read

![image-20240127142337785](1-TiDB性能优化.assets/image-20240127142337785.png)

#### Leader

![image-20240127142522848](1-TiDB性能优化.assets/image-20240127142522848.png)

#### Follower

![image-20240127142625195](1-TiDB性能优化.assets/image-20240127142625195.png)

### Coprocessor

![image-20240127143240990](1-TiDB性能优化.assets/image-20240127143240990.png)

​	

![image-20240127143423565](1-TiDB性能优化.assets/image-20240127143423565.png)

# PD

![image-20240127143938599](1-TiDB性能优化.assets/image-20240127143938599.png)

## PD（Placement Driver）架构

![image-20240127144002734](1-TiDB性能优化.assets/image-20240127144002734.png)

注意：

- 至少3个节点，一般奇数个节点；
- 全局时钟：
  - 读取时间戳
  - 事务时间戳
  - 提交时间戳

## PD主要功能

![image-20240127144403884](1-TiDB性能优化.assets/image-20240127144403884.png)

### 路由功能

![image-20240127144632677](1-TiDB性能优化.assets/image-20240127144632677.png)

### TSO分配

#### TSO分配-TSO概念

![image-20240127145847691](1-TiDB性能优化.assets/image-20240127145847691.png)

TSO：int64位的整数

physical time: Unix时间戳，精确到1ms;

logical time：1ms 分配 262144个TSO；

#### TSO分配-获取流程

![image-20240127150127171](1-TiDB性能优化.assets/image-20240127150127171.png)

#### TSO分配-时间窗口

![image-20240127150316806](1-TiDB性能优化.assets/image-20240127150316806.png)

- 单个TiDB每3秒向PD申请一次TSO，PD将未来3秒的所有TSO全部给TiDB；避免TiDB频繁请求TSO。



### 调度

#### 调度-总流程

![image-20240127151205831](1-TiDB性能优化.assets/image-20240127151205831.png)

#### 调度-信息收集

![image-20240127151224259](1-TiDB性能优化.assets/image-20240127151224259.png)



#### 调度-生成调度

![image-20240127151334444](1-TiDB性能优化.assets/image-20240127151334444.png)

#### 调度-执行调度

![image-20240127151522895](1-TiDB性能优化.assets/image-20240127151522895.png)



## Label与高可用

![image-20240127151616285](1-TiDB性能优化.assets/image-20240127151616285.png)



### Label的配置

![image-20240127151950310](1-TiDB性能优化.assets/image-20240127151950310.png)





# TiDB系统表使用

## TiDB系统表存储位置

![image-20240116121249439](1-TiDB性能优化.assets/image-20240116121249439.png)

## MySQL数据库

![image-20240116121324682](1-TiDB性能优化.assets/image-20240116121324682.png)

![image-20240116121355742](1-TiDB性能优化.assets/image-20240116121355742.png)

## INFORMATION_SCHEMA数据库

![image-20240116121535998](1-TiDB性能优化.assets/image-20240116121535998.png)



![image-20240116121550541](1-TiDB性能优化.assets/image-20240116121550541.png)



![image-20240116121602810](1-TiDB性能优化.assets/image-20240116121602810.png)

## 运维常用系统表

### 系统慢日志查询

#### 查询某个用户的Top N查询

![image-20240116121623535](1-TiDB性能优化.assets/image-20240116121623535.png)

#### 根据SQL指纹查询同类慢查询

![image-20240116121719903](1-TiDB性能优化.assets/image-20240116121719903.png)



![image-20240116121757564](1-TiDB性能优化.assets/image-20240116121757564.png)

#### 搜索统计信息为pseudo的慢查询SQL

![image-20240116121825045](1-TiDB性能优化.assets/image-20240116121825045.png)

### 系统读写热点表查询

#### 统计当前读写热点表

![image-20240116121840760](1-TiDB性能优化.assets/image-20240116121840760.png)

#### 统计当前读写热点STORE

![image-20240116121910356](1-TiDB性能优化.assets/image-20240116121910356.png)

## SQL阻塞查询

![image-20240116121926623](1-TiDB性能优化.assets/image-20240116121926623.png)



![image-20240116122023665](1-TiDB性能优化.assets/image-20240116122023665.png)



![image-20240116122146925](1-TiDB性能优化.assets/image-20240116122146925.png)



# TiDB数据库HTAP技术

## HTAP技术

![image-20240127152950450](1-TiDB性能优化.assets/image-20240127152950450.png)

## OLTP与OLAP

![image-20240127153048936](1-TiDB性能优化.assets/image-20240127153048936.png)

## HTAP的要求

![image-20240127153206129](1-TiDB性能优化.assets/image-20240127153206129.png)

## TiDB的HATP架构

![image-20240127153407299](1-TiDB性能优化.assets/image-20240127153407299.png)



TiFlash节点以Learner角色接入到Raft Group中，即没有投票权；

## TiDB的HTAP特性

![image-20240127153544675](1-TiDB性能优化.assets/image-20240127153544675.png)

MPP架构：对聚合和连接进行加速；

## MPP

![image-20240127153753402](1-TiDB性能优化.assets/image-20240127153753402.png)



## 混合负载场景（Mixed Workload Scenarios）

![image-20240127154456186](1-TiDB性能优化.assets/image-20240127154456186.png)

## 流式计算场景（Streaming Scenarios）

![image-20240127154538851](1-TiDB性能优化.assets/image-20240127154538851.png)



![image-20240127154656795](1-TiDB性能优化.assets/image-20240127154656795.png)



# TiFlash

## TiFlash架构

![image-20240127155025708](1-TiDB性能优化.assets/image-20240127155025708.png)

## TiFlash核心特性

![image-20240127155217515](1-TiDB性能优化.assets/image-20240127155217515.png)

### TiFlash的核心特性-异步复制

![image-20240127155408874](1-TiDB性能优化.assets/image-20240127155408874.png)



### TiFlash核心特性-一致性读取 *

![image-20240127155509077](1-TiDB性能优化.assets/image-20240127155509077.png)



![image-20240127155735687](1-TiDB性能优化.assets/image-20240127155735687.png)



![image-20240127155845678](1-TiDB性能优化.assets/image-20240127155845678.png)



![image-20240127155905237](1-TiDB性能优化.assets/image-20240127155905237.png)



![image-20240127155955086](1-TiDB性能优化.assets/image-20240127155955086.png)



![image-20240127161152266](1-TiDB性能优化.assets/image-20240127161152266.png)



![image-20240127160049620](1-TiDB性能优化.assets/image-20240127160049620.png)





![image-20240127160136190](1-TiDB性能优化.assets/image-20240127160136190.png)

### TiFlash核心特性-智能选择

![image-20240127160214590](1-TiDB性能优化.assets/image-20240127160214590.png)



## TiFlash部署

![image-20240127160503936](1-TiDB性能优化.assets/image-20240127160503936.png)

## TiFlash使用

![image-20240127160548138](1-TiDB性能优化.assets/image-20240127160548138.png)



# TiDB数据库事务设计

## 事务定义

![image-20240127164444971](1-TiDB性能优化.assets/image-20240127164444971.png)

### 隔离级别

![image-20240127164740537](1-TiDB性能优化.assets/image-20240127164740537.png)



![image-20240127164855736](1-TiDB性能优化.assets/image-20240127164855736.png)

不可重复读：在事务开始的时候读到的数据和事务过程中读到的数据不一致；

#### 未提交读

![image-20240127165209927](1-TiDB性能优化.assets/image-20240127165209927.png)

#### 提交读

![image-20240127165340562](1-TiDB性能优化.assets/image-20240127165340562.png)

#### 可重复读

![image-20240127165409770](1-TiDB性能优化.assets/image-20240127165409770.png)

#### 幻读

![image-20240127165516399](1-TiDB性能优化.assets/image-20240127165516399.png)

#### 隔离级别与现象

![image-20240127165640296](1-TiDB性能优化.assets/image-20240127165640296.png)

## 事务在分布式系统的挑战

![image-20240127165810394](1-TiDB性能优化.assets/image-20240127165810394.png)

### TCC（Try Confirm/Commit Cancel）

![image-20240127170029046](1-TiDB性能优化.assets/image-20240127170029046.png)



![image-20240127170246831](1-TiDB性能优化.assets/image-20240127170246831.png)

问题：完全由应用程序实现控制，对业务侵入比较大；

### SAGA

![image-20240127170346250](1-TiDB性能优化.assets/image-20240127170346250.png)



![image-20240127170510225](1-TiDB性能优化.assets/image-20240127170510225.png)

将业务的大事务拆分为一个个的小事务，需要为每个事务做一个回退方案；

一旦业务某个步骤失败，需要将属于业务的前面成功的小事务也回退；

对业务代码的侵入不大，但是需要为每个小事务准备回退方案；

### 2PC（两阶段提交）

![image-20240127170742442](1-TiDB性能优化.assets/image-20240127170742442.png)

每个Prepare和Commit阶段都是由分布式数据库去做，对业务代码0侵入；

Percolator模型：2PC的衍生版本，TiDB采用的分布式事务模型；

## Percolater模型

![image-20240128152518221](1-TiDB性能优化.assets/image-20240128152518221.png)

### Percolator背景

![image-20240128152421046](1-TiDB性能优化.assets/image-20240128152421046.png)

### 快照隔离级别

![image-20240128152742680](1-TiDB性能优化.assets/image-20240128152742680.png)

### 分布式时钟

![image-20240128153100222](1-TiDB性能优化.assets/image-20240128153100222.png)

### Percolator事务执行流程

![image-20240128153427819](1-TiDB性能优化.assets/image-20240128153427819.png)



![image-20240128153616664](1-TiDB性能优化.assets/image-20240128153616664.png)



![image-20240128153654783](1-TiDB性能优化.assets/image-20240128153654783.png)



![image-20240128153756429](1-TiDB性能优化.assets/image-20240128153756429.png)

- 先给主行加锁，再从行的Lock CF中添加一个指向主行锁的指针；（即实际只有主行有锁，这个过程称为Prewrite）



![image-20240128154033281](1-TiDB性能优化.assets/image-20240128154033281.png)

### Percolator案例

![image-20240128154107139](1-TiDB性能优化.assets/image-20240128154107139.png)



![image-20240128154333233](1-TiDB性能优化.assets/image-20240128154333233.png)



![image-20240128154516905](1-TiDB性能优化.assets/image-20240128154516905.png)



![image-20240128154557659](1-TiDB性能优化.assets/image-20240128154557659.png)



![image-20240128154810347](1-TiDB性能优化.assets/image-20240128154810347.png)

## TiDB事务的实现和优化

![image-20240128155714201](1-TiDB性能优化.assets/image-20240128155714201.png)

### 事务在TiDB中的存储

![image-20240128155759449](1-TiDB性能优化.assets/image-20240128155759449.png)



两阶段提交：

- Prewrite：先获取事务开始时间戳，并添加锁信息和内存中修改数据；
  - 锁信息写入到Lock CF;
  - 修改的数据写入Default CF；（注意：如果该行数据小于255字节的话，该行数据是写入到Write CF，这里统一假设事务数据大于255字节，所以默认写入到Default CF中）
- Commit：
  - 先获取事务提交时间戳；
  - 将事务提交信息写入Write CF；
    - key：业务id_事务提交时间戳；
    - value：事务开始时间戳；
  - 删除锁信息；

### 事务执行过程

#### 1、发起事务

![image-20240128161525321](1-TiDB性能优化.assets/image-20240128161525321.png)

事务只有在提交之后才有持久化特性；

#### 2、Prewrite版本检查

#### 事务执行过程（二）-Prewrite版本检查

![image-20240128161826565](1-TiDB性能优化.assets/image-20240128161826565.png)

#### 3、Prewrite锁冲突检测

![image-20240128162232605](1-TiDB性能优化.assets/image-20240128162232605.png)

#### 4、Prewrite加锁

![image-20240128162618934](1-TiDB性能优化.assets/image-20240128162618934.png)

#### 5、commit锁检查与提交

![image-20240128211829956](1-TiDB性能优化.assets/image-20240128211829956.png)



#### 6-commit 锁检查与提交

![image-20240128211220796](1-TiDB性能优化.assets/image-20240128211220796.png)

乐观锁：在Prewrite阶段加锁；

悲观锁：在Prewrite之前DML就加锁；

### TiDB悲观锁



![image-20240128211930725](1-TiDB性能优化.assets/image-20240128211930725.png)



![image-20240128212124330](1-TiDB性能优化.assets/image-20240128212124330.png)



![image-20240128212206402](1-TiDB性能优化.assets/image-20240128212206402.png)

DML上锁：<3,(W, pk, 00000000)>  ，因为目前还没有到提交阶段，所以提交时间戳设置的是占位符，即00000000；

![image-20240128212235631](1-TiDB性能优化.assets/image-20240128212235631.png)

Prewrite：<3,(W, pk, 12000000)>  ，这里的是提交时间戳了，即12000000；

![image-20240128212521679](1-TiDB性能优化.assets/image-20240128212521679.png)



![image-20240128212604116](1-TiDB性能优化.assets/image-20240128212604116.png)



![image-20240128214058397](1-TiDB性能优化.assets/image-20240128214058397.png)

## 事务的去中心化

![image-20240128214235993](1-TiDB性能优化.assets/image-20240128214235993.png)

## TiDB数据库的锁

![image-20240129082534920](1-TiDB性能优化.assets/image-20240129082534920.png)

### 乐观锁模式现象

![image-20240129084431647](1-TiDB性能优化.assets/image-20240129084431647.png)



![image-20240129084537717](1-TiDB性能优化.assets/image-20240129084537717.png)



![image-20240129084623374](1-TiDB性能优化.assets/image-20240129084623374.png)

- 乐观锁模式：在事务提交的时候才检测锁冲突；

![image-20240129084652327](1-TiDB性能优化.assets/image-20240129084652327.png)

### 悲观锁模式现象

![image-20240129084713248](1-TiDB性能优化.assets/image-20240129084713248.png)

悲观锁模式：在事务提交前就检测锁冲突；

- 修改行的时候做锁冲突检测，这个时候Session2中的delete操作被阻塞了。

- 即在DML阶段就检测锁冲突了，而不是在两阶段提交的Prewrite阶段才检测锁冲突（和乐观锁模式的重要区别）；

![image-20240129084751298](1-TiDB性能优化.assets/image-20240129084751298.png)

Session1提交之后，锁就释放了，Session2的delete操作不再被阻塞，执行成功；

![image-20240129084911037](1-TiDB性能优化.assets/image-20240129084911037.png)

## 写偏斜

- 写偏斜和隔离级别有关系，但是不涉及悲观事务模式或乐观模式，因为写偏斜修改的是不同的行，不涉及锁问题；
- 提交读隔离级别，出现写偏斜是没有问题的，因为其本身就只能读到已经提交到的数据，因为其不可重复读；
- 写偏斜一般出现在可重复读隔离级别或者快照读；
  - 事务执行过程中读取的数据和事务开始时的数据一致（即可重复读），这也是出现写偏斜原因的关键，所以可以通过查询时加for update实现当前读（即读取当前最新的数据，而不是事务开始时的数据），从而避免写偏斜。
- for update是当前读，即读取当前最新的数据；
- 悲观锁模式遇到锁冲突，默认会阻塞（有些数据库可能是回滚）。

![image-20240129082831707](1-TiDB性能优化.assets/image-20240129082831707.png)



![image-20240129085202279](1-TiDB性能优化.assets/image-20240129085202279.png)



![image-20240129085239149](1-TiDB性能优化.assets/image-20240129085239149.png)



![image-20240129085255461](1-TiDB性能优化.assets/image-20240129085255461.png)



![image-20240129085439573](1-TiDB性能优化.assets/image-20240129085439573.png)



![image-20240129085335441](1-TiDB性能优化.assets/image-20240129085335441.png)



**避免写倾斜的方法**：**查询时使用for update，实现当前读，即读取当前最新的数据，而不是读取事务开始时的数据**；

![image-20240129085635459](1-TiDB性能优化.assets/image-20240129085635459.png)



# TiDB数据库索引设计

## Schema的KV映射原理

![image-20240107172424818](1-TiDB性能优化.assets/image-20240107172424818.png)

### 唯一索引&非聚簇表的主键

![image-20240107172503670](1-TiDB性能优化.assets/image-20240107172503670.png)

### 二级索引

![image-20240107172542218](1-TiDB性能优化.assets/image-20240107172542218.png)

### 索引实例

![image-20240107173850083](1-TiDB性能优化.assets/image-20240107173850083.png)

### 索引的设计

- 在Percolate模型中一个事务的写入可能会一个Primary Key和多个Secondary Keys；
- TiDB在创建索引时不会阻塞的数据读写；

## 联合索引



![image-20240107170527524](1-TiDB性能优化.assets/image-20240107170527524.png)



![image-20240107170554290](1-TiDB性能优化.assets/image-20240107170554290.png)



## 表达式索引



![image-20240107170705456](1-TiDB性能优化.assets/image-20240107170705456.png)



![image-20240107170815872](1-TiDB性能优化.assets/image-20240107170815872.png)

## 不可见索引

![image-20240107170855455](1-TiDB性能优化.assets/image-20240107170855455.png)

## MySQL兼容性

![image-20240107171026314](1-TiDB性能优化.assets/image-20240107171026314.png)

## 查看索引的Region分布

![image-20240107171213972](1-TiDB性能优化.assets/image-20240107171213972.png)



# 基于索引的SQL优化

## TiDB 增加索引的原理

![image-20240107175643972](1-TiDB性能优化.assets/image-20240107175643972.png)

## 动态调整创建索引的速度

![image-20240107175851356](1-TiDB性能优化.assets/image-20240107175851356.png)



![image-20240107180051326](1-TiDB性能优化.assets/image-20240107180051326.png)

## 测试增加索引对线上业务的影响

### 目标列上存在频繁读写的场景

![image-20240107180214937](1-TiDB性能优化.assets/image-20240107180214937.png)

![image-20240107180439375](1-TiDB性能优化.assets/image-20240107180439375.png)

![image-20240107180550362](1-TiDB性能优化.assets/image-20240107180550362.png)

### 目标列只读场景

![image-20240107180617047](1-TiDB性能优化.assets/image-20240107180617047.png)

![image-20240107180705204](1-TiDB性能优化.assets/image-20240107180705204.png)

### 目标列不涉及读写的场景

![image-20240107180750661](1-TiDB性能优化.assets/image-20240107180750661.png)

![image-20240107180818329](1-TiDB性能优化.assets/image-20240107180818329.png)

### 总结

![image-20240107180836223](1-TiDB性能优化.assets/image-20240107180836223.png)

## Point Get与Batch Point Get

![image-20240107180946009](1-TiDB性能优化.assets/image-20240107180946009.png)

![image-20240107181141283](1-TiDB性能优化.assets/image-20240107181141283.png)

![image-20240107183143214](1-TiDB性能优化.assets/image-20240107183143214.png)

## Index Full Scan

![image-20240107181209992](1-TiDB性能优化.assets/image-20240107181209992.png)



![image-20240107181319977](1-TiDB性能优化.assets/image-20240107181319977.png)

## Index Range Scan

![image-20240107181559788](1-TiDB性能优化.assets/image-20240107181559788.png)



![image-20240107181700063](1-TiDB性能优化.assets/image-20240107181700063.png)

## 索引选择的维度

![image-20240107181738683](1-TiDB性能优化.assets/image-20240107181738683.png)



![image-20240107181900321](1-TiDB性能优化.assets/image-20240107181900321.png)



![image-20240107182109645](1-TiDB性能优化.assets/image-20240107182109645.png)



# TiDB SQL优化

## explain format='cost_trace'

参考：https://asktug.com/t/topic/1021096/20

> explain format=‘cost_trace’ 这个看执行计划有cost计算公式

这个问题有了一些进展，权当抛砖引玉好了。

我找了一张大表，大致有3.5亿数据，analyze之后，根据时间片查询的结果如下：
index scan:

[![index scan](1-TiDB性能优化.assets/b710909073feb46a1892fea95165a6b99989cf72_2_690x46.jpeg)index scan3805×258 169 KB](https://asktug.com/uploads/default/original/4X/b/7/1/b710909073feb46a1892fea95165a6b99989cf72.jpeg)


table scan:

[![tableScan](1-TiDB性能优化.assets/1a9fb2f9bab4dd742776c287d01f080d157fae8d_2_690x55.png)tableScan3639×294 125 KB](https://asktug.com/uploads/default/original/4X/1/a/9/1a9fb2f9bab4dd742776c287d01f080d157fae8d.png)



具体的公式格式化结果后，index scan:

[![11](1-TiDB性能优化.assets/7bab77ee8eaf8f0cb7ac9bfa88e7d12ed55f9789.png)11858×517 15.4 KB](https://asktug.com/uploads/default/original/4X/7/b/a/7bab77ee8eaf8f0cb7ac9bfa88e7d12ed55f9789.png)


table scan:

[![22](1-TiDB性能优化.assets/03de99b8b5c4961811a29460864a43e3ec2c75d6.png)22943×205 10.3 KB](https://asktug.com/uploads/default/original/4X/0/3/d/03de99b8b5c4961811a29460864a43e3ec2c75d6.png)



这个看着还是有点乱。让我们把重点突出一下。
在index scan的这个公式里面，起决定性作用的是：

> (doubleRead(tasks(36.685874119007586)*tidb_request_factor(6e+06)))
>
> )/tidb_executor_concurrency=5.00

因为tidb_request_factor的代价设置非常高，一次就是6000000.
https://github.com/pingcap/tidb/blob/master/pkg/planner/core/plan_cost_ver2.go#L940

所以当index scan发生的时候，其cost基本可以简化为上面这个式子，其他的影响权重非常小。
task的数量的公式为
https://github.com/pingcap/tidb/blob/master/pkg/planner/core/plan_cost_ver2.go#L275

> batchSize := float64(p.SCtx().GetSessionVars().IndexLookupSize)
> taskPerBatch := 32.0 // TODO: remove this magic number
> doubleReadTasks := doubleReadRows / batchSize * taskPerBatch

IndexLookupSize 可以通过`show variables like 'tidb_index_lookup_size'`查看，我这边设置为20000。
所以tasks=预估读取行数（22928）/tidb_index_lookup_size（20000）*taskPerBatch（32）=36.68左右。

因此，现在如果我们希望index scan的代价变低，比较有效的做法有两个，提高tidb_index_lookup_size和tidb_executor_concurrency的值。

同理，我们看看table scan的代价计算方式，主要的部分是：

> (
> (cpu(3.5718153e+07(总记录数)*filters(1)*tikv_cpu_factor(49.9))) +
> (scan(3.5718153e+07(总记录数)*logrowsize(312)*tikv_scan_factor(40.7)))
> )/(tidb_distsql_scan_concurrency)15.00

net的部分cost影响权重很小，可以直接忽略，cost值主要的影响因素在于全表记录大小和tidb_distsql_scan_concurrency的值的大小。
全表记录大小是没有办法调整的，为了降低table scan 的cost，那就只剩下调大tidb_distsql_scan_concurrency的值，这一个办法。

回到标题的问题。这个比例是不太好算的，但可以大致推算，当
（index scan的预估记录数/tidb_index_lookup_size*32*tidb_request_factor）/tidb_executor_concurrency小于
表的总记录数* logrowsize*tikv_scan_factor/tidb_distsql_scan_concurrency

大体就会选择index scan而非table scan。同时如果为了让优化器尽可能的选择index scan还可以通过2种参数设置来做到：
1，提高tidb_index_lookup_size的值
2，提高tidb_executor_concurrency的值



## 大量DML操作导致OOM

![image-20240107211905043](1-TiDB性能优化.assets/image-20240107211905043.png)



![image-20240107211918204](1-TiDB性能优化.assets/image-20240107211918204.png)



![image-20240107211959806](1-TiDB性能优化.assets/image-20240107211959806.png)



## 执行计划不稳定导致查询延迟增加

![image-20240107212103283](1-TiDB性能优化.assets/image-20240107212103283.png)



![image-20240107212228116](1-TiDB性能优化.assets/image-20240107212228116.png)



![image-20240107212241660](1-TiDB性能优化.assets/image-20240107212241660.png)

## SQL执行计划不准

![image-20240107212339102](1-TiDB性能优化.assets/image-20240107212339102.png)



![image-20240107215001035](1-TiDB性能优化.assets/image-20240107215001035.png)



![image-20240107215152823](1-TiDB性能优化.assets/image-20240107215152823.png)



![image-20240107215247401](1-TiDB性能优化.assets/image-20240107215247401.png)



![image-20240107215305048](1-TiDB性能优化.assets/image-20240107215305048.png)



![image-20240107215350182](1-TiDB性能优化.assets/image-20240107215350182.png)



![image-20240107215411706](1-TiDB性能优化.assets/image-20240107215411706.png)



![image-20240107215436050](1-TiDB性能优化.assets/image-20240107215436050.png)



# 查询优化案例选

## SQL优化原则

- SQL优化是一个系统工程，

![image-20240107215730471](1-TiDB性能优化.assets/image-20240107215730471.png)



![image-20240107215849091](1-TiDB性能优化.assets/image-20240107215849091.png)



![image-20240107220018469](1-TiDB性能优化.assets/image-20240107220018469.png)

> 注意：like也是范围查询！！！

![image-20240107220238442](1-TiDB性能优化.assets/image-20240107220238442.png)

> 注意：多表Join关联字段需要有索引，特别是被驱动表的关联字段！！！

![image-20240107220502242](1-TiDB性能优化.assets/image-20240107220502242.png)



![image-20240107220516891](1-TiDB性能优化.assets/image-20240107220516891.png)

## 隐式转换案例

![image-20240107220624072](1-TiDB性能优化.assets/image-20240107220624072.png)



![image-20240107220721744](1-TiDB性能优化.assets/image-20240107220721744.png)



![image-20240107220815720](1-TiDB性能优化.assets/image-20240107220815720.png)



![image-20240107220907593](1-TiDB性能优化.assets/image-20240107220907593.png)



![image-20240107221029114](1-TiDB性能优化.assets/image-20240107221029114.png)

![image-20240107221112511](1-TiDB性能优化.assets/image-20240107221112511.png)

![image-20240107221143780](1-TiDB性能优化.assets/image-20240107221143780.png)

## 深度分页

### 深度分页案例 

![image-20240108083846886](1-TiDB性能优化.assets/image-20240108083846886.png)



![image-20240108084000210](1-TiDB性能优化.assets/image-20240108084000210.png)



![image-20240108084013580](1-TiDB性能优化.assets/image-20240108084013580.png)



![image-20240108084040387](1-TiDB性能优化.assets/image-20240108084040387.png)



![image-20240108084200920](1-TiDB性能优化.assets/image-20240108084200920.png)

![image-20240108084410009](1-TiDB性能优化.assets/image-20240108084410009.png)



![image-20240108084349298](1-TiDB性能优化.assets/image-20240108084349298.png)



![image-20240108084501861](1-TiDB性能优化.assets/image-20240108084501861.png)



![image-20240108084720028](1-TiDB性能优化.assets/image-20240108084720028.png)



![image-20240108085010187](1-TiDB性能优化.assets/image-20240108085010187.png)

## 聚合操作

### 表中数据不经常修改

在计算节点TiDB上开启copr-cache（TiDB 4.0版本引入，默认开启）

![image-20240108224803858](1-TiDB性能优化.assets/image-20240108224803858.png)

### 表中数据经常修改案例

![image-20240108223357016](1-TiDB性能优化.assets/image-20240108223357016.png)

![image-20240108223711354](1-TiDB性能优化.assets/image-20240108223711354.png)



![image-20240108223802367](1-TiDB性能优化.assets/image-20240108223802367.png)



![image-20240108223817245](1-TiDB性能优化.assets/image-20240108223817245.png)



![image-20240108223842408](1-TiDB性能优化.assets/image-20240108223842408.png)



![image-20240108223857025](1-TiDB性能优化.assets/image-20240108223857025.png)

- 先插入一行，再写入redis之间：如果这时用户查询redis和TIDB数据库，会发现在这个时间差内两者结果会不一致；

![image-20240108223910147](1-TiDB性能优化.assets/image-20240108223910147.png)

- 先写入redis，再插入一行之间：如果这时用户查询redis和TIDB数据库，会发现在这个时间差内两者结果也会不一致；

![image-20240108224006803](1-TiDB性能优化.assets/image-20240108224006803.png)



**如何解决这种不一致问题**：不使用redis，直接通过数据库事务去解决，保证插入（insert）和统计（count）都在同一个事务里；

![image-20240108224030064](1-TiDB性能优化.assets/image-20240108224030064.png)

### 总结

![image-20240108224642420](1-TiDB性能优化.assets/image-20240108224642420.png)

## 多表连接

![image-20240108224924775](1-TiDB性能优化.assets/image-20240108224924775.png)



### Hash Join

![image-20240108225315147](1-TiDB性能优化.assets/image-20240108225315147.png)



![image-20240108225357495](1-TiDB性能优化.assets/image-20240108225357495.png)



![image-20240108225414570](1-TiDB性能优化.assets/image-20240108225414570.png)



![image-20240108225457218](1-TiDB性能优化.assets/image-20240108225457218.png)



![image-20240108225603585](1-TiDB性能优化.assets/image-20240108225603585.png)



### Merge Join

![image-20240108225636503](1-TiDB性能优化.assets/image-20240108225636503.png)



![image-20240108225653580](1-TiDB性能优化.assets/image-20240108225653580.png)



![image-20240108225736803](1-TiDB性能优化.assets/image-20240108225736803.png)



![image-20240108225750131](1-TiDB性能优化.assets/image-20240108225750131.png)

### INDEX JOIN

![image-20240108233006039](1-TiDB性能优化.assets/image-20240108233006039.png)



![image-20240108233018842](1-TiDB性能优化.assets/image-20240108233018842.png)



![image-20240108233035849](1-TiDB性能优化.assets/image-20240108233035849.png)



![image-20240108233047679](1-TiDB性能优化.assets/image-20240108233047679.png)

![image-20240108233108027](1-TiDB性能优化.assets/image-20240108233108027.png)



![image-20240108233210150](1-TiDB性能优化.assets/image-20240108233210150.png)

![image-20240108233226260](1-TiDB性能优化.assets/image-20240108233226260.png)



![image-20240108233240836](1-TiDB性能优化.assets/image-20240108233240836.png)

多表连接总结

- 小表驱动大表
- 关联字段需要有索引
  - 被驱动表的关联列上要有索引

# TiDB优化器原理

## TiDB优化器架构

![image-20240116084032026](1-TiDB性能优化.assets/image-20240116084032026.png)

## 预处理（pre process）阶段概述

![image-20240116084524614](1-TiDB性能优化.assets/image-20240116084524614.png)

### 对点查的优化

![image-20240116084556737](1-TiDB性能优化.assets/image-20240116084556737.png)

### 构造初始逻辑执行计划

![image-20240116084742053](1-TiDB性能优化.assets/image-20240116084742053.png)

## 逻辑优化

![image-20240116084902517](1-TiDB性能优化.assets/image-20240116084902517.png)

### 列剪裁

![image-20240116084932404](1-TiDB性能优化.assets/image-20240116084932404.png)

### 谓词下推

![image-20240116084954287](1-TiDB性能优化.assets/image-20240116084954287.png)

![image-20240116085206147](1-TiDB性能优化.assets/image-20240116085206147.png)

### 连接顺序调整

![image-20240116085341164](1-TiDB性能优化.assets/image-20240116085341164.png)

## 物理优化

### 物理优化的维度

![image-20240116085548405](1-TiDB性能优化.assets/image-20240116085548405.png)

### 物理优化的决策

![image-20240116085637311](1-TiDB性能优化.assets/image-20240116085637311.png)

### 物理优化的索引选择

![image-20240116085728500](1-TiDB性能优化.assets/image-20240116085728500.png)





# 理解执行计划

## EXPLAIN

![image-20240109084625164](1-TiDB性能优化.assets/image-20240109084625164.png)

## EXPLAIN ANALYZE

![image-20240109084657813](1-TiDB性能优化.assets/image-20240109084657813.png)

> 注意：
>
> 查看执行计划的方法：**由上至下，从右向左**。

## 执行计划中的算子

### 汇聚数据类算子

![image-20240109085431810](1-TiDB性能优化.assets/image-20240109085431810.png)

#### Hash Aggregate

![image-20240109085509375](1-TiDB性能优化.assets/image-20240109085509375.png)



![image-20240109085719101](1-TiDB性能优化.assets/image-20240109085719101.png)



![image-20240109090231233](1-TiDB性能优化.assets/image-20240109090231233.png)



![image-20240109090245735](1-TiDB性能优化.assets/image-20240109090245735.png)

#### Stream Aggregate

![image-20240109090323504](1-TiDB性能优化.assets/image-20240109090323504.png)

## 表关联算子

![image-20240110084641368](1-TiDB性能优化.assets/image-20240110084641368.png)

![image-20240110084730321](1-TiDB性能优化.assets/image-20240110084730321.png)



![image-20240110084800108](1-TiDB性能优化.assets/image-20240110084800108.png)

### Merge Join 

![image-20240110084827470](1-TiDB性能优化.assets/image-20240110084827470.png)

![image-20240110084921861](1-TiDB性能优化.assets/image-20240110084921861.png)



![image-20240110084934645](1-TiDB性能优化.assets/image-20240110084934645.png)



### Index Join

![image-20240110084958344](1-TiDB性能优化.assets/image-20240110084958344.png)



![image-20240110085218972](1-TiDB性能优化.assets/image-20240110085218972.png)

![image-20240110085256275](1-TiDB性能优化.assets/image-20240110085256275.png)



![image-20240110085310950](1-TiDB性能优化.assets/image-20240110085310950.png)

优化：

- 增加并行度（将表分成多个数据块，每个块用一个线程处理）
- 批量处理（一次处理多条数据）

## 管理执行计划

![image-20240110085531331](1-TiDB性能优化.assets/image-20240110085531331.png)



![image-20240110085636931](1-TiDB性能优化.assets/image-20240110085636931.png)

![image-20240110085819723](1-TiDB性能优化.assets/image-20240110085819723.png)

# 统计信息收集

## 统计信息简介

![image-20240110090026758](1-TiDB性能优化.assets/image-20240110090026758.png)



## 统计信息的基本组成

![image-20240110090102837](1-TiDB性能优化.assets/image-20240110090102837.png)

## 直方图

**直方图**：是指对某列的分布情况进行描述，通过直方图优化器可以知道某个具体列值在这个列的数据分布情况。

TiDB使用的是等深直方图，而不是等宽直方图。

![image-20240110090157716](1-TiDB性能优化.assets/image-20240110090157716.png)

> 等深直方图：深度一样或者高度一样
>
> 纵坐标：值的个数
>
> 横坐标：值的区间
>
> 以上图为例：值的个数都是3个，1.6到1.9、2.0到2.6、2.7到2.8、2.9到3.5这几个区间内都有3个值，显然2.0到2.6的区间最宽，即数据分布最不密集，而2.7到2.8的区间最窄，即数据分布最密集。很好理解，2.0到2.6的区间大小是0.6，但是其有3个值，而2.7到2.8的区间大小为0.1，但是其也有3个值，所以很明显可以看出哪个区间更加密集。

## Count-Min Sketch

![image-20240110121806400](1-TiDB性能优化.assets/image-20240110121806400.png)

![image-20240110121956722](1-TiDB性能优化.assets/image-20240110121956722.png)

## 统计信息收集方法

![image-20240110122028144](1-TiDB性能优化.assets/image-20240110122028144.png)

> CMSKETCH DEPTH（深度）和CMSKETCH WIDTH（宽度），两者越大则发生hash碰撞的几率更小；
>
> SAMPLES：抽样比例

## 控制ANALYZE的并发度

![image-20240110122253905](1-TiDB性能优化.assets/image-20240110122253905.png)



![image-20240117090856954](1-TiDB性能优化.assets/image-20240117090856954.png)



## 自动更新统计信息

![image-20240110122356399](1-TiDB性能优化.assets/image-20240110122356399.png)

> 默认60s更新一次；



## 导入导出统计信息

![image-20240117091000537](1-TiDB性能优化.assets/image-20240117091000537.png)



# TiDB Server关键性能参数和优化



# PD关键性能参数和优化

## PD 调度基本概念

![image-20240122122820583](1-TiDB性能优化.assets/image-20240122122820583.png)

![image-20240122123008598](1-TiDB性能优化.assets/image-20240122123008598.png)



## 调度流程                

![image-20240122123344716](1-TiDB性能优化.assets/image-20240122123344716.png)

## 调度limit参数

![image-20240122123608656](1-TiDB性能优化.assets/image-20240122123608656.png)



![image-20240122124032658](1-TiDB性能优化.assets/image-20240122124032658.png)

## 存储空间阈值参数

![image-20240122124148736](1-TiDB性能优化.assets/image-20240122124148736.png)

注意：**在high space里面不考虑剩余空间，在low space里面需要着重考虑剩余空间。**

## 其他参数

![image-20240122221346838](1-TiDB性能优化.assets/image-20240122221346838.png)

## pd-ctl基本操作

![image-20240122221656805](1-TiDB性能优化.assets/image-20240122221656805.png)

## 常见问题的处理

### 扩容后balance region调度速度慢

![image-20240122221823354](1-TiDB性能优化.assets/image-20240122221823354.png)

### Store节点故障后补副本的速度慢

![image-20240122222157024](1-TiDB性能优化.assets/image-20240122222157024.png)

### Region merge速度慢

![image-20240122222605640](1-TiDB性能优化.assets/image-20240122222605640.png)

# TiKV关键性能参数及优化

## TiKV架构

![image-20240123084309542](1-TiDB性能优化.assets/image-20240123084309542.png)

## TiKV主要模块和线程

![image-20240123084429423](1-TiDB性能优化.assets/image-20240123084429423.png)

## TiKV数据写入流程

![image-20240123084740383](1-TiDB性能优化.assets/image-20240123084740383.png)

scheduler pool：协调事务请求，处理事务并发，冲突时获得latch的才继续往下走，其他未获取latch则先等待；

scheduler pool 》raftstore pool 》apply pool

## 写入瓶颈排查

![image-20240123085112579](1-TiDB性能优化.assets/image-20240123085112579.png)

## TiKV写入参数优化

![image-20240123085227105](1-TiDB性能优化.assets/image-20240123085227105.png)

## TiKV数据读取流程

### 点查

![image-20240123085953283](1-TiDB性能优化.assets/image-20240123085953283.png)

### 非点查

![image-20240123090407186](1-TiDB性能优化.assets/image-20240123090407186.png)

这里的handler不再是KV查询，而是table scan、table range scan等表扫描

## 读取瓶颈分析

![image-20240123090558899](1-TiDB性能优化.assets/image-20240123090558899.png)

## 读取参数优化

![image-20240123090623265](1-TiDB性能优化.assets/image-20240123090623265.png)

storage.block-cache.capacity：建议调到TiKV内存的40%—60%之间。

split.qps-threshold和split.byte-threshold是流量打散的控制参数，当达到这两个参数的阈值时，会出现自动进行Region的分裂，实现流量的打散。

## 常见问题处理

### Write Stall

![image-20240123091255056](1-TiDB性能优化.assets/image-20240123091255056.png)



![image-20240123215032823](1-TiDB性能优化.assets/image-20240123215032823.png)

### TiKV Slow Query

![image-20240123215710833](1-TiDB性能优化.assets/image-20240123215710833.png)

# 硬件和操作系统优化

## 生产环境的推荐配置

![image-20240123220804697](1-TiDB性能优化.assets/image-20240123220804697.png)

![image-20240123221315325](1-TiDB性能优化.assets/image-20240123221315325.png)

## 服务组件的混合部署

![image-20240123221526494](1-TiDB性能优化.assets/image-20240123221526494.png)



## 操作系统版本要求

注意：目前的基准TiDB版本是V5.4.0

![image-20240124083159011](1-TiDB性能优化.assets/image-20240124083159011.png)



![image-20240124083419311](1-TiDB性能优化.assets/image-20240124083419311.png)

### 操作系统配置

![image-20240124083446316](1-TiDB性能优化.assets/image-20240124083446316.png)



![image-20240124083530409](1-TiDB性能优化.assets/image-20240124083530409.png)

**irqbalance：将中断均匀地分配到多个CPU上；**

![image-20240124083818523](1-TiDB性能优化.assets/image-20240124083818523.png)

- 倾向不使用虚拟内存（swap），即关闭swap；
  - vm.swappiness: 0
    - 先使用物理内存，物理内存使用100%了，再考虑使用swap虚拟内存；
- vm.overcommi_memory：一般0或者1；
  - 0：申请内存超过限制报错；
  - 1：申请内存超过限制不报错，申请多少给多少，也不管够不够，易OOM；
  - 2：申请超过限制（已有的），就不给；

![image-20240124084516035](1-TiDB性能优化.assets/image-20240124084516035.png)

![image-20240124084145687](1-TiDB性能优化.assets/image-20240124084145687.png)

## TiDB快速环境检查方法

### CLUSTER CHECK介绍

![image-20240124084924502](1-TiDB性能优化.assets/image-20240124084924502.png)

### 检测硬件基础要求

![image-20240124085014055](1-TiDB性能优化.assets/image-20240124085014055.png)

### 自动修复

![image-20240124085041815](1-TiDB性能优化.assets/image-20240124085041815.png)



# TiDB集群常用监控指标

## TiDB Server相关监控

### Duration相关

#### 整体延迟展示 

![image-20240114113453290](1-TiDB性能优化.assets/image-20240114113453290.png)

#### 按不同SQL类型展示

![image-20240114113605407](1-TiDB性能优化.assets/image-20240114113605407.png)

#### 按Instance维度展示

![image-20240114113653555](1-TiDB性能优化.assets/image-20240114113653555.png)

### QPS相关

QPS：每秒的查询次数（Query Per Second），宏观上反映数据库的负载高低和请求类型;

#### 按SQL类型展示

![image-20240114113824002](1-TiDB性能优化.assets/image-20240114113824002.png)

#### 按Instance维度展示CPS（Command Per Second）

CPS（Command Per Second）：每秒发生的command数（请求书），注意：一次发的请求，可能包含多个query；

```sql
-- 一次性请求发出的语句：这里相当于一个command，但是属于两个query；
select 1;select 1;
```

![image-20240114113934100](1-TiDB性能优化.assets/image-20240114113934100.png)



![image-20240114114609710](1-TiDB性能优化.assets/image-20240114114609710.png)

### Transaction相关

分为两种：乐观事务和悲观事务；（tidb_txn_mode）

![image-20240114114652099](1-TiDB性能优化.assets/image-20240114114652099.png)

- 事务内的语句量：如果事务内语句量比较大，说明有可能有大事务；

![image-20240114114759846](1-TiDB性能优化.assets/image-20240114114759846.png)

- 事务重试次数：重点在乐观事务场景下监控，悲观事务不用监控；

![image-20240114114934964](1-TiDB性能优化.assets/image-20240114114934964.png)



## 资源相关

![image-20240114115038199](1-TiDB性能优化.assets/image-20240114115038199.png)



![image-20240114115103280](1-TiDB性能优化.assets/image-20240114115103280.png)



- 默认TiDB是不限制客户端的连接数，限制连接数的参数：max_server_connection

![image-20240114115134779](1-TiDB性能优化.assets/image-20240114115134779.png)



- token_limit：拿到token才连接，达到token_limit，后面的连接需要等待，这里可以看出连接等待

![image-20240114115308525](1-TiDB性能优化.assets/image-20240114115308525.png)

### PD/TiKV相关

- 查询获取一次TSO，事务需要获取两次TSO；
- PD TSO Wait Duration包含PD TSO RPC Duration；

![image-20240114115509894](1-TiDB性能优化.assets/image-20240114115509894.png)



- 取数据
- 这两个监控反应的是：TiDB发送读写请求到TiKV的响应时间；

![image-20240114115701199](1-TiDB性能优化.assets/image-20240114115701199.png)

KV Request延迟高的可能的原因：

- TiDB和TiKV的网络延迟较高，带宽被打满；
- TiKV性能出现问题；

#### KV Backoff OPS 

KV Backoff：

- PD RPC（网络错误）
- Region Miss
- TiKV Server Busy
- TiKV RPC（网络错误）
- Txn Lock
- Txn Lock Fast 
- Update Leader...

![image-20240114120254690](1-TiDB性能优化.assets/image-20240114120254690.png)

#### KV BackOff Duration

![image-20240114120644056](1-TiDB性能优化.assets/image-20240114120644056.png)



# 性能监控使用实践

## 大量查询超时案例

![image-20240113230128051](1-TiDB性能优化.assets/image-20240113230128051.png)

![image-20240113230337801](1-TiDB性能优化.assets/image-20240113230337801.png)



![image-20240113230456139](1-TiDB性能优化.assets/image-20240113230456139.png)



![image-20240113230623910](1-TiDB性能优化.assets/image-20240113230623910.png)



![image-20240113231059898](1-TiDB性能优化.assets/image-20240113231059898.png)



![image-20240113231324429](1-TiDB性能优化.assets/image-20240113231324429.png)

DML语句写流程概要

- TSO
- Parse/Compile
- Execute

![image-20240113231426001](1-TiDB性能优化.assets/image-20240113231426001.png)



![image-20240113231507158](1-TiDB性能优化.assets/image-20240113231507158.png)

- PD TSO Wait Duration：是TiDB Server到PD Client，然后到PD并且按顺序返回的时间；
- PD TSO RPC Duration：是PD Client到PD然后返回的时间；



![image-20240113231922644](1-TiDB性能优化.assets/image-20240113231922644.png)



![image-20240114105531177](1-TiDB性能优化.assets/image-20240114105531177.png)



![image-20240114105549925](1-TiDB性能优化.assets/image-20240114105549925.png)



![image-20240114105752845](1-TiDB性能优化.assets/image-20240114105752845.png)



## 写入频繁抖动案例

![image-20240114110103513](1-TiDB性能优化.assets/image-20240114110103513.png)



![image-20240114110225797](1-TiDB性能优化.assets/image-20240114110225797.png)



![image-20240114110434886](1-TiDB性能优化.assets/image-20240114110434886.png)



![image-20240114110656864](1-TiDB性能优化.assets/image-20240114110656864.png)



![image-20240114110751876](1-TiDB性能优化.assets/image-20240114110751876.png)



![image-20240114110843445](1-TiDB性能优化.assets/image-20240114110843445.png)



![image-20240114111105596](1-TiDB性能优化.assets/image-20240114111105596.png)



![image-20240114111215969](1-TiDB性能优化.assets/image-20240114111215969.png)



![image-20240114111408624](1-TiDB性能优化.assets/image-20240114111408624.png)

## 磁盘容量影响PD调度

![image-20240114111649168](1-TiDB性能优化.assets/image-20240114111649168.png)



![image-20240114111858611](1-TiDB性能优化.assets/image-20240114111858611.png)



![image-20240114111949695](1-TiDB性能优化.assets/image-20240114111949695.png)



![image-20240114112039300](1-TiDB性能优化.assets/image-20240114112039300.png)



![image-20240114112128097](1-TiDB性能优化.assets/image-20240114112128097.png)



![image-20240114112200942](1-TiDB性能优化.assets/image-20240114112200942.png)



![image-20240114112250143](1-TiDB性能优化.assets/image-20240114112250143.png)



![image-20240114113151010](1-TiDB性能优化.assets/image-20240114113151010.png)



![image-20240114113121497](1-TiDB性能优化.assets/image-20240114113121497.png)



## TiKV

### 资源相关

![image-20240114205615427](1-TiDB性能优化.assets/image-20240114205615427.png)



![image-20240114205639937](1-TiDB性能优化.assets/image-20240114205639937.png)





MBps：每秒的流量；

可以看下每个TiKV节点的流量是否均衡，如果不均衡，说明可能存在热点，可以通过Dashboard的热力图查看热点位置

![image-20240114205700893](1-TiDB性能优化.assets/image-20240114205700893.png)



每个Store的Region的分布情况

如果Region较多

- 扩容

- 空Region Merge

- 静默Region，降低向PD的心跳反馈；

![image-20240114210133245](1-TiDB性能优化.assets/image-20240114210133245.png)



参考：监控最好不能超过 80%*Server.gRPC-concurency，即每个线程池最好不要超过80%；

- Server.gRPC-concurency：线程池中的并行线程数，

![image-20240114210409557](1-TiDB性能优化.assets/image-20240114210409557.png)

如果过高，说明读存在热点；

![image-20240114210728905](1-TiDB性能优化.assets/image-20240114210728905.png)



冲突检测

参考：不要高于90%*Storage.Scheduler_Worker_Pod_Size

![image-20240114210745997](1-TiDB性能优化.assets/image-20240114210745997.png)



参考：不要超过80%*raftstore.store_pod_size

![image-20240114210934637](1-TiDB性能优化.assets/image-20240114210934637.png)



参考：不超过90%*raftstore.apply_pool_size；

![image-20240114211115352](1-TiDB性能优化.assets/image-20240114211115352.png)



### Duration相关

gRPC message duration：仅为TiKV耗时；

![image-20240114211239214](1-TiDB性能优化.assets/image-20240114211239214.png)



一部分为网络耗时，一部分为TiKV耗时；

![image-20240114211327400](1-TiDB性能优化.assets/image-20240114211327400.png)



![image-20240114211509512](1-TiDB性能优化.assets/image-20240114211509512.png)



![image-20240114211533755](1-TiDB性能优化.assets/image-20240114211533755.png)



![image-20240114211613965](1-TiDB性能优化.assets/image-20240114211613965.png)



![image-20240114212002202](1-TiDB性能优化.assets/image-20240114212002202.png)



![image-20240114212208407](1-TiDB性能优化.assets/image-20240114212208407.png)



![image-20240114212340998](1-TiDB性能优化.assets/image-20240114212340998.png)



![image-20240114212354009](1-TiDB性能优化.assets/image-20240114212354009.png)



![image-20240114212608866](1-TiDB性能优化.assets/image-20240114212608866.png)



![image-20240114212628984](1-TiDB性能优化.assets/image-20240114212628984.png)



## PD相关监控

![image-20240114212719219](1-TiDB性能优化.assets/image-20240114212719219.png)



![image-20240114212802241](1-TiDB性能优化.assets/image-20240114212802241.png)



![image-20240114212910599](1-TiDB性能优化.assets/image-20240114212910599.png)



![image-20240114212953264](1-TiDB性能优化.assets/image-20240114212953264.png)



![image-20240114213018159](1-TiDB性能优化.assets/image-20240114213018159.png)



![image-20240114213028956](1-TiDB性能优化.assets/image-20240114213028956.png)

# TiDB SQL执行流程

## DML语句读流程

![image-20240121125304560](1-TiDB性能优化.assets/image-20240121125304560.png)

## DML语句写流程

![image-20240121125338661](1-TiDB性能优化.assets/image-20240121125338661.png)

## DDL流程

![image-20240121125853303](1-TiDB性能优化.assets/image-20240121125853303.png)

## SQL的Parse与Compile

![image-20240122083812711](1-TiDB性能优化.assets/image-20240122083812711.png)

## 读取的执行

![image-20240122084116212](1-TiDB性能优化.assets/image-20240122084116212.png)

![image-20240122084340571](1-TiDB性能优化.assets/image-20240122084340571.png)

## 写入的执行

![image-20240122084812209](1-TiDB性能优化.assets/image-20240122084812209.png)

![image-20240122085058259](1-TiDB性能优化.assets/image-20240122085058259.png)

## DDL的执行

![image-20240122085250168](1-TiDB性能优化.assets/image-20240122085250168.png)



![image-20240122085556035](1-TiDB性能优化.assets/image-20240122085556035.png)





# Schema设计

## 自增ID

![image-20240115222556675](1-TiDB性能优化.assets/image-20240115222556675.png)

![image-20240115222702284](1-TiDB性能优化.assets/image-20240115222702284.png)

![image-20240115222809616](1-TiDB性能优化.assets/image-20240115222809616.png)

前提是聚簇表,auto_random只能用在聚簇表中。

![image-20240115222957758](1-TiDB性能优化.assets/image-20240115222957758.png)

![image-20240115223528157](1-TiDB性能优化.assets/image-20240115223528157.png)

```sql
select @@global.tidb_enable_cluster_index;
```



![image-20240115223625784](1-TiDB性能优化.assets/image-20240115223625784.png)

## 数据类型

![image-20240115223826841](1-TiDB性能优化.assets/image-20240115223826841.png)

## 对象限制

![image-20240115223902512](1-TiDB性能优化.assets/image-20240115223902512.png)

## 高兼容性Schema

![image-20240115224002392](1-TiDB性能优化.assets/image-20240115224002392.png)

## 高性能Schema

![image-20240115224113552](1-TiDB性能优化.assets/image-20240115224113552.png)



- 非聚簇表如果在 where 条件中用主键过滤，也会回表；
- 