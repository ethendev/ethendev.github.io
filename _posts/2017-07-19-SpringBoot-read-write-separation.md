---
layout: post
title: SpringBoot中数据源读写分离配置
tags:  [SpringBoot, 读写分离, DB]
categories: [SpringBoot]
---

* content
{:toc}

开发中常用到主从数据库来提高系统的性能。怎么样才能方便的实现主从读写分离呢？通过事务注解里面的可读属性readOnly的取值来自动切换数据源, 从而实现数据库读写分离。




### 主备数据源配置
```
@Configuration
@PropertySources({@PropertySource("classpath:palette.properties")})
@EnableTransactionManagement
@MapperScan(basePackages = "com.naver.palette", annotationClass = Mapper.class, sqlSessionFactoryRef = "sqlSessionFactory")
public class DataSourceConfig {
 
    @Autowired
    private Environment props;
 
    /**
     * basic setting
     */
    private BasicDataSource abstractDataSource() {
        BasicDataSource abstractDataSource = new BasicDataSource();
        abstractDataSource.setDriverClassName(props.getProperty("db.driverClassName"));
        abstractDataSource.setUsername(props.getProperty("db.username"));
        abstractDataSource.setPassword(props.getProperty("db.password"));
        abstractDataSource.setTestOnBorrow(true);
        abstractDataSource.setTestWhileIdle(true);
        abstractDataSource.setValidationQuery("SELECT 1");
        abstractDataSource.setTimeBetweenEvictionRunsMillis(300000);
        abstractDataSource.setNumTestsPerEvictionRun(10);
        abstractDataSource.setMinEvictableIdleTimeMillis(-1);
        abstractDataSource.setPoolPreparedStatements(true);
        abstractDataSource.setMaxOpenPreparedStatements(100);
        return abstractDataSource;
    }
 
    /**
     * maste setting
     */
    @Bean(destroyMethod = "close", name="master")
    @Primary
    public BasicDataSource masterDataSource() {
        BasicDataSource masterDataSource = abstractDataSource();
        masterDataSource.setUrl(props.getProperty("db.master.url"));
        masterDataSource.setMaxTotal(Integer.parseInt(props.getProperty("db.master.maxActive")));
        masterDataSource.setMaxIdle(Integer.parseInt(props.getProperty("db.master.maxIdle")));
        masterDataSource.setMinIdle(Integer.parseInt(props.getProperty("db.master.maxIdle")));
        return masterDataSource;
    }
 
    /**
     * slave setting
     */
    @Bean(destroyMethod = "close", name="slave")
    public BasicDataSource slaveDataSource() {
        BasicDataSource masterDataSource = abstractDataSource();
        masterDataSource.setUrl(props.getProperty("db.slave.url"));
        masterDataSource.setMaxTotal(Integer.parseInt(props.getProperty("db.slave.maxActive")));
        masterDataSource.setMaxIdle(Integer.parseInt(props.getProperty("db.slave.maxIdle")));
        masterDataSource.setMinIdle(Integer.parseInt(props.getProperty("db.slave.maxIdle")));
        return masterDataSource;
    }
 
    @Bean(name="dynamicDataSource")
    public ReplicationRoutingDataSource dynamicDataSource() throws IOException {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("master", masterDataSource());
        targetDataSources.put("slave", slaveDataSource());
 
        ReplicationRoutingDataSource dynamicDataSource = new ReplicationRoutingDataSource();
        dynamicDataSource.setDefaultTargetDataSource(targetDataSources.get("master"));
        dynamicDataSource.setTargetDataSources(targetDataSources);
        return dynamicDataSource;
    }
 
    public DataSource dataSource() throws IOException {
        return new LazyConnectionDataSourceProxy(dynamicDataSource());
    }
 
    @Bean
    public SqlSessionFactory sqlSessionFactory() throws Exception {
        SqlSessionFactoryBean sqlFactory = new SqlSessionFactoryBean();
        sqlFactory.setDataSource(dataSource());
 
        sqlFactory.setConfigLocation(new ClassPathResource("mybatis-config.xml"));
        sqlFactory.setTypeAliasesPackage("com.naver.palette");
        return sqlFactory.getObject();
    }
 
    @Bean
    public DataSourceTransactionManager txManager() throws IOException {
        DataSourceTransactionManager dataSourceTXManager = new DataSourceTransactionManager();
        dataSourceTXManager.setDataSource(dataSource());
        AnnotationTransactionAspect.aspectOf().setTransactionManager(dataSourceTXManager);
        return dataSourceTXManager;
    }
 
}
```

### 数据源动态切换
```
@Slf4j
public class ReplicationRoutingDataSource extends AbstractRoutingDataSource {
 
    private static final String SLAVE_LOOKUP_KEY = "slave";
    private static final String MASTER_LOOKUP_KEY = "master";
 
    /**
     * Slave if the current transaction is in read-only mode, or master.
     */
    @Override
    protected Object determineCurrentLookupKey() {
        String lookupKey = TransactionSynchronizationManager.isCurrentTransactionReadOnly() ? SLAVE_LOOKUP_KEY : MASTER_LOOKUP_KEY;
        log.debug("connected DataSource :{}", lookupKey);
        return lookupKey;
    }
 
}
```
