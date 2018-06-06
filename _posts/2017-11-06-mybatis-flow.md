---
layout: post
title: mybatis运行过程
tags:  [java, mybatis]
categories: [java]
keywords: java,mybatis
---

* content
{:toc}

MyBatis的运行过程主要分为两步，第一部读取配置文件缓存到Configuration对象，用来创建SqlSessionFactory，第二部分是获取SqlSession以及使用SqlSession进行数据库操作。




## SqlSessionFactory

SqlSessionFactory是MyBatis的核心接口的核心类之一，其最重要的功能就是提供创建MyBatis的核心接口SqlSession。因为MyBatis比较复杂，所以它采用构造模式来创建SqlSessionFactory，构建的类名为SqlSessionFactoryBuilder。
SqlSessionFactory是一个接口，所以MyBatis提供了一个默认的SqlSessionFactory实现类，类名为DefaultSqlSessionFactory。构建过程分为两步：

* 通过XMLConfigBuilder解析配置xml文件, 将读取到的参数存到Configuration对象中。
* 使用Configuration对象去创建SqlSessionFactory。


## SqlSession

构建SqlSessionFactory之后就可以获取到SqlSession，SqlSession是一个接口，它提供了增删改查等方法，是MyBatis重要和最常用的接口之一。
SqlSession用到了几个很重要的类，Mapper执行的过程是通过其中的Executor、StatementHandler、ParameterHandler、ResultSetHandler来完成数据库的操作和结果返回的。

* Executor：执行器，用来调度StatementHandler、ParameterHandler、ResultHandler等来执行对应的SQL。
* StatementHandler：使用数据库的Statement执行操作，是四大对象的核心，起到承上启下的作用。
* ParameterHandler：用于对SQL参数的处理。
* ResultSetHandler：进行最后数据集ResultSet的封装返回处理的。


![运行流程](/img/mybatis_flow.png)
