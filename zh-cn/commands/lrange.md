返回存储在 `key` 的列表中指定的元素。
`start` 和 `stop` 是基于零的索引，其中 `0` 是列表的第一个元素（列表的头），`1` 是下一个元素，依此类推。

这些偏移量也可以是负数，表示从列表的末尾开始的偏移量。
例如，`-1` 是列表的最后一个元素，`-2` 是倒数第二个元素，依此类推。

## 与各种编程语言中的范围函数一致性

请注意，如果您有一个从0到100的数字列表，`LRANGE list 0 10`将返回11个元素，也就是说，右边的项目被包括在内。
这**可能也可能不**与您选择的编程语言中与范围相关的函数的行为一致（比如Ruby的`Range.new`，`Array#slice`或Python的`range()`函数）。

## 超出范围的索引

超出范围的索引不会产生错误。
如果`start`大于列表的结尾，将返回一个空列表。
如果`stop`大于实际列表的结束，Redis将将其视为列表的最后一个元素。

@examples

```cli
RPUSH mylist "one"
RPUSH mylist "two"
RPUSH mylist "three"
LRANGE mylist 0 0
LRANGE mylist -3 2
LRANGE mylist -100 100
LRANGE mylist 5 10
```
