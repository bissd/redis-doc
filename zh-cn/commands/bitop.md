使用位操作符对多个键（包含字符串值）执行位操作，并将结果存储在目标键中。

`BITOP` 命令支持四种位操作：**AND**（与）、**OR**（或）、**XOR**（异或）和**NOT**（非），因此调用命令的有效形式有：


* `BITOP AND destkey srckey1 srckey2 srckey3 ... srckeyN` 
* `BITOP OR  destkey srckey1 srckey2 srckey3 ... srckeyN` 
* `BITOP XOR destkey srckey1 srckey2 srckey3 ... srckeyN` 
* `BITOP NOT destkey srckey`

正如你能看到的**NOT**是特殊的，因为它只接受一个输入键，因为它执行位反转，所以只作为一个一元运算符才有意义。

操作的结果始终存储在 `destkey`。

## 处理长度不同的字符串

当对长度不同的字符串进行操作时，所有长度较短的字符串都会被视为零填充，直到达到最长字符串的长度。

对于不存在的键，同样适用，它们被视为零字节的流，长度与最长的字符串相同。

@examples

```cli
SET key1 "foobar"
SET key2 "abcdef"
BITOP AND dest key1 key2
GET dest
```

## 模式：使用位图的实时度量

`BITOP` 是`BITCOUNT`命令文档中所记录模式的良好补充。
可以将不同的位图组合在一起，以获得进行人口统计操作的目标位图。

请查看名为"[使用Redis位图进行快速、简单的实时度量指标][hbgc212fermurb]"的文章，这是一个有趣的用例。

[hbgc212fermurb]: http://blog.getspool.com/2011/11/29/fast-easy-realtime-metrics-using-redis-bitmaps

## 性能考虑

`BITOP`命令可能会很慢，因为它的时间复杂度是O(N)。
在对长输入字符串运行该命令时要小心。

对于涉及大量输入的实时度量和统计，一个好的方法是使用一个启用了副本只读选项的副本，其中执行按位操作以避免阻塞主实例。
