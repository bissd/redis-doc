使用`index`参数，将列表中的元素设置为`element`。
有关`index`参数的更多信息，请参阅`LINDEX`。

对于超出范围的索引返回错误。

@examples

```cli
RPUSH mylist "one"
RPUSH mylist "two"
RPUSH mylist "three"
LSET mylist 0 "four"
LSET mylist -2 "five"
LRANGE mylist 0 -1
```
