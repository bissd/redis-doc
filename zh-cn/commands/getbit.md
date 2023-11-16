在键为_key_的字符串值中偏移量为_offset_的位值。

当偏移量超过字符串长度时，字符串被假定为连续的空间，其包含0位。
当键不存在时，假定键是空字符串，所以偏移量总是超出范围，值也被假定为连续的空间，包含0位。

```cli
SETBIT mykey 7 1
GETBIT mykey 0
GETBIT mykey 7
GETBIT mykey 100
```
