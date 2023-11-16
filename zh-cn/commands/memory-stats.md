`MEMORY STATS`命令返回服务器的内存使用情况的**数组回复**。

关于内存使用情况的信息以指标及其相应的值的形式提供。报告以下指标：

*   `peak.allocated`：Redis消耗的峰值内存（以字节为单位）（参见`INFO`中的`used_memory_peak`）
*   `total.allocated`：Redis使用分配器分配的总字节数（参见`INFO`中的`used_memory`）
*   `startup.allocated`：Redis在启动时消耗的初始内存量（以字节为单位）（参见`INFO`中的`used_memory_startup`）
*   `replication.backlog`：复制积压的字节数（参见`INFO`中的`repl_backlog_active`）
*   `clients.slaves`：所有副本的开销总字节数（输出和查询缓冲区，连接上下文）
*   `clients.normal`：所有客户端的开销总字节数（输出和查询缓冲区，连接上下文）
*   `cluster.links`：集群链接的内存使用情况（在Redis 7.0中添加，参见`INFO`中的`mem_cluster_links`）
*   `aof.buffer`：AOF相关缓冲区的总字节数
*   `lua.caches`：Lua脚本缓存的开销总字节数
*   `dbXXX`：对于服务器的每个数据库，报告主字典和到期字典的开销（分别为`overhead.hashtable.main`和`overhead.hashtable.expires`）（以字节为单位）
*   `overhead.total`：所有开销的总和，即`startup.allocated`，`replication.backlog`，`clients.slaves`，`clients.normal`，`aof.buffer`以及在管理Redis键空间中使用的内部数据结构的开销（参见`INFO`中的`used_memory_overhead`）
*   `keys.count`：服务器中所有数据库中存储的键的总数
*   `keys.bytes-per-key`：**净内存使用量**（`total.allocated`减去`startup.allocated`）与`keys.count`之间的比率
*   `dataset.bytes`：数据集的大小，即`total.allocated`减去`overhead.total`（参见`INFO`中的`used_memory_dataset`）
*   `dataset.percentage`：`dataset.bytes`占总内存使用量的百分比
*   `peak.percentage`：`total.allocated`占`peak.allocated`的百分比
*   `fragmentation`：参见`INFO`中的`mem_fragmentation_ratio`

**关于本手册中使用的slave一词说明**：从Redis 5开始，为了不向后兼容，Redis项目不再使用slave一词。不幸的是，在此命令中，slave一词是协议的一部分，因此只有在此API自然废弃时，我们才能移除这样的使用情况。
