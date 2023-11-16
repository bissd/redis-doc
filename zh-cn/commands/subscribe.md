订阅客户端到指定的频道。

一旦客户端进入订阅状态，除了额外的 SUBSCRIBE、SSUBSCRIBE、PSUBSCRIBE、UNSUBSCRIBE、SUNSUBSCRIBE、PUNSUBSCRIBE、PING、RESET 和 QUIT 命令外，不应该发出任何其他命令。
然而，如果使用 RESP3（参见 HELLO），客户端在订阅状态下可以发出任何命令。

有关更多信息，请参阅[发布/订阅](/docs/interact/pubsub/)。

## 行为改变历史

*   `>= 6.2.0`: 可以调用`RESET`退出订阅状态。
