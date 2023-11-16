如果`newkey`不存在，将`key`重命名为`newkey`。
当`key`不存在时会返回一个错误。

在集群模式下，`key` 和 `newkey` 必须位于同一个**哈希槽**中，这意味着在实际操作中，只有具有相同哈希标记的键才能在集群中可靠地重命名。

以下是示例内容：

@examples

```cli
SET mykey "Hello"
SET myotherkey "World"
RENAMENX mykey myotherkey
GET myotherkey
```
