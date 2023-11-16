将给定的键设置为其相应的值。
`MSET`用新值替换现有值，就像常规的`SET`一样。
如果您不想覆盖现有值，请参阅`MSETNX`。

`MSET` 是原子操作，因此所有给定的键将同时设置。
客户端无法观察到其中一些键被更新而其他键保持不变。

@examples

```cli
MSET key1 "Hello" key2 "World"
GET key1
GET key2
```
