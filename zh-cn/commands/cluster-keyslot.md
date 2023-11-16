返回一个整数，标识指定键散列到的哈希槽。
此命令主要用于调试和测试，因为它通过API暴露了Redis哈希算法的底层实现。
此命令的示例用途包括：

1. 客户端库可以使用Redis来测试它们自己的哈希算法，生成随机的键并使用它们自己的实现和Redis的`CLUSTER KEYSLOT`命令进行哈希，然后检查结果是否相同。
2. 人们可以使用该命令来检查给定键的哈希槽和关联的Redis集群节点。

## 示例

```
> CLUSTER KEYSLOT somekey
(integer) 11058
> CLUSTER KEYSLOT foo{hash_tag}
(integer) 2515
> CLUSTER KEYSLOT bar{hash_tag}
(integer) 2515
```

请注意，该命令实现了完整的哈希算法，包括**哈希标签**的支持，这是 Redis 集群键哈希算法的特殊属性。 如果在键名中找到此类模式则仅哈希 `{` 和 `}` 之间的内容，以便强制多个键由同一节点处理。
