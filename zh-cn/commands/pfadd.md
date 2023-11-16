将所有的元素参数添加到以第一个参数指定的变量名存储的HyperLogLog数据结构中。

作为这个命令的副作用，HyperLogLog 内部可能会更新以反映迄今为止添加的唯一项数量的不同估计值（集合的基数）。

如果执行命令后，HyperLogLog估计的近似基数发生了变化，`PFADD`命令返回1，否则返回0。如果指定的键不存在，该命令会自动创建一个空的HyperLogLog结构（即具有指定长度和给定编码的Redis字符串）。

调用没有元素，只有变量名的命令是有效的，如果变量已经存在，这将导致没有操作执行，如果键不存在，则只会创建数据结构（在后一种情况下返回1）。

了解HyperLogLog数据结构的介绍，请查看[PFCOUNT](https://redis.io/commands/pfcount)命令页面。

@examples

```cli
PFADD hll a b c d e f g
PFCOUNT hll
```
