当只使用`key`参数调用时，从存储在`key`上的哈希值中返回一个随机字段。

如果提供的 `count` 参数是正数，则返回一个包含 **不同字段** 的数组。
数组的长度为 `count` 或哈希的字段数 (`HLEN`) 的较小者。

如果使用负数的`count`调用，则行为会发生变化，并且允许命令**多次返回相同字段**。
此时，返回的字段数量是指定`count`的绝对值。

可选的`WITHVALUES`修饰符可以更改回复，使其包含随机选择的哈希字段的相应值。

@examples

```cli
HMSET coin heads obverse tails reverse edge null
HRANDFIELD coin
HRANDFIELD coin
HRANDFIELD coin -5 WITHVALUES
```

## 在传递count参数时的行为规范

当 `count` 参数是正值时，此命令的行为如下：

* 不会返回重复字段。
* 如果 `count` 大于哈希中的字段数，命令将仅返回整个哈希而不包括额外的字段。
* 回复中字段的顺序并非真正随机，所以如果需要的话，由客户端来对它们进行洗牌。

当 `count` 的值为负数时，行为如下所示：

* 可以使用重复字段。
* 如果哈希为空（即键不存在），则始终返回`count`个字段或一个空数组。
* 返回的字段顺序是真正的随机。
