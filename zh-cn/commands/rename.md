将`key`重命名为`newkey`。
当`key`不存在时，返回错误。
如果`newkey`已经存在，则会进行覆盖。当发生覆盖时，`RENAME`执行隐式的`DEL`操作，所以如果被删除的键包含一个非常大的值，即使`RENAME`本身通常是一个常量时间操作，也可能引起高延迟。

在集群模式下，`key`和`newkey`必须位于相同的**哈希槽**中，这意味着实际上只有具有相同哈希标记的键才能在集群中可靠地重命名。

@examples

```cli
SET mykey "Hello"
RENAME mykey myotherkey
GET myotherkey
```

## 行为变更历史

* `>= 3.2.0`: 当源和目标名称相同时，该命令不再返回错误。
