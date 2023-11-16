`LATENCY HISTORY` 命令返回 `event` 的延迟峰值时间序列的原始数据。

这对于希望获取原始数据以进行监控、显示图表等操作的应用程序非常有用。

命令将返回最多160个时间戳-延迟对，用于`event`事件。

`event` 的有效值包括：
* `active-defrag-cycle`
* `aof-fsync-always`
* `aof-stat`
* `aof-rewrite-diff-write`
* `aof-rename`
* `aof-write`
* `aof-write-active-child`
* `aof-write-alone`
* `aof-write-pending-fsync`
* `command`
* `expire-cycle`
* `eviction-cycle`
* `eviction-del`
* `fast-command`
* `fork`
* `rdb-unlink-temp-file`

@examples

```
127.0.0.1:6379> latency history command
1) 1) (integer) 1405067822
   2) (integer) 251
2) 1) (integer) 1405067941
   2) (integer) 1001
```

有关更多信息，请参阅[延迟监控框架页面][lm]。

[lm]: /topics/latency-monitor
