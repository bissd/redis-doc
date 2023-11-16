返回所有与`pattern`匹配的键。

这个操作的时间复杂度为O(N)，但常数时间非常低。
例如，运行在入门级笔记本上的Redis可以在40毫秒内扫描一个100万个键的数据库。

**警告**：将`KEYS`视为只应在生产环境下极度谨慎使用的命令。
当针对大型数据库执行该命令时，可能会破坏性能。
此命令适用于调试和特殊操作，例如更改您的键空间布局。
不要在常规应用程序代码中使用`KEYS`。
如果您正在寻找在键空间子集中查找键的方法，请考虑使用`SCAN`或[集合][tdts]。

[tdts]: /topics/data-types#sets

支持的通配符样式：


* `h?llo` 匹配 `hello`、`hallo` 和 `hxllo`
* `h*llo` 匹配 `hllo` 和 `heeeello`
* `h[ae]llo` 匹配 `hello` 和 `hallo`，但不匹配 `hillo`
* `h[^e]llo` 匹配 `hallo`、`hbllo`，但不匹配 `hello`
* `h[a-b]llo` 匹配 `hallo` 和 `hbllo`

如果你想与他们完全匹配，请使用`\`来转义特殊字符。

@examples

```cli
MSET firstname Jack lastname Stuntman age 35
KEYS *name*
KEYS a??
KEYS *
```
