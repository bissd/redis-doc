这个命令将存储在`source`键中的值复制到`destination`键中。

默认情况下，`destination`键被创建在连接所使用的逻辑数据库中。`DB`选项允许指定目标键的替代逻辑数据库索引。

命令在“destination”键已经存在时返回零。 `REPLACE`选项在将值复制到`destination`键之前删除该键。

@examples

```
SET dolly "sheep"
COPY dolly clone
GET clone
```
