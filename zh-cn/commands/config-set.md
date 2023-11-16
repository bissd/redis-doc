`CONFIG SET`命令用于在运行时重新配置服务器，而无需重新启动Redis。
您可以使用此命令更改普通参数或切换到其他持久化选项。

支持`CONFIG SET`的配置参数列表可以通过发出`CONFIG GET *`命令来获取，这是用于获取运行中的Redis实例配置信息的对称命令。

所有使用 `CONFIG SET` 设置的配置参数都会立即被 Redis 加载，并将在执行下一个命令后生效。

所有支持的参数的含义与 [redis.conf][hgcarr22rc] 文件中使用的等效配置参数相同。

[hgcarr22rc]: http://github.com/redis/redis/raw/unstable/redis.conf

注意，你应该查看与你正在使用的版本相关的redis.conf文件，因为配置选项可能在不同版本之间发生变化。上面的链接是最新的开发版本。

可以使用`CONFIG SET`命令将持久性从RDB快照切换为追加只文件（或反之亦然）。
有关如何执行此操作的更多信息，请查看[persistence页面][tp]。

[tp]: /topics/persistence

总的来说，你应该知道的是将`appendonly`参数设置为`yes`将启动一个后台进程，保存初始的追加日志文件（从内存数据集中获得），并将所有后续的命令追加到追加日志文件中，从而实现与启用AOF的Redis服务器从开始时一样的效果。

如果你愿意，可以同时启用AOF和RDB快照，这两个选项并不互斥。
