---
layout: post
title:  SQL语句执行顺序
categories: [SQL]
tags:  [SQL]
---


数据库是大多数程序员经常接触的东西。虽然项目中经常写sql，相信有部分人对SQL的执行顺序了解的不多。这里简单的介绍一下。




### 基本语法
首先，SELECT语句的基本语法如下:

```
(8)     SELECT 
(9)     DISTINCT <select_list>
(1)     FROM <left_table>
(3)     <join_type> JOIN <right_table>
(2)     ON <join_condition>
(4)     WHERE <where_condition>
(5)     GROUP BY <group_by_list>
(6)     WITH <CUBE | RollUP>
(7)     HAVING <having_condition>
(10)    ORDER BY <order_by_condition>
(11)    LIMIT <limit_number>
```

### 执行顺序
在大数编程语言中，代码按编码顺序被处理，但是在SQL语言中，第一个被处理的子句是FROM子句，尽管SELECT语句第一个出现，但是几乎总是最后被处理。每个步骤都会产生一个虚拟表，该虚拟表被用作下一个步骤的输入。这些虚拟表对调用者（客户端应用程序或者外部查询）不可用。只是最后一步生成的表才会返回给调用者。如果没有在查询中指定某一子句，将跳过相应的步骤。下面是对mysql的各个逻辑步骤的简单描述。

1.  执行FROM语句，SQL语句的执行过程中，都会产生一个虚拟表，用来保存SQL语句的执行结果（这是重点），执行from语句之后会产生一个虚拟表暂时叫VT1（vitual table 1），VT1是根据笛卡尔积生成

2.  执行on进行过滤
    根据on后面的条件过滤掉不符合条件的数据。只有那些使<join_condition>为真的行才被插入VT2。

3.  执行链接的类型
    inner join内连接、left join左链接、right右链接、outer join 外链接、fullouter join 全连接，执行完产生VT3。

4.  执行where后面的条件 
    这时候使用WHERE条件的时候要注意：不能使用组函数、并且字段的别名不能放到条件中使用。只有使<where_     condition>为true的行才被插入VT4。
    例如: SELECT city as c FROM t WHERE c='shanghai'

5.  执行group by 对VT4中的行分组，生成VT5。

6.  执行CUBE\|ROLLUP：把超组(Suppergroups)插入VT5,生成VT6.

7.  执行having过滤
    HAVING子句主要和GROUP BY子句配合使用，having后面可以跟组函数的条件。对VT6应用HAVING筛选器。只有     使<having_condition>为true的组才会被插入VT7.

8.  执行select，使用聚集函数进行计算。产生VT8

9.  执行distinct，去掉重复的数据

10.  执行order by 语句排序

11.  执行分页语句