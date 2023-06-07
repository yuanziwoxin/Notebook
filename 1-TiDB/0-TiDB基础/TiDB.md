# 临时表

临时表主要用于解决业务中间计算结果的临时存储问题，让用户免于频繁地建表和删表等操作；用户可将业务上的中间计算数据存入临时表，用完数据后TiDB自动清理回收临时表。这避免了用户业务过于复杂，减少了表管理开销，并提升了性能。

## 使用场景

TiDB临时表主要使用场景：

- 缓存业务的中间临时数据，计算完成后将数据存储转存至普通表，临时表会自动释放；
- 短期内对统一数据多次DML操作；
- 快速批量导入中间临时数据，提升导入临时数据的性能；
- 批量更新数据。
  - 将数据批量导入数据库的临时表，修改完后再导出到文件；

## 临时表类型

TiDB的临时表分为：

- 本地临时表：表定义和表内数据只对当前会话可见，适用于暂存会话内的中间数据；
  - 本地临时表的数据对会话内的所有事务可见；
  - 在会话结束后，该会话创建的本地临时表会被自动删除；
  - 本地临时表可以与普通表同名，此时在DDL和DML语句中，普通表被隐藏，直到本地临时表被删除；
  - 创建、删除本地临时表时，不会自动提交当前事务；
  - 本地临时表不支持外键和分区表；
  - 创建本地临时表需要 `CREATE TEMPORARY TABLES` 权限，随后对该表的所有操作不需要权限；
  - TiDB 本地临时表不支持 `ALTER TABLE`；
  - 不同于 MySQL，TiDB 本地临时表都是外部表，SQL 语句不会创建内部临时表；

```sql
-- 创建本地临时表
CREATE TEMPORARY TABLE users (
    id BIGINT,
    name VARCHAR(100),
    city VARCHAR(50),
    PRIMARY KEY(id)
);
-- 删除本地临时表
drop table users;
drop temporary table;
```



- 全局临时表：**表定义对整个TiDB集群可见**，**表内数据只对当前事务可见**，适用于暂存事务内的中间数据。
  - 全局临时表的表定义会持久化，对所有会话可见；
  - 全局临时表的数据只对当前的事务内可见，事务结束后数据自动清空；
  - 全局临时表不能与普通表同名；

```sql
-- 创建全局临时表
CREATE GLOBAL TEMPORARY TABLE users (
    id BIGINT,
    name VARCHAR(100),
    city VARCHAR(50),
    PRIMARY KEY(id)
) ON COMMIT DELETE ROWS;
-- 
drop table TABLE users;
drop global temporary table users;
```



> 连接（Connection）是物理概念，
>
> 会话（Session）是逻辑概念
>
> 一个连接可以有多个会话，也可以没有会话；一条连接上的各个会话可以使用不同的用户身份；用一个连接上的不同会话之间不会相互影响；
>
> 两个会话之间的影响主要体现在锁和锁存上，即对相同资源的操作（对象定义或数据库）或请求（CPU/内存），它们的处理一般是按队列来处理，前面未处理好，后面就要等待；
>
> 如打电话，Connection好比你接通对方，这时Connection就建立了，而不管有没有通话；双方进行通话，则Session建立了，如果换人，则新的Session建立，原Session结束；类似的，可以在同一个Connection上进行多个会话。最后，挂机，Connection结束。
>
> 会话可以创建多个事务；
>
> - 比如使用客户端连接数据库，这样你就可以执行很多个事务了；
>
> 一个事务只能由一个会话产生；

## 临时表局限

- 1、临时表数据是放在TiDB节点的内存中，可能造成内存溢出！！！

## 限制临时表的内存占用

无论定义表时声明的 `ENGINE` 是哪种存储引擎，本地临时表和全局临时表的数据都只暂存在 TiDB 实例的内存中，不持久化。

为了避免内存溢出，用户可通过系统变量 [`tidb_tmp_table_max_size`](https://docs.pingcap.com/zh/tidb/stable/system-variables#tidb_tmp_table_max_size-从-v53-版本开始引入) 限制每张临时表的大小。当临时表大小超过限制后 TiDB 会报错。`tidb_tmp_table_max_size` 的默认值是 `64MB`。

例如，将每张临时表的大小限制为 `256MB`：

```sql
SET GLOBAL tidb_tmp_table_max_size=268435456;
```