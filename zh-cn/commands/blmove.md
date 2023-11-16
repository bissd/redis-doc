`BLMOVE` 是 `LMOVE` 的阻塞变体。当 `source` 包含元素时，该命令与 `LMOVE` 的行为完全相同。在 `MULTI`/`EXEC` 块内部使用时，该命令与 `LMOVE` 的行为完全相同。当 `source` 为空时，Redis 将阻塞连接，直到另一个客户端向其推送数据，或者达到指定的 `timeout`（以秒为单位的双精度数，指定阻塞的最大秒数）。`timeout` 的值为零表示无限期阻塞。

这个命令是替代了现已废弃的 `BRPOPLPUSH`。执行 `BLMOVE RIGHT LEFT` 等效。

请参阅 "LMOVE" 获取更多信息。

## 模式：可靠队列

请参阅`LMOVE`文档中的模式描述。

## 模式：循环列表

请参阅`LMOVE`文档中的模式说明。
