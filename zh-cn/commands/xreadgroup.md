`XREADGROUP` 命令是 `XREAD` 命令的特殊版本，支持消费者组。在阅读本页面之前，可能需要先理解 `XREAD` 命令才能理解其含义。

此外，如果您对流的使用还不熟悉，我们建议阅读我们的[Redis Streams简介](/topics/streams-intro)。
请确保在介绍中了解消费者组的概念，以便更容易理解此命令的工作原理。

## 30秒了解消费者群体情况

这个命令和普通的 `XREAD` 命令的区别在于，它支持消费者组。

没有消费者组，只使用`XREAD`命令，所有客户端都会收到流中所有到达的条目。而使用`XREADGROUP`命令以消费者组的方式，可以创建消费流中不同部分消息的客户端群组。例如，如果流中有条目A、B和C，同时有两个使用消费者组方式读取消息的客户端，其中一个客户端将会获得消息A和C，另一个客户端将会获得消息B，依此类推。

在消费者组内，给定的消费者（即，仅仅是从流中消费消息的客户端）必须使用唯一的*消费者名称*来进行标识。这个名称只是一个字符串。

消费者群体之一的保证是，给定的消费者只能看到传递给它的消息历史，因此一条消息只有一个所有者。然而，有一种特殊的功能称为*消息认领*，允许其他消费者在某些消费者无法恢复的故障发生时认领消息。为了实现这样的语义，消费者群体需要对消费者成功处理的消息进行显式确认，使用`XACK`命令。这是必需的，因为流将跟踪每个消费者组正在处理的消息。

这是如何理解何时使用消费者组的方式：

1. 如果您有一个流和多个客户端，并且您希望所有客户端都获取所有消息，则不需要消费者组。
2. 如果您有一个流和多个客户端，并且希望将流*分区*或*分片*到您的客户端，以便每个客户端都获取流中到达的子消息集，您需要一个消费者组。

## XREAD 和 XREADGROUP 的区别

从语法角度来看，这些命令几乎是相同的，
但是 `XREADGROUP` *需要* 一个特殊且强制的选项：

    GROUP <group-name> <consumer-name>

组名只是与流相关联的消费者组的名称。
使用`XGROUP`命令创建组。消费者名称是客户端用于在组内标识自己的字符串。
消费者在首次被检测到时会自动在消费者组内创建。不同的客户端应选择不同的消费者名称。

当您使用`XREADGROUP`读取时，服务器将*记住*某个特定的消息已经传递给您：该消息将被存储在消费者组中的一个叫做Pending Entries List (PEL)的列表中，这是一组已经传递但尚未被确认的消息ID。

客户端需要使用`XACK`确认消息处理，以便将待处理的条目从PEL中移除。可以使用`XPENDING`命令检查PEL。

`NOACK`子命令可用于在不需要可靠性并且偶尔丢失消息可接受的情况下，避免将消息添加到PEL中。这相当于在读取消息时确认该消息。

使用 `XREADGROUP` 时，在 **STREAMS** 选项中指定的 ID 可以是以下两者之一：

* 特殊的 `>` ID 表示消费者仅希望接收之前没有发送给其他消费者的消息。这意味着只给我新消息。
* 任何其他的 ID，比如 0 或者任何有效的 ID 或者不完整的 ID（只有毫秒时间部分），都会返回那些等待消费者发送命令的 ID 大于提供的 ID 的条目。所以基本上，如果 ID 不是 `>`，则该命令只会让客户端访问其待处理的条目：已发送给它但尚未确认的消息。请注意，在这种情况下，`BLOCK` 和 `NOACK` 都会被忽略。

类似于`XREAD`命令，`XREADGROUP`命令也可以以阻塞的方式使用。在这方面没有任何区别。

当消息传递给消费者时会发生什么？

两件事情：

1. 我们需要制定一个全面的市场营销策略。这将包括市场调研、目标客户分析、竞争对手分析，以及定制化的推广和广告计划。

2. 我们需要与供应商进行谈判，以获得更有利的价格和条款。这将包括价格谈判、合同条款的讨论，以及与供应商的长期合作计划。

1. 如果消息从未传递给任何人，即如果我们在谈论一条新消息，则创建一个PEL（待处理条目列表）。
2. 如果消息已经传递给了此消费者，并且它只是再次获取相同的消息，则"last delivery counter"会更新为当前时间，并且"number of deliveries"会增加一。您可以使用`XPENDING`命令访问这些消息属性。

## 使用示例

通常情况下，你可以使用以下命令来获取新的消息并处理它们。伪代码如下：

```
WHILE true
    entries = XREADGROUP GROUP $GroupName $ConsumerName BLOCK 2000 COUNT 10 STREAMS mystream >
    if entries == nil
        puts "Timeout... try again"
        CONTINUE
    end

    FOREACH entries AS stream_entries
        FOREACH stream_entries as message
            process_message(message.id,message.fields)

            # ACK the message as processed
            XACK mystream $GroupName message.id
        END
    END
END
```

以这种方式，示例的消费者代码将仅获取新消息，处理它们，并通过 `XACK` 确认。然而，上面的示例代码不完整，因为它没有处理崩溃后的恢复。如果我们在处理消息的中间发生崩溃，那么我们的消息将保留在待处理列表中，因此我们可以通过最初给 `XREADGROUP` 一个 ID 0，并执行相同的循环来访问我们的历史记录。一旦提供了 ID 0，回复为空的消息集，我们就知道我们已经处理和确认了所有待处理的消息：我们可以开始使用 `>` 作为 ID，以获取新消息并重新加入正在处理新事物的消费者。

请查看`XREAD`命令页面，以查看命令的实际回复方式。

## 当待处理的消息被删除时会发生什么？

流中的条目可能因为修剪或显式调用 `XDEL` 而被随时删除。
按照设计，Redis 并不阻止从流的 PELs 中删除条目。
当发生这种情况时，PELs 会保留已删除的条目的 ID，但实际的条目负载不再可用。
因此，当读取这些 PEL 条目时，Redis 将返回 null 值，而不是它们的实际数据。

示例：

```
> XADD mystream 1 myfield mydata
"1-0"
> XGROUP CREATE mystream mygroup 0
OK
> XREADGROUP GROUP mygroup myconsumer STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) "1-0"
         2) 1) "myfield"
            2) "mydata"
> XDEL mystream 1-0
(integer) 1
> XREADGROUP GROUP mygroup myconsumer STREAMS mystream 0
1) 1) "mystream"
   2) 1) 1) "1-0"
         2) (nil)
```

阅读[Redis Streams 简介](/topics/streams-intro) 是为了更好地理解流的整体行为和语义而强烈建议的。
