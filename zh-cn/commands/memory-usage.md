`MEMORY USAGE`命令报告了将一个键及其值存储在RAM中所需的字节数。

报告的使用情况是键及其值所需的数据分配和管理开销的总和。

对于嵌套的数据类型，可以提供可选的`SAMPLES`选项，其中`count`是采样嵌套值的数量。样本被平均以估计总大小。
默认情况下，此选项设置为`5`。要采样所有嵌套值，请使用`SAMPLES 0`。

@examples

使用Redis v7.2.0 64位和**jemalloc**，空字符串的度量结果如下所示：

```
> SET "" ""
OK
> MEMORY USAGE ""
(integer) 56
```

这些字节目前纯粹是额外开销，因为没有实际数据被存储，并且用于维护服务器的内部数据结构（包括内部分配器碎片化）。较长的键和值显示出渐近线性的使用情况。

```
> SET foo bar
OK
> MEMORY USAGE foo
(integer) 56
> SET foo2 mybar
OK
> MEMORY USAGE foo2
(integer) 64
> SET foo3 0123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789
OK
> MEMORY USAGE foo3
(integer) 160
```
