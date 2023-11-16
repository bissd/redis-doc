如果`field`在`key`中的哈希中尚不存在，则将`field`设置为`value`。
如果`key`不存在，则创建一个新的持有哈希的键。
如果`field`已经存在，则此操作无效。

@examples

```cli
HSETNX myhash field "Hello"
HSETNX myhash field "World"
HGET myhash field
```
