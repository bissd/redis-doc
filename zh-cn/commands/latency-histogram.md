`LATENCY HISTOGRAM`以直方图格式返回命令延迟的累计分布。

默认情况下，将返回所有可用的延迟直方图。
您可以通过提供特定的命令名称来筛选回复。

每个直方图包括以下字段：

* 命令名称
* 该命令的总调用次数
* 时间段的映射：
  * 每个时间段表示一个延迟范围
  * 每个时间段的范围是前一个时间段范围的两倍
  * 空的时间段不会包含在回复中
  * 跟踪的延迟在1微秒到大约1秒之间
  * 超过1秒的所有延迟都被视为+Inf
  * 最多会有log2(1,000,000,000)=30个时间段

该命令需要启用扩展延迟监控功能，默认已启用。
如果需要启用它，请调用 `CONFIG SET latency-tracking yes`。

为了删除延迟直方图数据，请使用 `CONFIG RESETSTAT` 命令。

@examples

```
127.0.0.1:6379> LATENCY HISTOGRAM set
1# "set" =>
   1# "calls" => (integer) 100000
   2# "histogram_usec" =>
      1# (integer) 1 => (integer) 99583
      2# (integer) 2 => (integer) 99852
      3# (integer) 4 => (integer) 99914
      4# (integer) 8 => (integer) 99940
      5# (integer) 16 => (integer) 99968
      6# (integer) 33 => (integer) 100000
```
