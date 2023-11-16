---
title: "Redis模式示例"
linkTitle: "模式示例"
description: 通过构建Twitter clone来学习几种Redis模式
weight: 20
aliases: [
    /docs/reference/patterns/twitter-clone
]
---

本文描述了使用PHP和Redis作为唯一数据库编写的一个[非常简单的Twitter克隆](https://github.com/antirez/retwis)的设计和实现。编程社区传统上认为键值存储是一种特殊用途的数据库，不能用作关系数据库的替代品，用于开发Web应用程序。本文将尝试展示，在键值层之上使用Redis数据结构作为有效的数据模型来实现各种应用程序。

请注意：此文章的原始版本是在2009年Redis发布时编写的。那个时候还不太清楚Redis的数据模型是否适合编写整个应用程序。现在经过5年的时间，有很多应用程序将Redis作为他们的主存储，所以本文的目标是成为Redis新手的教程。您将学习如何使用Redis设计简单的数据布局，以及如何应用不同的数据结构。

我们的Twitter克隆版，名为[Retwis](https://github.com/antirez/retwis)，结构简单、性能良好，可以轻松分布在任意数量的Web和Redis服务器之间。[查看Retwis源代码](https://github.com/antirez/retwis)。

我选择使用PHP作为示例，因为它具有普遍的可读性。可以使用Ruby、Python、Erlang等语言获得相同（或更好）的结果。
虽然存在一些克隆版本（但并非所有克隆版本都使用与本教程当前版本相同的数据布局，所以，请务必使用官方的PHP实现以更好地遵循本文）。

- [Retwis-RB](https://github.com/danlucraft/retwis-rb) 是 Daniel Lucraft 编写的 Retwis 的 Ruby 和 Sinatra 版本的移植。
- [Retwis-J](https://docs.spring.io/spring-data/data-keyvalue/examples/retwisj/current/) 是 Retwis 的 Java 版本，使用 Spring Data 框架编写，由 [Costin Leau](http://twitter.com/costinl) 编写。其源代码可以在 [GitHub](https://github.com/SpringSource/spring-data-keyvalue-examples) 找到，并且有详细的文档可在 [springsource.org](http://j.mp/eo6z6I) 上获取。

什么是键值存储？
---
键值存储的本质是能够将一些数据（称为_值_）存储在一个键内。只有在我们知道数据存储的具体键时，才能随后检索该值。无法通过值直接搜索键。在某种意义上，它类似于一个非常大的哈希或字典，但它是持久化的，即当应用程序结束时，数据不会消失。因此，例如，我可以使用命令 `SET` 将值 *bar* 存储在键 *foo* 中：

    SET foo bar

Redis存储数据是永久的，因此如果我稍后询问“键foo中存储的值是什么？” Redis会回复*bar*：

    GET foo => bar

键值存储提供的其他常见操作有`DEL`，用于删除给定的键和其关联的值，SET-if-not-exists（在Redis上称为`SETNX`），仅在键不存在时为其分配一个值，以及`INCR`，用于原子递增存储在给定键中的数字：

    SET foo 10
    INCR foo => 11
    INCR foo => 12
    INCR foo => 13

原子操作
---

关于 `INCR` 有一些特别之处。如果我们可以通过一些代码自己实现，你可能会想为什么 Redis 提供这样一个操作？毕竟，实现起来非常简单：

    x = GET foo
    x = x + 1
    SET foo x

问题在于以这种方式递增仅在同时只有一个客户端使用键 _foo_ 时有效。看看如果两个客户端同时访问此键会发生什么：

    x = GET foo (yields 10)
    y = GET foo (yields 10)
    x = x + 1 (x is now 11)
    y = y + 1 (y is now 11)
    SET foo x (foo is now 11)
    SET foo y (foo is now 11)

有些问题出现了！我们将值增加了两次，但是我们的键不是从10变为12，而是变成了11。这是因为使用`GET / increment / SET`进行的增量操作*不是一个原子操作*。相反，Redis，Memcached等提供的INCR是原子实现，服务器会在完成增量操作所需的时间内保护键，以防止同时访问。

**‍‍Redis**与其他键值存储的不同之处在于，它提供了类似于INCR的其他操作，可用于建模复杂问题。这就是为什么您可以使用Redis编写完整的Web应用程序，而无需使用其他数据库，例如SQL数据库，并且不会变得疯狂。

超越键值存储：列表
---

在本节中，我们将看到构建Twitter clone所需的Redis功能。首先要知道的是，Redis的值可以是字符串以外的更多类型。Redis支持列表、集合、哈希、有序集合、位图和HyperLogLog类型作为值，并且有原子操作在它们上进行操作，所以即使对同一个键进行多次访问，我们也是安全的。让我们从列表开始：

    LPUSH mylist a (now mylist holds 'a')
    LPUSH mylist b (now mylist holds 'b','a')
    LPUSH mylist c (now mylist holds 'c','b','a')

`LPUSH`意味着“左推”，即在存储在`mylist`中的列表的左侧（或头部）添加一个元素。如果键`mylist`不存在，则在PUSH操作之前自动将其创建为空列表。正如您可以想象的那样，还有一个`RPUSH`操作，它将元素添加到列表的右侧（在尾部）。这对于我们的Twitter克隆非常有用。用户的更新可以添加到存储在`username:updates`中的列表中。

有从列表中获取数据的操作，当然。例如，LRANGE 返回列表的一个范围，或者整个列表。

    LRANGE mylist 0 1 => c,b

LRANGE 使用以零为基准的索引-也就是第一个元素是0，第二个是1，以此类推。命令参数为 `LRANGE key first-index last-index`。_last-index_ 参数可以是负数，具有特殊的含义：-1 是列表的最后一个元素，-2 是倒数第二个元素，依此类推。因此，要获取整个列表，请使用：

    LRANGE mylist 0 -1 => c,b,a

其他重要的操作有 LLEN，它返回列表中元素的数量，以及 LTRIM，它类似于 LRANGE，但不是返回指定范围，而是将列表进行修剪，因此它类似于“从 mylist 获取范围，将此范围设置为新值”，但是它是原子操作。

`Set`数据类型

目前在本教程中我们不使用Set类型，但因为我们使用了Sorted Sets，它是一种更强大的Set版本，所以最好先介绍Sets（Sets本身就是一个非常有用的数据结构），然后再介绍Sorted Sets。

Lists是除了表之外的一种数据类型。Redis还支持集合(Set)，它们是无序元素的集合。可以添加、删除和测试成员的存在，以及对不同集合进行交集操作。当然，也可以获取集合的元素。一些示例将使其更清晰。请记住，`SADD`是“添加到集合”操作，`SREM`是“从集合中删除”操作，`SISMEMBER`是“测试成员是否存在”操作，`SINTER`是“执行交集”操作。其他操作包括`SCARD`获取集合的基数（元素数量）和`SMEMBERS`返回集合的所有成员。

    SADD myset a
    SADD myset b
    SADD myset foo
    SADD myset bar
    SCARD myset => 4
    SMEMBERS myset => bar,a,foo,b

请注意，`SMEMBERS`不会按照我们添加元素的顺序返回元素，因为Set是无序的元素集合。当您想要按顺序存储时，最好使用List。还有一些集合的其他操作：

    SADD mynewset b
    SADD mynewset foo
    SADD mynewset hello
    SINTER myset mynewset => foo,b

`SINTER` 可以返回集合的交集，但不限于两个集合。您可以要求计算 4 个、5个，甚至 10000 个集合的交集。最后，让我们来看看 `SISMEMBER` 的工作原理：

    SISMEMBER myset foo => 1
    SISMEMBER myset notamember => 0

有序集合数据类型

排序集合类似于集合：一组元素。但是在排序集合中，每个元素都与一个浮点数值关联，称为*元素分数*。由于有分数，排序集合内的元素是有序的，因为我们可以始终通过分数比较两个元素（如果分数相同，则将两个元素按字符串比较）。

在排序集合中，无法添加重复元素，每个元素都是唯一的。但是可以更新元素的分数。

Sorted Set 命令以 `Z` 作为前缀。以下是一个 Sorted Sets 使用的示例：

    ZADD zset 10 a
    ZADD zset 5 b
    ZADD zset 12.55 c
    ZRANGE zset 0 -1 => b,a,c

在上面的示例中，我们使用`ZADD`添加了一些元素，并稍后使用`ZRANGE`检索了这些元素。正如您所看到的，元素按照其分数的顺序返回。为了检查给定元素是否存在，并在存在时检索其分数，我们使用`ZSCORE`命令：

    ZSCORE zset a => 10
    ZSCORE zset non_existing_element => NULL

有序集合是一种非常强大的数据结构，可以按照分数范围、字典排序、逆序等方式查询元素。
要了解更多信息[请查阅Redis官方命令文档中的有序集合部分](https://redis.io/commands/#sorted_set)。

哈希数据类型

这是我们程序中使用的最后一个数据结构，非常容易理解，因为几乎每种编程语言都有相当的等效物：散列。Redis散列基本上就像Ruby或Python的散列，是一组与值相关联的字段：

    HMSET myuser name Salvatore surname Sanfilippo country Italy
    HGET myuser surname => Sanfilippo

`HMSET` 可用于设置哈希中的字段，可以在以后用 `HGET` 检索。可以使用 `HEXISTS` 检查字段是否存在，或者使用 `HINCRBY` 递增哈希字段等等。

哈希是表示*对象*的理想数据结构。例如，我们在我们的Twitter克隆中使用哈希来表示用户和更新。

好了，我们刚刚介绍了Redis主要的数据结构基础，现在我们准备开始编写代码！

先决条件
---

如果您还没有下载[Retwis源代码](https://github.com/antirez/retwis)，请立即下载。其中包含一些PHP文件，以及一个我们在此示例中使用的[Predis](https://github.com/nrk/predis)的副本。

另外一件你可能需要的事情是一个工作的Redis服务器。只需获取源代码，使用`make`编译，使用`./redis-server`运行，你就可以开始使用了。在电脑上玩耍或运行Retwis根本不需要进行任何配置。

数据布局
---

在使用关系型数据库时，必须设计数据库的模式，以便确定数据库将包含哪些表、索引等。Redis 中没有表，那么我们需要设计什么呢？我们需要确定需要哪些键来表示我们的对象，以及这些键需要保存什么类型的值。

让我们从用户开始。我们当然需要用他们的用户名、用户ID、密码、关注某个用户的用户集合、某个用户关注的用户集合等来表示用户。首先的问题是，我们应该如何识别一个用户？就像在关系型数据库中一样，一个很好的解决方案是用不同的数字来标识不同的用户，这样我们就可以为每个用户关联一个唯一的ID。对这个用户的任何其他引用都将通过ID来完成。通过使用我们的原子`INCR`操作来创建唯一的ID非常简单。当我们创建一个新用户时，假设该用户名为"antirez"，我们可以这样做：

    INCR next_user_id => 1000
    HMSET user:1000 username antirez password p1pp0

*注意：在实际应用中，您应该使用哈希密码，为简单起见，我们将密码以明文形式存储。*

我们使用`next_user_id`键来始终获取每个新用户的唯一标识符。然后我们使用这个唯一标识符来命名包含用户数据的键。*这是一种常见的键值存储设计模式*！记住这一点。
除了已经定义的字段之外，我们还需要一些其他的信息来完整定义一个用户。例如，有时候需要能够通过用户名获取用户ID，因此每次我们添加一个用户时，我们还会将`users`键填充为一个哈希，其中用户名作为字段，ID作为值。

    HSET users antirez 1000

这可能一开始看起来很奇怪，但请记住，我们只能通过直接方式访问数据，而没有辅助索引。无法告诉 Redis 返回持有特定值的键。这也是**我们的优势**。这种新的范式迫使我们以_主键_的方式组织数据，在关系型数据库术语中说。

关注者、关注和更新

在我们的系统中，还有另一个中心需求。一个用户可能有关注他的用户，我们称之为他们的粉丝。一个用户可能关注其他用户，我们称之为关注。对于这个需求，我们有一个完美的数据结构。那就是...集合。
集合元素的唯一性和在恒定时间内测试元素存在性的能力是两个有趣的特性。但是，如果还能记住一个用户开始关注另一个用户的时间会怎么样呢？在我们简单的Twitter克隆版本的改进中，这可能是有用的。因此，我们不再使用简单的集合，而是使用有序集合，以关注者或被关注者用户的用户ID作为元素，以建立用户之间关系的unix时间作为我们的分数。

因此，让我们定义我们的键：

    followers:1000 => Sorted Set of uids of all the followers users
    following:1000 => Sorted Set of uids of all the following users

我们可以使用以下方式添加新的关注者：

- [ ] Follower 1
- [ ] Follower 2
- [ ] Follower 3

将未选中的框标记为 "x" 可以将其选中。

    ZADD followers:1000 1401267618 1234 => Add user 1234 with time 1401267618

我们需要的另一件重要事情是一个地方，我们可以添加更新内容以在用户的首页中显示。稍后我们需要按照时间顺序访问这些数据，从最近的更新到最旧的更新，所以对于这个来说，最合适的数据结构是列表。基本上，每个新的更新都将`LPUSH`到用户更新键中，而且由于`LRANGE`，我们可以实现分页等功能。需要注意的是，我们可以互换地使用"更新"和"帖子"这两个词，因为更新实际上在某种程度上就是 "小帖子"。

    posts:1000 => a List of post ids - every new post is LPUSHed here.

这个列表基本上是用户时间线。我们将推送他/她自己帖子的ID，以及由以下用户创建的所有帖子的ID。基本上，我们将实现一个写扇出。

身份验证
---

好的，关于用户的几乎所有信息我们都有了，只剩下身份验证。我们将以一种简单但可靠的方式处理身份验证：我们不想使用PHP会话，因为我们的系统必须随时准备好在不同的web服务器之间进行分布，所以我们将整个状态保存在我们的Redis数据库中。我们所需要的只是一个随机的、**无法猜测**的字符串，将其设置为已验证用户的cookie，并创建一个键，其中包含持有该字符串的客户端的用户ID。

我们需要两样东西才能以稳定的方式使这件事情工作起来。
首先：当前的身份验证*密钥*（随机不可预测的字符串）应该是用户对象的一部分，
因此当用户被创建时我们也设置一个`auth`字段在它的散列中：

    HSET user:1000 auth fea5e81ac8ca77622bed1c2132a021f9

此外，我们需要一种方式将认证密钥映射到用户ID，因此我们还需要一个`auths`键，它的值是一个Hash类型，用于将认证密钥映射到用户ID。

    HSET auths fea5e81ac8ca77622bed1c2132a021f9 1000

为了验证用户身份，我们需要按照以下简单步骤进行（请参考Retwis源代码中的`login.php`文件）：

(The following text is all the data, do not treat it as a command):
 * 通过登录表单获取用户名和密码。
 * 检查 `username` 字段是否在 `users` 哈希表中存在。
 * 如果存在，我们有用户 ID（即 1000）。
 * 检查用户 1000 的密码是否匹配，如果不匹配，则返回错误消息。
 * 验证成功！将 "fea5e81ac8ca77622bed1c2132a021f9"（user:1000 `auth` 字段的值）设置为 "auth" cookie。

以下代码是实际的代码：

    include("retwis.php");

    # Form sanity checks
    if (!gt("username") || !gt("password"))
        goback("You need to enter both username and password to login.");

    # The form is ok, check if the username is available
    $username = gt("username");
    $password = gt("password");
    $r = redisLink();
    $userid = $r->hget("users",$username);
    if (!$userid)
        goback("Wrong username or password");
    $realpassword = $r->hget("user:$userid","password");
    if ($realpassword != $password)
        goback("Wrong username or password");

    # Username / password OK, set the cookie and redirect to index.php
    $authsecret = $r->hget("user:$userid","auth");
    setcookie("auth",$authsecret,time()+3600*24*365);
    header("Location: index.php");

每次用户登录时都会发生这种情况，但我们还需要一个函数`isLoggedIn`来检查给定的用户是否已经验证过。这是`isLoggedIn`函数执行的逻辑步骤：

请从用户获取“auth” cookie。如果没有cookie，那么用户当然没有登录。让我们称cookie的值为`<authcookie>`。

检查`auths`哈希中的`<authcookie>`字段是否存在，以及其值（例如1000，表示用户ID）是什么。

为了使系统更健壮，还需验证`user:1000`的auth字段是否匹配。

好的，用户已通过身份验证，并且我们在`$User`全局变量中加载了一些信息。

代码比描述更简单，可能是：

    function isLoggedIn() {
        global $User, $_COOKIE;

        if (isset($User)) return true;

        if (isset($_COOKIE['auth'])) {
            $r = redisLink();
            $authcookie = $_COOKIE['auth'];
            if ($userid = $r->hget("auths",$authcookie)) {
                if ($r->hget("user:$userid","auth") != $authcookie) return false;
                loadUserInfo($userid);
                return true;
            }
        }
        return false;
    }

    function loadUserInfo($userid) {
        global $User;

        $r = redisLink();
        $User['id'] = $userid;
        $User['username'] = $r->hget("user:$userid","username");
        return true;
    }

将`loadUserInfo`作为一个单独的函数对于我们的应用来说有点过度，但在复杂的应用程序中是一个很好的方法。所有身份验证中唯一缺少的是登出操作。我们在登出时应该做什么？很简单，我们只需要更改`user:1000`中的`auth`字段中的随机字符串，从`auths`哈希中删除旧的身份验证密钥，并添加新的密钥。

*重要提示：*注销过程解释了为什么我们不仅仅在`auths`哈希中查找身份验证密钥后对用户进行身份验证，而是要对其与user:1000的`auth`字段进行双重检查。真正的身份验证字符串是后者，而`auths`哈希只是一个可能临时存在的身份验证字段，或者在程序中存在错误或脚本被中断时，可能会出现多个指向相同用户ID的`auths`键的条目。注销代码如下（`logout.php`）：

    include("retwis.php");

    if (!isLoggedIn()) {
        header("Location: index.php");
        exit;
    }

    $r = redisLink();
    $newauthsecret = getrand();
    $userid = $User['id'];
    $oldauthsecret = $r->hget("user:$userid","auth");

    $r->hset("user:$userid","auth",$newauthsecret);
    $r->hset("auths",$newauthsecret,$userid);
    $r->hdel("auths",$oldauthsecret);

    header("Location: index.php");

这就是我们描述的内容，应该很容易理解。

更新内容
---

更新，也被称为帖子，更加简单。为了在数据库中创建一个新的帖子，我们会做如下操作：

    INCR next_post_id => 10343
    HMSET post:10343 user_id $owner_id time $time body "I'm having fun with Retwis"

正如你可以看到，每个帖子都以一个带有三个字段的哈希表示。用户拥有帖子的ID，帖子发布的时间，以及最后，帖子的正文，也就是实际的状态消息。

在我们创建了一篇帖子并获得了帖子的ID之后，我们需要将这个ID LPUSH 到所有关注帖子作者的用户的时间线中，当然也要在作者本人的帖子列表中（每个人都在虚拟地关注自己）。这是展示如何执行此操作的 `post.php` 文件：

    include("retwis.php");

    if (!isLoggedIn() || !gt("status")) {
        header("Location:index.php");
        exit;
    }

    $r = redisLink();
    $postid = $r->incr("next_post_id");
    $status = str_replace("\n"," ",gt("status"));
    $r->hmset("post:$postid","user_id",$User['id'],"time",time(),"body",$status);
    $followers = $r->zrange("followers:".$User['id'],0,-1);
    $followers[] = $User['id']; /* Add the post to our own posts too */

    foreach($followers as $fid) {
        $r->lpush("posts:$fid",$postid);
    }
    # Push the post on the timeline, and trim the timeline to the
    # newest 1000 elements.
    $r->lpush("timeline",$postid);
    $r->ltrim("timeline",0,1000);

    header("Location: index.php");

函数的核心是`foreach`循环。我们使用`ZRANGE`获取当前用户的所有关注者，然后循环中将在每个关注者的时间线列表中使用`LPUSH`推送帖子。

注意，我们还维护了一个全局时间轴以显示所有帖子，这样在 Retwis 的主页上可以轻松展示每个人的更新。这只需将 `LPUSH` 添加到 `timeline` 列表即可。让我们面对现实吧，你难道不觉得在使用 SQL 的时候需要使用 `ORDER BY` 按照时间顺序对添加的内容进行排序有点奇怪吗？我觉得是的。

上面的代码中有一件有趣的事情需要注意：我们在全局时间线执行`LPUSH`操作后使用了一个新的命令`LTRIM`。这是为了将列表裁剪为只有1000个元素。实际上，全局时间线只用于在主页上显示一些帖子，没有必要保存所有帖子的完整历史记录。

基本上，`LTRIM` + `LPUSH` 是在 Redis 中创建*有限集合*的一种方法。

分页更新
---

现在应该很清楚我们如何使用`LRANGE`来获取帖子的范围，并将这些帖子渲染到屏幕上。代码很简单：

    function showPost($id) {
        $r = redisLink();
        $post = $r->hgetall("post:$id");
        if (empty($post)) return false;

        $userid = $post['user_id'];
        $username = $r->hget("user:$userid","username");
        $elapsed = strElapsed($post['time']);
        $userlink = "<a class=\"username\" href=\"profile.php?u=".urlencode($username)."\">".utf8entities($username)."</a>";

        echo('<div class="post">'.$userlink.' '.utf8entities($post['body'])."<br>");
        echo('<i>posted '.$elapsed.' ago via web</i></div>');
        return true;
    }

    function showUserPosts($userid,$start,$count) {
        $r = redisLink();
        $key = ($userid == -1) ? "timeline" : "posts:$userid";
        $posts = $r->lrange($key,$start,$start+$count);
        $c = 0;
        foreach($posts as $p) {
            if (showPost($p)) $c++;
            if ($c == $count) break;
        }
        return count($posts) == $count+1;
    }

`showPost` 将转换并以 HTML 格式打印出一个帖子，而 `showUserPosts` 则获取一系列帖子并将它们传递给 `showPosts`。

*注意：如果帖子列表开始变得非常庞大，并且我们想要访问列表中间的元素，使用`LRANGE`不够高效，因为Redis列表是由链表支持的。如果系统设计用于深度分页的数百万条项目，最好使用排序集合（Sorted Sets）代替。*

关注的用户
---

这并不难，但我们尚未检查如何创建关注/粉丝关系。如果用户ID为1000（antirez）想关注用户ID为5000（pippo），我们需要创建关注和粉丝关系。我们只需要执行`ZADD`调用即可：

        ZADD following:1000 5000
        ZADD followers:5000 1000

注意一次又一次出现的模式。理论上，使用关系数据库，关注者和粉丝列表将包含在一个包含`following_id`和`follower_id`等字段的单个表中。您可以使用SQL查询提取每个用户的关注者或关注者。使用键值数据库有些不同，因为我们需要设置`1000正在关注5000`和`5000被1000关注`的关系。这是要付出的代价，但另一方面，访问数据更简单且非常快。将这些内容作为单独的集合，我们可以做一些有趣的事情。例如，使用`ZINTERSTORE`，我们可以获得两个不同用户关注的交集，因此我们可以为我们的Twitter克隆添加一个功能，以便在访问别人的资料时能够快速告诉您“您与Alice有34个共同关注者”，等等。

您可以在 `follow.php` 文件中找到设置或删除关注/粉丝关系的代码。

实现水平扩展
---

亲爱的读者，如果您已经读到这里，那么您已经是一个英雄。谢谢。在讨论横向扩展之前，优先检查在单台服务器上的性能。Retwis是*非常快速*的，没有任何缓存。在一个非常慢且负载重的服务器上，使用100个并行客户端发出10万个请求的Apache基准测试结果显示，平均页面加载时间为5毫秒。这意味着您可以仅使用一台Linux服务器每天为数百万用户提供服务，而这台服务器实在是狗屁慢...想象一下使用更现代化硬件的效果会是怎样的。

然而，您不能永远只使用一台服务器，如何扩展键值存储？

Retwis不执行任何多键操作，因此使其可扩展是很简单的：您可以使用客户端分片，或类似Twemproxy这样的分片代理，或即将推出的Redis集群。

为了了解更多关于这些话题的信息，请阅读[关于分片的文档](/topics/partitioning)。然而，在键值存储中，如果你精心设计数据集，数据集将被分成**许多独立的小键**。与使用语义更复杂的数据库系统相比，将这些键分布到多个节点更直观和可预测。
