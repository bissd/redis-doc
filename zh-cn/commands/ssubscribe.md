订阅客户端到指定的分片频道。

在Redis集群中，分片通道通过与分片键相同的算法分配到插槽。 
客户端可以订阅覆盖了插槽（主/副本）的节点以接收发布的消息。 
所有指定的分片通道都需要属于一个单独的插槽才能在给定的`SSUBSCRIBE`调用中进行订阅，
客户端可以通过分开的`SSUBSCRIBE`调用订阅不同插槽的通道。

有关分片 Pub/Sub 的更多信息，请参阅 [分片 Pub/Sub](/topics/pubsub#sharded-pubsub)。

@examples

```
> ssubscribe orders
Reading messages... (press Ctrl-C to quit)
1) "ssubscribe"
2) "orders"
3) (integer) 1
1) "smessage"
2) "orders"
3) "hello"
```
