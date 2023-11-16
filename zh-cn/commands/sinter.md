返回由给定集合间交集结果组成的集合的成员。

例如：

```
key1 = {a,b,c,d}
key2 = {c}
key3 = {a,c,e}
SINTER key1 key2 key3 = {c}
```

不存在的键被视为空集。
如果其中一个键是空集，那么结果集也是空集（因为与空集进行交集运算总是得到空集）。

@examples

```cli
SADD key1 "a"
SADD key1 "b"
SADD key1 "c"
SADD key2 "c"
SADD key2 "d"
SADD key2 "e"
SINTER key1 key2
```
