`CLIENT LIST` 命令以大多是人类可读的格式返回有关客户端连接服务器的信息和统计数据。

您可以使用其中一个可选的子命令来筛选列表。`TYPE type` 子命令根据客户端类型来筛选列表，其中 *type* 可以为 `normal`、`master`、`replica` 或 `pubsub` 中的一个。请注意，被 `MONITOR` 命令阻塞的客户端属于 `normal` 类。

`ID`过滤器只返回与`client-id`参数匹配的客户ID的条目。

以下是字段的含义：

* `id`：64位唯一客户端ID
* `addr`：客户端的地址/端口
* `laddr`：客户端连接的本地地址/端口（绑定地址）
* `fd`：与套接字对应的文件描述符
* `name`：客户端使用 `CLIENT SETNAME` 设置的名称
* `age`：连接的总持续时间（以秒为单位）
* `idle`：连接的空闲时间（以秒为单位）
* `flags`：客户端标志（见下文）
* `db`：当前数据库ID
* `sub`：订阅的频道数量
* `psub`：模式匹配订阅的数量
* `ssub`：分片订阅的频道数量。在Redis 7.0.3中添加
* `multi`：MULTI/EXEC上下文中的命令数量
* `qbuf`：查询缓冲区长度（0表示没有挂起的查询）
* `qbuf-free`：查询缓冲区的剩余空间（0表示缓冲区已满）
* `argv-mem`：下一个命令的不完整参数（已从查询缓冲区中提取）
* `multi-mem`：由已缓冲的MULTI命令使用的内存。在Redis 7.0中添加
* `obl`：输出缓冲区长度
* `oll`：输出列表长度（当缓冲区已满时，回复会排队在此列表中）
* `omem`：输出缓冲区的内存使用情况
* `tot-mem`：该客户端在其各种缓冲区中消耗的总内存
* `events`：文件描述符事件（见下文）
* `cmd`：最后一个执行的命令
* `user`：客户端的认证用户名
* `redir`：当前客户端跟踪重定向的客户端ID
* `resp`：客户端的RESP协议版本。在Redis 7.0中添加

客户端标志可以是以下组合：

```
A: connection to be closed ASAP
b: the client is waiting in a blocking operation
c: connection to be closed after writing entire reply
d: a watched keys has been modified - EXEC will fail
e: the client is excluded from the client eviction mechanism
i: the client is waiting for a VM I/O (deprecated)
M: the client is a master
N: no specific flag set
O: the client is a client in MONITOR mode
P: the client is a Pub/Sub subscriber
r: the client is in readonly mode against a cluster node
S: the client is a replica node connection to this instance
u: the client is unblocked
U: the client is connected via a Unix domain socket
x: the client is in a MULTI/EXEC context
t: the client enabled keys tracking in order to perform client side caching
T: the client will not touch the LRU/LFU of the keys it accesses
R: the client tracking target client is invalid
B: the client enabled broadcast tracking mode 
```

文件描述符事件可以是以下之一：

```
r: the client socket is readable (event loop)
w: the client socket is writable (event loop)
```

## 笔记

定期添加新字段用于调试目的。 有些字段将来可能会被删除。使用此命令的版本安全的Redis客户端应该相应地解析输出（即优雅地处理缺失字段，跳过未知字段）。
