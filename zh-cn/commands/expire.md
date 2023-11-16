设置`key`的超时时间。
超时时间到期后，key将自动被删除。
在Redis术语中，一个具有超时的key通常被称为_易失性_。

超时只能通过删除或覆盖键的命令来清除，包括 `DEL`、`SET`、`GETSET` 和所有的 `*STORE` 命令。
这意味着所有在概念上改变键存储的值而不是用新值替换它的操作都不会影响超时。
例如，使用 `INCR` 增加键的值，使用 `LPUSH` 将新值推入列表，或使用 `HSET` 改变哈希的字段值，这些操作都不会影响超时。

超时也可以被清除，将密钥转回成持久密钥，
使用 `PERSIST` 命令。

如果使用`RENAME`重命名键，则关联的生存时间将转移到新的键名。

如果一个键被“RENAME”覆盖，就像在现有键“Key_A”被一个像“RENAME Key_B Key_A”这样的调用覆盖的情况下，原始的“Key_A”是否有关联的超时时间是无关紧要的，新键“Key_A”将会继承“Key_B”的所有特性。

请注意，调用具有非正超时的`EXPIRE`/`PEXPIRE`或带有过去时间的`EXPIREAT`/`PEXPIREAT`将导致键被[删除][del]而不是过期（相应地，发出的[键事件][ntf]将是`del`而不是`expired`）。

[del]: /commands/del
[ntf]: /topics/notifications

## 选项

`EXPIRE` 命令支持一组选项：

* `NX` -- 仅当键没有过期时设置过期时间
* `XX` -- 仅当键有现有过期时间时设置过期时间
* `GT` -- 仅当新的过期时间大于当前的过期时间时设置过期时间
* `LT` -- 仅当新的过期时间小于当前的过期时间时设置过期时间

非易失性键对于`GT`和`LT`的目的被视为无限的TTL。
`GT`，`LT`和`NX`选项是互斥的。

## 刷新过期时间

可以使用一个已经存在过期时间的键作为参数来调用`EXPIRE`命令。在这种情况下，键的存活时间将会被更新为新的值。
这有许多有用的应用，下面的“导航会话”模式部分中有一个例子。

## Redis 2.1.3 之前的差异

Redis版本在2.1.3之前，使用修改命令修改带过期时间的键会导致键被完全删除。这种语义是因为复制层面的限制而需要的，而这些限制现在已经修复。

`EXPIRE`命令返回0且不改变已设置超时时间的键的超时时间。

经典的Hello World示例程序：

```
#include <stdio.h>

int main() {
    printf("Hello, World!");
    return 0;
}
```

求两个整数的和的示例程序：

```
#include <stdio.h>

int main() {
    int a = 10, b = 20;
    int sum = a + b;
    printf("The sum of %d and %d is %d", a, b, sum);
    return 0;
}
```

打印九九乘法表的示例程序：

```
#include <stdio.h>

int main() {
    int i, j;
    for (i = 1; i <= 9; i++) {
        for (j = 1; j <= i; j++) {
            printf("%d * %d = %d ", j, i, i * j);
        }
        printf("\n");
    }
    return 0;
}
```

计算斐波那契数列的示例程序：

```
#include <stdio.h>

int fibonacci(int n) {
    if (n <= 1) {
        return n;
    }
    return fibonacci(n - 1) + fibonacci(n - 2);
}

int main() {
    int n = 10;
    printf("Fibonacci series up to %d terms:\n", n);
    for (int i = 0; i < n; i++) {
        printf("%d ", fibonacci(i));
    }
    return 0;
}
```

```cli
SET mykey "Hello"
EXPIRE mykey 10
TTL mykey
SET mykey "Hello World"
TTL mykey
EXPIRE mykey 10 XX
TTL mykey
EXPIRE mykey 10 NX
TTL mykey
```

## 模式：导航会话

假设你有一个网络服务，你希望得到最近 N 个被用户访问的页面，每个页面的访问时间间隔不超过 60 秒。
从概念上来说，你可以将这一系列页面视图看作用户的导航会话，可能包含关于用户当前所寻找的产品类型的有趣信息，以便你可以推荐相关产品。

使用Redis可以很容易地对这种模式进行建模，采用以下策略：每当用户进行页面访问时，调用以下命令：

```
MULTI
RPUSH pagewviews.user:<userid> http://.....
EXPIRE pagewviews.user:<userid> 60
EXEC
```

如果用户空闲超过60秒，则键将被删除，只有接下来的页面浏览记录的时间差小于60秒的会被记录。

这种模式可以很容易地通过使用`INCR`而不是列表的`RPUSH`进行修改以使用计数器。

# 附录：Redis过期时间

## 有过期时间的键

通常情况下，Redis的键是没有关联的生存时间的。
该键将永远存在，除非用户以显式方式删除它，例如使用`DEL`命令。

`EXPIRE`命令系列能够为给定的键关联一个过期时间，在键所使用的额外内存的代价下。

当一个键设置了过期时间，Redis会确保在指定的时间过去后删除该键。

关键的生存时间可以使用`EXPIRE`和`PERSIST`命令（或其他严格相关的命令）进行更新或完全删除。

## 过期的准确性

在Redis 2.4中，到期时间可能不是非常精确，可能会有零到一秒钟的误差。

自Redis 2.6起，过期错误时间为0到1毫秒。

## 过期和持久性

Keys expiring information is stored as absolute Unix timestamps (in milliseconds in case of Redis version 2.6 or greater). 
This means that the time is flowing even when the Redis instance is not active.
过期键信息以绝对Unix时间戳存储（对于Redis版本2.6或更高版本以毫秒为单位）。这意味着即使Redis实例不活动，时间仍在流动。

为了使过期操作正常工作，电脑时间必须保持稳定。
如果您将一个RDB文件从两台时钟严重不同步的电脑移动到另一台电脑上，
可能会发生一些奇怪的事情（比如在加载时将所有键设置为过期）。

即使运行中的实例也会始终检查计算机时钟，所以例如，如果您设置了一个存活时间为1000秒的键，然后将计算机时间设置为未来2000秒，该键将立即过期，而不是持续1000秒。

## Redis 如何过期键值对

Redis键有两种过期方式：被动和主动。

当某个客户端试图访问某个键，并且发现该键已超时，这时键会被被动地过期。

当然，这些还不够，因为有些过期的键将不会再次被访问。
这些键应该被过期，所以Redis会定期随机测试一些设置了过期时间的键。
所有已过期的键将从键空间中删除。

具体来说，Redis 每秒钟执行以下操作 10 次：

1. 从带有到期时间的键集合中随机测试20个键。
2. 删除所有已过期的键。
3. 如果超过25％的键已过期，从步骤1重新开始。

这是一个微不足道的概率算法，基本假设是我们的样本代表了整个密钥空间，我们会继续过期，直到可能过期的密钥的百分比低于25%。

这意味着在任何给定时刻，已经过期且正在使用内存的键的最大数量等于每秒写入操作的最大数量除以4。

## 复制链接和AOF文件中的过期处理方式

为了在不牺牲一致性的前提下获得正确的行为，在键过期时，在AOF文件和其所有附属副本节点中合成“DEL”操作。
这样，过期处理过程集中在主实例中，不存在一致性错误的机会。

然而，尽管连接到主节点的副本不会独立过期键（但会等待来自主节点的`DEL`命令），但它们仍将获取数据集中现有过期键的全部状态，因此当副本被选为主节点时，它将能够独立过期键，并完全扮演主节点的角色。
