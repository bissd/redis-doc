获取`key`的值。
如果`key`不存在，则返回特殊值`nil`。
如果存储在`key`上的值不是字符串，则返回错误，因为`GET`只处理字符串值。

@examples

```cli
GET nonexisting
SET mykey "Hello"
GET mykey
```

### 代码示例

{{< clients-example set_and_get />}}
