---
title: "ACL"
linkTitle: "ACL"
weight: 1
description: Redis 访问控制列表
aliases: [
    /topics/acl,
    /docs/manual/security/acl,
    /docs/manual/security/acl.md
]
---

Redis ACL（Access Control List）是一种允许限制某些连接在执行命令和访问密钥方面受限的功能。它的工作原理是，在连接后，客户端需要提供用户名和有效密码进行身份验证。如果身份验证成功，连接将与给定的用户和用户的限制相关联。Redis可以配置为新连接已通过“default”用户进行身份验证（这是默认配置）。配置默认用户的副作用是只能向未明确进行身份验证的连接提供特定子集的功能。

在默认配置中，Redis 6（第一个具有ACL的版本）的工作方式与旧版本的Redis完全相同。每个新连接都可以调用每个可能的命令并访问每个键，因此ACL功能与旧客户端和应用程序向后兼容。此外，使用**requirepass**配置指令配置密码的旧方式仍然按预期工作。然而，它现在为默认用户设置密码。

Redis `AUTH` 命令在 Redis 6 中被扩展，因此现在可以以两个参数形式使用它：

    AUTH <username> <password>

这是一个旧表单的示例：

    AUTH <password>

发生的情况是使用的用户名进行身份验证的是"default"，所以仅指定密码意味着我们要对默认用户进行身份验证。这提供了向后兼容性。

## 当ACLs有用时

在使用ACL之前，您可能想问自己实施这个保护层的目标是什么。通常有两个主要目标可以通过ACL很好地实现:

1.你希望通过限制对命令和密钥的访问来提高安全性，以使不受信任的客户端无法访问，并且仅授予受信任的客户端数据库中执行所需工作的最低访问级别。例如，某些客户端可能只能执行只读命令。
2.你希望通过禁止访问Redis的进程或人为错误或手动错误损坏数据或配置来提高操作安全性。例如，从Redis获取延迟作业的工作者没有理由能够调用“FLUSHALL”命令。

另一个典型的 ACL 用法与托管的 Redis 实例有关。Redis 经常由公司的内部团队提供托管服务，这些团队为其他内部客户处理 Redis 基础设施，或者由云提供商以软件即服务的方式提供。在这两种设置中，我们希望确保将配置命令排除在客户之外。

## 使用ACL命令配置ACLs

ACLs 使用描述用户被允许做什么的特定领域语言 (DSL) 来定义。这些规则始终按照从左到右、从头到尾的顺序实施，因为有时规则的顺序对于了解用户真正能做什么很重要。

默认情况下有一个名为*default*的单个用户定义。我们可以使用`ACL LIST`命令来检查当前活动的ACL，并验证新启动的默认配置Redis实例的配置。

    > ACL LIST
    1) "user default on nopass ~* &* +@all"

上述命令以与Redis配置文件中使用的相同格式报告用户列表，通过将当前设置的ACL转换为用户的描述来实现。

```
每行中的前两个单词为 "user" 后跟用户名。接下来的单词是描述不同事物的 ACL 规则。我们将详细展示规则的工作方式，但现在只需说默认用户被配置为激活（on），不需要密码（nopass），可访问每个可能的键（`~*`）和发布/订阅频道（`&*`），并且能够调用每个可能的命令（`+@all`）。
```

此外，在默认用户的特殊情况下，具有*nopass*规则意味着新的连接将自动使用默认用户进行身份验证，不需要任何显式的`AUTH`调用。

## ACL规则

以下是有效的访问控制列表规则的列表。某些规则只是单词，用于激活或移除标志，或对用户ACL进行特定更改。其他规则是字符前缀，与命令或类别名称、键模式等连接在一起。

允许和禁止用户：

* `on`：启用用户：可以使用该用户进行身份验证。
* `off`：禁止用户：不再可以使用该用户进行身份验证；但是，先前验证的连接仍然有效。请注意，如果默认用户标记为 *off*，新的连接将作为非认证状态启动，并且需要用户发送 `AUTH` 或带有认证选项的 `HELLO` 以某种方式进行身份验证，无论默认用户配置如何。

允许和禁止命令：

* `+<命令>`: 将命令添加到用户可调用的命令列表中。可以与 `|` 一起使用以允许子命令（例如 "+config|get"）。
* `-<命令>`: 从用户可调用的命令列表中删除命令。Redis 7.0 开始，可以与 `|` 一起使用以阻止子命令（例如 "-config|set"）。
* `+@<分类>`: 将该分类中的所有命令添加到用户可以调用的命令列表中，有效的分类可以是 @admin、@set、@sortedset 等等，通过调用 `ACL CAT` 命令查看完整列表。特殊的分类 @all 表示所有命令，包括当前服务器中存在的命令和通过模块加载的未来命令。
* `-@<分类>`: 类似于 `+@<分类>`，但从客户端的命令列表中删除该分类中的命令。
* `+<命令>|第一个参数`: 允许一个否则被禁用的命令的特定第一个参数。此功能仅适用于没有子命令的命令，并且不支持负形式，例如 -SELECT|1，只能以 "+" 为前缀进行添加。此功能已被废弃，将来可能会被移除。
* `allcommands`: +@all 的别名。请注意，它意味着能够执行通过模块系统加载的所有未来命令。
* `nocommands`: -@all 的别名。

允许和禁止某些键和键权限：

* `~<pattern>`：添加可以作为命令一部分提及的键的模式。例如，`~*` 允许所有键。模式是一个类似于 `KEYS` 的 glob 模式。可以指定多个模式。
* `%R~<pattern>`：（在 Redis 7.0 及更高版本可用）添加指定的读取键模式。这与常规键模式类似，但仅允许对与给定模式匹配的键进行读取。有关更多信息，请参阅[键权限](#key-permissions)。
* `%W~<pattern>`：（在 Redis 7.0 及更高版本可用）添加指定的写入键模式。这与常规键模式类似，但仅允许对与给定模式匹配的键进行写入。有关更多信息，请参阅[键权限](#key-permissions)。
* `%RW~<pattern>`：（在 Redis 7.0 及更高版本可用）`~<pattern>` 的别名。
* `allkeys`：`~*` 的别名。
* `resetkeys`：清除允许的键模式列表。例如，ACL `~foo:* ~bar:* resetkeys ~objects:*` 将只允许客户端访问与模式 `objects:*` 匹配的键。

允许和禁止Pub/Sub频道：

* `&<pattern>`：（在 Redis 6.2 及更高版本可用）添加一个以通配符形式表示的 Pub/Sub 频道模式，用户可以通过此模式访问。可以指定多个频道模式。请注意，模式匹配仅针对 `PUBLISH` 和 `SUBSCRIBE` 提到的频道，而 `PSUBSCRIBE` 则需要其频道模式与允许用户访问的频道模式完全匹配。
* `allchannels`：`&*` 的别名，允许用户访问所有的 Pub/Sub 频道。
* `resetchannels`：清空允许的频道模式列表，并断开用户的 Pub/Sub 客户端连接，如果这些客户端无法再访问它们各自的频道和/或频道模式。

为用户配置有效密码：
```sh
# 设置密码最小长度
password-minlen 8

# 设置密码必须包含至少一个字母、一个数字和一个特殊字符
password-complexity requirements="length=8 dcredit=-1 ucredit=-1 lcredit=-1 ocredit=-1"

# 设置密码不能与用户名相同
password-notusername

# 设置密码不能与前五个历史密码相同
password-history remember=5

# 设置密码过期时间为90天，并在过期前30天提醒用户更换密码
password-maxdays 90
password-warndays 30
```

* `><password>`：将此密码添加到用户的有效密码列表中。例如`>mypass`将"mypass"添加到有效密码列表中。此指令会清除*nopass*标志（后续详见）。每个用户可以有任意数量的密码。
* `<<password>`：从有效密码列表中移除此密码。如果要移除的密码实际上没有设置，则会发出错误。
* `#<hash>`：将此SHA-256哈希值添加到用户的有效密码列表中。此哈希值将与输入给ACL用户的密码的哈希进行比较。这允许用户在`acl.conf`文件中存储哈希而不是存储明文密码。只接受SHA-256哈希值，因为密码哈希必须为64个字符，且只能包含小写十六进制字符。
* `!<hash>`：从有效密码列表中移除此哈希值。当您不知道由哈希值指定的密码但想要从用户中移除密码时，这将非常有用。
* `nopass`：删除用户的所有设置密码，并标记用户不需要密码：这意味着任何密码都可以对此用户进行认证。如果将此指令用于默认用户，则每个新连接都将立即使用默认用户进行身份验证，而不需要任何显式的AUTH命令。请注意，*resetpass*指令将清除此条件。
* `resetpass`：清除允许的密码列表并移除*nopass*状态。在*resetpass*之后，用户没有关联密码，除非添加密码（或稍后设置为*nopass*），否则无法进行身份验证。

*注意：如果用户没有被标记为 "nopass" 并且没有有效密码列表，则该用户将无法使用，因为将无法以该用户身份登录。*

为用户配置选择器：


* `(<rule list>)`: （自 Redis 7.0 及以后版本可用）创建一个新的选择器来匹配规则。选择器在用户权限之后进行评估，并根据定义的顺序进行评估。如果一个命令匹配用户权限或任何选择器，则允许执行。更多信息请参见[选择器](#selectors)。
* `clearselectors`: （自 Redis 7.0 及以后版本可用）删除用户关联的所有选择器。

重置用户：

* `reset` 执行以下操作：resetpass（重置密码）、resetkeys（重置密钥）、resetchannels（重置频道）, allchannels（如果设置了acl-pubsub-default许可的话）, off（关闭）、clearselectors（清除选择器）、-@all（取消所有指示器）。用户返回到其创建后的相同状态。

## 使用ACL SETUSER命令创建和编辑用户ACL

用户可以通过两种主要方式创建和修改：

1. 使用 ACL 命令及其 `ACL SETUSER` 子命令。
2. 修改服务器配置，定义用户并重新启动服务器。使用 *外部 ACL 文件*，只需调用 `ACL LOAD`。

在本节中，我们将学习如何使用 `ACL` 命令来定义用户。
具备这样的知识后，通过配置文件执行相同的操作将变得非常容易。在配置文件中定义用户将在单独的部分中讨论。

要开始，请尝试使用最简单的`ACL SETUSER`命令调用：

    > ACL SETUSER alice
    OK

`ACL SETUSER` 命令会接受用户名和一系列要应用到该用户的 ACL 规则。然而上述示例没有指定任何规则。如果用户不存在，这个命令将只会创建用户，同时使用新用户的默认设置。如果用户已存在，上述命令将不会有任何作用。

检查默认用户状态：

    > ACL LIST
    1) "user alice off resetchannels -@all"
    2) "user default on nopass ~* &* +@all"

新用户"alice"是：

* 在离线状态下，`AUTH` 命令对用户 "alice" 无效。
* 用户也没有设置任何密码。
* 无法访问任何命令。注意，默认情况下创建的用户不能访问任何命令，因此上面的输出中的 `-@all` 可以省略；但是，`ACL LIST` 命令更倾向于明确而不是隐含。
* 用户无法访问任何键模式。
* 用户无法访问任何 Pub/Sub 频道。

新用户默认创建时具有受限的权限。从 Redis 6.2 开始，ACL 还提供 Pub/Sub 通道访问管理。为了确保与版本 6.0 的后向兼容性，在升级到 Redis 6.2 时，新用户默认被授予 "allchannels" 权限。可以通过 `acl-pubsub-default` 配置指令将默认设置为 `resetchannels`。

从7.0开始，默认将`acl-pubsub-default`值设置为`resetchannels`以默认限制频道访问，提供更好的安全性。
可以通过`acl-pubsub-default`配置指令将默认值设置为`allchannels`，以保持与之前版本的兼容性。

这样的用户是完全没用的。让我们尝试定义用户，使其处于活动状态，设置密码，并且只能通过`GET`命令访问以字符串“cached:”开头的键名。

    > ACL SETUSER alice on >p1pp0 ~cached:* +get
    OK

用户现在可以做一些事情，但会拒绝做其他事情：

    > AUTH alice p1pp0
    OK
    > GET foo
    (error) NOPERM this user has no permissions to access one of the keys used as arguments
    > GET cached:1234
    (nil)
    > SET cached:1234 zap
    (error) NOPERM this user has no permissions to run the 'set' command

一切都在按预期运行。为了检查用户 alice 的配置（请记住，用户名区分大小写），可以使用与 `ACL LIST` 类似但更适合计算机读取的替代方法，其为 `ACL GETUSER` 提供了更易于理解的输出。

    > ACL GETUSER alice
    1) "flags"
    2) 1) "on"
    3) "passwords"
    4) 1) "2d9c75..."
    5) "commands"
    6) "-@all +get"
    7) "keys"
    8) "~cached:*"
    9) "channels"
    10) ""
    11) "selectors"
    12) (empty array)

`ACL GETUSER` 返回一个描述用户的字段值数组，以更易解析的方式呈现。输出包括一组标志、关键模式列表、密码等等。如果我们使用 RESP3 返回作为映射回复，那么输出可能更易读：

    > ACL GETUSER alice
    1# "flags" => 1~ "on"
    2# "passwords" => 1) "2d9c75273d72b32df726fb545c8a4edc719f0a95a6fd993950b10c474ad9c927"
    3# "commands" => "-@all +get"
    4# "keys" => "~cached:*"
    5# "channels" => ""
    6# "selectors" => (empty array)

*注：从现在开始，我们将继续使用 Redis 的默认协议，版本 2*

使用另一个`ACL SETUSER`命令（来自不同用户，因为alice无法运行ACL命令），我们可以为用户添加多个模式：

    > ACL SETUSER alice ~objects:* ~items:* ~public:*
    OK
    > ACL LIST
    1) "user alice on #2d9c75... ~cached:* ~objects:* ~items:* ~public:* resetchannels -@all +get"
    2) "user default on nopass ~* &* +@all"

现在内存中的用户表示与我们预期的一样。

## 多次调用ACL SETUSER

了解在多次调用`ACL SETUSER`时会发生什么非常重要。需要了解的关键信息是，每次调用`ACL SETUSER`都不会重置用户，而只是将ACL规则应用于现有用户。仅当之前未知用户时，才会重置用户。在这种情况下，将创建一个全新的用户，其ACL为零。该用户无法执行任何操作，被禁止访问，没有密码等。这是为了安全起见的最佳默认设置。

然而，后续的调用将仅以增量方式修改用户。例如，以下序列：

    > ACL SETUSER myuser +set
    OK
    > ACL SETUSER myuser +get
    OK

这将使myuser可以调用`GET`和`SET`。

    > ACL LIST
    1) "user default on nopass ~* &* +@all"
    2) "user myuser off resetchannels -@all +get +set"

## 命令类别

通过逐个指定所有命令来设置用户 ACLs 实在太麻烦了，所以我们改用这种方式：

    > ACL SETUSER antirez on +@all -@dangerous >42a979... ~*

通过使用+@all和-@dangerous，我们包含了所有的命令，并在Redis命令表中删除了标记为危险的命令。
请注意，命令类别**不包括模块命令**，除了+@all以外。如果使用+@all，用户可以执行所有的命令，包括通过模块系统加载的未来命令。然而，如果使用ACL规则+@read或其他规则，模块命令总是被排除在外。这点非常重要，因为您应该只信任Redis内部的命令表。模块可能会暴露危险的内容，在ACL的情况下，它只是添加性的，即以`+@all -...`的形式。您应该非常确信您不会包含您不想要的东西。

以下是命令类别及其含义的列表：


* **admin** - 管理命令。一般应用程序不需要使用这些命令。包括`REPLICAOF`、`CONFIG`、`DEBUG`、`SAVE`、`MONITOR`、`ACL`、`SHUTDOWN`等。
* **bitmap** - 数据类型：位图相关。
* **blocking** - 可能会阻塞连接直到被其他命令释放的命令。
* **connection** - 影响连接或其他连接的命令。包括`AUTH`、`SELECT`、`COMMAND`、`CLIENT`、`ECHO`、`PING`等。
* **dangerous** - 潜在危险的命令（出于各种原因，每个命令都需要谨慎考虑）。包括`FLUSHALL`、`MIGRATE`、`RESTORE`、`SORT`、`KEYS`、`CLIENT`、`DEBUG`、`INFO`、`CONFIG`、`SAVE`、`REPLICAOF`等。
* **geo** - 数据类型：地理空间索引相关。
* **hash** - 数据类型：哈希相关。
* **hyperloglog** - 数据类型：HyperLogLog相关。
* **fast** - 快速的O(1)命令。可能会根据参数的个数进行循环，但不会根据键中元素的个数进行循环。
* **keyspace** - 以一种类型无关的方式写入或读取键、数据库或其元数据。包括`DEL`、`RESTORE`、`DUMP`、`RENAME`、`EXISTS`、`DBSIZE`、`KEYS`、`EXPIRE`、`TTL`、`FLUSHALL`等。可能修改键空间、键或元数据的命令也将被归为`write`类别。只读取键空间、键或元数据的命令将被归为`read`类别。
* **list** - 数据类型：列表相关。
* **pubsub** - 与PubSub相关的命令。
* **read** - 从键中读取（值或元数据）。请注意，不与键交互的命令将不会具有`read`或`write`两个类别之一。
* **scripting** - 与脚本相关的命令。
* **set** - 数据类型：集合相关。
* **sortedset** - 数据类型：有序集合相关。
* **slow** - 所有不属于`fast`命令的命令。
* **stream** - 数据类型：流相关。
* **string** - 数据类型：字符串相关。
* **transaction** - 与`WATCH` / `MULTI` / `EXEC`相关的命令。
* **write** - 向键中写入（值或元数据）。

Redis还可以使用Redis `ACL CAT`命令以两种形式为您显示所有类别和各个类别包含的确切命令。

    ACL CAT -- Will just list all the categories available
    ACL CAT <category-name> -- Will list all the commands inside the category

     > ACL CAT
     1) "keyspace"
     2) "read"
     3) "write"
     4) "set"
     5) "sortedset"
     6) "list"
     7) "hash"
     8) "string"
     9) "bitmap"
    10) "hyperloglog"
    11) "geo"
    12) "stream"
    13) "pubsub"
    14) "admin"
    15) "fast"
    16) "slow"
    17) "blocking"
    18) "dangerous"
    19) "connection"
    20) "transaction"
    21) "scripting"

如你所见，到目前为止有21个不同的类别。现在让我们看看*geo*类别中包含哪些命令：

     > ACL CAT geo
     1) "geohash"
     2) "georadius_ro"
     3) "georadiusbymember"
     4) "geopos"
     5) "geoadd"
     6) "georadiusbymember_ro"
     7) "geodist"
     8) "georadius"
     9) "geosearch"
    10) "geosearchstore"

请注意，命令可能属于多个类别。例如，像`+@geo -@read`这样的ACL规则将导致某些地理命令被排除在外，因为它们是只读命令。

## 允许/阻止子命令

从Redis 7.0开始，子命令可以像其他命令一样被允许/阻止（使用命令和子命令之间的分隔符`|`，例如：`+config|get`或`-config|set`）

对于除DEBUG命令以外的所有命令都适用这一规则。为了允许/阻止特定的DEBUG子命令，请参见下一节。

允许被阻止命令的第一个参数

**注意：这个功能自Redis 7.0版本开始已被弃用，并可能在将来的版本中被移除。**

有时仅仅能够排除或包含一个命令或子命令是不够的。
许多部署可能不想提供在任何数据库上执行`SELECT`的能力，但仍然希望能够运行`SELECT 0`。

在这种情况下，我们可以以以下方式更改用户的ACL：

    ACL SETUSER myuser -select +select|0

首先，移除`SELECT`命令，然后添加允许的第一个参数。需要注意的是，**无法进行相反操作**，因为只能添加第一个参数，不能排除它。最好为某个用户指定所有有效的第一个参数，因为将来可能会添加新的第一个参数。

下面是另一个示例：

    ACL SETUSER myuser -debug +debug|digest

请注意，第一个参数匹配可能会增加一些性能损失；然而，即使使用合成基准测试，很难衡量出性能损失的具体数值。这种额外的 CPU 开销只会在调用这些命令时产生，而不会在调用其他命令时产生。

可以使用此机制以允许在 Redis 7.0 之前的版本中使用子命令（请参阅上面的部分）。

## +@全部 VS -@全部

在前一节中，我们观察到可以根据添加/删除单个命令来定义命令ACL。

## 选择器

从 Redis 7.0 开始，Redis 支持添加多个独立评估的规则集。
这些次要的权限集被称为选择器，并且通过将一组规则包裹在括号中来添加。
为了执行命令，必须匹配给定命令的根权限（在括号外定义的规则）或任何选择器（在括号内定义的规则）之一。
在内部，首先检查根权限，然后按添加顺序检查选择器。

例如，考虑一个拥有ACL规则`+GET ~key1 (+SET ~key2)`的用户。
此用户可以执行`GET key1`和`SET key2 hello`，但不能执行`GET key2`或者`SET key1 world`。

与用户的根权限不同，选择器在添加后无法修改。
取而代之的是，可以使用“clearselectors”关键字来移除所有已添加的选择器。
请注意，“clearselectors”不会移除根权限。

## 密钥权限

从Redis 7.0开始，键模式也可以用于定义命令如何触及键。
通过定义键权限的规则来实现。
键权限规则的形式为`%(<permission>)~<pattern>`。
权限被定义为映射到以下键权限的单个字符。

* W（写入）：可以更新或删除存储在键中的数据。
* R（读取）：处理、复制或返回键中用户提供的数据。请注意，这不包括元数据（例如`STRLEN`），类型信息（例如`TYPE`）或有关值是否存在于集合中的信息（例如`SISMEMBER`）等。

可以通过指定多个字符来组合权限。
将权限指定为'RW'被认为是完全访问，并类似于仅传入`~<模式>`。

对于一个具体的例子，考虑一个拥有ACL规则`+@all ~app1:* (+@read ~app2:*)`的用户。
此用户对于`app1:*`拥有完全访问权限，对于`app2:*`只有只读权限。
然而，某些命令支持从一个键中读取数据，进行一些转换，然后将其存储到另一个键中。
一个这样的命令是`COPY`命令，它将源键中的数据复制到目标键中。
上述示例的ACL规则集无法处理从`app2:user`复制数据到`app1:user`的请求，因为根权限和选择器都无法完全匹配该命令。
然而，使用键选择器，您可以定义一组能够处理此请求的ACL规则：`+@all ~app1:* %R~app2:*`。
第一个模式能够匹配`app1:user`，第二个模式能够匹配`app2:user`。

执行命令所需的权限类型在[key规范](/topics/key-specs#logical-operation-flags)中有说明。
权限类型基于键的逻辑操作标志。
插入、更新和删除标志映射到写入键权限。
访问标志映射到读取键权限。
如果键没有逻辑操作标志，如`EXISTS`，用户仍然需要键读取或键写入权限来执行命令。

注意：在评估是否需要读取权限来执行命令时，会忽略访问用户数据的副信道。
这意味着一些写命令只需要对键具有写权限就可以执行，而不需要返回有关修改键的元数据。
例如，考虑以下两个命令：

* `LPUSH key1 data`：修改"key1"，但只返回与之相关的元数据，即推送后列表的大小，因此该命令只需要对"key1"拥有写权限才能执行。
* `LPOP key2`：修改"key2"，同时从中返回数据，即列表中的最左边的项，所以该命令需要同时具有对"key2"的读写权限才能执行。

如果一个应用程序需要确保没有从密钥中访问到任何数据，包括侧信道，建议不要提供对密钥的任何访问。

## 密码在内部是如何存储的

Redis在内部使用SHA256进行密码哈希存储。如果您设置了密码并检查`ACL LIST`或`ACL GETUSER`的输出，您将看到一个长的伪随机十六进制字符串。以下是一个示例，因为在先前的示例中，为了简洁起见，长的十六进制字符串已经被截断：

    > ACL GETUSER default
    1) "flags"
    2) 1) "on"
    3) "passwords"
    4) 1) "2d9c75273d72b32df726fb545c8a4edc719f0a95a6fd993950b10c474ad9c927"
    5) "commands"
    6) "+@all"
    7) "keys"
    8) "~*"
    9) "channels"
    10) "&*"
    11) "selectors"
    12) (empty array)

使用SHA256提供了在不以明文方式存储密码的能力，同时仍然允许非常快速的`AUTH`命令，这是Redis的一个非常重要的特性，并且与客户端对Redis的期望一致。

然而ACL *passwords*实际上并不是密码。它们是服务器和客户端之间的共享密钥，因为密码并不是由人类使用的身份验证令牌。例如：

* 没有长度限制，密码只会在一些客户端软件中被记住。在这种情境下，没有人需要记住密码。
* ACL密码不会保护任何其他东西。例如，它永远不会成为某个电子邮件账户的密码。
* 通常情况下，当您能够访问哈希密码本身，通过完全访问给定服务器的Redis命令，或者破坏系统本身，您已经可以访问密码所保护的内容：Redis实例的稳定性和其中包含的数据。

由于这个原因，为了使用一个使用时间和空间来增加密码破解困难的算法，减慢密码验证的速度是一个非常糟糕的选择。相反，我们建议生成强密码，这样即使有哈希值，也没有人能够使用字典或暴力攻击来破解它。为了做到这一点，我们有一个特殊的ACL命令`ACL GENPASS`，它使用系统的加密伪随机生成器来生成密码：

    > ACL GENPASS
    "dd721260bfe1b3d9601e7fbab36de6d04e2e67b0ef1c53de59d45950db0dd3cc"

以下的命令将输出一个32字节（256位）的伪随机字符串转换为一个64字节的字母数字字符串。这个长度足够避免攻击，同时也足够容易管理、剪切、粘贴和存储。这是生成Redis密码的推荐方式。

## 使用外部ACL文件

在Redis配置中，有两种方法可以存储用户：

1. 可以直接在 `redis.conf` 文件中指定用户。
2. 可以指定一个外部的 ACL 文件。

两种方法*相互不兼容*，因此Redis会要求您使用其中一种。在`redis.conf`中指定用户对于简单的用例是不错的选择。当需要定义多个用户的时候，在复杂的环境中，我们建议您使用ACL文件。

`redis.conf`文件内部使用的格式和外部访问控制列表(ACL)文件的格式完全相同，因此从一个切换到另一个是很简单的操作，具体如下所示：

    user <username> ... acl rules ...

例如：

    user worker +@list +@connection ~jobs:* on >ffa9203c493aa99

当您想要使用外部ACL文件时，您需要指定名为`aclfile`的配置指令，如下所示：

    aclfile /etc/redis/users.acl

当您在`redis.conf`文件中直接指定一些用户时，可以使用`CONFIG REWRITE`将新的用户配置存储在文件中，而不是通过重写。

外部ACL文件的功能更加强大。您可以执行以下操作：

* 如果您手动修改了ACL文件并且希望Redis重新加载新配置，请使用`ACL LOAD`。请注意，此命令只能在所有用户都正确指定的情况下加载文件。否则，将向用户报告错误，并保留旧的配置。
* 使用`ACL SAVE`将当前ACL配置保存到ACL文件中。

请注意，`CONFIG REWRITE` 命令不会触发 `ACL SAVE` 命令。当你使用 ACL 文件时，配置和 ACL 是分开处理的。

## Sentinel 和备份的ACL规则

如果您不想为Redis副本和Redis Sentinel实例提供完全访问权限，以下是必须允许的命令集，以便一切正常运行。

对于 Sentinel，在主节点和从节点实例中都允许用户访问以下命令：

* AUTH，CLIENT，SUBSCRIBE，SCRIPT，PUBLISH，PING，INFO，MULTI，SLAVEOF，CONFIG，CLIENT，EXEC。

Sentinel不需要访问数据库中的任何键，但需要使用Pub/Sub，因此ACL规则如下（注意：不需要`AUTH`，因为它总是允许的）：

    ACL SETUSER sentinel-user on >somepassword allchannels +multi +slaveof +ping +exec +subscribe +config|rewrite +role +publish +info +client|setname +client|kill +script|kill

Redis复制需要在主实例上允许以下命令：

* PSYNC，REPLCONF，PING

无需访问任何键，因此将其翻译为以下规则：

    ACL setuser replica-user on >somepassword +psync +replconf +ping

请注意，您无需配置副本以允许主节点执行任何一组命令。从副本的角度来看，主节点始终被视为根用户进行身份验证。
