从`key`存储的集合中随机移除并返回一个或多个成员。

此操作类似于 `SRANDMEMBER`，从集合中返回一个或多个随机元素，但不会将其移除。

默认情况下，该命令从集合中弹出单个成员。当提供可选的`count`参数时，回复将包含最多`count`个成员，取决于集合的基数。

@examples

```cli
SADD myset "one"
SADD myset "two"
SADD myset "three"
SPOP myset
SMEMBERS myset
SADD myset "four"
SADD myset "five"
SPOP myset 3
SMEMBERS myset
```
## 返回元素的分布

请注意，在需要保证返回元素均匀分布的情况下，该命令可能不适用。有关 `SPOP` 使用的算法的更多信息，请查阅 Knuth 抽样和 Floyd 抽样算法。
