在Redis集群中，每个节点与集群中的每个对等节点都维护着一对长连接的TCP链路：一个用于向对等节点发送出站消息，一个用于接收对等节点的入站消息。

`CLUSTER LINKS` 输出所有对等链接信息的数组，其中每个数组元素都是一个包含每个链接的属性和值的映射。

@examples

以下是一个示例输出：

```
> CLUSTER LINKS
1)  1) "direction"
    2) "to"
    3) "node"
    4) "8149d745fa551e40764fecaf7cab9dbdf6b659ae"
    5) "create-time"
    6) (integer) 1639442739375
    7) "events"
    8) "rw"
    9) "send-buffer-allocated"
   10) (integer) 4512
   11) "send-buffer-used"
   12) (integer) 0
2)  1) "direction"
    2) "from"
    3) "node"
    4) "8149d745fa551e40764fecaf7cab9dbdf6b659ae"
    5) "create-time"
    6) (integer) 1639442739411
    7) "events"
    8) "r"
    9) "send-buffer-allocated"
   10) (integer) 0
   11) "send-buffer-used"
   12) (integer) 0
```

每个地图由相应群集链接的以下属性及其值组成：

1. `direction`：此链接由本地节点`to`对等方建立，或由本地节点`from`对等方接受。
2. `node`：对等方的节点ID。
3. `create-time`：链接的创建时间。（在`to`链接的情况下，这是本地节点创建TCP链接的时间，而不是实际建立的时间。）
4. `events`：当前为链接注册的事件。 `r`表示可读事件，`w`表示可写事件。
5. `send-buffer-allocated`：链接发送缓冲区的分配大小，用于缓冲向对等方发送的消息。
6. `send-buffer-used`：链接发送缓冲区当前持有数据（消息）的大小。
