---
title: "Node.js指南"
linkTitle: "Node.js"
description: 将您的Node.js应用程序连接到Redis数据库
weight: 4
aliases:
  - /docs/clients/nodejs/
  - /docs/redis-clients/nodejs/
---

为您的Node.js应用安装Redis和Redis客户端，然后连接到Redis数据库。

## node-redis

[节点-redis](https://github.com/redis/node-redis) 是一个现代、高性能的 Node.js Redis 客户端。
`node-redis` 需要正在运行的 Redis 或 [Redis 堆栈](https://redis.io/docs/getting-started/install-stack/) 服务器。有关 Redis 安装说明，请参阅 [入门指南](/docs/getting-started/)。

### 安装

要安装 node-redis，请运行以下命令:

```
npm install redis
```

### 连接

通过端口 6379 连接到本地主机。

```js
import { createClient } from 'redis';

const client = createClient();

client.on('error', err => console.log('Redis Client Error', err));

await client.connect();
```

存储和检索一个简单的字符串。

```js
await client.set('key', 'value');
const value = await client.get('key');
```

存储和检索地图。

```js
await client.hSet('user-session:123', {
    name: 'John',
    surname: 'Smith',
    company: 'Redis',
    age: 29
})

let userSession = await client.hGetAll('user-session:123');
console.log(JSON.stringify(userSession, null, 2));
/*
{
  "surname": "Smith",
  "name": "John",
  "company": "Redis",
  "age": "29"
}
 */
```

要连接到不同的主机或端口，请使用以下格式的连接字符串：`redis[s]：//[[用户名][:密码]@][主机][:端口][/db-number]`：

```js
createClient({
  url: 'redis://alice:foobared@awesome.redis.server:6380'
});
```
要检查客户端是否已连接并准备好发送命令，请使用 `client.isReady`，它会返回一个布尔值。`client.isOpen` 也可用。当客户端的底层套接字打开时，它会返回 `true`，当它没有打开时（例如，在客户端仍在连接或在网络错误后重新连接时），它会返回 `false`。

#### 连接到Redis集群

要连接到一个Redis集群，请使用 `createCluster`。

```js
import { createCluster } from 'redis';

const cluster = createCluster({
    rootNodes: [
        {
            url: 'redis://127.0.0.1:16379'
        },
        {
            url: 'redis://127.0.0.1:16380'
        },
        // ...
    ]
});

cluster.on('error', (err) => console.log('Redis Cluster Error', err));

await cluster.connect();

await cluster.set('foo', 'bar');
const value = await cluster.get('foo');
console.log(value); // returns 'bar'

await cluster.quit();
```

#### 使用TLS连接您的生产Redis

当您部署应用程序时，请使用TLS并遵循[Redis安全](/docs/management/security/)指南。

```js
const client = createClient({
    username: 'default', // use your Redis user. More info https://redis.io/docs/management/security/acl/
    password: 'secret', // use your password here
    socket: {
        host: 'my-redis.cloud.redislabs.com',
        port: 6379,
        tls: true,
        key: readFileSync('./redis_user_private.key'),
        cert: readFileSync('./redis_user.crt'),
        ca: [readFileSync('./redis_ca.pem')]
    }
});

client.on('error', (err) => console.log('Redis Client Error', err));

await client.connect();

await client.set('foo', 'bar');
const value = await client.get('foo');
console.log(value) // returns 'bar'

await client.disconnect();
```

您还可以使用离散参数和UNIX套接字。详情请参阅[客户端配置指南](https://github.com/redis/node-redis/blob/master/docs/client-configuration.md)。

### 示例：索引和查询JSON文档

确保你已经安装了Redis Stack和`node-redis`。导入依赖项：

```js
import {AggregateSteps, AggregateGroupByReducers, createClient, SchemaFieldTypes} from 'redis';
const client = createClient();
await client.connect();
```

创建索引。

```js
try {
    await client.ft.create('idx:users', {
        '$.name': {
            type: SchemaFieldTypes.TEXT,
            SORTABLE: true
        },
        '$.city': {
            type: SchemaFieldTypes.TEXT,
            AS: 'city'
        },
        '$.age': {
            type: SchemaFieldTypes.NUMERIC,
            AS: 'age'
        }
    }, {
        ON: 'JSON',
        PREFIX: 'user:'
    });
} catch (e) {
    if (e.message === 'Index already exists') {
        console.log('Index exists already, skipped creation.');
    } else {
        // Something went wrong, perhaps RediSearch isn't installed...
        console.error(e);
        process.exit(1);
    }
}
```

用于添加到数据库的JSON文档。

```js
await Promise.all([
    client.json.set('user:1', '$', {
        "name": "Paul John",
        "email": "paul.john@example.com",
        "age": 42,
        "city": "London"
    }),
    client.json.set('user:2', '$', {
        "name": "Eden Zamir",
        "email": "eden.zamir@example.com",
        "age": 29,
        "city": "Tel Aviv"
    }),
    client.json.set('user:3', '$', {
        "name": "Paul Zamir",
        "email": "paul.zamir@example.com",
        "age": 35,
        "city": "Tel Aviv"
    }),
]);
```

找到用户'Paul'并根据年龄进行结果筛选。

```js
let result = await client.ft.search(
    'idx:users',
    'Paul @age:[30 40]'
);
console.log(JSON.stringify(result, null, 2));
/*
{
  "total": 1,
  "documents": [
    {
      "id": "user:3",
      "value": {
        "name": "Paul Zamir",
        "email": "paul.zamir@example.com",
        "age": 35,
        "city": "Tel Aviv"
      }
    }
  ]
}
 */
```

只返回城市字段。

```js
result = await client.ft.search(
    'idx:users',
    'Paul @age:[30 40]',
    {
        RETURN: ['$.city']
    }
);
console.log(JSON.stringify(result, null, 2));

/*
{
  "total": 1,
  "documents": [
    {
      "id": "user:3",
      "value": {
        "$.city": "Tel Aviv"
      }
    }
  ]
}
 */
```
 
在相同城市中计算所有用户的数量。

```js
result = await client.ft.aggregate('idx:users', '*', {
    STEPS: [
        {
            type: AggregateSteps.GROUPBY,
            properties: ['@city'],
            REDUCE: [
                {
                    type: AggregateGroupByReducers.COUNT,
                    AS: 'count'
                }
            ]
        }
    ]
})
console.log(JSON.stringify(result, null, 2));

/*
{
  "total": 2,
  "results": [
    {
      "city": "London",
      "count": "1"
    },
    {
      "city": "Tel Aviv",
      "count": "2"
    }
  ]
}
 */

await client.quit();
```

### 了解更多

* [Redis 命令](https://redis.js.org/#node-redis-usage-redis-commands)
* [可编程性](https://redis.js.org/#node-redis-usage-programmability)
* [集群](https://redis.js.org/#node-redis-usage-clustering)
* [GitHub](https://github.com/redis/node-redis)
 
