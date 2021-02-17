# ClickHouse存储部分 - MergeTree

> 初稿: 2021-02-16

### MergeTree系列

ClickHouse提供了许多表引擎，数据在不同的表引擎下会以**不同**的方式存储。MergeTree系列是**最广泛使用**的表引擎系列，该系列包括以下引擎：

* MergeTree
* ReplacingMergeTree
* SummingMergeTree
* AggregatingMergeTree
* CollapsingMergeTree
* VersionedCollapsingMergeTree
* GraphiteMergeTree

### MergeTree引擎的特点

#### 数据写入效率高

数据和索引存放在part文件夹中,  每一次写入生成新的part文件夹即结束。

#### 数据读取效率高

读取效率高有多方面原因:

* 数据按列存储，每个列有自己的数据文件和索引文件。在OLAP场景下，只需读取特定的列，IO效率高\(列存的优势一\)。
* 数据按列存储，因为列数据的相关性，按列压缩效率高。相同的IO下，读取的数据量更大\(列存的优势二\)。
* 数据按主键\(排序键\)排序，主键稀疏索引。依赖于有序性和稀疏索引，快速定位数据位置。
* 后台相同partition的part文件夹会发生合并，减少小文件数量。读取时，只需要读合并后的文件即可

### 观察MergeTree引擎在硬盘上的文件

数据文件的位置定义在配置文件config.xml中

```bash
# cat /etc/clickhouse-server/config.xml
<path>/var/lib/clickhouse</path>

# ls -l /var/lib/clickhouse
drwxr-x---  2 clickhouse clickhouse 4096 Feb 17 11:22 access
drwxr-x---  4 clickhouse clickhouse 4096 Feb 17 11:22 data
drwxr-x---  2 clickhouse clickhouse 4096 Feb 17 11:22 dictionaries_lib
drwxr-x---  2 clickhouse clickhouse 4096 Feb 17 11:22 flags
drwxr-x---  2 clickhouse clickhouse 4096 Feb 17 11:22 format_schemas
drwxr-x---  2 clickhouse clickhouse 4096 Feb 17 11:22 metadata
drwxr-x---  2 clickhouse clickhouse 4096 Feb 17 11:22 metadata_dropped
drwxr-x---  2 clickhouse clickhouse 4096 Feb 17 11:22 preprocessed_configs
-rw-r-----  1 clickhouse clickhouse   61 Feb 17 11:22 status
drwxr-x--- 10 clickhouse clickhouse 4096 Feb 17 11:26 store
drwxr-x---  2 clickhouse clickhouse 4096 Feb 17 11:22 tmp
drwxr-x---  2 clickhouse clickhouse 4096 Feb 17 11:22 user_files
```

重点关注data文件夹，其层次结构如下 data/&lt;db\_name&gt;/&lt;tb\_name&gt;/

```bash
# tree data/ 
data
|-- default #默认数据库 
|   `-- t1  #MergeTree表名
|        |-- all_1_1_0 #Part名
|        |-- detached
|        `-- format_version.txt
`-- system #系统表数据库
    |-- asynchronous_metric_log
    |-- metric_log
    |-- query_log 
    |-- query_thread_log 
    `-- trace_log
```

Part文件夹命名方式为 PartitionID\_MinBlockID\_MaxBlockID\_Level

* PartitionID: 属于相同partition的part会拥有同样的PartitionID, 后台线程会将相同PartitionID的part进行合并。
* MinBlockID和MaxBlockID还有Level的意义在观察一个数据合并的过程就清楚了



![Part&#x5408;&#x5E76;&#x8FC7;&#x7A0B;](.gitbook/assets/image%20%282%29.png)

观察上图合并过程有几个发现:

1. 插入数据生成Part的时候，MinBlockID和MaxBlockID相同, Level为0
2. 先后两次的插入，BlockID有**递增**关系
3. Part合并后，MinBlockID变为两个Part中MinBlockID较小值, MaxBlockID变为两个Part中MaxBlockID中较大值，Level增1
4. Part合并后，因为合并后的Part已经含有合并前两个Part的数据，因此合并前的Part可以被后台线程回收

通过MinBlockID和MaxBlockID，我们能知道这个Part下有几个数据Block。通过Level我们能知道合并了几次。





