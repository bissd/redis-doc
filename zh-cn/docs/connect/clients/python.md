---
title: "Python指南"
linkTitle: "Python"
description: 将您的Python应用程序连接到Redis数据库
weight: 5
aliases:
  - /docs/clients/python/
  - /docs/redis-clients/python/
---

# 安装 Redis 和 Redis 客户端，然后将您的 Python 应用程序连接到 Redis 数据库。

## redis-py
redis-py是Python的一个Redis客户端库。它使您能够与Redis服务器进行交互，执行各种操作，例如设置和获取键值、发送命令和执行事务等。redis-py是一个功能强大且易于使用的库，提供了完整的Redis命令支持，并具有高效的连接池管理和线程安全等特性。无论是在开发Web应用程序还是构建数据处理管道等场景中，redis-py都是一个值得选择的工具。

使用 [redis-py](https://github.com/redis/redis-py) 作为 Redis 的客户端进行开始。

`redis-py`需要一个正在运行的Redis或[Redis Stack](/docs/getting-started/install-stack/)服务器。请参阅[入门指南](/docs/getting-started/)获取Redis安装说明。

### 安装

要安装 `redis-py`，输入以下命令：

```bash
pip install redis
```

为了提高性能，请安装支持[`hiredis`](https://github.com/redis/hiredis)的Redis。这提供了一个编译好的响应解析器，并且对于大多数情况下不需要任何代码更改。默认情况下，如果可用的`hiredis`版本大于等于1.0，`redis-py`会尝试使用它来解析响应。

```bash
pip install redis[hiredis]
```

### 连接

使用 Python 连接到本地主机的 6379 端口，并在 Redis 中设置一个值，然后将其检索出来。所有的响应都以字节形式返回。要接收解码后的字符串，请设置 `decode_responses=True`。有关更多的连接选项，请参见[这些示例](https://redis.readthedocs.io/en/stable/examples.html)。

```python
r = redis.Redis(host='localhost', port=6379, decode_responses=True)
```

存储和检索一个简单的字符串。

```python
r.set('foo', 'bar')
# True
r.get('foo')
# bar
```

存储和检索一个字典。

```python
r.hset('user-session:123', mapping={
    'name': 'John',
    "surname": 'Smith',
    "company": 'Redis',
    "age": 29
})
# True

r.hgetall('user-session:123')
# {'surname': 'Smith', 'name': 'John', 'company': 'Redis', 'age': '29'}
```

#### 连接到 Redis 集群

要连接到一个 Redis 集群，使用 `RedisCluster`。

```python
from redis.cluster import RedisCluster

rc = RedisCluster(host='localhost', port=16379)

print(rc.get_nodes())
# [[host=127.0.0.1,port=16379,name=127.0.0.1:16379,server_type=primary,redis_connection=Redis<ConnectionPool<Connection<host=127.0.0.1,port=16379,db=0>>>], ...

rc.set('foo', 'bar')
# True

rc.get('foo')
# b'bar'
```
更多信息，请参阅[redis-py集群](https://redis-py.readthedocs.io/en/stable/clustering.html)。

#### 使用TLS连接到您的生产Redis

在部署应用程序时，请使用TLS，并遵循[Redis安全指南](/docs/management/security/)。

```python
import redis

r = redis.Redis(
    host="my-redis.cloud.redislabs.com", port=6379,
    username="default", # use your Redis user. More info https://redis.io/docs/management/security/acl/
    password="secret", # use your Redis password
    ssl=True,
    ssl_certfile="./redis_user.crt",
    ssl_keyfile="./redis_user_private.key",
    ssl_ca_certs="./redis_ca.pem",
)
r.set('foo', 'bar')
# True

r.get('foo')
# b'bar'
```
请查看[redis-py TLS示例](https://redis-py.readthedocs.io/en/stable/examples/ssl_connection_examples.html)获取更多信息。

### 示例：对JSON文档进行索引和查询

确保您已安装Redis Stack和`redis-py`。导入依赖项：

```python
import redis
from redis.commands.json.path import Path
import redis.commands.search.aggregation as aggregations
import redis.commands.search.reducers as reducers
from redis.commands.search.field import TextField, NumericField, TagField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType
from redis.commands.search.query import NumericFilter, Query
```

连接到您的 Redis 数据库。

```python
r = redis.Redis(host='localhost', port=6379)
```

让我们创建一些测试数据来添加到您的数据库中。

```python
user1 = {
    "name": "Paul John",
    "email": "paul.john@example.com",
    "age": 42,
    "city": "London"
}
user2 = {
    "name": "Eden Zamir",
    "email": "eden.zamir@example.com",
    "age": 29,
    "city": "Tel Aviv"
}
user3 = {
    "name": "Paul Zamir",
    "email": "paul.zamir@example.com",
    "age": 35,
    "city": "Tel Aviv"
}
```

使用 `schema` 定义索引字段及其数据类型。使用 JSON 路径表达式将特定的 JSON 元素映射到模式字段。

```python
schema = (
    TextField("$.name", as_name="name"), 
    TagField("$.city", as_name="city"), 
    NumericField("$.age", as_name="age")
)
```

创建索引。在此示例中，将对所有带有键前缀 `user:` 的 JSON 文档进行索引。有关更多信息，请参阅[查询语法](/docs/interact/search-and-query/query/)。

```python
rs = r.ft("idx:users")
rs.create_index(
    schema,
    definition=IndexDefinition(
        prefix=["user:"], index_type=IndexType.JSON
    )
)
# b'OK'
```

使用`JSON.SET`在指定路径中设置每个用户的值。

```python
r.json().set("user:1", Path.root_path(), user1)
r.json().set("user:2", Path.root_path(), user2)
r.json().set("user:3", Path.root_path(), user3)
```

让我们找到用户“Paul”，并根据年龄筛选结果。

```python
res = rs.search(
    Query("Paul @age:[30 40]")
)
# Result{1 total, docs: [Document {'id': 'user:3', 'payload': None, 'json': '{"name":"Paul Zamir","email":"paul.zamir@example.com","age":35,"city":"Tel Aviv"}'}]}
```

使用JSON路径表达式查询。

```python
rs.search(
    Query("Paul").return_field("$.city", as_field="city")
).docs
# [Document {'id': 'user:1', 'payload': None, 'city': 'London'}, Document {'id': 'user:3', 'payload': None, 'city': 'Tel Aviv'}]
```

使用`FT.AGGREGATE`来聚合你的结果。

```python
req = aggregations.AggregateRequest("*").group_by('@city', reducers.count().alias('count'))
print(rs.aggregate(req).rows)
# [[b'city', b'Tel Aviv', b'count', b'2'], [b'city', b'London', b'count', b'1']]
```

### 了解更多

* [命令参考](https://redis-py.readthedocs.io/en/stable/commands.html)
* [教程](https://redis.readthedocs.io/en/stable/examples.html)
* [GitHub](https://github.com/redis/redis-py)
 
