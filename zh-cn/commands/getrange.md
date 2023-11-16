存储在 `key` 中的字符串值的子字符串，该子字符串由偏移量 `start` 和 `end`（两者都包括）确定。
可以使用负偏移量来提供从字符串末尾开始的偏移量。
因此，-1 表示最后一个字符，-2 表示倒数第二个字符，依此类推。

函数通过将结果范围限制在字符串的实际长度以处理超出范围的请求。

@examples

```cli
SET mykey "This is a string"
GETRANGE mykey 0 3
GETRANGE mykey -3 -1
GETRANGE mykey 0 -1
GETRANGE mykey 10 100
```
