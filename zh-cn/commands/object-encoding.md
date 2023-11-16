返回存储在`<key>`上的Redis对象的内部编码

Redis对象可以以不同的方式编码：

*字符串可以编码为：*

    - `raw`, normal string encoding.
    - `int`, strings representing integers in a 64-bit signed interval, encoded in this way to save space.
    - `embstr`, an embedded string, which is an object where the internal simple dynamic string, `sds`, is an unmodifiable string allocated in the same chuck as the object itself.
      `embstr` can be strings with lengths up to the hardcoded limit of `OBJ_ENCODING_EMBSTR_SIZE_LIMIT` or 44 bytes. 

* 列表可以编码为`ziplist`或`linkedlist`。`ziplist`是用于节省小列表空间的特殊表示。
* 集合可以编码为`intset`或`hashtable`。`intset`是一种专门用于由整数组成的小集合的编码方式。
* 哈希可以编码为`ziplist`或`hashtable`。`ziplist`是一种专门用于小哈希的编码方式。
* 有序集合可以以`ziplist`或`skiplist`格式编码。至于列表类型的小型有序集合，可以特殊编码为`ziplist`，而`skiplist`编码则适用于任何大小的有序集合。

所有特殊编码的数据类型在执行一项操作后，如果导致Redis无法保留节省空间的编码方式，将自动转换为一般类型。
