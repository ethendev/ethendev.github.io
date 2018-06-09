---
layout: post
title: Mybatis运行过程简介
tags:  [java, Mybatis]
categories: [java]
keywords: java,Mybatis
excerpt: MyBatis的运行过程主要分为两步，第一步读取配置文件缓存到Configuration对象，用来创建SqlSessionFactory，第二步是获取SqlSession以及使用SqlSession进行数据库操作。
---

* content
{:toc}

### 1. MyBatis的基本构成
在介绍之前，先简单回顾一下MyBatis的核心类。
* SqlSessionFactoryBuilder(构造器): 根据配置信息或者代码来生成SqlSessionFactory
* SqlSessionFactory(工厂接口): 主要功能是生成SqlSession。
* SqlSession: 可以发送SQL去执行并返回结果，也可以获取Mapper的接口。
* Mapper(映射器): 由java接口和XML文件(或注解)组成。


MyBatis的运行过程主要分为两步，第一步读取配置文件缓存到Configuration对象，用来创建SqlSessionFactory，第二步是获取SqlSession以及使用SqlSession进行数据库操作。

### 2. 构建SqlSessionFactory

&emsp;&emsp;SqlSessionFactory是MyBatis的核心接口的核心类之一，其最重要的功能就是提供创建MyBatis的核心接口SqlSession。每次程序访问需要数据库，都要先通过SqlSessionFactory创建SqlSession，它存在于MyBatis的整个生命周期中。MyBatis比较复杂，采用构造模式来创建SqlSessionFactory，可以通过SqlSessionFactoryBuilder来获得SqlSessionFactory实例。

&emsp;&emsp;SqlSessionFactory是一个接口，所以MyBatis为它提供了两个实现类，DefaultSqlSessionFactory和SqlSessionManager。SqlSessionFactory构建过程分为两步：

* 通过XMLConfigBuilder解析配置xml文件, 将读取到的参数存到Configuration对象中。
* 使用Configuration对象去创建SqlSessionFactory。


### 3. 构建SqlSession

&emsp;&emsp;构建SqlSessionFactory之后就可以获取到SqlSession接口，它提供了增删改查、获取Mapper等方法，是MyBatis重要和最常用的接口之一。每个线程都有一个SqlSession实例，注意SqlSession并不是线程安全的。

#### 3.1 Mapper映射器

开发中经常使用的Mapper只是个接口，没有实现类，它是如何运行的呢，答案是动态代理。
![Mapper接口](https://i.loli.net/2018/06/06/5b17fa309d7c2.png)


下面是MapperProxyFactory.java的部分代码实现：
```java
public class MapperProxyFactory<T> {
    ...
    protected T newInstance(MapperProxy<T> mapperProxy) {
        return Proxy.newProxyInstance(mapperInterface.getClassLoader(), 
            new Class[]{mapperInterface}, mapperProxy);
    }

    public T newInstance(SqlSession sqlSession) {
        MapperProxy<T> mapperProxy = new MapperProxy(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }
}
```
从代码中可以看出使用动态代理对接口进行了绑定，它的作用是生成动态代理对象，而代理的方法被放到了MapperProxy类中。


接下里再看看MapperProxy的代码：
```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
    ...
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            if (Object.class.equals(method.getDeclaringClass())) {
                return method.invoke(this, args);
            }

            if (this.isDefaultMethod(method)) {
                return this.invokeDefaultMethod(proxy, method, args);
            }
        } catch (Throwable e) {
            throw ExceptionUtil.unwrapThrowable(e);
        }

        MapperMethod mapperMethod = this.cachedMapperMethod(method);
        return mapperMethod.execute(this.sqlSession, args);
    }
    ...
```
一旦mapper是代理对象，就会运行到invoke方法中。invoke首先判断它是否是一个类，因为Mapper是接口，会判断失败，那么就会生成MapperMethod对象，通过cachedMapperMethod方法对其初始化，然后执行execute方法。


```java
public class MapperMethod {
    ...
    public Object execute(SqlSession sqlSession, Object[] args) {
        Object param;
        Object result;
        switch(this.command.getType()) {
        case INSERT:
            param = this.method.convertArgsToSqlCommandParam(args);
            result = this.rowCountResult(sqlSession.insert(this.command.getName(), param));
            break;
        case UPDATE:
            param = this.method.convertArgsToSqlCommandParam(args);
            result = this.rowCountResult(sqlSession.update(this.command.getName(), param));
            break;
        case DELETE:
            param = this.method.convertArgsToSqlCommandParam(args);
            result = this.rowCountResult(sqlSession.delete(this.command.getName(), param));
            break;
        case SELECT:
            if (this.method.returnsVoid() && this.method.hasResultHandler()) {
                this.executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (this.method.returnsMany()) {
                result = this.executeForMany(sqlSession, args);
            } else if (this.method.returnsMap()) {
                result = this.executeForMap(sqlSession, args);
            } else if (this.method.returnsCursor()) {
                result = this.executeForCursor(sqlSession, args);
            } else {
                param = this.method.convertArgsToSqlCommandParam(args);
                result = sqlSession.selectOne(this.command.getName(), param);
            }
            break;
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + this.command.getName());
        }

        if (result == null && this.method.getReturnType().isPrimitive() && !this.method.returnsVoid()) {
            throw new BindingException("Mapper method '" + this.command.getName() + " attempted to return null from a method with a primitive return type (" + this.method.getReturnType() + ").");
        } else {
            return result;
        }
    }

    // 查询返回多条记录
    private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
        Object param = this.method.convertArgsToSqlCommandParam(args);
        List result;
        if (this.method.hasRowBounds()) {
            RowBounds rowBounds = this.method.extractRowBounds(args);
            result = sqlSession.selectList(this.command.getName(), param, rowBounds);
        } else {
            result = sqlSession.selectList(this.command.getName(), param);
        }

        if (!this.method.getReturnType().isAssignableFrom(result.getClass())) {
            return this.method.getReturnType().isArray() ? this.convertToArray(result) : 
                this.convertToDeclaredCollection(sqlSession.getConfiguration(), result);
        } else {
            return result;
        }
    }
}
```
上面的代码是MapperMethod类的execute方法，代码很长，我们只需要重点看executeForMany方法就可以了。MapperMethod采用了命令模式，根据查询类型，跳转不同的方法。最后通过sqlSession对象去执行SQL操作。

看完上面的介绍，相信大家已经了解了为什么MyBatis只用Mapper接口就可以查询数据库了。


#### 3.2 SqlSession四大对象
&emsp;&emsp;SqlSession用到了几个很重要的类，Mapper执行的过程是通过其中的Executor、StatementHandler、ParameterHandler、ResultSetHandler来完成数据库的操作和结果返回的。

* Executor：用来调度StatementHandler、ParameterHandler、ResultHandler等来执行对应的SQL。
* StatementHandler：使用数据库的Statement执行操作，是四大对象的核心，起到承上启下的作用。
* ParameterHandler：用于对SQL参数的处理。
* ResultSetHandler：进行最后数据集ResultSet的封装返回处理的。


下图是这4个类运行的流程图：
![运行流程](/img/mybatis_flow.png)