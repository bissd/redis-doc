为指定的事件生成ASCII艺术风格的图形。

`LATENCY GRAPH`通过最先进的可视化方式直观地展示了`event`的延迟趋势。在不用解析从`LATENCY HISTORY`或外部工具获取的原始数据之前，可以用它来快速把握情况。

`event` 的有效值有：
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
127.0.0.1:6379> latency reset command
(integer) 0
127.0.0.1:6379> debug sleep .1
OK
127.0.0.1:6379> debug sleep .2
OK
127.0.0.1:6379> debug sleep .3
OK
127.0.0.1:6379> debug sleep .5
OK
127.0.0.1:6379> debug sleep .4
OK
127.0.0.1:6379> latency graph command
command - high 500 ms, low 101 ms (all time high 500 ms)
--------------------------------------------------------------------------------
   #_
  _||
 _|||
_||||

11186
542ss
sss
```

每个图表列下面的垂直标签代表事件发生的秒数、分钟数、小时数或天数。例如，“15s”表示图表中的第一个事件发生在15秒前。

图表按照最小-最大刻度进行归一化处理，因此在底部一行中的下划线表示最小值，而在顶部一行中的#表示最大值。

有关更多信息，请参阅[延迟监控框架页面][lm]。

[lm]: /topics/latency-monitor
