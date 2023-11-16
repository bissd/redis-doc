从存储在`key`中的列表中移除并返回最后一个元素。

默认情况下，该命令从列表末尾弹出一个元素。
当提供可选的 "count" 参数时，答复的元素数将取决于列表的长度，最多为 "count" 个。

@examples

```cli
RPUSH mylist "one" "two" "three" "four" "five"
RPOP mylist
RPOP mylist 2
LRANGE mylist 0 -1
```
