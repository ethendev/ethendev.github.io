---
layout: post
title: MySQL EXPLAIN 命令详解
tags:  [MySQL,EXPLAIN]
categories: [MySQL]
keywords: MySQL,EXPLAIN
---

EXPLAIN 语句提供有关 MySQL 如何执行语句的信息。 解释与 SELECT， DELETE， INSERT， REPLACE，和 UPDATE 语句有关的工作。




EXPLAIN 返回 SELECT 语句中使用到的每个表的一行信息 。它按照 MySQL 在处理语句时读取它们的顺序列出表。MySQL 使用嵌套循环连接方法解析所有的 join 。这意味着 MySQL 从第一张表中读取一行数据，然后依次在第二张表，第三张表中找到匹配的行，依此类推。

先看看 EXPLAIN 会输出哪些东西，下面看一个简单的例子。
```
mysql> EXPLAIN SELECT * FROM student WHERE name = '李四';
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | student   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |    16.67 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

```

### EXPLAIN 输出列

|列名         |含义                  |
|:-           |:-                    |
|id           |该 SELECT 标识符      |
|select_type  |该 SELECT 类型        |
|table        |表名                  |
|partitions   |匹配的分区             |
|type         |JOIN 类型             |
|possible_keys|可供选择的索引         |
|key          |实际选择的索引         |
|key_len      |所选索引的长度         |
|ref          |与索引进行比较的列     |
|rows         |估计要检查的行数       |
|filtered     |按查询条件过滤的行的百分比|
|Extra        |附加信息              |


#### 1. id
标识符，查询的顺序编号。如果该行引用其他行的联合结果, 则该值可以为 NULL。

#### 2. select_type
查询的类型，取值为下表中任意一种。

|select_type 值      | 含义                 |
|:-                  |:-                    |
|SIMPLE              |简单 SELECT（不使用 UNION 或子查询）  |
|PRIMARY             |最外层 SELECT|	
|UNION               |UNION 中第二个或之后的SELECT语句|
|DEPENDENT UNION     |UNION 中的第二个或之后的SELECT语句 ，取决于外部查询|	
|UNION RESULT        |UNION 的结果|
|SUBQUERY            |子查询中的第一个 SELECT|
|DEPENDENT SUBQUERY  |子查询中的第一个SELECT，取决于外部的查询|
|DERIVED             |派生表（FROM 子句的子查询）|
|MATERIALIZED        |物化子查询|
|UNCACHEABLE SUBQUERY|结果集不能被缓存的子查询，必须重新评估外部查询的每一行|
|UNCACHEABLE UNION   |UNION中第二个或之后的SELECT，属于无法缓存的子查询|	

#### 3. table
输出行所引用的表的名称。有时候并不是真实的表名，可以是下列值之一:
* unionM,N: 该行引用 id 值为 M 和 N 的行的并集。
* derivedN: 该行引用 id 值为 N 的行的派生表结果。例如，派生表可能来自FROM子句中的子查询。
* subqueryN: 该行引用 id 值为 N 的行的具体化物化子查询的结果。

N 表示数字。

```
mysql> EXPLAIN SELECT * FROM (SELECT first_name, count(1) FROM employees group by first_name) t;
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra           |
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299297 |   100.00 | NULL            |
|  2 | DERIVED     | employees  | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299297 |   100.00 | Using temporary |
+----+-------------+------------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
2 rows in set, 1 warning (0.00 sec)
```

#### 4. partitions
其记录与查询匹配的分区，无分区表的值为 NULL。

#### 5. type
type 列描述了表是如何连接的。下表描述了从最佳到最差的连接类型：

<style>
table th:first-of-type {
    width: 20%;
}
</style>

|type值     | 含义                 |
|:-         |:-                    |
|system     |该表只有一行。这是const连接类型的特例。|
|const      |该表最多有一个匹配行, 在查询开始时读取。 因为只有一行，所以优化器的其余部分可以将此行中列的值视为常量。 const 表非常快，因为它们只读一次。|
|eq_ref     |最多从该表中匹配一行。 当使用索引的所有部分并且索引是PRIMARY KEY或UNIQUE NOT NULL索引时使用它。|
|ref        |从表中读取匹配索引值的所有行。如果连接仅使用最左前缀索引或者索引不是 PRIMARY KEY 或 UNIQUE 索引（换句话说，如果连接不能基于索引查询单行数据），则使用ref。 ref可用于使用=或<=>运算符进行比较的索引列。|
|fulltext   |使用FULLTEXT索引进行连接。|
|ref_or_null|这种连接类型与ref类似，但 MySQL 会对包含NULL值的行进行额外搜索|
|index_merge|表示使用了Index Merge优化。|
|unique_subquery|此类型替换某些 IN 子查询的eq_ref|
|index_subquery|类似于unique_subquery。它取代了IN子查询，适用于子查询中的非唯一索引：|
|range      |仅检索给定范围内的行，使用索引选择行。使用 =，<>，>，> =，<，<=，IS NULL 中的任何一个将主键与常量进行比较时，可以使用range|
|index      |Full Index Scan，index 与 ALL 区别为index类型只遍历索引树|
|ALL        |MySQL将扫描全表以找到匹配的行|


下面我们来看几个 type 取不同值的例子。

表结构如下：
```
CREATE TABLE `employees` (
  `emp_no` int(11) NOT NULL,
  `birth_date` date NOT NULL,
  `first_name` varchar(14) NOT NULL,
  `last_name` varchar(16) NOT NULL,
  `gender` enum('M','F') NOT NULL,
  `hire_date` date NOT NULL,
  `avatar` varchar(120) DEFAULT null,
  `city` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`emp_no`),
  KEY `idx_city` (`city`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

```
mysql> EXPLAIN SELECT * from employees where emp_no = 10001;
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

```
mysql> EXPLAIN SELECT * from employees a, titles b where a.emp_no = b.emp_no;
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref                | rows   | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------+
|  1 | SIMPLE      | b     | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL               | 442129 |   100.00 | NULL  |
|  1 | SIMPLE      | a     | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | employees.b.emp_no |      1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------------+--------+----------+-------+
2 rows in set, 1 warning (0.00 sec)
```

```
mysql> EXPLAIN SELECT * FROM employees WHERE city = 'new york';
+----+-------------+-----------+------------+------+---------------+----------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+------+---------------+----------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | ref  | idx_city      | idx_city | 63      | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+---------------+----------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

```
mysql> EXPLAIN SELECT * FROM employees WHERE city = 'new york' or city is null;
+----+-------------+-----------+------------+-------------+---------------+----------+---------+-------+--------+----------+-----------------------+
| id | select_type | table     | partitions | type        | possible_keys | key      | key_len | ref   | rows   | filtered | Extra                 |
+----+-------------+-----------+------------+-------------+---------------+----------+---------+-------+--------+----------+-----------------------+
|  1 | SIMPLE      | employees | NULL       | ref_or_null | idx_city      | idx_city | 63      | const | 149649 |   100.00 | Using index condition |
+----+-------------+-----------+------------+-------------+---------------+----------+---------+-------+--------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

```
mysql> EXPLAIN SELECT * FROM employees WHERE emp_no BETWEEN 10001 and 10010;
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |   10 |   100.00 | Using where |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

```
mysql> EXPLAIN SELECT emp_no FROM employees;
+----+-------------+-----------+------------+-------+---------------+----------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key      | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+----------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | index | NULL          | idx_city | 63      | NULL | 299297 |   100.00 | Using index |
+----+-------------+-----------+------------+-------+---------------+----------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

```
mysql> EXPLAIN SELECT * FROM employees WHERE first_name = 'Georgi';
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299297 |    10.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

#### 6. possible_keys
指出 MySQL 可以从中选择查找此表中的行的索引，需要注意的是查询中不一定会使用到。

#### 7. key
表示 MySQL 实际决定使用的键 (索引)。

要强制MySQL使用或忽略 possible_keys 列中列出的索引，可以在查询中使用 FORCE INDEX，USE INDEX 或 IGNORE INDEX。

#### 8. key_len
key_len 列表示 MySQL 实际使用的索引的长度。如果 key 列为 null, 则 len_len 列也为 null。

在不损失精确性的情况下，长度越短越好。

#### 9. ref
表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值

#### 10. rows
rows 列表示 MySQL 认为必须检查以执行查询的行数。注意，此值是估算值。

#### 11. filtered
filtered 列表示按查询条件过滤的行的百分比。最大值为 100, 表示未发生行过滤。

#### 12. Extra
Extra 列包含有关 MySQL 如何解析查询的其他信息，有如下取值：

|Extra 值       | 含义                 |
|:-             |:-                    |
|Using where    |表示条件查询          |
|Using index    |表示使用到索引        |
|Using temporary|表示MySQL需要使用临时表来存储结果集，常见于排序和分组查询|
|Using filesort |MySQL 中无法利用索引完成的排序操作称为"文件排序"|

Extra 取值范围很多,这里不一一列举，其他的取值请参考 [官方文档](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain-extra-information)

### 参考资料

> MySQL官网介绍：https://dev.mysql.com/doc/refman/5.7/en/explain-output.html