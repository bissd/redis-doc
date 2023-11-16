返回存储在`key`上的字符串值的长度。
当`key`保持非字符串值时，返回错误。

@examples

```cli
SET mykey "Hello world"
STRLEN mykey
STRLEN nonexisting
```
