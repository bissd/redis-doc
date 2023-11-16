该命令类似于`SINTER`，但不返回结果集，仅返回结果的基数。
返回所有给定集合交集的基数。

不存在的键被视为空集。
如果其中一个键是空集，则结果集也为空（因为与空集相交的集合总是为空集）。

默认情况下，该命令计算所有给定集合的交集的基数。
当提供可选的 `LIMIT` 参数（默认为0，表示无限制）时，如果交集基数在计算过程中达到限制值，算法将退出并将限制值作为基数。
这样的实现可以显著加快查询速度，尤其是限制值低于实际交集基数的情况下。

@examples

```cli
SADD key1 "a"
SADD key1 "b"
SADD key1 "c"
SADD key1 "d"
SADD key2 "c"
SADD key2 "d"
SADD key2 "e"
SINTER key1 key2
SINTERCARD 2 key1 key2
SINTERCARD 2 key1 key2 LIMIT 1
```
