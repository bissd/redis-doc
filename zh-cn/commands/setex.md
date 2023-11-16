将 `key` 设置为保存字符串 `value`，并设置 `key` 在给定的秒数后超时。
此命令等效于：

```
SET key value EX seconds
```

当`seconds`无效时，会返回一个错误。

@examples

```cli
SETEX mykey 10 "Hello"
TTL mykey
GET mykey
```
## 另请参阅

`TTL`
