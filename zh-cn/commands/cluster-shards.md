`CLUSTER SHARDS` 返回有关群集分片的详细信息。
分片被定义为一组为同一组槽提供服务并彼此复制的节点集合。
一个分片在给定时间只能有一个主节点，但可以有多个或没有副本。
一个分片可以在不提供任何槽的情况下仍然有副本。

This command replaces the `CLUSTER SLOTS` command, by providing a more efficient and extensible representation of the cluster.

这个命令适用于Redis集群客户端库，用于了解集群的拓扑结构。
客户端在启动时应该发出此命令，以检索将集群“哈希槽”与实际节点信息相关联的映射。
这个映射应该用于将命令发送到可能服务于给定命令相关的槽的节点。
如果命令被发送到了错误的节点，并收到了“-MOVED”重定向消息，那么可以使用此命令来更新集群的拓扑结构。

该命令返回一个由分片组成的数组，每个分片都包含两个字段，即“slots”和“nodes”。

'slots' 字段是由该分片服务的插槽范围的列表，以一对表示范围的包含起始和结束插槽的整数存储。
例如，如果一个节点拥有插槽1、2、3、5、7、8和9，插槽范围将被存储为[1-3]、[5-5]、[7-9]。
因此，插槽字段将由以下整数列表表示。

```
1) 1) "slots"
   2) 1) (integer) 1
      2) (integer) 3
      3) (integer) 5
      4) (integer) 5
      5) (integer) 7
      6) (integer) 9
```

节点'nodes'字段包含了分片中的所有节点列表。
每个单独的节点是一个描述节点的属性映射。
一些属性是可选的，未来可能会添加更多属性。
当前属性列表：

* id：此节点的唯一节点ID。
* endpoint：用于访问节点的首选终端点，有关此字段可能的值的更多信息，请参见下文。
* ip：用于向此节点发送请求的IP地址。
* hostname（可选）：发送请求到此节点的公告主机名。
* port（可选）：节点的TCP（非TLS）端口。至少会出现port或tls-port中的一个。
* tls-port（可选）：节点的TLS端口。至少会出现port或tls-port中的一个。
* role：此节点的复制角色。
* replication-offset：此节点的复制偏移量。此信息可用于向最新的副本发送命令。
* health：可为`online`、`failed`或`loading`。此信息应用于确定应向哪些节点发送流量。`loading`健康状态用于知道节点当前不能服务流量，但将来可能能够服务。

端点和端口一起定义了客户端应该用来发送给定槽位的请求的位置。
端点的值为NULL表示节点的端点未知，客户端应该连接到与发送`CLUSTER SHARDS`命令相同的端点，但端口应该是命令返回的端口。
当Redis节点位于Redis不知道端点的负载均衡器之后时，这种未知的端点配置非常有用。
设置端点是由`cluster-preferred-endpoint-type`配置确定的。
空字符串`""`是端点字段的另一个异常值，ip字段也是如此，如果节点不知道自己的IP地址，则返回该值。
这可能发生在只有一个节点组成的群集中，或者该节点尚未加入群集的其他部分。
如果节点被错误地配置为使用已宣布的主机名但没有使用`cluster-announce-hostname`配置主机名，则显示值`?`。
客户端可以将空字符串视为NULL的同样方式，也就是与它用来发送当前命令的端点相同，而`"?"`应该被视为一个未知节点，不一定是为当前命令提供服务的同一节点。

```
> CLUSTER SHARDS
1) 1) "slots"
   2) 1) (integer) 0
      2) (integer) 5460
   3) "nodes"
   4) 1)  1) "id"
          2) "e10b7051d6bf2d5febd39a2be297bbaea6084111"
          3) "port"
          4) (integer) 30001
          5) "ip"
          6) "127.0.0.1"
          7) "endpoint"
          8) "127.0.0.1"
          9) "role"
         10) "master"
         11) "replication-offset"
         12) (integer) 72156
         13) "health"
         14) "online"
      2)  1) "id"
          2) "1901f5962d865341e81c85f9f596b1e7160c35ce"
          3) "port"
          4) (integer) 30006
          5) "ip"
          6) "127.0.0.1"
          7) "endpoint"
          8) "127.0.0.1"
          9) "role"
         10) "replica"
         11) "replication-offset"
         12) (integer) 72156
         13) "health"
         14) "online"
2) 1) "slots"
   2) 1) (integer) 10923
      2) (integer) 16383
   3) "nodes"
   4) 1)  1) "id"
          2) "fd20502fe1b32fc32c15b69b0a9537551f162f1f"
          3) "port"
          4) (integer) 30003
          5) "ip"
          6) "127.0.0.1"
          7) "endpoint"
          8) "127.0.0.1"
          9) "role"
         10) "master"
         11) "replication-offset"
         12) (integer) 72156
         13) "health"
         14) "online"
      2)  1) "id"
          2) "6daa25c08025a0c7e4cc0d1ab255949ce6cee902"
          3) "port"
          4) (integer) 30005
          5) "ip"
          6) "127.0.0.1"
          7) "endpoint"
          8) "127.0.0.1"
          9) "role"
         10) "replica"
         11) "replication-offset"
         12) (integer) 72156
         13) "health"
         14) "online"
3) 1) "slots"
   2) 1) (integer) 5461
      2) (integer) 10922
   3) "nodes"
   4) 1)  1) "id"
          2) "a4a3f445ead085eb3eb9ee7d8c644ec4481ec9be"
          3) "port"
          4) (integer) 30002
          5) "ip"
          6) "127.0.0.1"
          7) "endpoint"
          8) "127.0.0.1"
          9) "role"
         10) "master"
         11) "replication-offset"
         12) (integer) 72156
         13) "health"
         14) "online"
      2)  1) "id"
          2) "da6d5847aa019e9b9d2a8aa24a75f856fd3456cc"
          3) "port"
          4) (integer) 30004
          5) "ip"
          6) "127.0.0.1"
          7) "endpoint"
          8) "127.0.0.1"
          9) "role"
         10) "replica"
         11) "replication-offset"
         12) (integer) 72156
         13) "health"
         14) "online"
```
