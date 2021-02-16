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











