---
title: "Redis Streams"
linkTitle: "Streams"
weight: 60
description: >
    Redis Streams简介
aliases:
    - /topics/streams-intro
    - /docs/manual/data-types/streams    
    - /docs/data-types/streams-tutorial/ 
---

Redis流是一种数据结构，它像追加日志一样运作，但同时实现了一些克服传统追加日志的限制的操作。这些操作包括O(1)时间内的随机访问和复杂的消费策略，比如消费者组。
您可以使用流来记录和同时实时传输事件。
Redis流的使用案例示例包括：

* 事件溯源（例如跟踪用户操作、点击等）
* 传感器监控（例如野外设备的读数）
* 通知（例如将每个用户的通知记录存储在单独的流中）

Redis为每个流条目生成一个唯一ID。
您可以使用这些ID以后检索相关条目，或者读取并处理流中的所有后续条目。
请注意，因为这些ID与时间有关，所以此处显示的ID可能会有所变化，并且与您在自己的Redis实例中看到的ID不同。

Redis流支持多种修剪策略（以防止流无限增长），以及多个消费策略（请参阅`XREAD`、`XREADGROUP`和`XRANGE`）。

## 基本命令
* `XADD` 向流中添加新条目。
* `XREAD` 从给定位置开始向前按时间顺序读取一个或多个条目。
* `XRANGE` 返回两个给定条目ID之间的一系列条目。
* `XLEN` 返回流的长度。
 
请查看[完整的流命令列表](https://redis.io/commands/?group=stream)。


## 示例

* 当我们的赛车手通过一个检查点时，我们为每个赛车手添加一个流条目，其中包括赛车手的姓名、速度、位置和位置ID:
{{< clients-example stream_tutorial xadd >}}
> XADD race:france * rider Castilla speed 30.2 position 1 location_id 1
"1692632086370-0"
> XADD race:france * rider Norem speed 28.8 position 3 location_id 1
"1692632094485-0"
> XADD race:france * rider Prickett speed 29.7 position 2 location_id 1
"1692632102976-0"
{{< /clients-example >}}

* 读取从ID `1692632086370-0`开始的两个流条目：
{{< clients-example stream_tutorial xrange >}}
> XRANGE race:france 1692632086370-0 + COUNT 2
1) 1) "1692632086370-0"
   2) 1) "rider"
      2) "Castilla"
      3) "speed"
      4) "30.2"
      5) "position"
      6) "1"
      7) "location_id"
      8) "1"
2) 1) "1692632094485-0"
   2) 1) "rider"
      2) "Norem"
      3) "speed"
      4) "28.8"
      5) "position"
      6) "3"
      7) "location_id"
      8) "1"
{{< /clients-example >}}

* 从流的末尾开始阅读最多100条新的流入项，并在没有流入项被写入时，阻塞最多300毫秒：
{{< clients-example stream_tutorial xread_block >}}
> XREAD COUNT 100 BLOCK 300 STREAMS race:france $
(nil)
{{< /clients-example >}}

## 性能

将条目添加到流的时间复杂度是O(1)。
访问任意单个条目的时间复杂度是O(n)，其中_n_是ID的长度。
由于流ID通常是短且固定长度，这有效地减少为常数时间查找。
有关详细信息，请注意流是如何实现为[基数树](https://en.wikipedia.org/wiki/Radix_tree)的。

简单来说，Redis流提供了高效的插入和读取功能。
有关详细信息，请参阅每个命令的时间复杂度。


## 流基础

流是一种只能追加的数据结构。被称为`XADD`的基本写命令会将新条目追加到指定的流中。

每个流入项由一个或多个字段-值对组成，有些类似于字典或Redis哈希表：

{{< clients-example stream_tutorial xadd_2 >}}
> XADD race:france * rider Castilla speed 29.9 position 1 location_id 2
"1692632147973-0"
{{< /clients-example >}}

以上对`XADD`命令的调用将条目`rider: Castilla, speed: 29.9, position: 1, location_id: 2`添加到键`race:france`所对应的流中，使用的是由该命令返回的自动生成的条目ID，即`1692632147973-0`。它的第一个参数是键名`race:france`，第二个参数是标识流中每个条目的条目ID。然而，在这种情况下，我们传递了`*`，因为我们希望服务器为我们生成一个新的ID。每个新的ID都会单调递增，所以简单来说，每个新添加的条目的ID都会比过去的所有条目都高。服务器自动生成ID几乎总是你想要的，手动指定ID的原因非常罕见。我们稍后会详细讨论这一点。每个流条目都有一个ID，这与日志文件的相似之处在于，可以使用行号或文件中的字节偏移来标识特定的条目。回到我们的`XADD`示例，键名和ID之后，接下来的参数是组成我们流条目的字段-值对。

使用`XLEN`命令就可以获取流中项目的数量。

{{< clients-example stream_tutorial xlen >}}
> XLEN race:france
(integer) 4
{{< /clients-example >}}

### 条目 ID（Entry IDs）

`XADD`命令返回的条目ID用于唯一标识给定流中的每个条目，由两部分组成：

```
<millisecondsTime>-<sequenceNumber>
```

毫秒时间部分实际上是本地Redis节点生成流ID时的本地时间，然而如果当前毫秒时间恰好小于上一条目的时间，则使用上一条目的时间，因此如果时钟倒退，单调递增的ID属性仍然成立。序列号用于在同一毫秒内创建的条目。由于序列号是64位宽度的，在实际情况下，在同一毫秒内可生成的条目数量没有限制。

这样的ID格式一开始可能看起来很奇怪，温柔的读者可能会想为什么时间是ID的一部分。原因是Redis流支持通过ID进行范围查询。由于ID与生成条目的时间有关，这使得可以基本免费地查询时间范围。在讲解`XRANGE`命令时，我们将很快看到这一点。

如果由于某种原因用户需要与时间无关但与其他外部系统ID实际相关的增量ID，如前所述，`XADD` 命令可以使用显式ID而不是触发自动生成的 `*` 通配符ID，例如以下示例：

{{< clients-example stream_tutorial xadd_id >}}
> XADD race:usa 0-1 racer Castilla
0-1
> XADD race:usa 0-2 racer Norem
0-2
{{< /clients-example >}}

请注意，在这种情况下，最小的ID是0-1，并且命令不会接受相等或小于之前的ID。

{{< clients-example stream_tutorial xadd_bad_id >}}
> XADD race:usa 0-1 racer Prickett
(error) ERR The ID specified in XADD is equal or smaller than the target stream top item
{{< /clients-example >}}

如果你在运行Redis 7或更高版本，你还可以提供一个仅包含毫秒部分的显式ID。在这种情况下，ID的序列部分将自动生成。要实现这一点，请使用以下语法：

{{< clients-example stream_tutorial xadd_7 >}}
> XADD race:usa 0-* racer Prickett
0-3
{{< /clients-example >}}

## 从流中获取数据

现在我们终于能够通过`XADD`在我们的流中追加条目。然而，尽管追加数据到流中很明显，但用于提取数据的查询流的方法就不那么明显。如果我们继续使用日志文件的类比，一种明显的方式是模仿我们通常使用的Unix命令`tail -f`，即我们可以开始监听以获取追加到流中的新消息。注意，与Redis的阻塞列表操作不同，其中给定的元素将到达一个正在进行*弹出式*操作（如`BLPOP`）的单个客户端，使用流我们希望多个消费者看到追加到流中的新消息（就像许多`tail -f`进程可以看到添加到日志中的内容一样）。使用传统的术语来说，我们希望流能够将消息*分发*给多个客户端。

然而，这只是一种潜在的访问模式。我们还可以以完全不同的方式看待流：不是作为消息系统，而是作为*时间序列存储*。在这种情况下，获取新消息追加可能也很有用，但另一种自然的查询模式是按时间范围获取消息，或者使用游标迭代消息，以逐步检查所有历史记录。这绝对是另一种有用的访问模式。

如果我们从消费者的角度看待流，我们可能希望以另一种方式访问流，即作为一系列可以被分割成多个消费者处理的消息流，这样消费者组只能看到单个流中到达的消息子集。通过这种方式，可以将消息处理在不同的消费者之间扩展，而不需要单个消费者处理所有消息：每个消费者只会获取不同的消息进行处理。这基本上就是 Kafka（TM）与消费者组一起完成的工作。通过消费者组阅读消息是从 Redis Stream 中读取消息的另一种有趣方式。

Redis Streams支持上述的所有三种查询模式，通过不同的命令来实现。接下来的几节将展示它们，从最简单和最直接的使用方式开始：范围查询。

### 按范围查询：XRANGE 和 XREVRANGE

要通过范围查询流，我们只需要指定两个ID，*start*和*end*。返回的范围将包括具有start或end作为ID的元素，因此范围是包容的。两个特殊的ID“-”和“+”分别表示最小和最大的可能ID。

{{< clients-example stream_tutorial xrange_all >}}
> XRANGE race:france - +
1) 1) "1692632086370-0"
   2) 1) "rider"
      2) "Castilla"
      3) "speed"
      4) "30.2"
      5) "position"
      6) "1"
      7) "location_id"
      8) "1"
2) 1) "1692632094485-0"
   2) 1) "rider"
      2) "Norem"
      3) "speed"
      4) "28.8"
      5) "position"
      6) "3"
      7) "location_id"
      8) "1"
3) 1) "1692632102976-0"
   2) 1) "rider"
      2) "Prickett"
      3) "speed"
      4) "29.7"
      5) "position"
      6) "2"
      7) "location_id"
      8) "1"
4) 1) "1692632147973-0"
   2) 1) "rider"
      2) "Castilla"
      3) "speed"
      4) "29.9"
      5) "position"
      6) "1"
      7) "location_id"
      8) "2"
{{< /clients-example >}}

每个返回的条目是一个包含两个元素的数组：ID 和字段值对的列表。我们已经说过，条目的 ID 与时间有关，因为在 `-` 字符左侧的部分是创建流条目时本地节点的 Unix 时间（以毫秒为单位），而条目被创建的时间（但请注意，流是通过完整指定的 `XADD` 命令进行复制的，因此副本的 ID 与主节点相同）。这意味着我可以使用 `XRANGE` 查询一段时间范围。然而，为了这样做，我可能希望省略 ID 的序列部分：如果省略，在范围的起始部分将假定为 0，而在结束部分将假定为可用的最大序列号。这样，只使用两个毫秒级的 Unix 时间进行查询，我们可以得到在那个时间范围内生成的所有条目，以包含的方式。例如，如果我想查询一个两毫秒的时间段，我可以使用：

{{< clients-example stream_tutorial xrange_time >}}
> XRANGE race:france 1692632086369 1692632086371
1) 1) "1692632086370-0"
   2) 1) "rider"
      2) "Castilla"
      3) "speed"
      4) "30.2"
      5) "position"
      6) "1"
      7) "location_id"
      8) "1"
{{< /clients-example >}}

我在此范围中只有一个条目。然而在真实的数据集中，我可以查询不同小时的范围，或者可能会有许多项目仅在两毫秒内，而返回的结果可能非常庞大。因此，`XRANGE`在末尾支持一个可选的**COUNT**选项。通过指定一个计数，我可以只获取前面的*N*个项目。如果我想获取更多，我可以获取上次返回的最后一个ID，将序列部分加一，然后再次查询。让我们通过下面的示例来看一下。假设流`race:france`已经添加了4个项目。为了开始我的迭代，每个命令获取2个项目，我从整个范围开始，但计数为2。

{{< clients-example stream_tutorial xrange_step_1 >}}
> XRANGE race:france - + COUNT 2
1) 1) "1692632086370-0"
   2) 1) "rider"
      2) "Castilla"
      3) "speed"
      4) "30.2"
      5) "position"
      6) "1"
      7) "location_id"
      8) "1"
2) 1) "1692632094485-0"
   2) 1) "rider"
      2) "Norem"
      3) "speed"
      4) "28.8"
      5) "position"
      6) "3"
      7) "location_id"
      8) "1"
{{< /clients-example >}}

为了继续迭代下两个项，我必须选择最后返回的ID，即 `1692632094485-0`，并在其前添加前缀 `(`。因此，得到的独占范围区间，即在这种情况下为 `(1692632094485-0`，现在可以作为下一次 `XRANGE` 调用的新 *start* 参数。

{{< clients-example stream_tutorial xrange_step_2 >}}
> XRANGE race:france (1692632094485-0 + COUNT 2
1) 1) "1692632102976-0"
   2) 1) "rider"
      2) "Prickett"
      3) "speed"
      4) "29.7"
      5) "position"
      6) "2"
      7) "location_id"
      8) "1"
2) 1) "1692632147973-0"
   2) 1) "rider"
      2) "Castilla"
      3) "speed"
      4) "29.9"
      5) "position"
      6) "1"
      7) "location_id"
      8) "2"
{{< /clients-example >}}

既然我们从只有4个条目的流中取出了4个项目，如果我们尝试去取更多的项目，我们将得到一个空数组：

{{< clients-example stream_tutorial xrange_empty >}}
> XRANGE race:france (1692632147973-0 + COUNT 2
(empty array)
{{< /clients-example >}}

由于`XRANGE`命令的复杂度为*O(log(N))*用于搜索，然后*O(M)*用于返回M个元素，所以命令的时间复杂度是对数级的，这意味着迭代的每一步都很快速。因此，`XRANGE`也是事实上的*流迭代器*，不需要**XSCAN**命令。

命令`XREVRANGE`相当于`XRANGE`，但返回的元素顺序是反向的，因此`XREVRANGE`的一个实际用途是检查 Stream 中的最后一项：

{{< clients-example stream_tutorial xrevrange >}}
> XREVRANGE race:france + - COUNT 1
1) 1) "1692632147973-0"
   2) 1) "rider"
      2) "Castilla"
      3) "speed"
      4) "29.9"
      5) "position"
      6) "1"
      7) "location_id"
      8) "2"
{{< /clients-example >}}

注意，`XREVRANGE`命令的*start*和*stop*参数顺序是相反的。

使用XREAD监听新项目

当我们不想通过流中的范围访问项目时，通常我们需要的是订阅流中到达的新项目。这个概念可能与Redis的发布/订阅相关，你可以订阅一个频道，或者与Redis阻塞列表相关，在那里你等待一个键获取新元素来获取，但在你消费流的方式上存在根本的区别：

1. 一个流可以有多个等待数据的客户端（消费者）。默认情况下，每个新项都将传递给在给定流中等待数据的*每个消费者*。这种行为与阻塞列表不同，阻塞列表中每个消费者将获得不同的元素。然而，能够向多个消费者*分发*数据类似于发布/订阅。
2. 在发布/订阅中，消息是一种*发出即忘*的形式，永远不会存储，而在使用阻塞列表时，当客户端收到消息时，它会从列表中被*弹出*（实际上被移除）。而流以一种根本不同的方式工作。所有的消息都会不断地附加到流中（除非用户显式要求删除条目）：不同的消费者将通过记住上次接收到的消息的ID知道什么是新消息。
3. 流的消费者组提供了一种Pub/Sub或阻塞列表无法实现的控制级别，同一个流可以有不同的组，允许对已处理项进行显式确认，能够检查未处理项，申请未处理消息，并为每个单独的客户端提供一致的历史可见性，只能看到自己的消息历史。

`XREAD` 命令提供了监听流中新消息到达的能力。它比 `XRANGE` 稍微复杂一些，所以我们首先展示简单形式的使用方式，然后提供整个命令的布局。

{{< clients-example stream_tutorial xread >}}
> XREAD COUNT 2 STREAMS race:france 0
1) 1) "race:france"
   2) 1) 1) "1692632086370-0"
         2) 1) "rider"
            2) "Castilla"
            3) "speed"
            4) "30.2"
            5) "position"
            6) "1"
            7) "location_id"
            8) "1"
      2) 1) "1692632094485-0"
         2) 1) "rider"
            2) "Norem"
            3) "speed"
            4) "28.8"
            5) "position"
            6) "3"
            7) "location_id"
            8) "1"
{{< /clients-example >}}

以上是`XREAD`的非阻塞形式。请注意，**COUNT**选项不是必需的，实际上，该命令的唯一必需选项是**STREAMS**选项，它指定了一个键的列表，以及调用消费者已经看到的每个流的相应最大ID，以便命令仅为客户端提供大于我们指定的ID的消息。

在上述命令中，我们写入了 `STREAMS race:france 0`，这样我们就可以获取 Stream `race:france` 中大于 `0-0` 的所有消息。可以看到在上面的示例中，命令返回了键名，因为实际上可以同时从不同的流中调用此命令来读取数据。例如，我可以这样写: `STREAMS race:france race:italy 0 0`。注意，在 **STREAMS** 选项之后，我们需要提供键名，然后是 ID。因此，**STREAMS** 选项必须始终是最后一个选项。
任何其他选项必须在 **STREAMS** 选项之前。

除了`XREAD`可以同时访问多个流以外，并且我们可以指定我们拥有的最后一个ID来只获取更新的消息之外，在这个简单的表单中，该命令与`XRANGE`相比没有太大的区别。然而，有意思的是，我们可以通过指定**BLOCK**参数轻松地将`XREAD`转换为*阻塞命令*：

```
> XREAD BLOCK 0 STREAMS race:france $
```

请注意，在上面的示例中，除了删除**COUNT**外，我使用了超时设置为0毫秒的新**BLOCK**选项（表示永不超时）。此外，我传递了一个特殊ID `$`，而不是传递一个普通的ID给流 `mystream`。这个特殊ID意味着 `XREAD` 应该使用流`mystream`中已存储的最大ID作为最后的ID，这样我们只会接收到从我们开始监听的时间以来的*新*消息。这在某种程度上类似于Unix命令`tail -f`。

注意，当使用 **BLOCK** 选项时，我们不必使用特殊的ID `$`。我们可以使用任何有效的ID。如果命令能够立即满足我们的请求而不阻塞，它将这样做；否则它将会阻塞。通常情况下，如果我们想要从新条目开始消费流，我们会以ID `$` 开始，并在此之后继续使用最后接收到的消息的ID来进行下一次调用，如此类推。

`XREAD` 的阻塞形式也可以同时监听多个流，只需指定多个键名即可。如果请求可以同步处理，因为至少有一个流的元素大于我们指定的相应ID，它将返回结果。否则，该命令将会阻塞，并返回第一个获取新数据的流的项目（根据指定的ID）。

与阻塞列表操作类似，阻塞流读取从等待数据的客户端的角度来看是“公平”的，因为其语义是FIFO式的。对于给定流程而阻塞的第一个客户端，在有新条目可用时将首先被解除阻塞。

`XREAD` 没有其他选项，只有 **COUNT** 和 **BLOCK**，所以它是一个非常基本的命令，具有将消费者附加到一个或多个流的特定目的。使用消费者组 API 可用于消费流的更强大功能，但通过消费者组进行读取是通过一个名为 `XREADGROUP` 的不同命令实现的，在本指南的下一节中介绍。

## 消费者群体

当任务需要从不同的客户端消费相同的数据流时，`XREAD`已经提供了一种*扩展到N个客户端*的方法，还可以使用副本来提供更高的读取可伸缩性。然而，在某些问题中，我们想要做的不是向多个客户端提供相同的消息流，而是向多个客户端提供来自相同消息流的*不同子集*的消息。一个明显有用的情况是处理速度缓慢的消息：拥有N个不同的工作器接收数据流的不同部分，可以通过将不同的消息路由到准备做更多工作的不同工作者，以实现消息处理的可扩展性。

以实际情况为基础，假设我们有三个消费者C1，C2，C3，以及一个包含消息1、2、3、4、5、6、7的流。我们希望按照以下图表的方式提供消息：

```
1 -> C1
2 -> C2
3 -> C3
4 -> C1
5 -> C2
6 -> C3
7 -> C1
```

为了实现这一点，Redis使用了一个称为*消费者组*的概念。很重要的一点是需要理解，从实现的角度来看，Redis消费者组与Kafka(TM)消费者组没有任何关联。然而它们在功能上是相似的，所以我决定保留Kafka(TM)的术语，因为它最初是这个概念的流行者。

消费者组类似于从流中获取数据的伪消费者，实际上为多个消费者提供服务，并提供一定的保证：

1. 每个消息都会被分配给不同的消费者，因此不会有相同的消息被发送给多个消费者。
2. 在消费者组内，消费者通过一个名称来进行标识，该名称是一个区分大小写的字符串，实现消费者的客户端必须选择。这意味着即使在断开连接后，流消费者组仍保留所有状态，因为客户端将再次声明自己是同一个消费者。然而，这也意味着客户端必须提供唯一标识符。
3. 每个消费者组都有“从未被消费的第一个消息ID”的概念，因此当消费者请求新消息时，它可以仅提供之前未传递的消息。
4. 但是，消费消息需要使用特定命令进行明确的确认。Redis将确认解释为：这条消息已被正确处理，因此可以从消费者组中删除。
5. 消费者组跟踪当前挂起的所有消息，即已经传递给消费者组中某个消费者但尚未被确认处理的消息。由于这个特性，当访问流的消息历史时，每个消费者只会看到已经传递给它的消息。

在某种程度上，可以将消费者群体想象为有关流的某种*状态*的数量：

```
+----------------------------------------+
| consumer_group_name: mygroup           |
| consumer_group_stream: somekey         |
| last_delivered_id: 1292309234234-92    |
|                                        |
| consumers:                             |
|    "consumer-1" with pending messages  |
|       1292309234234-4                  |
|       1292309234232-8                  |
|    "consumer-42" with pending messages |
|       ... (and so forth)               |
+----------------------------------------+
```

如果你从这个角度来看，理解一个消费者组可以做什么，它如何能够仅提供消费者的待处理消息历史记录，并且消费者请求新消息时只会得到大于`last_delivered_id`的消息ID，就非常简单了。同时，如果你把消费者组看作是Redis流的辅助数据结构，很明显一个流可以有多个不同集合的消费者组。实际上，同一个流甚至可以有一些客户端通过`XREAD`读取而没有消费者组，而另一些客户端通过不同的消费者组通过`XREADGROUP`进行读取。

现在是时候放大来看基本的消费者群组命令了。它们如下所示：

*使用`XGROUP`命令来创建、销毁和管理消费者组。
*使用`XREADGROUP`命令通过消费者组从流中读取消息。
*使用`XACK`命令允许消费者将未决消息标记为已正确处理。

## 创建消费者群组

假设我已经有一个名为`race:france`的流类型键存在，为了创建一个消费者组，我只需要按照以下步骤进行操作：

{{< clients-example stream_tutorial xgroup_create >}}
> XGROUP CREATE race:france france_riders $
OK
{{< /clients-example >}}

正如您在上面的命令中所看到的，创建消费者组时我们必须指定一个ID，而在示例中只是使用了`$`。这是必需的，因为消费者组在首次连接时必须知道要提供哪条消息，也就是在消费者组刚刚创建时的*最后一条消息ID*。如果我们像上面那样指定`$`，那么只有从现在开始流中到达的新消息才会提供给消费者组中的消费者。如果我们指定`0`，那么消费者组将消费流历史中的*所有*消息来开始。当然，您可以指定任何其他有效的ID。您需要知道的是，消费者组将开始传递大于您指定ID的消息。因为`$`表示流中的当前最大ID，指定`$`将只消费新消息的效果。

`XGROUP CREATE` 还支持自动创建流，如果不存在的话，可以使用可选的 `MKSTREAM` 子命令作为最后一个参数：

{{< clients-example stream_tutorial xgroup_create_mkstream >}}
> XGROUP CREATE race:italy italy_riders $ MKSTREAM
OK
{{< /clients-example >}}

现在创建了消费者组，我们可以立即尝试使用`XREADGROUP`命令通过消费者组来读取消息。我们将从名为Alice和Bob的消费者读取消息，以查看系统如何将不同的消息返回给Alice或Bob。

`XREADGROUP`与`XREAD`非常相似，提供了相同的**BLOCK**选项，除此之外它是一个同步命令。但是它还有一个必需的选项，必须始终指定，即**GROUP**，它有两个参数：消费者组的名称，以及正在尝试读取的消费者的名称。选项**COUNT**也被支持，并且与`XREAD`中的选项相同。

我们将向比赛中添加骑手：意大利流，并尝试使用消费者组来读取一些内容：
注意：*这里的骑手是字段名，名称是相关联的值。记住，流项目是小字典。*

{{< clients-example stream_tutorial xgroup_read >}}
> XADD race:italy * rider Castilla
"1692632639151-0"
> XADD race:italy * rider Royce
"1692632647899-0"
> XADD race:italy * rider Sam-Bodden
"1692632662819-0"
> XADD race:italy * rider Prickett
"1692632670501-0"
> XADD race:italy * rider Norem
"1692632678249-0"
> XREADGROUP GROUP italy_riders Alice COUNT 1 STREAMS race:italy >
1) 1) "race:italy"
   2) 1) 1) "1692632639151-0"
         2) 1) "rider"
            2) "Castilla"
{{< /clients-example >}}

`XREADGROUP` 的回复与 `XREAD` 的回复一样。但请注意上述提供的 `GROUP <group-name> <consumer-name>`。它声明了我要使用消费者组 `mygroup` 以及我是消费者 `Alice`，每当消费者在消费者组中执行操作时，都必须指定其名称，以在组内唯一标识该消费者。

在上述命令行中还有另一个非常重要的细节，在必填的**STREAMS**选项之后，对于关键字`mystream`所请求的ID是特殊ID`>`。这个特殊ID仅在消费者组的上下文中有效，意思是：**迄今为止未发送给其他消费者的消息**。

这几乎总是您想要的，然而也可以指定一个真实的ID，比如`0`或任何其他有效的ID，在这种情况下，然而，发生的情况是我们从`XREADGROUP`请求仅提供给我们**待处理消息的历史记录**，并且在这种情况下，将永远看不到组中的新消息。所以基本上`XREADGROUP`根据我们指定的ID有以下行为：

* 如果ID是特殊ID`>`，则命令将仅返回到目前为止从未传递给其他消费者的新消息，并作为副作用更新消费者组的*最新ID*。
* 如果ID是任何其他有效数字ID，则命令将允许我们访问我们的*待处理消息历史记录*。也就是说，这是一组消息，这些消息已传递给指定的消费者（由提供的名称识别），但迄今尚未使用`XACK`确认。

我们可以立即测试这种行为，指定一个ID为0，不使用任何**COUNT**选项：我们只能看到唯一的待处理消息，即Castilla的消息:

{{< clients-example stream_tutorial xgroup_read_id >}}
> XREADGROUP GROUP italy_riders Alice STREAMS race:italy 0
1) 1) "race:italy"
   2) 1) 1) "1692632639151-0"
         2) 1) "rider"
            2) "Castilla"
{{< /clients-example >}}

然而，如果我们将信息视为已处理，它将不再作为待处理消息历史的一部分，因此系统将不再报告任何内容：

{{< clients-example stream_tutorial xack >}}
> XACK race:italy italy_riders 1692632639151-0
(integer) 1
> XREADGROUP GROUP italy_riders Alice STREAMS race:italy 0
1) 1) "race:italy"
   2) (empty array)
{{< /clients-example >}}

请不用担心，如果您还不知道`XACK`如何工作，其实这个想法就是处理过的消息不再是我们可以访问的历史的一部分。

现在轮到Bob朗读一些东西：

{{< clients-example stream_tutorial xgroup_read_bob >}}
> XREADGROUP GROUP italy_riders Bob COUNT 2 STREAMS race:italy >
1) 1) "race:italy"
   2) 1) 1) "1692632647899-0"
         2) 1) "rider"
            2) "Royce"
      2) 1) "1692632662819-0"
         2) 1) "rider"
            2) "Sam-Bodden"
{{< /clients-example >}}

Bob要求最多获得两条消息，并通过相同的组“mygroup”进行阅读。因此，Redis只报告*新的*消息。正如您所看到的，“Castilla”消息未被传递，因为它已经传递给了Alice，所以Bob获得了Royce和Sam-Bodden等等。

以这种方式，Alice，Bob和组内的其他消费者可以从同一流中读取不同的消息，读取它们尚未处理的消息历史，或将消息标记为已处理。这允许创建从流中消费消息的不同拓扑和语义。

有几件事需要记住：

* 首次提及时，消费者会自动创建，无需显式创建。
* 即使使用`XREADGROUP`，您也可以同时从多个键中读取数据。但是为了使其正常工作，需要在每个流中创建一个同名的消费者组。这并不是常见的需求，但值得一提的是，技术上是可行的。
* `XREADGROUP`是一个*写命令*，因为即使它从流中读取数据，但是作为读取的副作用，消费者组会发生修改，所以它只能在主实例上调用。

以下是一个使用消费者组的消费者实现示例，使用 Ruby 语言编写。即使不了解 Ruby 语言，这段 Ruby 代码对于任何有经验的程序员来说都很容易阅读：

```ruby
require 'redis'

if ARGV.length == 0
    puts "Please specify a consumer name"
    exit 1
end

ConsumerName = ARGV[0]
GroupName = "mygroup"
r = Redis.new

def process_message(id,msg)
    puts "[#{ConsumerName}] #{id} = #{msg.inspect}"
end

$lastid = '0-0'

puts "Consumer #{ConsumerName} starting..."
check_backlog = true
while true
    # Pick the ID based on the iteration: the first time we want to
    # read our pending messages, in case we crashed and are recovering.
    # Once we consumed our history, we can start getting new messages.
    if check_backlog
        myid = $lastid
    else
        myid = '>'
    end

    items = r.xreadgroup('GROUP',GroupName,ConsumerName,'BLOCK','2000','COUNT','10','STREAMS',:my_stream_key,myid)

    if items == nil
        puts "Timeout!"
        next
    end

    # If we receive an empty reply, it means we were consuming our history
    # and that the history is now empty. Let's start to consume new messages.
    check_backlog = false if items[0][1].length == 0

    items[0][1].each{|i|
        id,fields = i

        # Process the message
        process_message(id,fields)

        # Acknowledge the message as processed
        r.xack(:my_stream_key,GroupName,id)

        $lastid = id
    }
end
```

正如你所看到的，这里的想法是先消费历史记录，也就是我们待处理消息的列表。这是有用的，因为消费者可能在之前崩溃了，因此在重新启动时，我们希望重新读取已发送但未被确认的消息。请注意，我们可能会多次或一次处理一条消息（至少在消费者故障的情况下），但还涉及到Redis持久性和复制的限制，请参阅有关此主题的详细部分。

一旦历史消息被消耗完并且我们得到一个空的消息列表，我们可以切换到使用特殊的 `>` ID 来消耗新的消息。

## 从永久故障中恢复

上面的例子允许我们编写参与相同消费者组的消费者，每个消费者处理一部分消息，并在故障恢复时重新读取已经传递给它们的待处理消息。然而，在现实世界中，消费者可能永久性失败并且永远无法恢复。停止运行后，那些永远无法恢复的消费者的待处理消息会发生什么情况？

Redis消费者组提供了一种功能，用于在这些情况下*声明*给定消费者的待处理消息，以便这些消息的所有权发生变化，并重新分配给其他消费者。该功能非常明确。消费者必须检查待处理消息列表，并使用特殊命令声明特定消息，否则服务器将永远保留这些消息，并分配给旧消费者。这样，不同的应用程序可以选择是否使用此功能，以及如何使用它。

这个过程的第一步只是一个命令，用于提供消费者组中待处理条目的可观察性，命令叫做 `XPENDING`。
这是一个只读命令，始终可以安全调用，不会改变任何消息的所有权。
其最简单的形式是使用两个参数调用该命令，即流的名称和消费者组的名称。

{{< clients-example stream_tutorial xpending >}}
> XPENDING race:italy italy_riders
1) (integer) 2
2) "1692632647899-0"
3) "1692632662819-0"
4) 1) 1) "Bob"
      2) "2"
{{< /clients-example >}}

以这种方式调用时，该命令将输出消费者组中待处理消息的总数（此处为2），待处理消息中的较低和较高消息ID，最后是消费者列表和它们拥有的待处理消息数量。
我们只有Bob有两条待处理消息，因为Alice请求的单条消息已经使用`XACK`确认。

通过向`XPENDING`提供更多的参数，我们可以询问更多的信息，因为完整的命令签名如下所示：

```
XPENDING <key> <groupname> [[IDLE <min-idle-time>] <start-id> <end-id> <count> [<consumer-name>]]
```

通过提供起始和结束的ID（可以使用`XRANGE`中的`-`和`+`），以及一个用于控制命令返回的信息量的计数，我们可以更多地了解待处理的消息。可选的最后一个参数，消费者名称，如果我们只想限制输出为特定消费者的待处理消息，则可使用此功能，但在下面的示例中不使用此功能。

{{< clients-example stream_tutorial xpending_plus_minus >}}
> XPENDING race:italy italy_riders - + 10
1) 1) "1692632647899-0"
   2) "Bob"
   3) (integer) 74642
   4) (integer) 1
2) 1) "1692632662819-0"
   2) "Bob"
   3) (integer) 74642
   4) (integer) 1
{{< /clients-example >}}

现在我们有每条消息的详细信息：ID、消费者名称、*空闲时间*（以毫秒为单位，表示自上次将消息传递给某个消费者以来经过的毫秒数），以及给定消息传递的次数。
我们有两条来自Bob的消息，它们的空闲时间为60000+毫秒，大约一分钟。

请注意，我们可以使用`XRANGE`命令来检查第一条消息的内容，没有人会阻止我们这样做。

这是一个示例数据，仅供参考。

```
XRANGE race:italy 1692632647899-0 1692632647899-0
1) 1) "1692632647899-0"
   2) 1) "rider"
      2) "Royce"
```

我们只需将相同的ID在参数中重复两次。现在我们有了一些想法，Alice可能会决定，如果Bob在1分钟内没有处理消息，他可能不会迅速恢复，这时就是*认领*这些消息并代替Bob继续处理的时候了。为了这样做，我们使用`XCLAIM`命令。

该命令非常复杂，并且在其完整形式中有很多选项，因为它用于复制消费者组更改，但我们通常只会使用我们需要的参数。在这种情况下，它非常简单，如下所示：

```
XCLAIM <key> <group> <consumer> <min-idle-time> <ID-1> <ID-2> ... <ID-N>
```

基本上，我们说，对于特定的键和组合，我希望指定的消息ID将改变所有权，并分配给指定的消费者名称 `<consumer>`。然而，我们还提供了一个最小空闲时间，这样操作只会在所提到的消息的空闲时间大于指定的空闲时间时才起作用。这很有用，因为可能有两个客户端同时尝试声明一条消息。

```
Client 1: XCLAIM race:italy italy_riders Alice 60000 1692632647899-0
Client 2: XCLAIM race:italy italy_riders Lora 60000 1692632647899-0
```

然而，作为副作用，声明消息将重置其空闲时间，并增加其交付次数计数器，因此第二个客户端将无法声明该消息。通过这种方式，我们避免了消息的琐碎重新处理（即使在一般情况下，您不能获得完全一次的处理）。

这是命令执行的结果：

{{<clients-example stream_tutorial xclaim >}}
> XCLAIM race:italy italy_riders Alice 60000 1692632647899-0
1) 1) "1692632647899-0"
   2) 1) "骑手"
      2) "Royce"
{{< /clients-example >}}

Alice成功领取了消息，现在可以处理消息并予以确认，即使原始消费者没有恢复也可以推动事情发展。

上面的示例清楚地表明，成功声明给定消息的副作用是返回它。然而，这不是强制的。**JUSTID**选项可用于仅返回成功声明的消息ID。如果您想减少客户端和服务器之间使用的带宽（以及命令的性能），并且您对消息不感兴趣，因为您的使用者的实现方式会不时重新扫描悬而未决的消息历史记录，这将非常有用。

声明还可以通过一个单独的进程实现：该进程仅检查待处理消息的列表，并将空闲消息分配给看起来活动的消费者。可以使用Redis流的可观察性特征之一获取活动消费者。这是下一节的主题。

## 自动认领

在Redis 6.2中新增的`XAUTOCLAIM`命令实现了我们所描述的认领过程。
`XPENDING`和`XCLAIM`为不同类型的恢复机制提供了基本构建块。
该命令通过让Redis进行管理，优化了通用过程，并提供了大多数恢复需求的简单解决方案。

`XAUTOCLAIM` 识别空闲的待处理消息并将其所有权转给消费者。
命令的签名如下所示：

```
XAUTOCLAIM <key> <group> <consumer> <min-idle-time> <start> [COUNT count] [JUSTID]
```

所以，在上面的示例中，我可以使用自动认领来认领一个单独的消息，就像这样：

{{< clients-example stream_tutorial xgroup_read_id >}}
> XREADGROUP GROUP italy_riders Alice STREAMS race:italy 0
1) 1) "race:italy"
   2) 1) 1) "1692632639151-0"
         2) 1) "rider"
            2) "Castilla"
{{< /clients-example >}}

类似于 `XCLAIM` 命令，该命令返回一个包含已认领消息的数组，同时它还返回一个流ID，允许迭代待处理的条目。
流ID是一个游标，我可以在下一次调用中使用它，以继续认领空闲的待处理消息：

{{< clients-example stream_tutorial xack >}}
> XACK race:italy italy_riders 1692632639151-0
(integer) 1
> XREADGROUP GROUP italy_riders Alice STREAMS race:italy 0
1) 1) "race:italy"
   2) (empty array)
{{< /clients-example >}}

当`XAUTOCLAIM`将"0-0"流ID作为游标返回时，这意味着它已经达到了消费者组未处理条目列表的末尾。
这并不意味着没有新的空闲未处理消息，所以进程会继续从流的开头调用`XAUTOCLAIM`。

## 领取和交付柜台

您在XPENDING输出中观察到的计数器是每条消息的投递次数。计数器在两种情况下递增：通过XCLAIM成功认领消息或使用XREADGROUP调用以访问待处理消息的历史记录时。

当出现故障时，消息会被多次传递是正常的，但通常最终会被处理和确认。但是，由于某些特定消息被损坏或以某种触发处理代码错误的方式进行了制作，可能会出现处理问题。在这种情况下，消费者将持续失败以处理此特定消息。因为我们有交付尝试计数器，我们可以利用该计数器来检测由于某种原因而无法处理的消息。因此，一旦交付计数器达到您选择的一个较大的数值，最明智的做法可能是将此类消息放入另一个流并向系统管理员发送通知。这基本上是 Redis Streams 实现“死信（dead letter）”概念的方式。

## 流观察

缺乏可观察性的消息系统很难处理。不知道谁在消费消息、有哪些待处理的消息、在给定流中活跃的消费者群组集合，使一切都变得模糊不清。因此，Redis Streams 和消费者群组有不同的方式来观察发生的情况。我们已经涵盖了 `XPENDING`，它允许我们检查正在处理中的消息列表，以及它们的空闲时间和传递次数。

然而，我们可能希望做更多的事情，`XINFO` 命令是一个可观察性接口，可以与子命令一起使用，以便获取有关流或消费者组的信息。

此命令使用子命令来显示有关流和其消费者组状态的不同信息。例如，**XINFO STREAM <key>** 提供有关流本身的信息。

{{< clients-example stream_tutorial xinfo >}}
> XINFO STREAM race:italy
 1) "length"
 2) (integer) 5
 3) "radix-tree-keys"
 4) (integer) 1
 5) "radix-tree-nodes"
 6) (integer) 2
 7) "last-generated-id"
 8) "1692632678249-0"
 9) "groups"
10) (integer) 1
11) "first-entry"
12) 1) "1692632639151-0"
    2) 1) "rider"
       2) "Castilla"
13) "last-entry"
14) 1) "1692632678249-0"
    2) 1) "rider"
       2) "Norem"
{{< /clients-example >}}

输出显示有关流内部编码方式的信息，并显示流中的第一条和最后一条消息。另外，还可以获取与此流关联的消费者组数量的信息。我们可以进一步询问有关消费者组的更多信息。

{{< clients-example stream_tutorial xinfo_groups >}}
> XINFO GROUPS race:italy
1) 1) "name"
   2) "italy_riders"
   3) "consumers"
   4) (integer) 3
   5) "pending"
   6) (integer) 2
   7) "last-delivered-id"
   8) "1692632662819-0"
{{< /clients-example >}}

如您在本次和之前的输出中所见，`XINFO`命令会输出一系列的字段-值项。由于这是一个可观察性命令，它使得人类用户可以立即理解报告的信息，并且允许命令在将来通过添加更多字段而不破坏与旧客户端的兼容性来报告更多信息。而其他需要更高带宽效率的命令（比如`XPENDING`）只会报告信息而不包含字段名称。

上面示例的输出，使用了 **GROUPS** 命令，通过观察字段名称，应该是清晰的。我们可以通过检查已注册到该组的消费者来更详细地查看特定消费者组的状态。

{{< clients-example stream_tutorial xinfo_consumers >}}
> XINFO CONSUMERS race:italy italy_riders
1) 1) "name"
   2) "Alice"
   3) "pending"
   4) (integer) 1
   5) "idle"
   6) (integer) 177546
2) 1) "name"
   2) "Bob"
   3) "pending"
   4) (integer) 0
   5) "idle"
   6) (integer) 424686
3) 1) "name"
   2) "Lora"
   3) "pending"
   4) (integer) 1
   5) "idle"
   6) (integer) 72241
{{< /clients-example >}}

如果你不记得该命令的语法，只需向命令本身询问帮助：

```
> XINFO HELP
1) XINFO <subcommand> [<arg> [value] [opt] ...]. Subcommands are:
2) CONSUMERS <key> <groupname>
3)     Show consumers of <groupname>.
4) GROUPS <key>
5)     Show the stream consumer groups.
6) STREAM <key> [FULL [COUNT <count>]
7)     Show information about the stream.
8) HELP
9)     Prints this help.
```

## 与Kafka（TM）分区的区别

Redis streams的消费者组在某种程度上类似于基于分区的Kafka（TM）消费者组，但请注意，从实际角度来看，Redis streams是非常不同的。分区仅仅是*逻辑上*存在的，消息只是放入单个Redis键中，因此不同客户端的服务方式是基于谁准备好处理新消息，而不是从哪个分区读取。例如，如果消费者C3在某个时间点永久性失败，Redis将继续向C1和C2提供所有新到达的消息，就像现在只有两个*逻辑*分区一样。

同样地，如果一个给定的消费者在处理消息方面比其他消费者更快，那么该消费者将在同样的时间单位内获得比例上更多的消息。这是可能的，因为Redis显式地跟踪所有未确认的消息，并记住谁接收了哪条消息以及第一条消息从未传递给任何消费者的ID。

然而，这也意味着在Redis中，如果您确实想将同一流中的消息分区到多个Redis实例中，您必须使用多个键和一些分片系统，例如Redis集群或其他特定于应用程序的分片系统。单个Redis流不会自动分区为多个实例。

我们可以说，如下所示是符合逻辑的：

* 如果您使用1 个流-> 1 个消费者，则按顺序处理消息。
* 如果您使用N 个流和N 个消费者，以便仅有一个消费者命中N 个流的子集，您可以扩展上述模型1 个流-> 1 个消费者。
* 如果您使用1 个流-> N 个消费者，则对N 个消费者进行负载均衡，然而在这种情况下，同一逻辑项的消息可能会以不同的顺序被消费，因为一个给定的消费者可以比另一个消费者更快地处理消息3。

所以基本上，Kafka的分区更类似于使用N个不同的Redis键，而Redis的消费者群组是将给定流中的消息从服务器端负载均衡到N个不同的消费者的系统。

## 有上限的流

有许多应用程序不想将数据永远收集到流中。有时在流中最多存储指定数量的项目是有用的，其他时候一旦达到指定大小，将数据从Redis移动到非内存且不像Redis那样快速但适合用于存储长达几十年的历史数据的存储也是有用的。Redis流对此提供了一些支持。其中之一是`XADD`命令的**MAXLEN**选项。该选项非常简单使用：


{{< clients-example stream_tutorial maxlen >}}
> XADD race:italy MAXLEN 2 * rider Jones
"1692633189161-0"
> XADD race:italy MAXLEN 2 * rider Wood
"1692633198206-0"
> XADD race:italy MAXLEN 2 * rider Henshaw
"1692633208557-0"
> XLEN race:italy
(integer) 2
> XRANGE race:italy - +
1) 1) "1692633198206-0"
   2) 1) "rider"
      2) "Wood"
2) 1) "1692633208557-0"
   2) 1) "rider"
      2) "Henshaw"
{{< /clients-example >}}

使用**MAXLEN**，当达到指定长度时，旧的条目会自动被删除，这样流的大小就保持不变。目前没有选项告诉流仅保留不超过给定时间的条目，因为这样的命令为了保持一致性可能会长时间阻塞以删除条目。例如，想象一下如果插入有一个峰值，然后长时间暂停，再插入一个条目，都有相同的最大时间。在暂停期间，流将阻塞以删除过时的数据。因此，用户需要做一些规划并理解所需的最大流长度是多少。此外，流的长度与占用的内存成比例，通过时间进行剪裁则不那么容易控制和预测：它取决于插入速率，而插入速率通常会随时间改变（当不改变时，仅按大小剪裁是简单的）。

然而，使用**MAXLEN**修剪可能是昂贵的：流通过宏节点表示为基数树，以便非常节省内存。修改由几十个元素组成的单个宏节点是不理想的。因此，可以以以下特殊形式使用该命令：

```
XADD race:italy MAXLEN ~ 1000 * ... entry fields here ...
```

`~` 参数在 **MAXLEN** 选项和实际计数之间的作用是，我并不需要确切地保留 1000 个项目。可以是 1000、1010 或 1030，只要保证至少保存 1000 个项目。有了这个参数，只有在我们能够删除一个完整节点时才会执行修剪操作。这使得它更加高效，并且通常是你想要的。请注意这里客户端库有不同的实现。例如，Python 客户端默认为近似值，并且必须显式设置为真实长度。

还有`XTRIM`命令，它执行的操作与上面的**MAXLEN**选项非常相似，只不过它可以独立运行：

{{< clients-example stream_tutorial xtrim >}}
> XTRIM race:italy MAXLEN 10
(integer) 0
{{< /clients-example >}}

关于`XADD`选项：

{{< clients-example stream_tutorial xtrim2 >}}
> XTRIM mystream MAXLEN ~ 10
(integer) 0
{{< /clients-example >}}

然而，`XTRIM`的设计目的是接受不同的修剪策略。另一种修剪策略是**MINID**，它会清除比指定的ID更小的条目。

由于“XTRIM”是一个明确的命令，用户应该知道不同修剪策略可能存在的缺点。

将来可能添加到`XTRIM`的另一种有用的淘汰策略是根据ID范围进行删除，以便更方便地使用`XRANGE`和`XTRIM`将数据从Redis移动到其他存储系统。

## 流API中的特殊ID

你可能已经注意到在Redis API中有几个特殊的ID可以使用。以下是一个简短的复习，以便将来更容易理解。

首两个特殊ID分别为`-`和`+`，并与`XRANGE`命令一起用于范围查询。这两个ID分别表示最小可能ID（基本上是`0-1`）和最大可能ID（即`18446744073709551615-18446744073709551615`）。如你所见，使用`-`和`+`代替这些数字更加简洁。

接下来有一些API，我们想表达的是，在流中具有最大ID的项的ID。这就是`$`的含义。例如，如果我只想获取使用`XREADGROUP`的新条目，我可以使用此ID表示我已经具有所有现有的条目，但还没有将来要插入的新条目。类似地，当我创建或设置一个消费者组的ID时，我可以将最后交付的项目设置为`$`，以便将新条目传递给群组中的消费者。

正如您所看到的，$并不意味着+，它们是两个不同的东西，因为+是每个可能流中可能的最大ID，而$是给定流中包含给定条目的最大ID。此外，API通常只会理解+或$，但避免将给定符号加载多个含义很有用。

另一个特殊的 ID 是 `>`，它只与消费者群组有关，且仅在使用 `XREADGROUP` 命令时有效。这个特殊的 ID 表示我们只想获取那些在此之前从未被传递给其他消费者的条目。所以基本上，`>` ID 是一个消费者群组的*最后已传递 ID*。

最后特殊ID`*`只能与`XADD`命令一起使用，表示自动为新条目选择一个ID。

所以我们有`-`，`+`，`$`，`>`和`*`，每个符号都有不同的意义，大多数情况下，可以在不同的上下文中使用。

## 持久性，复制和消息安全性

流，就像任何其他Redis数据结构一样，会异步地复制到副本并持久化到AOF和RDB文件中。然而可能不太明显的是，消费者组的完整状态也会传播到AOF、RDB和副本中，因此如果主节点上有待处理的消息，副本也会有相同的信息。类似地，在重新启动后，AOF将还原消费者组的状态。

然而，请注意，Redis流和消费者组是使用Redis默认复制进行持久化和复制的，因此：

* 如果在您的应用程序中对消息的持久化很重要，则必须使用具有强制fsync策略的AOF。
* 默认情况下，异步复制不会保证“XADD”命令或消费者组状态更改的复制：故障转移后，根据副本从主节点接收数据的能力，可能会有一些数据丢失。
* 可以使用“WAIT”命令来强制将更改传播到一组副本。然而，请注意，虽然这使丢失数据的可能性非常低，但由Sentinel或Redis Cluster操作的Redis故障转移进程仅对升级为最新的副本进行最大努力检查，并且在某些特定的故障条件下可能会升级缺少一些数据的副本。

在使用Redis流和消费者组设计应用程序时，请确保理解应用程序在失败时应具有的语义属性，并相应地配置事物，评估是否对您的用例足够安全。

## 从流中移除单个项

流还有一个特殊命令，可以通过ID从流中删除项。对于只追加的数据结构来说，这可能看起来是一个奇怪的特性，但对于涉及隐私规定的应用程序而言，它实际上是有用的。该命令名为`XDEL`，接受流的名称，后跟要删除的ID：

{{< clients-example stream_tutorial xdel >}}
> XRANGE race:italy - + COUNT 2
1) 1) "1692633198206-0"
   2) 1) "rider"
      2) "Wood"
2) 1) "1692633208557-0"
   2) 1) "rider"
      2) "Henshaw"
> XDEL race:italy 1692633208557-0
(integer) 1
> XRANGE race:italy - + COUNT 2
1) 1) "1692633198206-0"
   2) 1) "rider"
      2) "Wood"
{{< /clients-example >}}

然而在当前的实现中，只有当一个宏节点完全为空时，内存才会真正被回收，因此您不应滥用这个功能。

## 零长度流

存储Redis数据结构之间的一个区别是，当其他数据结构没有元素时，调用删除元素的命令会将key本身删除。因此，例如，当调用`ZREM`删除有序集合中的最后一个元素时，有序集合将完全被删除。另一方面，流允许保持零个元素，既可以通过使用**MAXLEN**选项并将计数设置为零（使用`XADD`和`XTRIM`命令），也可以通过调用`XDEL`命令实现。

存在这种不对称的原因是因为流可能有关联的消费者组，而我们不希望仅仅因为流中没有任何项目，就丢失消费者组定义的状态。目前，即使流没有关联的消费者组，它也不会被删除。

## 消费消息的总延迟

像`XRANGE`和`XREAD` 或者没有BLOCK选项的`XREADGROUP`等非阻塞流命令和任何其他Redis命令一样以同步方式进行服务，因此讨论此类命令的延迟是没有意义的：更有意义的是检查Redis文档中这些命令的时间复杂度。可以说，当提取范围时流命令至少与有序集合命令一样快，并且如果使用流水线传输方式，`XADD`非常快，平均情况下每秒可以轻松插入50万到100万条数据。

然而，延迟在理解处理消息的延迟时变得很有意义，尤其是在消费者组中阻塞消费者的情况下。从消息通过 `XADD` 产生到通过 `XREADGROUP` 接收消息返回给消费者的这段时间。

## 如何处理被封锁的消费者

在提供执行测试结果之前，了解Redis在路由流消息方面使用的模型以及实际上如何管理等待数据的任何阻塞操作将是有趣的。

* 阻塞客户端被引用在一个哈希表中，该哈希表将至少有一个阻塞消费者的键映射为等待该键的消费者列表。这样，给定一个接收到数据的键，我们可以解析出所有等待该数据的客户端。
* 当发生写操作时，例如调用`XADD`命令时，它会调用`signalKeyAsReady()`函数。这个函数会将键放入一个需要处理的键列表中，因为这些键可能有新的数据用于阻塞消费者。注意，这些*准备好的键*将在稍后处理，所以在同一事件循环周期内，键可能会接收到其他写操作。
* 最后，在返回到事件循环之前，*准备好的键*会被最终处理。对于每个键，会扫描等待数据的客户端列表，如果适用，这些客户端将接收到新到达的数据。对于流，数据是消费者请求的适用范围内的消息。

正如您所见，在返回到事件循环之前，调用 `XADD` 的客户端和阻塞消费消息的客户端在输出缓冲区中都会有它们的回复，因此调用 `XADD` 的调用方应该在消费者接收到新消息时大致同时收到Redis的回复。

这个模型是*推送式*的，因为通过调用`XADD`来直接将数据添加到消费者的缓冲区中，所以延迟往往是相当可预测的。

## 延迟测试结果

为了检查这些延迟特性，使用多个Ruby程序实现了一个测试，推送带有计算机毫秒时间作为附加字段的消息，然后由Ruby程序从消费者组中读取消息并进行处理。消息处理步骤包括将当前计算机时间与消息时间戳进行比较，以了解总延迟时间。

获得的结果：

```
Processed between 0 and 1 ms -> 74.11%
Processed between 1 and 2 ms -> 25.80%
Processed between 2 and 3 ms -> 0.06%
Processed between 3 and 4 ms -> 0.01%
Processed between 4 and 5 ms -> 0.02%
```

99.9％的请求的延迟≤2毫秒，剩下的异常值也非常接近平均值。

在流中添加几百万未被确认的消息并不改变基准测试的核心，大多数查询仍然以非常短的延迟进行处理。

几点说明：

* 在这里，我们每次迭代处理多达10,000条消息，这意味着`XREADGROUP`的`COUNT`参数被设置为10000。这会增加很多延迟，但是为了让慢消费者能够跟上消息流，这是必需的。因此，您可以期望真实世界的延迟要小得多。
* 这个基准测试所使用的系统与现今的标准相比非常慢。




## 了解更多

* [Redis Streams 教程](/docs/data-types/streams-tutorial) 通过许多例子解释了 Redis Streams。
* [Redis Streams 详解](https://www.youtube.com/watch?v=Z8qcpXyMAiA) 是一个有趣的 Redis Streams 入门视频。
* [Redis University RU202](https://university.redis.com/courses/ru202/) 是一门免费的在线课程，专门讲授 Redis Streams。
