`XTRIM` 根据需要，通过清除较旧的条目（具有较低的 ID）来修剪流。

修剪流可以使用以下策略之一进行操作：

* `MAXLEN`：只要流的长度超过指定的`threshold`，就会逐出条目，其中`threshold`是一个正整数。
* `MINID`：只要条目的ID低于`threshold`，就会逐出该条目，其中`threshold`是一个流ID。

例如，这将截取流的最新1000个项目：

```
XTRIM mystream MAXLEN 1000
```

在这个例子中，所有ID小于649085820-0的条目都将被清除：

```
XTRIM mystream MINID 649085820
```

默认情况下，或者当提供可选的 `=` 参数时，该命令执行精确修剪。

根据策略，精确修剪意味着：

* `MAXLEN`：修剪后的流的长度将恰好为其原始长度和指定的 `threshold` 之间的最小值。
* `MINID`：流中最旧的 ID 将恰好为其原始最旧 ID 和指定的 `threshold` 之间的最大值。

几乎完全修剪

由于精确的修剪可能需要Redis服务器额外的工作量，可以提供可选的`~`参数以使其更有效。

例如：

```
XTRIM mystream MAXLEN ~ 1000
```

`MAXLEN` 策略和 `threshold` 之间的 `~` 参数意味着用户请求修剪流的长度**至少**为 `threshold`，但可能略多。
在这种情况下，当性能提升时（例如，无法删除数据结构中的整个宏节点时），Redis 会提前停止修剪。
这使修剪更加高效，通常是您想要的，尽管在修剪之后，流可能会超过 `threshold` 有几十个额外的条目。

另一种控制使用`~`命令时工作量的方法是使用`LIMIT`子句。 
当使用时，它指定将被驱逐的最大条目数量。
当未指定`LIMIT`和`count`时，默认值为100乘以宏节点中条目数量将被隐式地用作`count`。
将值0指定为`count`将完全禁用限制机制。

@examples

```cli
XADD mystream * field1 A field2 B field3 C field4 D
XTRIM mystream MAXLEN 2
XRANGE mystream - +
```
