这个命令类似于 `ZINTER`，但是不返回结果集，只返回结果的基数。

不存在的键被视为空集。
如果其中一个键是空集，那么结果集合也是空集（因为与空集求交集的结果始终为空集）。

默认情况下，该命令计算所有给定集合的交集的基数。
当提供可选的`LIMIT`参数时（默认为0，表示无限制），如果交集基数在计算过程中达到限制，算法将退出并将限制作为基数输出。
此类实现确保对于限制小于实际交集基数的查询，可以显著加快速度。

@examples

```cli
ZADD zset1 1 "one"
ZADD zset1 2 "two"
ZADD zset2 1 "one"
ZADD zset2 2 "two"
ZADD zset2 3 "three"
ZINTER 2 zset1 zset2
ZINTERCARD 2 zset1 zset2
ZINTERCARD 2 zset1 zset2 LIMIT 1
```
