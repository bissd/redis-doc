`LATENCY RESET` 命令重置所有或部分事件的延迟峰值时间序列。

当不带参数调用该命令时，将重置所有事件，丢弃当前记录的延迟峰值事件，并重置最大事件时间注册表。

可以通过提供特定的事件名称作为参数来重置只有特定事件。

“event” 的有效值包括：
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

如需更多信息，请参考[延迟监控框架页面][lm]。

[慢查询监控]: /topics/latency-monitor
