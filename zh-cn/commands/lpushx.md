使用指定的值插入到存储在`key`中的列表的头部，仅当`key`已经存在并且保存的是一个列表时。
与`LPUSH`相反，当`key`尚不存在时不会执行任何操作。

@examples

```cli
LPUSH mylist "World"
LPUSHX mylist "Hello"
LPUSHX myotherlist "Hello"
LRANGE mylist 0 -1
LRANGE myotherlist 0 -1
```
