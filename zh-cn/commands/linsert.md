在存储在`key`位置的列表中，将`element`插入参考值`pivot`的前面或后面。

当`key`不存在时，被视为一个空列表，不进行任何操作。

当`key`存在但不保存列表值时，会返回错误。

@examples

```cli
RPUSH mylist "Hello"
RPUSH mylist "World"
LINSERT mylist BEFORE "World" "There"
LRANGE mylist 0 -1
```
