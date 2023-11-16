返回指定分片频道的订阅者数量。

请注意，可以在没有频道的情况下调用此命令，此时它将返回一个空列表。

集群节点注意事项：在Redis集群中，“PUBSUB”命令的回复仅报告节点的发布/订阅上下文中的信息，而不是整个集群的信息。

@examples

```
> PUBSUB SHARDNUMSUB orders
1) "orders"
2) (integer) 1
```
