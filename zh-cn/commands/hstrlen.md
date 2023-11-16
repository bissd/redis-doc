返回存储在`key`中的哈希中与`field`相关联的值的字符串长度。如果`key`或`field`不存在，则返回0。

@examples

```cli
HMSET myhash f1 HelloWorld f2 99 f3 -256
HSTRLEN myhash f1
HSTRLEN myhash f2
HSTRLEN myhash f3
```
