从 `source` 集合中将 `member` 移动到 `destination` 集合。
此操作是原子操作。
在任何给定的时刻，元素将在其他客户端的 `source` **或** `destination` 中出现。

如果源集合不存在或不包含指定元素，则不执行操作并返回`0`。
否则，该元素将从源集合中移除并添加到目标集合中。
当指定元素已经存在于目标集合中时，它仅从源集合中移除。

如果`source`或`destination`没有设置值，将返回错误。

@examples

```cli
SADD myset "one"
SADD myset "two"
SADD myotherset "three"
SMOVE myset myotherset "two"
SMEMBERS myset
SMEMBERS myotherset
```
