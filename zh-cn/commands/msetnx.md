以给定的键设置它们各自的值。
即使只有一个键已经存在，`MSETNX`也不会执行任何操作。

由于这个语义，可以使用`MSETNX`来设置不同的键，代表唯一逻辑对象的不同字段，确保要么设置全部字段，要么不设置。

`MSETNX`是原子操作，因此所有给定的键将同时设置。
客户端无法看到某些键的更新同时其他键保持不变。

@examples

```cli
MSETNX key1 "Hello" key2 "there"
MSETNX key2 "new" key3 "world"
MGET key1 key2 key3
```
