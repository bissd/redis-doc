将多个HyperLogLog值合并为一个唯一的值，该值将近似于源HyperLogLog结构观察到的集合的并集的基数。

计算合并的HyperLogLog被设置为目标变量，如果目标变量不存在则创建（默认为一个空的HyperLogLog）。

如果目标变量存在，它将被视为源集合之一，并且它的基数将包含在计算的HyperLogLog的基数中。

@examples

```cli
PFADD hll1 foo bar zap a
PFADD hll2 a b c foo
PFMERGE hll3 hll1 hll2
PFCOUNT hll3
```
