---
title: "C#/.NET 指南"
linkTitle: "C#/.NET"
description: 将您的.NET应用程序连接到一个Redis数据库
weight: 1
aliases:
  - /docs/clients/dotnet/
  - /docs/redis-clients/dotnet/
---

请安装Redis和Redis客户端，然后将您的.NET应用程序连接到Redis数据库。

## NRedisStack

[NRedisStack](https://github.com/redis/NRedisStack) 是一个用于 Redis 的 .NET 客户端。
`NredisStack` 需要运行中的 Redis 或 [Redis Stack](https://redis.io/docs/getting-started/install-stack/) 服务器。请参阅 [入门指南](/docs/getting-started/) 获取 Redis 的安装说明。

### 安装

使用 `dotnet` CLI，运行：

```
dotnet add package NRedisStack
```

### 连接

连接到本地主机的6379端口。

```
using NRedisStack;
using NRedisStack.RedisStackCommands;
using StackExchange.Redis;
//...
ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("localhost");
IDatabase db = redis.GetDatabase();
```

存储和检索一个简单的字符串。

```csharp
db.StringSet("foo", "bar");
Console.WriteLine(db.StringGet("foo")); // prints bar
```

使用HashMap存储和检索数据。

```csharp
var hash = new HashEntry[] { 
    new HashEntry("name", "John"), 
    new HashEntry("surname", "Smith"),
    new HashEntry("company", "Redis"),
    new HashEntry("age", "29"),
    };
db.HashSet("user-session:123", hash);

var hashFields = db.HashGetAll("user-session:123");
Console.WriteLine(String.Join("; ", hashFields));
// Prints: 
// name: John; surname: Smith; company: Redis; age: 29
```

要访问Redis堆栈功能，您应该像这样使用适当的界面：

```
IBloomCommands bf = db.BF();
ICuckooCommands cf = db.CF();
ICmsCommands cms = db.CMS();
IGraphCommands graph = db.GRAPH();
ITopKCommands topk = db.TOPK();
ITdigestCommands tdigest = db.TDIGEST();
ISearchCommands ft = db.FT();
IJsonCommands json = db.JSON();
ITimeSeriesCommands ts = db.TS();
```

#### 连接到 Redis 集群

要连接到 Redis 集群，您只需在客户端配置中指定一个或所有集群端点即可：

```csharp
ConfigurationOptions options = new ConfigurationOptions
{
    //list of available nodes of the cluster along with the endpoint port.
    EndPoints = {
        { "localhost", 16379 },
        { "localhost", 16380 },
        // ...
    },            
};

ConnectionMultiplexer cluster = ConnectionMultiplexer.Connect(options);
IDatabase db = cluster.GetDatabase();

db.StringSet("foo", "bar");
Console.WriteLine(db.StringGet("foo")); // prints bar
```

#### 使用TLS连接到您的生产Redis

在部署应用程序时，请使用TLS并遵循[Redis安全](/docs/management/security/)指南。

连接应用程序到启用了TLS的Redis服务器之前，请确保您的证书和私钥处于正确的格式。

要将用户证书和私钥从PEM格式转换为`pfx`格式，请使用以下命令：

```bash
openssl pkcs12 -inkey redis_user_private.key -in redis_user.crt -export -out redis.pfx
```

请输入密码以保护您的 `pfx` 文件。

使用以下片段与您的Redis数据库建立安全连接。

```python
import redis
from redis import Redis
from redis import ConnectionPool

# Redis信息
host = 'your_redis_host'
port = your_redis_port
password = 'your_redis_password'

# 创建连接池
pool = ConnectionPool(host=host, port=port, password=password)

# 创建Redis实例
r = Redis(connection_pool=pool)

# 进行操作，例如设置键值对
r.set('key', 'value')
```
确保将`your_redis_host`替换为您的Redis主机地址，`your_redis_port`替换为您的Redis端口号，`your_redis_password`替换为您的Redis密码。

```csharp
ConfigurationOptions options = new ConfigurationOptions
{
    EndPoints = { { "my-redis.cloud.redislabs.com", 6379 } },
    User = "default",  // use your Redis user. More info https://redis.io/docs/management/security/acl/
    Password = "secret", // use your Redis password
    Ssl = true,
    SslProtocols = System.Security.Authentication.SslProtocols.Tls12                
};

options.CertificateSelection += delegate
{
    return new X509Certificate2("redis.pfx", "secret"); // use the password you specified for pfx file
};
options.CertificateValidation += ValidateServerCertificate;

bool ValidateServerCertificate(
        object sender,
        X509Certificate? certificate,
        X509Chain? chain,
        SslPolicyErrors sslPolicyErrors)
{
    if (certificate == null) {
        return false;       
    }

    var ca = new X509Certificate2("redis_ca.pem");
    bool verdict = (certificate.Issuer == ca.Subject);
    if (verdict) {
        return true;
    }
    Console.WriteLine("Certificate error: {0}", sslPolicyErrors);
    return false;
}

ConnectionMultiplexer muxer = ConnectionMultiplexer.Connect(options);   
            
//Creation of the connection to the DB
IDatabase conn = muxer.GetDatabase();

//send SET command
conn.StringSet("foo", "bar");

//send GET command and print the value
Console.WriteLine(conn.StringGet("foo"));   
```

### 示例：索引和查询JSON文档

该示例展示了如何使用`NRedisStack`将Redis搜索结果转换为JSON格式。

请确保您已安装Redis Stack和`NRedisStack`。

导入依赖项并连接到Redis服务器：

```csharp
using NRedisStack;
using NRedisStack.RedisStackCommands;
using NRedisStack.Search;
using NRedisStack.Search.Aggregation;
using NRedisStack.Search.Literals.Enums;
using StackExchange.Redis;

// ...

ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("localhost");
```

获取数据库的引用，用于搜索和JSON命令。

```csharp
var db = redis.GetDatabase();
var ft = db.FT();
var json = db.JSON();
```

让我们创建一些测试数据以添加到您的数据库中。

```csharp
var user1 = new {
    name = "Paul John",
    email = "paul.john@example.com",
    age = 42,
    city = "London"
};

var user2 = new {
    name = "Eden Zamir",
    email = "eden.zamir@example.com",
    age = 29,
    city = "Tel Aviv"
};

var user3 = new {
    name = "Paul Zamir",
    email = "paul.zamir@example.com",
    age = 35,
    city = "Tel Aviv"
};
```

在此示例中，将索引具有键前缀`user:`的所有JSON文档。有关更多信息，请参阅[查询语法](/docs/interact/search-and-query/query/)。

```csharp
var schema = new Schema()
    .AddTextField(new FieldName("$.name", "name"))
    .AddTagField(new FieldName("$.city", "city"))
    .AddNumericField(new FieldName("$.age", "age"));

ft.Create(
    "idx:users",
    new FTCreateParams().On(IndexDataType.JSON).Prefix("user:"),
    schema);
```

请使用`JSON.SET`在指定路径上设置每个用户的值。

```csharp
json.Set("user:1", "$", user1);
json.Set("user:2", "$", user2);
json.Set("user:3", "$", user3);
```

让我们找到用户 `Paul` 并按年龄筛选结果。

```csharp
var res = ft.Search("idx:users", new Query("Paul @age:[30 40]")).Documents.Select(x => x["json"]);
Console.WriteLine(string.Join("\n", res)); 
// Prints: {"name":"Paul Zamir","email":"paul.zamir@example.com","age":35,"city":"Tel Aviv"}
```

仅返回`city`字段。

```csharp
var res_cities = ft.Search("idx:users", new Query("Paul").ReturnFields(new FieldName("$.city", "city"))).Documents.Select(x => x["city"]);
Console.WriteLine(string.Join(", ", res_cities)); 
// Prints: London, Tel Aviv
```

在同一个城市中统计所有用户。

```csharp
var request = new AggregationRequest("*").GroupBy("@city", Reducers.Count().As("count"));
var result = ft.Aggregate("idx:users", request);

for (var i=0; i<result.TotalResults; i++)
{
    var row = result.GetRow(i);
    Console.WriteLine($"{row["city"]} - {row["count"]}");
}
// Prints:
// London - 1
// Tel Aviv - 2
```

### 了解更多

* [GitHub](https://github.com/redis/NRedisStack)
 
