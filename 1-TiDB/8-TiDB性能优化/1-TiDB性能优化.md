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
- 在right join中，过滤条件放在on后面，只对左表有效；过滤条件放在where后，表示在left join关联之后的结果集进行过滤，对左表和右表都有效；

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