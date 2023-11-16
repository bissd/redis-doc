---
title: "Java指南"
linkTitle: "Java"
description: 将您的Java应用程序连接到Redis数据库
weight: 3
aliases:
  - /docs/clients/java/
  - /docs/redis-clients/java/
---

安装Redis和Redis客户端，然后将您的Java应用程序连接到Redis数据库。

## Jedis

[Jedis](https://github.com/redis/jedis) 是一个为了性能和易用性而设计的 Redis 的 Java 客户端。

### 安装

要将 `Jedis` 作为应用程序的依赖项，编辑依赖文件，如下所示。

* 如果您使用 **Maven**：

  ```xml
  <dependency>
      <groupId>redis.clients</groupId>
      <artifactId>jedis</artifactId>
      <version>4.3.1</version>
  </dependency>
  ```

* 如果您使用**Gradle**：

  ```
  repositories {
      mavenCentral()
  }
  //...
  dependencies {
      implementation 'redis.clients:jedis:4.3.1'
      //...
  }
  ```

* 如果您使用JAR文件，请从[Maven Central](https://central.sonatype.com/)或其他任何Maven仓库下载最新的Jedis和Apache Commons Pool2 JAR文件。

* 从[源码](https://github.com/redis/jedis)构建

### 连接

对于许多应用程序，最好使用连接池。您可以像这样实例化和使用 `Jedis` 连接池：

```java
package org.example;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;

public class Main {
    public static void main(String[] args) {
        JedisPool pool = new JedisPool("localhost", 6379);

        try (Jedis jedis = pool.getResource()) {
            // Store & Retrieve a simple string
            jedis.set("foo", "bar");
            System.out.println(jedis.get("foo")); // prints bar
            
            // Store & Retrieve a HashMap
            Map<String, String> hash = new HashMap<>();;
            hash.put("name", "John");
            hash.put("surname", "Smith");
            hash.put("company", "Redis");
            hash.put("age", "29");
            jedis.hset("user-session:123", hash);
            System.out.println(jedis.hgetAll("user-session:123"));
            // Prints: {name=John, surname=Smith, company=Redis, age=29}
        }
    }
}
```

因为为每个命令添加一个`try-with-resources`块可能很麻烦，所以考虑使用`JedisPooled`作为连接池的更简单方式。

```java
import redis.clients.jedis.JedisPooled;

//...

JedisPooled jedis = new JedisPooled("localhost", 6379);
jedis.set("foo", "bar");
System.out.println(jedis.get("foo")); // prints "bar"
```

#### 连接到 Redis 集群

要连接到一个 Redis 集群，使用 `JedisCluster`。

```java
import redis.clients.jedis.JedisCluster;
import redis.clients.jedis.HostAndPort;

//...

Set<HostAndPort> jedisClusterNodes = new HashSet<HostAndPort>();
jedisClusterNodes.add(new HostAndPort("127.0.0.1", 7379));
jedisClusterNodes.add(new HostAndPort("127.0.0.1", 7380));
JedisCluster jedis = new JedisCluster(jedisClusterNodes);
```

#### 使用TLS连接到您的生产Redis

在部署应用程序时，请使用TLS并遵循Redis安全指南。

在将您的应用程序连接到启用 TLS 的 Redis 服务器之前，请确保您的证书和私钥处于正确的格式中。

将用户证书和私钥从PEM格式转换为`pkcs12`格式，请运行以下命令：

```
openssl pkcs12 -export -in ./redis_user.crt -inkey ./redis_user_private.key -out redis-user-keystore.p12 -name "redis"
```

输入密码来保护您的`pkcs12`文件。

使用 JDK 随附的 [keytool](https://docs.oracle.com/en/java/javase/12/tools/keytool.html) 将服务器 (CA) 证书转换为 JKS 格式。

```
keytool -importcert -keystore truststore.jks \ 
  -storepass REPLACE_WITH_YOUR_PASSWORD \
  -file redis_ca.pem
```

使用此代码片段与您的Redis数据库建立安全连接。

```java
package org.example;

import redis.clients.jedis.*;

import javax.net.ssl.*;
import java.io.FileInputStream;
import java.io.IOException;
import java.security.GeneralSecurityException;
import java.security.KeyStore;

public class Main {

    public static void main(String[] args) throws GeneralSecurityException, IOException {
        HostAndPort address = new HostAndPort("my-redis-instance.cloud.redislabs.com", 6379);

        SSLSocketFactory sslFactory = createSslSocketFactory(
                "./truststore.jks",
                "secret!", // use the password you specified for keytool command
                "./redis-user-keystore.p12",
                "secret!" // use the password you specified for openssl command
        );

        JedisClientConfig config = DefaultJedisClientConfig.builder()
                .ssl(true).sslSocketFactory(sslFactory)
                .user("default") // use your Redis user. More info https://redis.io/docs/management/security/acl/
                .password("secret!") // use your Redis password
                .build();

        JedisPooled jedis = new JedisPooled(address, config);
        jedis.set("foo", "bar");
        System.out.println(jedis.get("foo")); // prints bar
    }

    private static SSLSocketFactory createSslSocketFactory(
            String caCertPath, String caCertPassword, String userCertPath, String userCertPassword)
            throws IOException, GeneralSecurityException {

        KeyStore keyStore = KeyStore.getInstance("pkcs12");
        keyStore.load(new FileInputStream(userCertPath), userCertPassword.toCharArray());

        KeyStore trustStore = KeyStore.getInstance("jks");
        trustStore.load(new FileInputStream(caCertPath), caCertPassword.toCharArray());

        TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance("X509");
        trustManagerFactory.init(trustStore);

        KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance("PKIX");
        keyManagerFactory.init(keyStore, userCertPassword.toCharArray());

        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(keyManagerFactory.getKeyManagers(), trustManagerFactory.getTrustManagers(), null);

        return sslContext.getSocketFactory();
    }
}
```

### 示例：索引和查询JSON文档

请确保您已经安装了Redis Stack和`Jedis`。

导入依赖项并添加一个示例的 `User` 类：

```java
import redis.clients.jedis.JedisPooled;
import redis.clients.jedis.search.*;
import redis.clients.jedis.search.aggr.*;
import redis.clients.jedis.search.schemafields.*;

class User {
    private String name;
    private String email;
    private int age;
    private String city;

    public User(String name, String email, int age, String city) {
        this.name = name;
        this.email = email;
        this.age = age;
        this.city = city;
    }

    //...
}
```

使用 `JedisPooled` 连接到您的 Redis 数据库。

```java
JedisPooled jedis = new JedisPooled("localhost", 6379);
```

让我们创建一些测试数据，添加到您的数据库中。

```java
User user1 = new User("Paul John", "paul.john@example.com", 42, "London");
User user2 = new User("Eden Zamir", "eden.zamir@example.com", 29, "Tel Aviv");
User user3 = new User("Paul Zamir", "paul.zamir@example.com", 35, "Tel Aviv");
```

创建索引。在此示例中，索引了所有键前缀为 `user:` 的 JSON 文档。有关更多信息，请参阅 [查询语法](/docs/interact/search-and-query/query/)。

```java
jedis.ftCreate("idx:users",
    FTCreateParams.createParams()
            .on(IndexDataType.JSON)
            .addPrefix("user:"),
    TextField.of("$.name").as("name"),
    TagField.of("$.city").as("city"),
    NumericField.of("$.age").as("age")
);
```

使用`JSON.SET`在指定路径设置每个用户的值。

```java
jedis.jsonSetWithEscape("user:1", user1);
jedis.jsonSetWithEscape("user:2", user2);
jedis.jsonSetWithEscape("user:3", user3);
```

让我们找到用户 `Paul` 并根据年龄筛选结果。

```java
var query = new Query("Paul @age:[30 40]");
var result = jedis.ftSearch("idx:users", query).getDocuments();
System.out.println(result);
// Prints: [id:user:3, score: 1.0, payload:null, properties:[$={"name":"Paul Zamir","email":"paul.zamir@example.com","age":35,"city":"Tel Aviv"}]]
```

只返回`city`字段。

```java
var city_query = new Query("Paul @age:[30 40]");
var city_result = jedis.ftSearch("idx:users", city_query.returnFields("city")).getDocuments();
System.out.println(city_result);
// Prints: [id:user:3, score: 1.0, payload:null, properties:[city=Tel Aviv]]
```

统计同一城市的所有用户数量。

```java
AggregationBuilder ab = new AggregationBuilder("*")
        .groupBy("@city", Reducers.count().as("count"));
AggregationResult ar = jedis.ftAggregate("idx:users", ab);

for (int idx=0; idx < ar.getTotalResults(); idx++) {
    System.out.println(ar.getRow(idx).getString("city") + " - " + ar.getRow(idx).getString("count"));
}
// Prints:
// London - 1
// Tel Aviv - 2
```

### 了解更多

* [Jedis API参考](https://www.javadoc.io/doc/redis.clients/jedis/latest/index.html)
* [GitHub](https://github.com/redis/jedis)
