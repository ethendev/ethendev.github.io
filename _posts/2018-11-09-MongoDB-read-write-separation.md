---
layout: post
title: MongoDB 复制集实现读写分离
tags:  [Java, MongoDB,读写分离]
categories: [Java]
keywords: MongoDB,Java,Replica Set
---

Mongodb 早期版本使用类似于 MySQL 的 master-slave 方式，但 slave 为只读，当 master 宕机后，slave 不能自动切换为 master。目前已经废弃，改为了复制集方式。



### 复制集(Replica Set)
MongoDB 的 复制集 是由一组 mongod 实例所组成的，并提供了数据冗余与高可用性。复制集中的成员有以下几种：

* Primary
* Secondaries

其中 Primary 是主节点，负责处理客户端请求，Secondaries 是从节点，负责复制主节点上的数据。我们也可以为复制集设置一个 投票节点。投票节点其本身并不包含数据集。但是，一旦当前的主节点不可用时，投票节点就会参与到新的主节点选举的投票中。

默认情况下，读写都指定到副本集中的 Primary 节点。对于读多写少的情况我们可以使用读写分离来减轻 DB 的压力。MongoDB 驱动程序支持五种读取首选项(Read Preference) 模式。


Read Preference | 描述
---|---
primary | 默认模式。 所有操作都从当前副本集 primary 读取。
primaryPreferred | 在大多数情况下，从 primary 读取，但如果不可用，则从 secondary 读取。
secondary | 所有操作都从 secondary 中读取。
secondaryPreferred | 在大多数情况下，从 secondary 读取，但如果没有 secondary  可用，则从 primary 读取。
nearest | 无论成员的类型如何，操作都从具有最小网络延迟的副本集成员读取

### 读写分离配置

要实现读写分离，需要将 ReadPreference 修改为 secondaryPreferred。

首先配置 Mongodb 地址等参数
```
mongodb:
  database: monitor
  username: test
  password: pwd
  replica-set:
    - host: 127.0.0.1
      port: 27017
    - host: 127.0.0.1
      port: 27018
    - host: 127.0.0.1
      port: 27019
```

配置类代码
```
@Setter
@Getter
@Component
@ConfigurationProperties(prefix = "mongodb")
public class MongoConfig {
    private String database;
    private String username;
    private char[] password;
    private List<Server> replicaSet;

    @Setter
    @Getter
    public static class Server {
        private String host;
        private int port;
    }
}
```

配置 MongoDbFactory
```
@Configuration
@EnableMongoRepositories("com.mongotest.repository")
public class DataSourceConfig {

    @Autowired
    private MongoConfig mongodbConfig;

    @Bean
    public MongoDbFactory mongoDbFactory() {
        List<ServerAddress> seeds = new ArrayList<>();
        for (MongoConfig.Server server : mongodbConfig.getReplicaSet()) {
            ServerAddress address = new ServerAddress(server.getHost(), server.getPort());
            seeds.add(address);
        }

        MongoCredential mongoCredential = MongoCredential.createScramSha1Credential(
                mongodbConfig.getUsername(),
                mongodbConfig.getDatabase(),
                mongodbConfig.getPassword());

        MongoClientOptions options = MongoClientOptions.builder()
                .readPreference(ReadPreference.secondaryPreferred()).build();
        MongoClient client = new MongoClient(seeds, mongoCredential, options);
        return new SimpleMongoDbFactory(client, mongodbConfig.getDatabase());
    }

}

```

至此，读写分离就配置好了。