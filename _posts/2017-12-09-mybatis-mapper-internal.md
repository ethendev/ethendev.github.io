---
layout: post
title: Mybatis之Mapper内部组成
tags:  [Java, Mybatis]
categories: [Java, Mybatis]
keywords: Java,Mybatis,Mapper
---


上一篇文章介绍了MyBatis的Mapper接口是怎么运行的，在这篇文章中将简单介绍一下Mapper映射器的内部组成。




一般而言，一个映射器主要由3部分组成的：MappedStatement、SqlSource和BoundSql。

下面我们来了解一下这3类之间的关系，下图只列举了主要的属性和方法:
![类关系](https://i.loli.net/2018/06/09/5b1bb83300a8c.png)

### MappedStatement  
它保存映射器的一个节点(select \| insert \| delete \| update), 包括许多我们配置的 SQL、SQL的id、parameterType、resultType等重要的内容，例如：
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.chy.pagetest.user.UserMapper">

    <select id="getList" resultType="com.chy.pagetest.user.UserVo">
        SELECT user_id userId, nick_name nickName, sex FROM user where nick_name = #{nickName}
    </select>
    <delete id="delete">
        delete FROM user where userId = #{userId}
    </delete>
</mapper>
```

MappedStatement源码中的一些属性对应xml中元素的配置，详细的配置信息可以参考[官网介绍](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html#), 代码如下:
```
public final class MappedStatement {

  private String resource;
  private Configuration configuration;
  private String id;
  private Integer fetchSize;
  private Integer timeout;
  // STATEMENT，PREPARED 或 CALLABLE 的一个，默认值：PREPARED。
  private StatementType statementType;
  private ResultSetType resultSetType;
  private SqlSource sqlSource;
  private Cache cache;
  private ParameterMap parameterMap;
  private List<ResultMap> resultMaps;
  private boolean flushCacheRequired;
  private boolean useCache;
  private boolean resultOrdered;
  private SqlCommandType sqlCommandType;
  private KeyGenerator keyGenerator;
  private String[] keyProperties;
  private String[] keyColumns;
  private boolean hasNestedResultMaps;
  private String databaseId;
  private Log statementLog;
  private LanguageDriver lang;
  private String[] resultSets;

  ...
}
```

### SqlSource  
它是提供 BoundSql 对象的地方，是 MappedStatement 的一个属性。
  
接口源码如下：
```
/**
 * Represents the content of a mapped statement read from an XML file or an annotation. 
 * It creates the SQL that will be passed to the database out of the input parameter received from the user.
 *
 * @author Clinton Begin
 */
public interface SqlSource {
    BoundSql getBoundSql(Object parameterObject);
}
```
getBoundSql方法根据用户传递的parameterObject参数，动态地生成SQL语句并将信息封装到BoundSql对象中返回。其中paramenterObject为运行sql里的实际参数。

SqlSource有4个实现类
![实现类](https://i.loli.net/2018/06/10/5b1d41668c966.png)

下面简单看一下其中一个实现类StaticSqlSource的源码：
```
public class StaticSqlSource implements SqlSource {

  private final String sql;
  private final List<ParameterMapping> parameterMappings;
  private final Configuration configuration;

  public StaticSqlSource(Configuration configuration, String sql) {
    this(configuration, sql, null);
  }

  public StaticSqlSource(Configuration configuration, String sql, List<ParameterMapping> parameterMappings) {
    this.sql = sql;
    this.parameterMappings = parameterMappings;
    this.configuration = configuration;
  }

  @Override
  public BoundSql getBoundSql(Object parameterObject) {
    return new BoundSql(configuration, sql, parameterMappings, parameterObject);
  }

}
```

### BoundSql  
它是建立 SQL 和参数的地方。它有3个常用的属性：sql、parameterObject 和 parameterMappings。

源码如下：
```
/**
 * An actual SQL String got from an {@link SqlSource} after having processed any dynamic content.
 * The SQL may have SQL placeholders "?" and an list (ordered) of an parameter mappings 
 * with the additional information for each parameter (at least the property name of the input object to read 
 * the value from). 
 * </br>
 * Can also have additional parameters that are created by the dynamic language (for loops, bind...).
 *
 * @author Clinton Begin
 */
public class BoundSql {

  private final String sql;
  private final List<ParameterMapping> parameterMappings;
  private final Object parameterObject;
  private final Map<String, Object> additionalParameters;
  private final MetaObject metaParameters;

  public BoundSql(Configuration configuration, String sql, List<ParameterMapping> parameterMappings, Object parameterObject) {
    this.sql = sql;
    this.parameterMappings = parameterMappings;
    this.parameterObject = parameterObject;
    this.additionalParameters = new HashMap<String, Object>();
    this.metaParameters = configuration.newMetaObject(additionalParameters);
  }
  //省略set和get方法
}
```

调用SqlSource的getBoundSql方法，生成BoundSql对象，里面就包含预编译后的SQL以及参数等数据。
![BoundSql对象](https://i.loli.net/2018/06/10/5b1d42c6a2ac0.png)

#### 参数传递
简单回顾一下Mapper接口的参数设置方式
```
@Mapper
public interface UserMapper {
    UserVo getUserById(String id);

    List<UserVo> getList(UserVo user);

    List<UserVo> getListByPage(@Param("page") PageParam page, @Param("user") UserVo user);
}
```

```
<select id="getListByPage" resultType="com.test.user.UserVo">
    SELECT user_id userId, nick_name nickName, sex FROM user where sex = #{user.sex}
</select>
```

当参数个数大于等于2时，我们通常使用`@Param`来给参数命名，然后在***Mapper.XML中通过参数名获取参数值。下面就来讲讲为什么要这么做。 

上面我们说到，BoundSql有一个parameterObject属性，这个属性就是用来传递Mapper接口参数的。  

下面我们看一下使用了`@Param`注解时的parameterObject的值  
![有@Param注解时的参数](https://i.loli.net/2018/06/09/5b1bd18e166d1.png)

不使用`@Param`注解时的parameterObject的值
![没有有@Param注解时的参数](https://i.loli.net/2018/06/09/5b1bcbe52816a.png)

从上面parameterObject的调试信息可以看出，当有多个参数时，parameterObject使用HashMap来存储参数。使用了`@Param`注解，key值就是设置的参数名，否则就是arg0这样的形式。


#### 属性介绍

BoundSql的3个主要属性及其作用

* parameterObject: 它为参数本身，有如下几个规则
  * 传递简单对象时，如int类型，MyBatis会把参数变为其对应的包装类Integer对象传递。
  * 传递POJO或者Map，那么parameterObject就是传入的类型。
  * 传递多个参数，MyBatis会把parameterObject变成一个Map<String, Object>对象，键值与`@Param`注解有关
     * 没用@Param注解，其键值是按顺序规划的，类似{"arg0":p1, "arg1":p2, "param1":p1, "param2":p2}，所以我们可以使用#{param1}或者是#{arg0}去引用第1个参数。
     * 使用@param注解，键值换成@param注解的键，如@Param("key1")String p1, @Param("key2")int p2, 键值为{"key1": p1, "key2": p2, "param1": p1, "param2": p2}。
    
* parameterMappings: 它是一个List，每一个元素都是ParameterMapping的对象，这个对象会描述我们的参数。参数包括属性、名称、表达式、javaType、jdbcType、typeHandler等重要信息。通过它可以实现参数和SQL的结合，以便PreparedStatement能够通过它找到parameterObject对象的属性并设置参数，使得程序准确运行。

* sql: 就是我们书写在映射器里面的一条SQL。