
1. 查看数据库容量、行数、压缩率

```
SELECT 
    sum(rows) AS `总行数`,
    formatReadableSize(sum(data_uncompressed_bytes)) AS `原始大小`,
    formatReadableSize(sum(data_compressed_bytes)) AS `压缩大小`,
    round((sum(data_compressed_bytes) / sum(data_uncompressed_bytes)) * 100, 0) AS `压缩率`
FROM system.parts;
```
执行结果如下：
```
SELECT 
    sum(rows) AS `总行数`, 
    formatReadableSize(sum(data_uncompressed_bytes)) AS `原始大小`, 
    formatReadableSize(sum(data_compressed_bytes)) AS `压缩大小`, 
    round((sum(data_compressed_bytes) / sum(data_uncompressed_bytes)) * 100, 0) AS `压缩率`
FROM system.parts

┌────总行数─┬─原始大小───┬─压缩大小─┬─压缩率─┐
│ 390715560 │ 105.40 GiB │ 4.10 GiB │      4 │
└───────────┴────────────┴──────────┴────────┘

1 rows in set. Elapsed: 0.037 sec. Processed 1.72 thousand rows, 878.91 KB (45.93 thousand rows/s., 23.51 MB/s.)
```

2. 查看数据表容量、行数、压缩率

```
--在此查询一张临时表的信息
SELECT 
    table AS `表名`,
    sum(rows) AS `总行数`,
    formatReadableSize(sum(data_uncompressed_bytes)) AS `原始大小`,
    formatReadableSize(sum(data_compressed_bytes)) AS `压缩大小`,
    round((sum(data_compressed_bytes) / sum(data_uncompressed_bytes)) * 100, 0) AS `压缩率`
FROM system.parts
WHERE table IN ('temp_1')
GROUP BY table
```
执行结果如下：
```

```

3. 查看数据表分区信息
```
--查看测试表在19年12月的分区信息
SELECT 
    partition AS `分区`,
    sum(rows) AS `总行数`,
    formatReadableSize(sum(data_uncompressed_bytes)) AS `原始大小`,
    formatReadableSize(sum(data_compressed_bytes)) AS `压缩大小`,
    round((sum(data_compressed_bytes) / sum(data_uncompressed_bytes)) * 100, 0) AS `压缩率`
FROM system.parts
WHERE (database IN ('default')) AND (table IN ('temp_1')) AND (partition LIKE '2019-12-%')
GROUP BY partition
ORDER BY partition ASC
```


4. 查看数据表字段的信息

```
SELECT 
    column AS `字段名`,
    any(type) AS `类型`,
    formatReadableSize(sum(column_data_uncompressed_bytes)) AS `原始大小`,
    formatReadableSize(sum(column_data_compressed_bytes)) AS `压缩大小`,
    sum(rows) AS `行数`
FROM system.parts_columns
WHERE (database = 'default') AND (table = 'temp_1')
GROUP BY column
ORDER BY column ASC
```

4. 查看后台进程并杀死进程
ClickHouse自带用于记录系统信息的系统库system，通过processes表，我们可以查看当前连接的进程信息，也就是正在运行的sql的信息:
```
select
    query_id,read_rows,total_rows_approx,memory_usage,
    initial_user,initial_address,elapsed,query
from system.processes;
# 字段含义
# query_id 查询id，
# read_rows 从表中读取的行数，
# total_rows_approx 应读取的行总数的近似值，
# memory_usage 请求使用的内存量
# initial_user 进行查询的用户
# initial_address 请求的 IP 地址
# elapsed 求执行开始以来的秒数
# query 查询语句
```

通过sql语句的查询行数和查询已经执行的时间来判断sql是不是在慢查询，或者是同事在查询的时候没有日期限定而直接查全表。一般的话如果grafana监控的CK节点出现cpu飙升的情况，就需要我们去判断CK中是否有垃圾sql在执行，根据query_id杀死该进程
```
kill query where query_id='70442d9b-7fc5-4a0e-81be-9543431a4882';

```


