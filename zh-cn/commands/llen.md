存储在 `key` 上的列表的长度。
如果 `key` 不存在，则它被解释为空列表，返回 `0`。
当存储在 `key` 上的值不是列表时，返回错误。

@examples

```cli
LPUSH mylist "World"
LPUSH mylist "Hello"
LLEN mylist
```
