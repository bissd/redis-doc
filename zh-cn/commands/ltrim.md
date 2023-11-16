修剪现有的列表，使其仅包含指定范围的元素。
`start`和`stop`都是从零开始的索引，其中`0`是列表的第一个元素（头部），`1`是下一个元素，依此类推。

例如：`LTRIM foobar 0 2` 将修改存储在 `foobar` 中的列表，只保留列表的前三个元素。

`start` 和 `end` 也可以是负数，表示距离列表末尾的偏移量，其中 `-1` 是列表的最后一个元素，`-2` 是倒数第二个元素，依此类推。

超出范围的索引不会产生错误：如果 `start` 大于列表的末尾，或者 `start > end`，结果将是一个空列表（会导致 `key` 被移除）。
如果 `end` 大于列表的末尾，Redis 将把它当作列表的最后一个元素处理。

`LTRIM` can be used to remove elements from a list. The command takes a list key, a start index, and an end index as arguments. It removes all elements from the list that are not within the specified range and returns the length of the trimmed list.

Here's an example:

```
LPUSH mylist "element1"
LPUSH mylist "element2"
LPUSH mylist "element3"
LPUSH mylist "element4"
LPUSH mylist "element5"

LTRIM mylist 0 2
```

After running the above commands, the list `mylist` will contain the following elements:

```
1. "element5"
2. "element4"
3. "element3"
```

The `LTRIM` command ensures that only the elements with indices 0, 1, and 2 are kept in the list.

```
LPUSH mylist someelement
LTRIM mylist 0 99
```

这对命令将在列表中添加一个新元素，同时确保列表不会超过100个元素。
当使用Redis存储日志时，这非常有用。
重要的是要注意，以这种方式使用时，`LTRIM`是一个O(1)操作，因为在平均情况下，仅从列表的尾部删除一个元素。

@examples

```cli
RPUSH mylist "one"
RPUSH mylist "two"
RPUSH mylist "three"
LTRIM mylist 1 -1
LRANGE mylist 0 -1
```
