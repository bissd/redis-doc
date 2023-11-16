`LATENCY LATEST`命令报告最新记录的延迟事件。

每个报告的事件具有以下字段：

* 事件名称。
* 事件的最新延迟峰值的Unix时间戳。
* 事件的最新延迟，以毫秒为单位。
* 此事件的历史最大延迟。

"All-time" 表示自 Redis 实例启动以来的最大延迟，或者事件重置时的时间 `LATENCY RESET`。

@examples

```
127.0.0.1:6379> debug sleep 1
OK
(1.00s)
127.0.0.1:6379> debug sleep .25
OK
127.0.0.1:6379> latency latest
1) 1) "command"
   2) (integer) 1405067976
   3) (integer) 251
   4) (integer) 1001
```

有关更多信息，请参阅[延迟监控框架页面][lm]。

[lm]: /topics/延迟监控
