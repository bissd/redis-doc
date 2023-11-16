移除并返回存储在 `key` 处的列表的第一个元素。

默认情况下，该命令从列表的开头弹出一个元素。
当提供可选的 `count` 参数时，答复将包含最多 `count` 个元素，取决于列表的长度。

@examples

```cli
RPUSH mylist "one" "two" "three" "four" "five"
LPOP mylist
LPOP mylist 2
LRANGE mylist 0 -1
```
