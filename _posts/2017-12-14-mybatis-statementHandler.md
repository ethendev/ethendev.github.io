---
layout: post
title: Mybatis之MappedStatement源码分析
tags:  [Java, MyBatis, MappedStatement]
categories: [Java, MyBatis]
keywords: Java,MyBatis,MappedStatement源码
---


本篇文章将简单分析Mybatis的StatementHandler源码




首先来看一下StatementHandler接口定义
```
public interface StatementHandler {
  // 编译SQL
  Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException;
  // 设置参数
  void parameterize(Statement statement) throws SQLException;
  // 批量处理
  void batch(Statement statement) throws SQLException;

  int update(Statement statement) throws SQLException;

  <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException;
  
  <E> Cursor<E> queryCursor(Statement statement) throws SQLException;

  BoundSql getBoundSql();

  ParameterHandler getParameterHandler();

}
```

其中prepare、parameterize、update、query是StatementHandler中比较重要也比较常用的几个方法。StatementHandler的实现类有4个，如下图所示：
![4个实现类](https://i.loli.net/2018/06/10/5b1d2a2900d75.png)

### BaseStatementHandler
其中抽象类BaseStatementHandler实现了StatementHandler中的部分接口，同时也声明了几个自己的方法，包括setStatementTimeout、setFetchSize、closeStatement，以及生成主键的方法generateKeys。
```
public abstract class BaseStatementHandler implements StatementHandler {

  protected final Configuration configuration;
  protected final ObjectFactory objectFactory;
  protected final TypeHandlerRegistry typeHandlerRegistry;
  protected final ResultSetHandler resultSetHandler;
  protected final ParameterHandler parameterHandler;

  protected final Executor executor;
  protected final MappedStatement mappedStatement;
  protected final RowBounds rowBounds;

  protected BoundSql boundSql;

  // 此处省略构造函数、get方法

  @Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      statement = instantiateStatement(connection);
      setStatementTimeout(statement, transactionTimeout);
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }

  protected abstract Statement instantiateStatement(Connection connection) throws SQLException;

  // 设置超时时间
  protected void setStatementTimeout(Statement stmt, Integer transactionTimeout) throws SQLException {
    Integer queryTimeout = null;
    if (mappedStatement.getTimeout() != null) {
      queryTimeout = mappedStatement.getTimeout();
    } else if (configuration.getDefaultStatementTimeout() != null) {
      queryTimeout = configuration.getDefaultStatementTimeout();
    }
    if (queryTimeout != null) {
      stmt.setQueryTimeout(queryTimeout);
    }
    StatementUtil.applyTransactionTimeout(stmt, queryTimeout, transactionTimeout);
  }

  // 批量返回的结果行数
  protected void setFetchSize(Statement stmt) throws SQLException {
    Integer fetchSize = mappedStatement.getFetchSize();
    if (fetchSize != null) {
      stmt.setFetchSize(fetchSize);
      return;
    }
    Integer defaultFetchSize = configuration.getDefaultFetchSize();
    if (defaultFetchSize != null) {
      stmt.setFetchSize(defaultFetchSize);
    }
  }

  protected void closeStatement(Statement statement) {
    try {
      if (statement != null) {
        statement.close();
      }
    } catch (SQLException e) {
      //ignore
    }
  }
 
  // 生成主键
  protected void generateKeys(Object parameter) {
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    ErrorContext.instance().store();
    keyGenerator.processBefore(executor, mappedStatement, null, parameter);
    ErrorContext.instance().recall();
  }

}
```

### RoutingStatementHandler
RoutingStatementHandler不提供具体的实现，而是根据StatementType，创建不同的类型StatementHandler。

```
public class RoutingStatementHandler implements StatementHandler {

  private final StatementHandler delegate;

  public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }
  }
  
  @Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    return delegate.prepare(connection, transactionTimeout);
  }
  ......
}
```

### SimpleStatementHandler
SimpleStatementHandler用于没有预编译参数的SQL的运行。

```
public class SimpleStatementHandler extends BaseStatementHandler {
  ......
  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    statement.execute(sql);
    return resultSetHandler.<E>handleResultSets(statement);
  }
  ......
}

```

### PreparedStatementHandler
PreparedStatementHandler用于预编译参数SQL的运行。

```
public class PreparedStatementHandler extends BaseStatementHandler {
  ......
  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.<E> handleResultSets(ps);
  }
  ......
}
```

### CallableStatementHandler
CallableStatementHandler用于存储过程的调度。

```
public class CallableStatementHandler extends BaseStatementHandler {
  ......
  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    CallableStatement cs = (CallableStatement) statement;
    cs.execute();
    List<E> resultList = resultSetHandler.<E>handleResultSets(cs);
    resultSetHandler.handleOutputParameters(cs);
    return resultList;
  }
  ......
}
```


Executor会通过Configuration对象的newStatementHandler方法生成StatementHandler对象，准确的说是生成它的实现类RoutingStatementHandler对象。然后RoutingStatementHandler根据Executor的类型去创建对应的statementHandler对象。


Configuration的newStatementHandler方法源码：
```
public class Configuration {
......
  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }
}
......
```

![](https://i.loli.net/2018/06/10/5b1d32b95473f.png)