从存储在`key`中的列表中删除第一个等于`element`的`count`次出现。

`count`参数以以下方式影响操作：

* `count > 0`：从头到尾移除等于 `element` 的元素。
* `count < 0`：从尾到头移除等于 `element` 的元素。
* `count = 0`：移除所有等于 `element` 的元素。

举个例子，`LREM list -2 "hello"` 将会从存储在`list`的列表中删除最后两次出现的`"hello"`。

请注意，不存在的键被视为空列表，所以当`key`不存在时，该命令将始终返回`0`。

@examples

```cli
RPUSH mylist "hello"
RPUSH mylist "hello"
RPUSH mylist "foo"
RPUSH mylist "hello"
LREM mylist -2 "hello"
LRANGE mylist 0 -1
```
