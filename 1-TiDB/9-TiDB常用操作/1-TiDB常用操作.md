### 如何预估 TiDB 中一张表的大小？

要预估 TiDB 中一张表的大小，你可以参考使用以下查询语句：

```sql
SELECT
    db_name,
    table_name,
    ROUND(SUM(total_size / cnt), 2) Approximate_Size,
    ROUND(SUM(total_size / cnt / (SELECT
                    ROUND(AVG(value), 2)
                FROM
                    METRICS_SCHEMA.store_size_amplification
                WHERE
                    value > 0)),
            2) Disk_Size
FROM
    (SELECT
        db_name,
            table_name,
            region_id,
            SUM(Approximate_Size) total_size,
            COUNT(*) cnt
    FROM
        information_schema.TIKV_REGION_STATUS
    WHERE
        db_name = @dbname
            AND table_name IN (@table_name)
    GROUP BY db_name , table_name , region_id) tabinfo
GROUP BY db_name , table_name;
```

在使用以上语句时，你需要根据实际情况填写并替换掉语句里的以下字段：

- `@dbname`：数据库名称。
- `@table_name`：目标表的名称。

此外，以上语句中：

- `store_size_amplification` 表示集群压缩比的平均值。除了使用 `SELECT * FROM METRICS_SCHEMA.store_size_amplification;` 语句进行查询以外，你还可以查看 Grafana 监控 **PD - statistics balance** 面板下各节点的 `Size amplification` 指标来获取该信息，集群压缩比的平均值即为所有节点的 `Size amplification` 平均值。
- `Approximate_Size` 表示压缩前表的单副本大小，该值为估算值，并非准确值。
- `Disk_Size` 表示压缩后表的大小，可根据 `Approximate_Size` 和 `store_size_amplification` 得出估算值。