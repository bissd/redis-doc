该命令返回存储在 `<key>` 的流中属于 `<groupname>` 消费者组的消费者列表。

下面为群组中每个消费者提供以下信息：

* **名称**: 消费者的名称
* **待处理**: PEL（挂起消息列表）中的条目数量: 代表已经传递但尚未确认的消息
* **空闲**: 自消费者上次尝试交互（例如：`XREADGROUP`、`XCLAIM`、`XAUTOCLAIM`）以来的毫秒数
* **不活跃**: 自消费者上次成功交互（例如：`XREADGROUP`实际将一些条目读入 PEL、`XCLAIM`/`XAUTOCLAIM`实际声明一些条目）以来的毫秒数

@examples

```
> XINFO CONSUMERS mystream mygroup
1) 1) name
   2) "Alice"
   3) pending
   4) (integer) 1
   5) idle
   6) (integer) 9104628
   7) inactive
   8) (integer) 18104698
2) 1) name
   2) "Bob"
   3) pending
   4) (integer) 1
   5) idle
   6) (integer) 83841983
   7) inactive
   8) (integer) 993841998
```
