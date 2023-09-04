# TiDB 锁冲突问题处理

TiDB 支持完整的分布式事务，自 v3.0 版本起，提供[乐观事务](https://docs.pingcap.com/zh/tidb/dev/optimistic-transaction)与[悲观事务](https://docs.pingcap.com/zh/tidb/dev/pessimistic-transaction)两种事务模式。本文介绍如何使用 Lock View 排查锁相关的问题，以及如何处理使用乐观事务或者悲观事务的过程中常见的锁冲突问题。

## 使用 Lock View 排查锁相关的问题

自 v5.1 版本起，TiDB 支持 Lock View 功能。该功能在 `information_schema` 中内置了若干系统表，用于提供更多关于锁冲突和锁等待的信息。



注意

Lock View 功能目前仅提供悲观锁的冲突和等待信息。

关于这些系统表的详细说明，请参考以下文档：

- [`TIDB_TRX` 与 `CLUSTER_TIDB_TRX`](https://docs.pingcap.com/zh/tidb/dev/information-schema-tidb-trx)：提供当前 TiDB 节点上或整个集群上所有运行中的事务的信息，包括事务是否处于等锁状态、等锁时间和事务曾经执行过的语句的 Digest 等信息。
- [`DATA_LOCK_WAITS`](https://docs.pingcap.com/zh/tidb/dev/information-schema-data-lock-waits)：提供关于 TiKV 内的悲观锁等锁信息，包括阻塞和被阻塞的事务的 `start_ts`、被阻塞的 SQL 语句的 Digest 和发生等待的 key。
- [`DEADLOCKS` 与 `CLUSTER_DEADLOCKS`](https://docs.pingcap.com/zh/tidb/dev/information-schema-deadlocks)：提供当前 TiDB 节点上或整个集群上最近发生过的若干次死锁的相关信息，包括死锁环中事务之间的等待关系、事务当前正在执行的语句的 Digest 和发生等待的 key。



注意

Lock View 所属的系统表中展示的 SQL 语句为归一化的 SQL 语句（即去除了格式和参数的 SQL 语句），通过内部查询从 SQL Digest 获得，因而无法获取包括格式和参数在内的完整语句。有关 SQL Digest 和归一化 SQL 语句的详细介绍，请参阅 [Statement Summary Tables](https://docs.pingcap.com/zh/tidb/dev/statement-summary-tables)。

以下为排查部分问题的示例。

### 死锁错误

要获取最近发生的死锁错误的信息，可查询 `DEADLOCKS` 或 `CLUSTER_DEADLOCKS` 表。

以查询 `DEADLOCKS` 表为例，请执行以下 SQL 语句：

```sql
select * from information_schema.deadlocks;
```

示例输出：

```sql
+-------------+----------------------------+-----------+--------------------+------------------------------------------------------------------+-----------------------------------------+----------------------------------------+----------------------------------------------------------------------------------------------------+--------------------+
| DEADLOCK_ID | OCCUR_TIME                 | RETRYABLE | TRY_LOCK_TRX_ID    | CURRENT_SQL_DIGEST                                               | CURRENT_SQL_DIGEST_TEXT                 | KEY                                    | KEY_INFO                                                                                           | TRX_HOLDING_LOCK   |
+-------------+----------------------------+-----------+--------------------+------------------------------------------------------------------+-----------------------------------------+----------------------------------------+----------------------------------------------------------------------------------------------------+--------------------+
|           1 | 2021-08-05 11:09:03.230341 |         0 | 426812829645406216 | 22230766411edb40f27a68dadefc63c6c6970d5827f1e5e22fc97be2c4d8350d | update `t` set `v` = ? where `id` = ? ; | 7480000000000000355F728000000000000002 | {"db_id":1,"db_name":"test","table_id":53,"table_name":"t","handle_type":"int","handle_value":"2"} | 426812829645406217 |
|           1 | 2021-08-05 11:09:03.230341 |         0 | 426812829645406217 | 22230766411edb40f27a68dadefc63c6c6970d5827f1e5e22fc97be2c4d8350d | update `t` set `v` = ? where `id` = ? ; | 7480000000000000355F728000000000000001 | {"db_id":1,"db_name":"test","table_id":53,"table_name":"t","handle_type":"int","handle_value":"1"} | 426812829645406216 |
+-------------+----------------------------+-----------+--------------------+------------------------------------------------------------------+-----------------------------------------+----------------------------------------+----------------------------------------------------------------------------------------------------+--------------------+
```

查询结果会显示死锁错误中多个事务之间的等待关系和各个事务当前正在执行的 SQL 语句的归一化形式（即去掉参数和格式的形式），以及发生冲突的 key 及其从 key 中解读出的一些信息。

例如在上述例子中，第一行意味着 ID 为 `426812829645406216` 的事务当前正在执行形如 `update `t` set `v` = ? where `id` = ? ;` 的语句，被另一个 ID 为 `426812829645406217` 的事务阻塞；而 `426812829645406217` 同样也在执行一条形如 `update `t` set `v` = ? where `id` = ? ;` 的语句，并被 ID 为 `426812829645406216` 的事务阻塞，两个事务因而构成死锁。

### 少数热点 key 造成锁排队

`DATA_LOCK_WAITS` 系统表提供 TiKV 节点上的等锁情况。查询该表时，TiDB 将自动从所有 TiKV 节点上获取当前时刻的等锁信息。当少数热点 key 频繁被上锁并阻塞较多事务时，你可以查询 `DATA_LOCK_WAITS` 表并按 key 对结果进行聚合，以尝试找出经常发生问题的 key：

```sql
select `key`, count(*) as `count` from information_schema.data_lock_waits group by `key` order by `count` desc;
```

示例输出：

```sql
+----------------------------------------+-------+
| key                                    | count |
+----------------------------------------+-------+
| 7480000000000000415F728000000000000001 |     2 |
| 7480000000000000415F728000000000000002 |     1 |
+----------------------------------------+-------+
```

为避免偶然性，你可考虑进行多次查询。

如果已知频繁出问题的 key，可尝试从 `TIDB_TRX` 或 `CLUSTER_TIDB_TRX` 表中获取试图上锁该 key 的事务的信息。

需要注意 `TIDB_TRX` 和 `CLUSTER_TIDB_TRX` 表所展示的信息也是对其进行查询的时刻正在运行的事务的信息，并不展示已经结束的事务。如果并发的事务数量很大，该查询的结果集也可能很大，可以考虑添加 limit 子句，或用 where 子句筛选出等锁时间较长的事务。需要注意，对 Lock View 中的多张表进行 join 时，不同表之间的数据并不保证在同一时刻获取，因而不同表中的信息可能并不同步。

以用 where 子句筛选出等锁时间较长的事务为例，请执行以下 SQL 语句：

```sql
select trx.* from information_schema.data_lock_waits as l left join information_schema.tidb_trx as trx on l.trx_id = trx.id where l.key = "7480000000000000415F728000000000000001"\G
```

示例输出：

```sql
*************************** 1. row ***************************
                     ID: 426831815660273668
             START_TIME: 2021-08-06 07:16:00.081000
     CURRENT_SQL_DIGEST: 06da614b93e62713bd282d4685fc5b88d688337f36e88fe55871726ce0eb80d7
CURRENT_SQL_DIGEST_TEXT: update `t` set `v` = `v` + ? where `id` = ? ;
                  STATE: LockWaiting
     WAITING_START_TIME: 2021-08-06 07:16:00.087720
        MEM_BUFFER_KEYS: 0
       MEM_BUFFER_BYTES: 0
             SESSION_ID: 77
                   USER: root
                     DB: test
        ALL_SQL_DIGESTS: ["0fdc781f19da1c6078c9de7eadef8a307889c001e05f107847bee4cfc8f3cdf3","06da614b93e62713bd282d4685fc5b88d688337f36e88fe55871726ce0eb80d7"]
*************************** 2. row ***************************
                     ID: 426831818019569665
             START_TIME: 2021-08-06 07:16:09.081000
     CURRENT_SQL_DIGEST: 06da614b93e62713bd282d4685fc5b88d688337f36e88fe55871726ce0eb80d7
CURRENT_SQL_DIGEST_TEXT: update `t` set `v` = `v` + ? where `id` = ? ;
                  STATE: LockWaiting
     WAITING_START_TIME: 2021-08-06 07:16:09.290271
        MEM_BUFFER_KEYS: 0
       MEM_BUFFER_BYTES: 0
             SESSION_ID: 75
                   USER: root
                     DB: test
        ALL_SQL_DIGESTS: ["0fdc781f19da1c6078c9de7eadef8a307889c001e05f107847bee4cfc8f3cdf3","06da614b93e62713bd282d4685fc5b88d688337f36e88fe55871726ce0eb80d7"]
2 rows in set (0.00 sec)
```

### 事务被长时间阻塞

如果已知一个事务被另一事务（或多个事务）阻塞，且已知当前事务的 `start_ts`（即事务 ID），则可使用如下方式获取导致该事务阻塞的事务的信息。注意对 Lock View 中的多张表进行 join 时，不同表之间的数据并不保证在同一时刻获取，因而可能不同表中的信息可能并不同步。

```sql
select l.key, trx.*, tidb_decode_sql_digests(trx.all_sql_digests) as sqls from information_schema.data_lock_waits as l join information_schema.cluster_tidb_trx as trx on l.current_holding_trx_id = trx.id where l.trx_id = 426831965449355272\G
```

示例输出：

```sql
*************************** 1. row ***************************
                    key: 74800000000000004D5F728000000000000001
               INSTANCE: 127.0.0.1:10080
                     ID: 426832040186609668
             START_TIME: 2021-08-06 07:30:16.581000
     CURRENT_SQL_DIGEST: 06da614b93e62713bd282d4685fc5b88d688337f36e88fe55871726ce0eb80d7
CURRENT_SQL_DIGEST_TEXT: update `t` set `v` = `v` + ? where `id` = ? ;
                  STATE: LockWaiting
     WAITING_START_TIME: 2021-08-06 07:30:16.592763
        MEM_BUFFER_KEYS: 1
       MEM_BUFFER_BYTES: 19
             SESSION_ID: 113
                   USER: root
                     DB: test
        ALL_SQL_DIGESTS: ["0fdc781f19da1c6078c9de7eadef8a307889c001e05f107847bee4cfc8f3cdf3","a4e28cc182bdd18288e2a34180499b9404cd0ba07e3cc34b6b3be7b7c2de7fe9","06da614b93e62713bd282d4685fc5b88d688337f36e88fe55871726ce0eb80d7"]
                   sqls: ["begin ;","select * from `t` where `id` = ? for update ;","update `t` set `v` = `v` + ? where `id` = ? ;"]
1 row in set (0.01 sec)
```

上述查询中，对 `CLUSTER_TIDB_TRX` 表的 `ALL_SQL_DIGESTS` 列使用了 [`TIDB_DECODE_SQL_DIGESTS`](https://docs.pingcap.com/zh/tidb/dev/tidb-functions#tidb_decode_sql_digests) 函数，目的是将该列（内容为一组 SQL Digest）转换为其对应的归一化 SQL 语句，便于阅读。

如果当前事务的 `start_ts` 未知，可以尝试从 `TIDB_TRX` / `CLUSTER_TIDB_TRX` 表或者 [`PROCESSLIST` / `CLUSTER_PROCESSLIST`](https://docs.pingcap.com/zh/tidb/dev/information-schema-processlist) 表中的信息进行判断。

## 处理乐观锁冲突问题

以下介绍乐观事务模式下常见的锁冲突问题的处理方式。

### 读写冲突

在 TiDB 中，读取数据时，会获取一个包含当前物理时间且全局唯一递增的时间戳作为当前事务的 start_ts。事务在读取时，需要读到目标 key 的 commit_ts 小于这个事务的 start_ts 的最新的数据版本。当读取时发现目标 key 上存在 lock 时，因为无法知道上锁的那个事务是在 Commit 阶段还是 Prewrite 阶段，所以就会出现读写冲突的情况，如下图：

![读写冲突](https://download.pingcap.com/images/docs-cn/troubleshooting-lock-pic-04.png)

分析：

Txn0 完成了 Prewrite，在 Commit 的过程中 Txn1 对该 key 发起了读请求，Txn1 需要读取 start_ts > commit_ts 最近的 key 的版本。此时，Txn1 的 `start_ts > Txn0` 的 lock_ts，需要读取的 key 上的锁信息仍未清理，故无法判断 Txn0 是否提交成功，因此 Txn1 与 Txn0 出现读写冲突。

你可以通过如下两种途径来检测当前环境中是否存在读写冲突：

1. TiDB 监控及日志

   - 通过 TiDB Grafana 监控分析：

     观察 KV Errors 下 Lock Resolve OPS 面板中的 not_expired/resolve 监控项以及 KV Backoff OPS 面板中的 tikvLockFast 监控项，如果有较为明显的上升趋势，那么可能是当前的环境中出现了大量的读写冲突。其中，not_expired 是指对应的锁还没有超时，resolve 是指尝试清锁的操作，tikvLockFast 代表出现了读写冲突。

     ![KV-backoff-txnLockFast-optimistic](https://download.pingcap.com/images/docs-cn/troubleshooting-lock-pic-09.png)![KV-Errors-resolve-optimistic](https://download.pingcap.com/images/docs-cn/troubleshooting-lock-pic-08.png)

   - 通过 TiDB 日志分析：

     在 TiDB 的日志中可以看到下列信息：

     ```log
     [INFO] [coprocessor.go:743] ["[TIME_COP_PROCESS] resp_time:406.038899ms txnStartTS:416643508703592451 region_id:8297 store_addr:10.8.1.208:20160 backoff_ms:255 backoff_types:[txnLockFast,txnLockFast] kv_process_ms:333 scan_total_write:0 scan_processed_write:0 scan_total_data:0 scan_processed_data:0 scan_total_lock:0 scan_processed_lock:0"]
     ```

     - txnStartTS：发起读请求的事务的 start_ts，如上面示例中的 416643508703592451
     - backoff_types：读写发生了冲突，并且读请求进行了 backoff 重试，重试的类型为 txnLockFast
     - backoff_ms：读请求 backoff 重试的耗时，单位为 ms，如上面示例中的 255
     - region_id：读请求访问的目标 region 的 id

2. 通过 TiKV 日志分析：

   在 TiKV 的日志可以看到下列信息：

   ```log
   [ERROR] [endpoint.rs:454] [error-response] [err=""locked primary_lock:7480000000000004D35F6980000000000000010380000000004C788E0380000000004C0748 lock_version: 411402933858205712 key: 7480000000000004D35F7280000000004C0748 lock_ttl: 3008 txn_size: 1""]
   ```

   这段报错信息表示出现了读写冲突，当读数据时发现 key 有锁阻碍读，这个锁包括未提交的乐观锁和未提交的 prewrite 后的悲观锁。

   - primary_lock：锁对应事务的 primary lock。
   - lock_version：锁对应事务的 start_ts。
   - key：表示被锁的 key。
   - lock_ttl: 锁的 TTL。
   - txn_size：锁所在事务在其 Region 的 key 数量，指导清锁方式。

处理建议：

- 在遇到读写冲突时会有 backoff 自动重试机制，如上述示例中 Txn1 会进行 backoff 重试，单次初始 10 ms，单次最大 3000 ms，总共最大 20000 ms

- 可以使用 TiDB Control 的子命令 [decoder](https://docs.pingcap.com/zh/tidb/dev/tidb-control#decoder-命令) 来查看指定 key 对应的行的 table id 以及 rowid：

  ```sh
  ./tidb-ctl decoder "t\x00\x00\x00\x00\x00\x00\x00\x1c_r\x00\x00\x00\x00\x00\x00\x00\xfa"
  format: table_row
  table_id: -9223372036854775780
  row_id: -9223372036854775558
  ```

### KeyIsLocked 错误

事务在 Prewrite 阶段的第一步就会检查是否有写写冲突，第二步会检查目标 key 是否已经被另一个事务上锁。当检测到该 key 被 lock 后，会在 TiKV 端报出 KeyIsLocked。目前该报错信息没有打印到 TiDB 以及 TiKV 的日志中。与读写冲突一样，在出现 KeyIsLocked 时，后台会自动进行 backoff 重试。

你可以通过 TiDB Grafana 监控检测 KeyIsLocked 错误：

观察 KV Errors 下 Lock Resolve OPS 面板中的 resolve 监控项以及 KV Backoff OPS 面板中的 txnLock 监控项，会有比较明显的上升趋势，其中 resolve 是指尝试清锁的操作，txnLock 代表出现了写冲突。

![KV-backoff-txnLockFast-optimistic-01](https://download.pingcap.com/images/docs-cn/troubleshooting-lock-pic-07.png)![KV-Errors-resolve-optimistic-01](https://download.pingcap.com/images/docs-cn/troubleshooting-lock-pic-08.png)

处理建议：

- 监控中出现少量 txnLock，无需过多关注。后台会自动进行 backoff 重试，单次初始 100 ms，单次最大 3000 ms。
- 如果出现大量的 txnLock，需要从业务的角度评估下冲突的原因。
- 使用悲观锁模式。

### 锁被清除 (LockNotFound) 错误

TxnLockNotFound 错误是由于事务提交的慢了，超过了 TTL 的时间。当要提交时，发现被其他事务给 Rollback 掉了。在开启 TiDB [自动重试事务](https://docs.pingcap.com/zh/tidb/dev/system-variables#tidb_retry_limit)的情况下，会自动在后台进行事务重试（注意显示和隐式事务的差别）。

你可以通过如下两种途径来查看 LockNotFound 报错信息：

1. 查看 TiDB 日志

   如果出现了 TxnLockNotFound 的报错，会在 TiDB 的日志中看到下面的信息：

   ```log
   [WARN] [session.go:446] ["commit failed"] [conn=149370] ["finished txn"="Txn{state=invalid}"] [error="[kv:6]Error: KV error safe to retry tikv restarts txn: Txn(Mvcc(TxnLockNotFound{ start_ts: 412720515987275779, commit_ts: 412720519984971777, key: [116, 128, 0, 0, 0, 0, 1, 111, 16, 95, 114, 128, 0, 0, 0, 0, 0, 0, 2] })) [try again later]"]
   ```

   - start_ts：出现 TxnLockNotFound 报错的事务的 start_ts，如上例中的 412720515987275779
   - commit_ts：出现 TxnLockNotFound 报错的事务的 commit_ts，如上例中的 412720519984971777

2. 查看 TiKV 日志

   如果出现了 TxnLockNotFound 的报错，在 TiKV 的日志中同样可以看到相应的报错信息：

   ```log
   Error: KV error safe to retry restarts txn: Txn(Mvcc(TxnLockNotFound)) [ERROR [Kv.rs:708] ["KvService::batch_raft send response fail"] [err=RemoteStoped]
   ```

处理建议：

- 通过检查 start_ts 和 commit_ts 之间的提交间隔，可以确认是否超过了默认的 TTL 的时间。

  查看提交间隔：

  ```sh
  tiup ctl:v<CLUSTER_VERSION> pd tso [start_ts]
  tiup ctl:v<CLUSTER_VERSION> pd tso [commit_ts]
  ```

- 建议检查下是否是因为写入性能的缓慢导致事务提交的效率差，进而出现了锁被清除的情况。

- 在关闭 TiDB 事务重试的情况下，需要在应用端捕获异常，并进行重试。

## 处理悲观锁冲突问题

以下介绍悲观事务模式下常见的锁冲突问题的处理方式。



注意

即使设置了悲观事务模式，autocommit 事务仍然会优先尝试使用乐观事务模式进行提交，并在发生冲突后、自动重试时切换为悲观事务模式。

### 读写冲突

报错信息以及处理建议同乐观锁模式。

### pessimistic lock retry limit reached

在冲突非常严重的场景下，或者当发生 write conflict 时，乐观事务会直接终止，而悲观事务会尝试用最新数据重试该语句直到没有 write conflict。因为 TiDB 的加锁操作是一个写入操作，且操作过程是先读后写，需要 2 次 RPC。如果在这中间发生了 write conflict，那么会重试。每次重试都会打印日志，不用特别关注。重试次数由 [pessimistic-txn.max-retry-count](https://docs.pingcap.com/zh/tidb/dev/tidb-configuration-file#max-retry-count) 定义。

可通过查看 TiDB 日志查看报错信息：

悲观事务模式下，如果发生 write conflict，并且重试的次数达到了上限，那么在 TiDB 的日志中会出现含有下述关键字的报错信息。如下：

```log
err="pessimistic lock retry limit reached"
```

处理建议：

- 如果上述报错出现的比较频繁，建议从业务的角度进行调整。
- 如果业务中包含对同一行（同一个 key）的高并发上锁而频繁冲突，可以尝试启用系统变量 [`tidb_pessimistic_txn_fair_locking`](https://docs.pingcap.com/zh/tidb/dev/system-variables#tidb_pessimistic_txn_fair_locking-从-v700-版本开始引入)。需要注意启用该选项可能对存在锁冲突的事务带来一定程度的吞吐下降（平均延迟上升）的代价。对于新部署的集群，该选项默认启用 (`ON`) 。

### Lock wait timeout exceeded

在悲观锁模式下，事务之间出现会等锁的情况。等锁的超时时间由 TiDB 的 [innodb_lock_wait_timeout](https://docs.pingcap.com/zh/tidb/dev/system-variables#innodb_lock_wait_timeout) 参数来定义，这个是 SQL 语句层面的最大允许等锁时间，即一个 SQL 语句期望加锁，但锁一直获取不到，超过这个时间，TiDB 不会再尝试加锁，会向客户端返回相应的报错信息。

可通过查看 TiDB 日志查看报错信息：

当出现等锁超时的情况时，会向客户端返回下述报错信息：

```log
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

处理建议：

- 如果出现的次数非常频繁，建议从业务逻辑的角度来进行调整。

### TTL manager has timed out

除了有不能超出 GC 时间的限制外，悲观锁的 TTL 有上限，默认为 1 小时，所以执行时间超过 1 小时的悲观事务有可能提交失败。这个超时时间由 TiDB 参数 [performance.max-txn-ttl](https://github.com/pingcap/tidb/blob/master/config/config.toml.example) 指定。

可通过查看 TiDB 日志查看报错信息：

当悲观锁的事务执行时间超过 TTL 时，会出现下述报错：

```log
TTL manager has timed out, pessimistic locks may expire, please commit or rollback this transaction
```

处理建议：

- 当遇到该报错时，建议确认下业务逻辑是否可以进行优化，如将大事务拆分为小事务。在未使用[大事务](https://docs.pingcap.com/zh/tidb/dev/tidb-configuration-file#txn-total-size-limit)的前提下，大事务可能会触发 TiDB 的事务限制。
- 可适当调整相关参数，使其符合事务要求。

### Deadlock found when trying to get lock

死锁是指两个或两个以上的事务在执行过程中，由于竞争资源而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去，将永远在互相等待。此时，需要终止其中一个事务使其能够继续推进下去。

TiDB 在使用悲观锁的情况下，多个事务之间出现了死锁，必定有一个事务 abort 来解开死锁。在客户端层面行为和 MySQL 一致，在客户端返回表示死锁的 Error 1213。如下：

```log
[err="[executor:1213]Deadlock found when trying to get lock; try restarting transaction"]
```

处理建议：

- 如果难以确认产生死锁的原因，对于 v5.1 及以后的版本，建议尝试查询 `INFORMATION_SCHEMA.DEADLOCKS` 或 `INFORMATION_SCHEMA.CLUSTER_DEADLOCKS` 系统表来获取死锁的等待链信息。详情请参考[死锁错误](https://docs.pingcap.com/zh/tidb/dev/troubleshoot-lock-conflicts#死锁错误)小节和 [`DEADLOCKS` 表](https://docs.pingcap.com/zh/tidb/dev/information-schema-deadlocks)文档。
- 如果死锁出现非常频繁，需要调整业务代码来降低发生概率。 



# TIDB_TRX

`TIDB_TRX` 表提供了当前 TiDB 节点上正在执行的事务的信息。

```sql
USE INFORMATION_SCHEMA;
DESC TIDB_TRX;
```

输出结果如下：

```sql
+-------------------------+-----------------------------------------------------------------+------+------+---------+-------+
| Field                   | Type                                                            | Null | Key  | Default | Extra |
+-------------------------+-----------------------------------------------------------------+------+------+---------+-------+
| ID                      | bigint(21) unsigned                                             | NO   | PRI  | NULL    |       |
| START_TIME              | timestamp(6)                                                    | YES  |      | NULL    |       |
| CURRENT_SQL_DIGEST      | varchar(64)                                                     | YES  |      | NULL    |       |
| CURRENT_SQL_DIGEST_TEXT | text                                                            | YES  |      | NULL    |       |
| STATE                   | enum('Idle','Running','LockWaiting','Committing','RollingBack') | YES  |      | NULL    |       |
| WAITING_START_TIME      | timestamp(6)                                                    | YES  |      | NULL    |       |
| MEM_BUFFER_KEYS         | bigint(64)                                                      | YES  |      | NULL    |       |
| MEM_BUFFER_BYTES        | bigint(64)                                                      | YES  |      | NULL    |       |
| SESSION_ID              | bigint(21) unsigned                                             | YES  |      | NULL    |       |
| USER                    | varchar(16)                                                     | YES  |      | NULL    |       |
| DB                      | varchar(64)                                                     | YES  |      | NULL    |       |
| ALL_SQL_DIGESTS         | text                                                            | YES  |      | NULL    |       |
| RELATED_TABLE_IDS       | text                                                            | YES  |      | NULL    |       |
+-------------------------+-----------------------------------------------------------------+------+------+---------+-------+
```

`TIDB_TRX` 表中各列的字段含义如下：

- `ID`：事务 ID，即事务的开始时间戳 `start_ts`。

- `START_TIME`：事务的开始时间，即事务的 `start_ts` 所对应的物理时间。

- `CURRENT_SQL_DIGEST`：该事务当前正在执行的 SQL 语句的 Digest。

- `CURRENT_SQL_DIGEST_TEXT`：该事务当前正在执行的 SQL 语句的归一化形式，即去除了参数和格式的 SQL 语句。与 `CURRENT_SQL_DIGEST` 对应。

- ```
  STATE
  ```

  ：该事务当前所处的状态。其可能的值包括：

  - `Idle`：事务处于闲置状态，即正在等待用户输入查询。
  - `Running`：事务正在正常执行一个查询。
  - `LockWaiting`：事务处于正在等待悲观锁上锁完成的状态。需要注意的是，事务刚开始进行悲观锁上锁操作时即进入该状态，无论是否有被其它事务阻塞。
  - `Committing`：事务正在提交过程中。
  - `RollingBack`：事务正在回滚过程中。

- `WAITING_START_TIME`：当 `STATE` 值为 `LockWaiting` 时，该列显示等待的开始时间。

- `MEM_BUFFER_KEYS`：当前事务写入内存缓冲区的 key 的个数。

- `MEM_BUFFER_BYTES`：当前事务写入内存缓冲区的 key 和 value 的总字节数。

- `SESSION_ID`：该事务所属的 session 的 ID。

- `USER`：执行该事务的用户名。

- `DB`：执行该事务的 session 当前的默认数据库名。

- `ALL_SQL_DIGESTS`：该事务已经执行过的语句的 Digest 的列表，表示为一个 JSON 格式的字符串数组。每个事务最多记录前 50 条语句。通过 [`TIDB_DECODE_SQL_DIGESTS`](https://docs.pingcap.com/zh/tidb/dev/tidb-functions#tidb_decode_sql_digests) 函数可以将该列的信息变换为对应的归一化 SQL 语句的列表。

- `RELATED_TABLE_IDS`：该事务访问的表、视图等对象的 ID。



注意

- 仅拥有 [PROCESS](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_process) 权限的用户可以获取该表中的完整信息。没有 PROCESS 权限的用户则只能查询到当前用户所执行的事务的信息。
- `CURRENT_SQL_DIGEST` 和 `ALL_SQL_DIGESTS` 列中的信息 (SQL Digest) 为 SQL 语句进行归一化后计算得到的哈希值。`CURRENT_SQL_DIGEST_TEXT` 列中的信息和函数 `TIDB_DECODE_SQL_DIGESTS` 所得到的结果均为内部从 Statements Summary 系列表中查询得到，因而存在内部查询不到对应语句的可能性。关于 SQL Digest 和 Statements Summary 相关表的详细说明，请参阅[Statement Summary Tables](https://docs.pingcap.com/zh/tidb/dev/statement-summary-tables)。
- [`TIDB_DECODE_SQL_DIGESTS`](https://docs.pingcap.com/zh/tidb/dev/tidb-functions#tidb_decode_sql_digests) 函数调用开销较大，如果对大量事务的信息调用该函数查询历史 SQL，可能查询耗时较长。如果集群规模较大、同一时刻并发运行的事务较多，请避免直接在查询 `TIDB_TRX` 全表的同时直接将该函数用于 `ALL_SQL_DIGEST` 列（即尽量避免 `SELECT *, tidb_decode_sql_digests(all_sql_digests) FROM TIDB_TRX` 这样的用法）。
- 目前 `TIDB_TRX` 表暂不支持显示 TiDB 内部事务相关的信息。

## 示例

查看 `TIDB_TRX` 表中的信息：

```sql
SELECT * FROM INFORMATION_SCHEMA.TIDB_TRX\G
```

输出结果如下：

```sql
*************************** 1. row ***************************
                     ID: 426789913200689153
             START_TIME: 2021-08-04 10:51:54.883000
     CURRENT_SQL_DIGEST: NULL
CURRENT_SQL_DIGEST_TEXT: NULL
                  STATE: Idle
     WAITING_START_TIME: NULL
        MEM_BUFFER_KEYS: 1
       MEM_BUFFER_BYTES: 29
             SESSION_ID: 7
                   USER: root
                     DB: test
        ALL_SQL_DIGESTS: ["e6f07d43b5c21db0fbb9a31feac2dc599787763393dd5acbfad80e247eb02ad5","04fa858fa491c62d194faec2ab427261cc7998b3f1ccf8f6844febca504cb5e9","b83710fa8ab7df8504920e8569e48654f621cf828afbe7527fd003b79f48da9e"]
*************************** 2. row ***************************
                     ID: 426789921471332353
             START_TIME: 2021-08-04 10:52:26.433000
     CURRENT_SQL_DIGEST: 38b03afa5debbdf0326a014dbe5012a62c51957f1982b3093e748460f8b00821
CURRENT_SQL_DIGEST_TEXT: update `t` set `v` = `v` + ? where `id` = ?
                  STATE: LockWaiting
     WAITING_START_TIME: 2021-08-04 10:52:35.106568
        MEM_BUFFER_KEYS: 0
       MEM_BUFFER_BYTES: 0
             SESSION_ID: 9
                   USER: root
                     DB: test
        ALL_SQL_DIGESTS: ["e6f07d43b5c21db0fbb9a31feac2dc599787763393dd5acbfad80e247eb02ad5","38b03afa5debbdf0326a014dbe5012a62c51957f1982b3093e748460f8b00821"]
2 rows in set (0.01 sec)
```

此示例的查询结果表示：当前节点有两个运行中的事务，第一个事务正在闲置状态（`STATE` 为 `Idle`，`CURRENT_SQL_DIGEST` 为 `NULL`），该事务已经执行过 3 条语句（`ALL_SQL_DIGESTS` 列表中有三条记录，分别为执行过的 3 条语句的 Digest）；第二个事务正在执行一条语句并正在等锁（`STATE` 为 `LockWaiting`，`WAITING_START_TIME` 显示了等锁开始的时间），该事务已经执行过两条语句，当前正在执行的语句形如 `"update `t` set `v` = `v` + ? where `id` = ?"`。

```sql
SELECT id, all_sql_digests, tidb_decode_sql_digests(all_sql_digests) AS all_sqls FROM INFORMATION_SCHEMA.TIDB_TRX\G
```

输出结果如下：

```sql
*************************** 1. row ***************************
             id: 426789913200689153
all_sql_digests: ["e6f07d43b5c21db0fbb9a31feac2dc599787763393dd5acbfad80e247eb02ad5","04fa858fa491c62d194faec2ab427261cc7998b3f1ccf8f6844febca504cb5e9","b83710fa8ab7df8504920e8569e48654f621cf828afbe7527fd003b79f48da9e"]
       all_sqls: ["begin","insert into `t` values ( ... )","update `t` set `v` = `v` + ?"]
*************************** 2. row ***************************
             id: 426789921471332353
all_sql_digests: ["e6f07d43b5c21db0fbb9a31feac2dc599787763393dd5acbfad80e247eb02ad5","38b03afa5debbdf0326a014dbe5012a62c51957f1982b3093e748460f8b00821"]
       all_sqls: ["begin","update `t` set `v` = `v` + ? where `id` = ?"]
```

此查询对 `TIDB_TRX` 表的 `ALL_SQL_DIGESTS` 列调用了 [`TIDB_DECODE_SQL_DIGESTS`](https://docs.pingcap.com/zh/tidb/dev/tidb-functions#tidb_decode_sql_digests) 函数，将 SQL Digest 的数组通过系统内部的查询转换成归一化 SQL 语句的数组，以便于直观地获取事务历史执行过的语句的信息。但是需要注意上述查询扫描了 `TIDB_TRX` 全表，并对每一行都调用了 `TIDB_DECODE_SQL_DIGESTS` 函数；而 `TIDB_DECODE_SQL_DIGESTS` 函数调用的开销很大，所以如果集群中并发事务数量较多，请尽量避免这种查询。

## CLUSTER_TIDB_TRX

`TIDB_TRX` 表仅提供单个 TiDB 节点中正在执行的事务信息。如果要查看整个集群上所有 TiDB 节点中正在执行的事务信息，需要查询 `CLUSTER_TIDB_TRX` 表。与 `TIDB_TRX` 表的查询结果相比，`CLUSTER_TIDB_TRX` 表的查询结果额外包含了 `INSTANCE` 字段。`INSTANCE` 字段展示了集群中各节点的 IP 地址和端口，用于区分事务所在的 TiDB 节点。

```sql
USE INFORMATION_SCHEMA;
DESC CLUSTER_TIDB_TRX;
```

输出结果如下：

```sql
+-------------------------+-----------------------------------------------------------------+------+------+---------+-------+
| Field                   | Type                                                            | Null | Key  | Default | Extra |
+-------------------------+-----------------------------------------------------------------+------+------+---------+-------+
| INSTANCE                | varchar(64)                                                     | YES  |      | NULL    |       |
| ID                      | bigint(21) unsigned                                             | NO   | PRI  | NULL    |       |
| START_TIME              | timestamp(6)                                                    | YES  |      | NULL    |       |
| CURRENT_SQL_DIGEST      | varchar(64)                                                     | YES  |      | NULL    |       |
| CURRENT_SQL_DIGEST_TEXT | text                                                            | YES  |      | NULL    |       |
| STATE                   | enum('Idle','Running','LockWaiting','Committing','RollingBack') | YES  |      | NULL    |       |
| WAITING_START_TIME      | timestamp(6)                                                    | YES  |      | NULL    |       |
| MEM_BUFFER_KEYS         | bigint(64)                                                      | YES  |      | NULL    |       |
| MEM_BUFFER_BYTES        | bigint(64)                                                      | YES  |      | NULL    |       |
| SESSION_ID              | bigint(21) unsigned                                             | YES  |      | NULL    |       |
| USER                    | varchar(16)                                                     | YES  |      | NULL    |       |
| DB                      | varchar(64)                                                     | YES  |      | NULL    |       |
| ALL_SQL_DIGESTS         | text                                                            | YES  |      | NULL    |       |
| RELATED_TABLE_IDS       | text                                                            | YES  |      | NULL    |       |
+-------------------------+-----------------------------------------------------------------+------+------+---------+-------+
14 rows in set (0.00 sec)
```



# DATA_LOCK_WAITS

`DATA_LOCK_WAITS` 表展示了集群中所有 TiKV 节点上当前正在发生的等锁情况，包括悲观锁的等锁情况和乐观事务被阻塞的信息。

```sql
USE information_schema;
DESC data_lock_waits;
+------------------------+---------------------+------+------+---------+-------+
| Field                  | Type                | Null | Key  | Default | Extra |
+------------------------+---------------------+------+------+---------+-------+
| KEY                    | text                | NO   |      | NULL    |       |
| KEY_INFO               | text                | YES  |      | NULL    |       |
| TRX_ID                 | bigint(21) unsigned | NO   |      | NULL    |       |
| CURRENT_HOLDING_TRX_ID | bigint(21) unsigned | NO   |      | NULL    |       |
| SQL_DIGEST             | varchar(64)         | YES  |      | NULL    |       |
| SQL_DIGEST_TEXT        | text                | YES  |      | NULL    |       |
+------------------------+---------------------+------+------+---------+-------+
```

`DATA_LOCK_WAITS` 表中各列的字段含义如下：

- `KEY`：正在发生等锁的 key，以十六进制编码的形式显示。
- `KEY_INFO`：对 `KEY` 进行解读得出的一些详细信息，见 [KEY_INFO](https://docs.pingcap.com/zh/tidb/dev/information-schema-data-lock-waits#key_info)。
- `TRX_ID`：正在等锁的事务 ID，即 `start_ts`。
- `CURRENT_HOLDING_TRX_ID`：当前持有锁的事务 ID，即 `start_ts`。
- `SQL_DIGEST`：当前正在等锁的事务中被阻塞的 SQL 语句的 Digest。
- `SQL_DIGEST_TEXT`：当前正在等锁的事务中被阻塞的 SQL 语句的归一化形式，即去除了参数和格式的 SQL 语句。与 `SQL_DIGEST` 对应。



警告

- 仅拥有 [PROCESS](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_process) 权限的用户可以查询该表。
- 由于实现限制，目前对于乐观事务被阻塞情况的 `SQL_DIGEST` 和 `SQL_DIGEST_TEXT` 字段为 `null`。如果需要知道导致阻塞的 SQL 语句，可以将此表与 [`CLUSTER_TIDB_TRX`](https://docs.pingcap.com/zh/tidb/dev/information-schema-tidb-trx) 进行 `JOIN` 来获得对应事务的所有 SQL 语句。
- `DATA_LOCK_WAITS` 表中的信息是在查询时，从所有 TiKV 节点实时获取的。目前，即使加上了 `WHERE` 查询条件，也无法避免对所有 TiKV 节点都进行信息收集。如果集群规模很大、负载很高，查询该表有造成性能抖动的潜在风险，因此请根据实际情况使用。
- 来自不同 TiKV 节点的信息不能保证是同一时间点的快照。
- `SQL_DIGEST` 列中的信息（SQL Digest）为 SQL 语句进行归一化后计算得到的哈希值。`SQL_DIGEST_TEXT` 列中的信息为内部从 Statements Summary 系列表中查询得到，因而存在内部查询不到对应语句的可能性。关于 SQL Digest 和 Statements Summary 相关表的详细说明，请参阅[Statement Summary Tables](https://docs.pingcap.com/zh/tidb/dev/statement-summary-tables)。

## `KEY_INFO`

`KEY_INFO` 列中展示了对 `KEY` 列中所给出的 key 的详细信息，以 JSON 格式给出。其包含的信息如下：

- `"db_id"`：该 key 所属的数据库（schema）的 ID。

- `"db_name"`：该 key 所属的数据库（schema）的名称。

- `"table_id"`：该 key 所属的表的 ID。

- `"table_name"`：该 key 所属的表的名称。

- `"partition_id"`：该 key 所在的分区（partition）的 ID。

- `"partition_name"`：该 key 所在的分区（partition）的名称。

- ```
  "handle_type"
  ```

  ：该 row key （即储存一行数据的 key）的 handle 类型，其可能的值有：

  - `"int"`：handle 为 int 类型，即 handle 为 row ID
  - `"common"`：非 int64 类型的 handle，在启用 clustered index 时非 int 类型的主键会显示为此类型
  - `"unknown"`：当前暂不支持的 handle 类型

- `"handle_value"`：handle 的值。

- `"index_id"`：该 index key （即储存索引的 key）所属的 index ID。

- `"index_name"`：该 index key 所属的 index 名称。

- `"index_values"`：该 index key 中的 index value。

其中，不适用或当前无法查询到的信息会被省略。比如，row key 的信息中不会包含 `index_id`、`index_name` 和 `index_values`；index key 不会包含 `handle_type` 和 `handle_value`；非分区表不会显示 `partition_id` 和 `partition_name`；已经被删除掉的表中的 key 的信息无法获取 `table_name`、`db_id`、`db_name`、`index_name` 等 schema 信息，且无法区分是否为分区表。



注意

如果一个 key 来自一张启用了分区的表，而在查询时，由于某些原因（例如，其所属的表已经被删除）导致无法查询其所属的 schema 信息，则其所属的分区的 ID 可能会出现在 `table_id` 字段中。这是因为，TiDB 对不同分区的 key 的编码方式与对几张独立的表的 key 的编码方式一致，因而在缺失 schema 信息时无法确认该 key 属于一张未分区的表还是某张表的一个分区。

## 示例

```sql
select * from information_schema.data_lock_waits\G
*************************** 1. row ***************************
                   KEY: 7480000000000000355F728000000000000001
              KEY_INFO: {"db_id":1,"db_name":"test","table_id":53,"table_name":"t","handle_type":"int","handle_value":"1"}
                TRX_ID: 426790594290122753
CURRENT_HOLDING_TRX_ID: 426790590082449409
            SQL_DIGEST: 38b03afa5debbdf0326a014dbe5012a62c51957f1982b3093e748460f8b00821
       SQL_DIGEST_TEXT: update `t` set `v` = `v` + ? where `id` = ?
1 row in set (0.01 sec)
```

以上查询结果显示，ID 为 `426790594290122753` 的事务在执行 Digest 为 `"38b03afa5debbdf0326a014dbe5012a62c51957f1982b3093e748460f8b00821"`、形如 `update `t` set `v` = `v` + ? where `id` = ?` 的语句的过程中，试图在 `"7480000000000000355F728000000000000001"` 这个 key 上获取悲观锁，但是该 key 上的锁目前被 ID 为 `426790590082449409` 的事务持有。

# DEADLOCKS

`DEADLOCKS` 表提供当前 TiDB 节点上最近发生的若干次死锁错误的信息。

```sql
USE INFORMATION_SCHEMA;
DESC DEADLOCKS;
```

输出结果如下：

```sql
+-------------------------+---------------------+------+------+---------+-------+
| Field                   | Type                | Null | Key  | Default | Extra |
+-------------------------+---------------------+------+------+---------+-------+
| DEADLOCK_ID             | bigint(21)          | NO   |      | NULL    |       |
| OCCUR_TIME              | timestamp(6)        | YES  |      | NULL    |       |
| RETRYABLE               | tinyint(1)          | NO   |      | NULL    |       |
| TRY_LOCK_TRX_ID         | bigint(21) unsigned | NO   |      | NULL    |       |
| CURRENT_SQL_DIGEST      | varchar(64)         | YES  |      | NULL    |       |
| CURRENT_SQL_DIGEST_TEXT | text                | YES  |      | NULL    |       |
| KEY                     | text                | YES  |      | NULL    |       |
| KEY_INFO                | text                | YES  |      | NULL    |       |
| TRX_HOLDING_LOCK        | bigint(21) unsigned | NO   |      | NULL    |       |
+-------------------------+---------------------+------+------+---------+-------+
```

`DEADLOCKS` 表中需要用多行来表示同一个死锁事件，每行显示参与死锁的其中一个事务的信息。当该 TiDB 节点记录了多次死锁错误时，需要按照 `DEADLOCK_ID` 列来区分，相同的 `DEADLOCK_ID` 表示同一个死锁事件。需要注意，`DEADLOCK_ID` **并不保证全局唯一，也不会持久化**，因而其只能在同一个结果集里表示同一个死锁事件。

`DEADLOCKS` 表中各列的字段含义如下：

- `DEADLOCK_ID`：死锁事件的 ID。当表内存在多次死锁错误的信息时，需要使用该列来区分属于不同死锁错误的行。
- `OCCUR_TIME`：发生该次死锁错误的时间。
- `RETRYABLE`：该次死锁错误是否可重试。关于可重试的死锁错误的说明，参见[可重试的死锁错误](https://docs.pingcap.com/zh/tidb/dev/information-schema-deadlocks#可重试的死锁错误)小节。
- `TRY_LOCK_TRX_ID`：试图上锁的事务 ID，即事务的 `start_ts`。
- `CURRENT_SQL_DIGEST`：试图上锁的事务中当前正在执行的 SQL 语句的 Digest。
- `CURRENT_SQL_DIGEST_TEXT`：试图上锁的事务中当前正在执行的 SQL 语句的归一化形式。
- `KEY`：该事务试图上锁、但是被阻塞的 key，以十六进制编码的形式显示。
- `KEY_INFO`：对 `KEY` 进行解读得出的一些详细信息，详见 [`KEY_INFO`](https://docs.pingcap.com/zh/tidb/dev/information-schema-deadlocks#key_info)。
- `TRX_HOLDING_LOCK`：该 key 上当前持锁并导致阻塞的事务 ID，即事务的 `start_ts`。

要调整 `DEADLOCKS` 表中可以容纳的死锁事件数量，可通过 TiDB 配置文件中的 [`pessimistic-txn.deadlock-history-capacity`](https://docs.pingcap.com/zh/tidb/dev/tidb-configuration-file#deadlock-history-capacity) 配置项进行调整，默认容纳最近 10 次死锁错误的信息。



注意

- 仅拥有 [PROCESS](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_process) 权限的用户可以查询该表。
- `CURRENT_SQL_DIGEST` 列中的信息 (SQL Digest) 为 SQL 语句进行归一化后计算得到的哈希值。`CURRENT_SQL_DIGEST_TEXT` 列中的信息为内部从 Statements Summary 系列表中查询得到，因而存在内部查询不到对应语句的可能性。关于 SQL Digest 和 Statements Summary 相关表的详细说明，请参阅[Statement Summary Tables](https://docs.pingcap.com/zh/tidb/dev/statement-summary-tables)。

## `KEY_INFO`

`KEY_INFO` 列中展示了对 `KEY` 列中所给出的 key 的详细信息，以 JSON 格式给出。其包含的信息如下：

- `"db_id"`：该 key 所属的数据库 (schema) 的 ID。

- `"db_name"`：该 key 所属的数据库 (schema) 的名称。

- `"table_id"`：该 key 所属的表的 ID。

- `"table_name"`：该 key 所属的表的名称。

- `"partition_id"`：该 key 所在的分区 (partition) 的 ID。

- `"partition_name"`：该 key 所在的分区 (partition) 的名称。

- ```
  "handle_type"
  ```

  ：该 row key（即储存一行数据的 key）的 handle 类型，其可能的值有：

  - `"int"`：handle 为 int 类型，即 handle 为 row ID
  - `"common"`：非 int64 类型的 handle，在启用 clustered index 时非 int 类型的主键会显示为此类型
  - `"unknown"`：当前暂不支持的 handle 类型

- `"handle_value"`：handle 的值。

- `"index_id"`：该 index key（即储存索引的 key）所属的 index ID。

- `"index_name"`：该 index key 所属的 index 名称。

- `"index_values"`：该 index key 中的 index value。

其中，不适用或当前无法查询到的信息会被省略。比如，row key 的信息中不会包含 `index_id`、`index_name` 和 `index_values`；index key 不会包含 `handle_type` 和 `handle_value`；非分区表不会显示 `partition_id` 和 `partition_name`；已经被删除掉的表中的 key 的信息无法获取 `table_name`、`db_id`、`db_name`、`index_name` 等 schema 信息，且无法区分是否为分区表。



注意

如果一个 key 来自一张启用了分区的表，而在查询时，由于某些原因（例如，其所属的表已经被删除）导致无法查询其所属的 schema 信息，则其所属的分区的 ID 可能会出现在 `table_id` 字段中。这是因为，TiDB 对不同分区的 key 的编码方式与对几张独立的表的 key 的编码方式一致，因而在缺失 schema 信息时无法确认该 key 属于一张未分区的表还是某张表的一个分区。

## 可重试的死锁错误



注意

`DEADLOCKS` 表中默认不收集可重试的死锁错误的信息。如果需要收集，可通过 TiDB 配置文件中的 [`pessimistic-txn.deadlock-history-collect-retryable`](https://docs.pingcap.com/zh/tidb/dev/tidb-configuration-file#deadlock-history-collect-retryable) 配置项进行调整。

当事务 A 被另一个事务 B 已经持有的锁阻塞，而事务 B 直接或间接地被当前事务 A 持有的锁阻塞，将会引发一个死锁错误。这里：

- 情况一：事务 B 可能（直接或间接地）被事务 A 开始后到被阻塞前这段时间内已经执行完成的语句产生的锁阻塞
- 情况二：事务 B 也可能被事务 A 目前正在执行的语句阻塞

对于情况一，TiDB 将会向事务 A 的客户端报告死锁错误，并终止该事务；而对于情况二，事务 A 当前正在执行的语句将在 TiDB 内部被自动重试。例如，假设事务 A 执行了如下语句：

```sql
UPDATE t SET v = v + 1 WHERE id = 1 OR id = 2;
```

事务 B 则先后执行如下两条语句：

```sql
UPDATE t SET v = 4 WHERE id = 2;
UPDATE t SET v = 2 WHERE id = 1;
```

那么如果事务 A 先后对 `id = 1` 和 `id = 2` 的两行分别上锁，且两个事务以如下时序运行：

1. 事务 A 对 `id = 1` 的行上锁
2. 事务 B 执行第一条语句并对 `id = 2` 的行上锁
3. 事务 B 执行第二条语句试图对 `id = 1` 的行上锁，被事务 A 阻塞
4. 事务 A 试图对 `id = 2` 的行上锁，被 B 阻塞，形成死锁

对于情况二，由于事务 A 阻塞其它事务的语句也是当前正在执行的语句，因而可以解除当前语句所上的悲观锁（使得事务 B 可以继续运行），并重试当前语句。TiDB 内部使用 key 的哈希值来判断是否属于这种情况。

当可重试的死锁发生时，内部自动重试并不会引起事务报错，因而对客户端透明，但是这种情况的频繁发生可能影响性能。当这种情况发生时，在 TiDB 的日志中可以观察到 `single statement deadlock, retry statement` 字样的日志。

## 示例 1

假设有如下表定义和初始数据：

```sql
CREATE TABLE t (id int primary key, v int);
INSERT INTO t VALUES (1, 10), (2, 20);
```

使两个事务按如下顺序执行：

| 事务 1                              | 事务 2                              | 说明                |
| :---------------------------------- | :---------------------------------- | :------------------ |
| `UPDATE t SET v = 11 WHERE id = 1;` |                                     |                     |
|                                     | `UPDATE t SET v = 21 WHERE id = 2;` |                     |
| `UPDATE t SET v = 12 WHERE id = 2;` |                                     | 事务 1 阻塞         |
|                                     | `UPDATE t SET v = 22 WHERE id = 1;` | 事务 2 报出死锁错误 |

接下来，事务 2 将报出死锁错误。此时，查询 `DEADLOCKS` 表：

```sql
SELECT * FROM INFORMATION_SCHEMA.DEADLOCKS;
```

输出结果如下：

```sql
+-------------+----------------------------+-----------+--------------------+------------------------------------------------------------------+-----------------------------------------+----------------------------------------+----------------------------------------------------------------------------------------------------+--------------------+
| DEADLOCK_ID | OCCUR_TIME                 | RETRYABLE | TRY_LOCK_TRX_ID    | CURRENT_SQL_DIGEST                                               | CURRENT_SQL_DIGEST_TEXT                 | KEY                                    | KEY_INFO                                                                                           | TRX_HOLDING_LOCK   |
+-------------+----------------------------+-----------+--------------------+------------------------------------------------------------------+-----------------------------------------+----------------------------------------+----------------------------------------------------------------------------------------------------+--------------------+
|           1 | 2021-08-05 11:09:03.230341 |         0 | 426812829645406216 | 22230766411edb40f27a68dadefc63c6c6970d5827f1e5e22fc97be2c4d8350d | update `t` set `v` = ? where `id` = ? ; | 7480000000000000355F728000000000000002 | {"db_id":1,"db_name":"test","table_id":53,"table_name":"t","handle_type":"int","handle_value":"2"} | 426812829645406217 |
|           1 | 2021-08-05 11:09:03.230341 |         0 | 426812829645406217 | 22230766411edb40f27a68dadefc63c6c6970d5827f1e5e22fc97be2c4d8350d | update `t` set `v` = ? where `id` = ? ; | 7480000000000000355F728000000000000001 | {"db_id":1,"db_name":"test","table_id":53,"table_name":"t","handle_type":"int","handle_value":"1"} | 426812829645406216 |
+-------------+----------------------------+-----------+--------------------+------------------------------------------------------------------+-----------------------------------------+----------------------------------------+----------------------------------------------------------------------------------------------------+--------------------+
```

该表中产生了两行数据，两行的 `DEADLOCK_ID` 字段皆为 1，表示这两行数据包含同一次死锁错误的信息。第一行显示 ID 为 `426812829645406216` 的事务，在 `"7480000000000000355F728000000000000002"` 这个 key 上，被 ID 为 `"426812829645406217"` 的事务阻塞了；第二行则显示 ID 为 `"426812829645406217"` 的事务在 `"7480000000000000355F728000000000000001"` 这个 key 上被 ID 为 `426812829645406216` 的事务阻塞了，构成了相互阻塞的状态，形成了死锁。

## 示例 2

假设查询 `DEADLOCKS` 表得到了如下结果集：

```sql
+-------------+----------------------------+-----------+--------------------+------------------------------------------------------------------+-----------------------------------------+----------------------------------------+----------------------------------------------------------------------------------------------------+--------------------+
| DEADLOCK_ID | OCCUR_TIME                 | RETRYABLE | TRY_LOCK_TRX_ID    | CURRENT_SQL_DIGEST                                               | CURRENT_SQL_DIGEST_TEXT                 | KEY                                    | KEY_INFO                                                                                           | TRX_HOLDING_LOCK   |
+-------------+----------------------------+-----------+--------------------+------------------------------------------------------------------+-----------------------------------------+----------------------------------------+----------------------------------------------------------------------------------------------------+--------------------+
|           1 | 2021-08-05 11:09:03.230341 |         0 | 426812829645406216 | 22230766411edb40f27a68dadefc63c6c6970d5827f1e5e22fc97be2c4d8350d | update `t` set `v` = ? where `id` = ? ; | 7480000000000000355F728000000000000002 | {"db_id":1,"db_name":"test","table_id":53,"table_name":"t","handle_type":"int","handle_value":"2"} | 426812829645406217 |
|           1 | 2021-08-05 11:09:03.230341 |         0 | 426812829645406217 | 22230766411edb40f27a68dadefc63c6c6970d5827f1e5e22fc97be2c4d8350d | update `t` set `v` = ? where `id` = ? ; | 7480000000000000355F728000000000000001 | {"db_id":1,"db_name":"test","table_id":53,"table_name":"t","handle_type":"int","handle_value":"1"} | 426812829645406216 |
|           2 | 2021-08-05 11:09:21.252154 |         0 | 426812832017809412 | 22230766411edb40f27a68dadefc63c6c6970d5827f1e5e22fc97be2c4d8350d | update `t` set `v` = ? where `id` = ? ; | 7480000000000000355F728000000000000002 | {"db_id":1,"db_name":"test","table_id":53,"table_name":"t","handle_type":"int","handle_value":"2"} | 426812832017809413 |
|           2 | 2021-08-05 11:09:21.252154 |         0 | 426812832017809413 | 22230766411edb40f27a68dadefc63c6c6970d5827f1e5e22fc97be2c4d8350d | update `t` set `v` = ? where `id` = ? ; | 7480000000000000355F728000000000000003 | {"db_id":1,"db_name":"test","table_id":53,"table_name":"t","handle_type":"int","handle_value":"3"} | 426812832017809414 |
|           2 | 2021-08-05 11:09:21.252154 |         0 | 426812832017809414 | 22230766411edb40f27a68dadefc63c6c6970d5827f1e5e22fc97be2c4d8350d | update `t` set `v` = ? where `id` = ? ; | 7480000000000000355F728000000000000001 | {"db_id":1,"db_name":"test","table_id":53,"table_name":"t","handle_type":"int","handle_value":"1"} | 426812832017809412 |
+-------------+----------------------------+-----------+--------------------+------------------------------------------------------------------+-----------------------------------------+----------------------------------------+----------------------------------------------------------------------------------------------------+--------------------+
```

以上查询结果中的 `DEADLOCK_ID` 列表明，前两行共同表示一次死锁错误的信息，两条事务相互等待构成了死锁；而后三行共同表示另一次死锁信息，三个事务循环等待构成了死锁。

## CLUSTER_DEADLOCKS

`CLUSTER_DEADLOCKS` 表返回整个集群上每个 TiDB 节点中最近发生的数次死锁错误的信息，即将每个节点上的 `DEADLOCKS` 表内的信息合并在一起。`CLUSTER_DEADLOCKS` 还包含额外的 `INSTANCE` 列展示所属节点的 IP 地址和端口，用以区分不同的 TiDB 节点。

需要注意的是，由于 `DEADLOCK_ID` 并不保证全局唯一，所以在 `CLUSTER_DEADLOCKS` 表的查询结果中，需要 `INSTANCE` 和 `DEADLOCK_ID` 两个字段共同区分结果集中的不同死锁错误的信息。

```sql
USE INFORMATION_SCHEMA;
DESC CLUSTER_DEADLOCKS;
+-------------------------+---------------------+------+------+---------+-------+
| Field                   | Type                | Null | Key  | Default | Extra |
+-------------------------+---------------------+------+------+---------+-------+
| INSTANCE                | varchar(64)         | YES  |      | NULL    |       |
| DEADLOCK_ID             | bigint(21)          | NO   |      | NULL    |       |
| OCCUR_TIME              | timestamp(6)        | YES  |      | NULL    |       |
| RETRYABLE               | tinyint(1)          | NO   |      | NULL    |       |
| TRY_LOCK_TRX_ID         | bigint(21) unsigned | NO   |      | NULL    |       |
| CURRENT_SQL_DIGEST      | varchar(64)         | YES  |      | NULL    |       |
| CURRENT_SQL_DIGEST_TEXT | text                | YES  |      | NULL    |       |
| KEY                     | text                | YES  |      | NULL    |       |
| KEY_INFO                | text                | YES  |      | NULL    |       |
| TRX_HOLDING_LOCK        | bigint(21) unsigned | NO   |      | NULL    |       |
+-------------------------+---------------------+------+------+---------+-------+
```