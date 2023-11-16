---
title: Redis Pub/Sub
linkTitle: "Pub/sub"
weight: 40
description: 如何在Redis中使用发布/订阅频道
aliases:
  - /topics/pubsub
  - /docs/manual/pub-sub
  - /docs/manual/pubsub
---

`SUBSCRIBE`，`UNSUBSCRIBE`和`PUBLISH`实现了[Publish/Subscribe](http://en.wikipedia.org/wiki/Publish/subscribe)的消息传递范式，其中（引用维基百科）发送方（发布者）不被编程为将其消息发送给特定的接收方（订阅者）。
相反，发布的消息被归类到通道中，而不知道可能存在的订阅者。
订阅者对一个或多个通道表达兴趣，只接收感兴趣的消息，而不知道是否存在任何发布者。
发布者和订阅者的解耦允许更大的可扩展性和更动态的网络拓扑。

例如，要订阅频道 "channel11" 和 "ch:00"，客户端发出 `SUBSCRIBE` 命令，并提供频道名称：

```bash
SUBSCRIBE channel11 ch:00
```

其他客户端发送到这些频道的消息将由Redis推送给所有订阅的客户端。
订阅者按照消息发布的顺序接收这些消息。

客户端订阅一个或多个频道时不应发出命令，虽然它可以订阅和取消订阅其他频道。
订阅和取消订阅操作的回复以消息的形式发送，这样客户端就可以只读取一系列连贯的消息，其中的第一个元素指示消息的类型。
在已订阅 RESP2 客户端的上下文中允许使用的命令有：

* `PING`
* `PSUBSCRIBE`
* `PUNSUBSCRIBE`
* `QUIT`
* `RESET`
* `SSUBSCRIBE`
* `SUBSCRIBE`
* `SUNSUBSCRIBE`
* `UNSUBSCRIBE`


然而，如果使用 RESP3（见 `HELLO`），客户端可以在订阅状态下发出任何命令。

请注意，在使用 `redis-cli` 时，订阅模式下的命令如 `UNSUBSCRIBE` 和 `PUNSUBSCRIBE` 无法使用，因为 `redis-cli` 不会接受任何命令，只能通过 `Ctrl-C` 退出订阅模式。

## 传递语义学

Redis的Pub/Sub展示的是最多一次的消息传递语义。
顾名思义，这意味着一条消息要么会被传递一次，要么不传递。
一旦消息由Redis服务器发送，就不会再有机会发送。
如果订阅者无法处理消息（例如，由于错误或网络断开连接），消息将永远丢失。

如果您的应用程序需要更强的传递保证，请了解[Redis Streams](/docs/data-types/streams-tutorial)。
流中的消息是持久的，并支持_最多一次_和_最少一次_交付语义。

## 推送消息的格式

消息是一个具有三个元素的数组回复。

第一个元素是消息的类型：

* `subscribe`：表示我们成功订阅了回复中作为第二个元素给出的频道。
  第三个参数表示我们目前订阅的频道数量。

* `unsubscribe`：表示我们已成功取消订阅回复中给出的第二个元素所表示的频道。
  第三个参数表示我们当前订阅的频道数量。
  当最后一个参数为零时，表示我们不再订阅任何频道，并且客户端可以发出任何类型的Redis命令，因为我们已经退出了Pub/Sub状态。

* `message`: 它是由另一个客户端发出的`PUBLISH`命令所接收到的消息。
  第二个元素是消息来源频道的名称，第三个参数是实际的消息负载。

## 数据库和作用域

Pub/Sub与键空间没有关系。
它设计为不与任何级别的键空间干扰，包括数据库编号。

在 db 10 上发布的内容会被 db 1 上的订阅者收听到。

如果您需要某种类型的作用域，请在频道前面加上环境的名称（测试，暂存，生产...）。

## 电线协议示例

```
SUBSCRIBE first second
*3
$9
subscribe
$5
first
:1
*3
$9
subscribe
$6
second
:2
```

在此时，我们从另一个客户端对名为`second`的频道执行`PUBLISH`操作：

```
> PUBLISH second Hello
```

这是第一个客户端收到的内容：

```
*3
$7
message
$6
second
$5
Hello
```

现在客户端使用`UNSUBSCRIBE`命令且没有附加参数，自动取消订阅所有频道。

```
UNSUBSCRIBE
*3
$11
unsubscribe
$6
second
:1
*3
$11
unsubscribe
$5
first
:0
```

## 模式匹配订阅

Redis Pub/Sub实现支持模式匹配。
客户端可以订阅通配符样式的模式，以接收发送到与给定模式匹配的频道名称的所有消息。

例如：


```
PSUBSCRIBE news.*
```

将接收发送到频道`news.art.figurative`、`news.music.jazz`等的所有消息。
所有的glob样式模式都是有效的，因此支持多个通配符。

```
PUNSUBSCRIBE news.*
```

然后将客户取消订阅该模式。
此调用不会影响其他订阅。

使用模式匹配收到的消息以不同的格式发送：

* 消息类型为 `pmessage`: 它是从另一个客户端发出的`PUBLISH`命令导致的结果消息，匹配了一个模式订阅。
  第二个元素是原始匹配的模式，第三个元素是源频道的名称，最后一个元素是实际的消息负载。

类似于 `SUBSCRIBE` 和 `UNSUBSCRIBE`， `PSUBSCRIBE` 和 `PUNSUBSCRIBE` 命令也通过发送 `psubscribe` 和 `punsubscribe` 类型的消息来确认，并使用与 `subscribe` 和 `unsubscribe` 消息格式相同的格式。

## 匹配模式和频道订阅的消息

如果客户端订阅了与已发布消息匹配的多个模式，或者同时订阅了与该消息匹配的模式和频道，则可能会多次接收到同一条消息。
以下示例说明了这种情况：

```
SUBSCRIBE foo
PSUBSCRIBE f*
```

在上面的示例中，如果向频道`foo`发送消息，则客户端将收到两条消息：一条类型为`message`，一条类型为`pmessage`。

## 订阅计数的含义与模式匹配

在`subscribe`、`unsubscribe`、`psubscribe`和`punsubscribe`消息类型中，最后一个参数是订阅仍然活动的数量。
这个数字是客户端仍然订阅的通道和模式的总数。
所以当通过取消订阅所有通道和模式导致此计数降至零时，客户端将退出Pub/Sub状态。

## 分片发布/订阅

从Redis 7.0开始，引入了分片的Pub/Sub，其中分片通道通过与分配键槽使用的相同算法被分配到槽上。
必须将分片消息发送到拥有被哈希到的分片通道的槽的节点。
集群确保已发布的分片消息转发到分片中的所有节点，因此客户端可以通过连接到负责槽的主节点或任何副本之一来订阅分片通道。
使用`SSUBSCRIBE`、`SUNSUBSCRIBE`和`SPUBLISH`来实现分片的Pub/Sub。

Sharded Pub/Sub可帮助以集群模式扩展Pub/Sub的使用。它将消息的传播限定在集群的分片内。因此，与全局Pub/Sub相比，通过集群总线传输的数据量受到限制，而全局Pub/Sub中每条消息都传播到集群中的每个节点。这使用户可以通过添加更多分片来水平扩展Pub/Sub的使用。
 
## 编程示例

Pieter Noordhuis 使用 EventMachine 和 Redis 提供了一个很好的示例，用于创建 [一个多用户高性能的Web聊天](https://gist.github.com/pietern/348262)。

## 客户端库实现提示

由于所有收到的消息都包含原始订阅，导致消息传递（对于消息类型而言是通道，对于pmessage类型而言是原始模式），客户端库可以将原始订阅绑定到回调函数（可以是匿名函数、块、函数指针），使用哈希表。

当收到消息时，可以通过O(1)的查找操作将消息传递给已注册的回调函数。
