---
layout: post
title: SpringBoot实现JPA读写分离
tags:  [Java, SpringBoot,JPA,MySQL]
categories: [Java]
keywords: SpringBoot,JPA,MySQL,读写分离
---

前面的文章讲解了 MyBatis 和 MongoDb 的读写分离配置，今天将讲解一下 JPA 的读写分离配置, 通过事务的只读属性值来切主从换数据源。



### 添加依赖

本文使用 Gradle 作为构建工具， 首先在 build.gradle 中添加 jpa 和 MySQL 依赖。
```
buildscript {
    ext {
        springBootVersion = '2.1.1.RELEASE'
    }
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}

apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'

group = 'com.jpatest'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = 1.8

repositories {
    maven {url "http://maven.aliyun.com/nexus/content/groups/public/" }
    mavenCentral()
}


dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web')
    implementation('org.springframework.boot:spring-boot-starter-data-jpa')
    runtimeOnly('mysql:mysql-connector-java:8.0.13')
    testImplementation('org.springframework.boot:spring-boot-starter-test')

    compile group: 'com.alibaba', name: 'druid', version: '1.1.12'

}

```

###  配置数据源
SpringBoot 配置文件 application.yml 中添加数据源信息
```
server:
  port: 8080

spring:
  jpa:
    database: mysql
    generate-ddl: true
    show-sql: true
    hibernate:
      ddl-auto: update
      naming:
        physical-strategy: org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy

dataSource:
  driverClass: com.mysql.cj.jdbc.Driver
  master:
    url: jdbc:mysql://localhost:3306/test?autoReconnect=true&useUnicode=true&serverTimezone=UTC&useSSL=false
    username: root
    password: 666788
    maxActive: 10
    minIdle: 0
  slave:
    url: jdbc:mysql://localhost:3306/test?autoReconnect=true&useUnicode=true&serverTimezone=UTC&useSSL=false
    username: root
    password: 666788
    maxActive: 10
    minIdle: 0
```


配置数据源
```
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(basePackages = {"com.jpatest.demo.repository"})
public class DataSourceConfig {

    @Autowired
    private Environment props;

    /**
     * basic setting
     */
    private DruidDataSource abstractDataSource() {
        DruidDataSource abstractDataSource = new DruidDataSource();
        abstractDataSource.setDriverClassName(props.getProperty("dataSource.driverClass"));
        abstractDataSource.setTestOnBorrow(true);
        abstractDataSource.setTestWhileIdle(true);
        abstractDataSource.setValidationQuery("SELECT 1");
        abstractDataSource.setMinEvictableIdleTimeMillis(30000);
        abstractDataSource.setPoolPreparedStatements(true);
        abstractDataSource.setMaxOpenPreparedStatements(100);
        return abstractDataSource;
    }

    /**
     * maste setting
     */
    @Bean(destroyMethod = "close", name="master")
    @Primary
    public DruidDataSource masterDataSource() {
        DruidDataSource masterDataSource = abstractDataSource();
        masterDataSource.setUrl(props.getProperty("dataSource.master.url"));
        masterDataSource.setUsername(props.getProperty("dataSource.master.username"));
        masterDataSource.setPassword(props.getProperty("dataSource.master.password"));
        masterDataSource.setMaxActive(Integer.parseInt(props.getProperty("dataSource.master.maxActive")));
        masterDataSource.setMinIdle(Integer.parseInt(props.getProperty("dataSource.master.minIdle")));
        return masterDataSource;
    }

    /**
     * slave setting
     */
    @Bean(destroyMethod = "close", name="slave")
    public DruidDataSource slaveDataSource() {
        DruidDataSource slaveDataSource = abstractDataSource();
        slaveDataSource.setUrl(props.getProperty("dataSource.slave.url"));
        slaveDataSource.setUsername(props.getProperty("dataSource.slave.username"));
        slaveDataSource.setPassword(props.getProperty("dataSource.slave.password"));
        slaveDataSource.setMaxActive(Integer.parseInt(props.getProperty("dataSource.slave.maxActive")));
        slaveDataSource.setMinIdle(Integer.parseInt(props.getProperty("dataSource.slave.minIdle")));
        return slaveDataSource;
    }

    @Bean(name="dynamicDataSource")
    public DataSource dynamicDataSource() throws IOException {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("master", masterDataSource());
        targetDataSources.put("slave", slaveDataSource());

        AbstractRoutingDataSource dynamicDataSource = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                String lookupKey = TransactionSynchronizationManager.isCurrentTransactionReadOnly() ? "slave" : "master";
                System.out.println("connected DataSource :" + lookupKey);
                return lookupKey;
            }
        };

        dynamicDataSource.setDefaultTargetDataSource(targetDataSources.get("master"));
        dynamicDataSource.setTargetDataSources(targetDataSources);
        return dynamicDataSource;
    }

    @Bean
    public DataSource dataSource() throws IOException {
        return new LazyConnectionDataSourceProxy(dynamicDataSource());
    }

    @Bean(name = "entityManagerFactory")
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(EntityManagerFactoryBuilder builder) throws IOException {
        return builder
                .dataSource(dataSource())
                .packages("com.jpatest.demo")
                .build();
    }

    @Bean(name = "transactionManager")
    JpaTransactionManager transactionManager(EntityManagerFactoryBuilder builder) throws IOException {
        return new JpaTransactionManager(entityManagerFactory(builder).getObject());
    }

}

```

### 测试
添加User实体类
```
@Data
@Entity
public class User {
    @Id
    @GeneratedValue
    private Integer id;
    private String name;
    private Integer sex;
    private LocalDateTime createTime;
    private LocalDateTime updateTime;
}
```

添加 Service 类
```
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    @Transactional(readOnly = true)
    public List<User> getAll() {
        return userRepository.findAll();
    }

    @Transactional(rollbackFor = Exception.class)
    public User save(User user) {
        return userRepository.save(user);
    }
}
```

添加 UserRepository 类
```
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
}
```

添加 UserServiceTest 测试类
```
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserServiceTest {

    @Autowired
    private UserService userService;

    @Test
    public void getAll() {
        List<User> list = userService.getAll();
        list.forEach(e -> System.out.println(e));
    }

    @Test
    public void save() {
        User user = new User();
        user.setName("小婷");
        user.setSex(0);
        user.setCreateTime(LocalDateTime.now());
        user = userService.save(user);
        System.out.println(user);
    }
}
```

运行测试类，如果出现下面的结果就说明读写分离正常工作了。
```
connected DataSource :slave
2018-12-22 18:12:01.424  INFO 7080 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-2} inited
User{id=1, name='zhansan', sex=1, createTime=null, updateTime=null}
User{id=2, name='xiaohong', sex=0, createTime=null, updateTime=null}

......
Hibernate: update hibernate_sequence set next_val= ? where next_val=?
Hibernate: insert into User (createTime, name, sex, updateTime, id) values (?, ?, ?, ?, ?)
connected DataSource :master
User{id=5, name='小婷', sex=0, createTime=2018-12-17T18:12:01.506687300, updateTime=null}
```
