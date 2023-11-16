列出当前**活跃的分片频道**。

一个活动的 shard 频道是一个具有一个或多个订阅者的 Pub/Sub shard 频道。

如果没有指定`pattern`，则列出所有的频道；否则，仅列出与指定的glob样式模式匹配的频道。

有关活动的分片通道的返回信息是在分片级别而不是集群级别。

@examples

```
> PUBSUB SHARDCHANNELS
1) "orders"
> PUBSUB SHARDCHANNELS o*
1) "orders"
```
