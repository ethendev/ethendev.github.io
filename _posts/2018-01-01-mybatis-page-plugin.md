---
layout: post
title: Mybatis自动分页插件
tags:  [Java, Mybatis, 分页插件]
categories: [Java, Mybatis]
keywords: Java,Mybatis,分页插件
---

* content
{:toc}

自己实现了一个比较简单的Mybatis分页插件。在讲解如何实现分页插件之前，我们先简单介绍一下Mybatis中的一些重要的对象。我们通过映射器Mapper对数据库进行增删改操作时，Mapper执行的过程是通过Executor、StatementHandler、ParameterHandler和ResultHandler来完成对数据库的操作和返回结果的。




> * Executor代表执行器，由它来调度StatementHandler、ParameterHandler、ResultHandler等来执行对应的SQL。
> * StatementHandler的作用是使用数据库的Statement(PreparedStatement)执行操作，是上面提到的四个对象的核心。
> * ParameterHandler用于SQL对参数的处理。
> * ResultHandler是进行最后数据集(ResultSet)的封装返回处理的。


### 前提条件
要编写Mybatis插件，我们就必须要实现Interceptor接口，下面先来看看这个接口里面的方法：

```
public interface Interceptor {
    Object intercept(Invocation var1) throws Throwable;

    Object plugin(Object var1);

    void setProperties(Properties var1);
}
```
> * intercept方法是插件的核心方法，它有个Invocation类型的参数，通过这个参数可以反射调度原来对象的方法。
> * plugin方法的作用是给被拦截的对象生成一个代理对象并返回。
> * setProperties方法允许在plugin元素中配置所需参数。



### 分页拦截器

#### 拦截器签名
```
@Intercepts({@Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
)})
```

#### 实现自己的拦截器
这里我要拦截的是Executor的query方法，先判断有没有PageParam类型的分页参数，如果有的话先查询符合条件的数据条数count，再获取具体的数据list，将count和list封装在Page类型的对象里面返回。
```
@Intercepts({@Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
)})
public class EasyPage implements Interceptor {

    Logger logger = LoggerFactory.getLogger(EasyPage.class);

    private static int MAPPED_STATEMENT_INDEX = 0;
    private static int PARAMETER_INDEX = 1;

    public Object intercept(Invocation invocation) throws Throwable {
        Object[] queryArgs = invocation.getArgs();
        MappedStatement ms = (MappedStatement) queryArgs[MAPPED_STATEMENT_INDEX];
        Object parameter = queryArgs[PARAMETER_INDEX];

        PageParam page = new PageParam();
        String pageKey = "";// 分页参数前缀
        if (parameter instanceof PageParam) {// 只有分页参数一个参数
            page = (PageParam) parameter;
        } else if (parameter instanceof PageParam || parameter instanceof HashMap) {// 2个及以上参数
            HashMap<String, Object> parameterMap = (HashMap<String, Object>) parameter;
            for (String key : parameterMap.keySet()) {
                if (parameterMap.get(key) instanceof PageParam) {
                    page = (PageParam) parameterMap.get(key);
                    pageKey = key + ".";
                    break;
                }
            }
        }

        // 判断是否需要分页，当参数不是默认值的时候就进行分页
        if (page != null && page.getIndex() != 0 && page.getRows() != Integer.MAX_VALUE) {
            int index = page.getIndex();
            int rows = page.getRows();

            BoundSql boundSql = ms.getBoundSql(parameter);
            int total = this.getCount(ms, parameter, boundSql);
            List list = Collections.EMPTY_LIST;
            if (total > 0) {
                Dialect dialect = new Dialect();
                BoundSql newBoundSql = dialect.getBoungSQL(ms, boundSql, (index - 1) * rows, pageKey);

                MappedStatement newMs = copyFromMappedStatement(ms, new MySqlSource(newBoundSql));
                queryArgs[MAPPED_STATEMENT_INDEX] = newMs;
                list = (List) invocation.proceed();
            }
            return new Page(list, index, rows, total);
        }
        return invocation.proceed();
    }
```

获取数据总数方法
```
/**
     * 获取数据总条数
     * @param mappedStatement
     * @param parameter
     * @param boundSql
     * @return
     * @throws SQLException
     */
    public int getCount(MappedStatement mappedStatement, Object parameter, BoundSql boundSql) throws SQLException {
        StringBuilder sqlBuilder = new StringBuilder();
        sqlBuilder.append("select count(1) from (");
        sqlBuilder.append(clearOrderBy(boundSql.getSql())).append(") tmp");

        Connection connection;
        PreparedStatement countStmt = null;
        ResultSet rs = null;
        int count = 0;
        try {
            connection = mappedStatement.getConfiguration().getEnvironment().getDataSource().getConnection();
            countStmt = connection.prepareStatement(sqlBuilder.toString());
            DefaultParameterHandler handler = new DefaultParameterHandler(mappedStatement, parameter, boundSql);
            handler.setParameters(countStmt);
            rs = countStmt.executeQuery();
            if (rs.next()) {
                count = rs.getInt(1);
            }
            logger.debug("==> Preparing: {}", sqlBuilder.toString());
            logger.debug("<== Total: {}", count);
        } finally {
            try {
                if (rs != null) {
                    rs.close();
                }
            } finally {
                if (countStmt != null) {
                    countStmt.close();
                }
            }
        }
        return count;
    }
```

根据数据库类型设置BoundSql
```
/**
     * 根据数据库类型设置参数，不需要在配置中设置数据库类型，通过DatabaseMetaData对象可以取到数据库名称
     * @param ms
     * @param boundSql
     * @param offset
     * @param pageKey
     * @return
     * @throws SQLException
     */
    public BoundSql getBoungSQL(MappedStatement ms, BoundSql boundSql, int offset, String pageKey) throws SQLException {
        DatabaseMetaData dbmd = ms.getConfiguration().getEnvironment().getDataSource().getConnection().getMetaData();
        String dbType = dbmd.getDatabaseProductName();

        String sql = boundSql.getSql();
        if (dbType != null) {
            switch (dbType) {
                case "MySQL":
                    sql = MysqlDialect.getLimitString(boundSql.getSql(), offset);
                    break;
                case "Oracle":
                    sql = OracleDialect.getLimitString(boundSql.getSql(), offset);
                    break;
                default:
                    throw new IllegalArgumentException("Not supported dialect:" + dbType);
            }
        }

        // copy a new list, if use "list=boundSql.getParameterMappings()" will throws UnsupportedOperationException
        List<ParameterMapping> list = new ArrayList<ParameterMapping>(boundSql.getParameterMappings());
        if (offset > 0) {
            list.add(new ParameterMapping.Builder(ms.getConfiguration(), pageKey + "offset", Integer.class).build());
            list.add(new ParameterMapping.Builder(ms.getConfiguration(), pageKey + "rows", Integer.class).build());
        } else {
            list.add(new ParameterMapping.Builder(ms.getConfiguration(), pageKey + "rows", Integer.class).build());
        }

        BoundSql newboundSql = new BoundSql(ms.getConfiguration(), sql, list, boundSql.getParameterObject());
        return newboundSql;
    }
```

访问 http://127.0.0.1:8080/user/getAll?index=1&rows=10 ，就可以获取到分页后的结果了。

![分页结果](https://raw.githubusercontent.com/ethendev/easypage/master/page_result.png)

上面只是几个比较重要的方法，完整的代码在 [github](https://github.com/ethendev/easypage.git) 。