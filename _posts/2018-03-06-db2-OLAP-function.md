---
layout: post
author: "杨小定定"
title:  "DB2 数据库OLAP函数使用"
description: "本文介绍了DB2的OLAP函数的使用，示例。"
date:   2018-03-05 14:05:00 +0800
categories: technology
---

> OLAP是数据仓库系统的主要应用，支持复杂的分析操作，侧重决策支持，并且提供直观易懂的查询结果。  
> 本文我们主要就介绍了DB2数据库OLAP函数的使用，希望能够对您有所收获！

DB2数据库OLAP函数的使用是本文我们主要要介绍的内容。

我们知道，当今的数据处理大致可以分成两大类：

* 联机事务处理OLTP（On-Line Transaction Processing）
* 联机分析处理OLAP（On-Line Analytical Processing）

OLTP是传统的关系型数据库的主要应用，主要是基本的、日常的事务处理，例如银行交易。  
OLAP是数据仓库系统的主要应用，支持复杂的分析操作，侧重决策支持，并且提供直观易懂的查询结果。

下表列出了OLTP与OLAP之间的比较。

| - | OLTP | OLAP |
|:--- |:--- |:--- |
| 用户 | 操作人员,低层管理人员 | 决策人员,高级管理人员 |
| 功能 | 日常操作处理 | 分析决策
| DB设计 | 面向应用 | 面向主题 |
| 数据 | 当前的, 最新的细节的, 二维的分立的 | 历史的, 聚集的, 多维的集成的, 统一的 |
| 存取 | 读/写数十条记录 | 读上百万条记录 |
| 工作单位 | 简单的事务 | 复杂的查询 |
| 用户数 | 上千个 | 上百个 |
| DB大小 | 100MB-GB | 100GB-TB |

联机分析处理 (OLAP) 可以用很好很强大来形容。这项功能特别适用于各种统计查询，这些查询用通常的SQL很难实现，或者根本就无发实现。

首先，我们从一个简单的例子开始，来一步一步揭开它神秘的面纱，请看下面的SQL：

```sql
SELECT    
    ROW_NUMBER() OVER(ORDER BY SALARY) AS 序号,    
    NAME AS 姓名,    
    DEPT AS 部门,    
    SALARY AS 工资    
FROM    
    (--姓名    部门  工资    
     VALUES    
        ('张三','市场部',4000),    
        ('赵红','技术部',2000),    
        ('李四','市场部',5000),    
        ('李白','技术部',5000),    
        ('王五','市场部',NULL),    
        ('王蓝','技术部',4000)    
    ) AS EMPLOY(NAME,DEPT,SALARY)
;
```

查询结果如下：

```
序号       姓名       部门       工资    
1     赵红       技术部    2000    
2     张三       市场部    4000    
3     王蓝       技术部    4000    
4     李四       市场部    5000    
5     李白       技术部    5000    
6     王五       市场部    (null)
```

看到上面的`ROW_NUMBER() OVER()`了吗？很多人非常不理解，怎么两个函数能这么写呢？甚至有人怀疑上面的SQL语句是不是真的能执行。

其实，`ROW_NUMBER`是个函数没错，它的作用从它的名字也可以看出来，就是给查询结果集编号。  
但是，OVER并不是一个函数，而是一个表达式，它的作用是定义一个作用域（或者可以说是结果集），`OVER`前面的函数只对`OVER`定义的结果集起作用。

怎么样，不明白？没关系，我们后面还会详细介绍。

从上面的SQL我们可以看出，典型的 DB2 在线分析处理的格式包括两部分：函数部分和`OVER`表达式部分。

那么，函数部分可以有哪些函数呢？如下：

* `ROW_NUMBER`
* `RANK`
* `DENSE_RANK`
* `FIRST_VALUE`
* `LAST_VALUE`
* `LAG`
* `LEAD`
* `COUNT`
* `MIN`
* `MAX`
* `AVG`
* `SUM`

*上面这些函数的作用，我们在后面逐步介绍，大家可以根据函数名猜测一下函数的作用。*

假设我想在不改变上面语句的查询结果的情况下，追加对部门员工的平均工资和全体员工的平均工资的查询，怎么办呢？

用通常的SQL很难查询，但是用OLAP函数则非常简单，如下SQL所示：

```sql
SELECT    
    ROW_NUMBER() OVER() AS 序号,
    ROW_NUMBER() OVER(PARTITION BY DEPT ORDER BY SALARY) AS 部门序号,
    NAME AS 姓名,
    DEPT AS 部门,
    SALARY AS 工资,
    AVG(SALARY) OVER(PARTITION BY DEPT) AS 部门平均工资,
    AVG(SALARY) OVER() AS 全员平均工资
FROM    
    (--姓名    部门  工资    
     VALUES    
        ('张三','市场部',4000),    
        ('赵红','技术部',2000),    
        ('李四','市场部',5000),    
        ('李白','技术部',5000),    
        ('王五','市场部',NULL),    
        ('王蓝','技术部',4000)    
    ) AS EMPLOY(NAME,DEPT,SALARY)
;
```

查询结果如下：

```
序号       部门序号       姓名       部门       工资       部门平均工资       全员平均工资    
1            1          张三       市场部    4000       4500                     4000    
2            2          李四       市场部    5000       4500                     4000    
3            3          王五       市场部    (null)     4500                     4000    
4            1          赵红       技术部    2000       3666                     4000    
5            2          王蓝       技术部    4000       3666                     4000    
6            3          李白       技术部    5000       3666                     4000
```

请注意序号和部门序号之间的区别，我们在查询部门序号的时候，在`OVER`表达式中多了两个子句，分别是`PARTITION BY`和`ORDER BY`。它们有什么作用呢？

在介绍它们的作用之前，我们先来回顾一下`OVER`的作用，还记得吗？

`OVER`是一个表达式，它的作用是定义一个作用域（或者可以说是结果集），`OVER`前面的函数只对`OVER`定义的结果集起作用。

`ORDER BY`的作用大家应该非常熟悉，用来对结果集排序。  
`PARTITION BY`的作用其实也很简单，和`GROUP BY`的作用相同，用来对结果集分组。

到此为止，大家应该对OLAP函数的套路有一定的了解和体会了吧。

大家看一下上面SQL的结果集，发现王五的工资是null，当我们按工资排序时，null被放到最后，我们想把null放在前边该怎么办呢？

使用`NULLS FIRST`关键字即可，默认是`NULLS LAST`，请看下面的SQL：

```sql
SELECT    
    ROW_NUMBER() OVER(ORDER BY SALARY desc NULLS FIRST) AS RN,
    RANK() OVER(ORDER BY SALARY desc NULLS FIRST) AS RK,
    DENSE_RANK() OVER(ORDER BY SALARY desc NULLS FIRST) AS D_RK,
    NAME AS 姓名,
    DEPT AS 部门,
    SALARY AS 工资
FROM    
    (--姓名    部门  工资
     VALUES    
        ('张三','市场部',4000),
        ('赵红','技术部',2000),
        ('李四','市场部',5000),
        ('李白','技术部',5000),
        ('王五','市场部',NULL),
        ('王蓝','技术部',4000)
    ) AS EMPLOY(NAME,DEPT,SALARY)
;
```

查询结果如下：

```
RN  RK   D_RK     姓名       部门       工资    
1     1     1     王五       市场部    (null)    
2     2     2     李四       市场部    5000    
3     2     2     李白       技术部    5000    
4     4     3     张三       市场部    4000    
5     4     3     王蓝       技术部    4000    
6     6     4     赵红       技术部    2000
```

请注意`ROW_NUMBER`和`RANK`之间的区别。

`RANK`是等级，排名的意思，李四和李白的工资都是5000，他们并列排名第二。

张三和王蓝的工资都是4000，怎么`RANK`函数的排名是第四，而`DENSE_RANK`的排名是第三呢？

这正是这两个函数之间的区别。由于有两个第二名，所以`RANK`函数默认没有第三名。

现在又有个新问题，假设让你查询一下每个员工的工资以及工资小于他的所有员工的平均工资，该怎么办呢？

怎么？没听明白问题？不要紧，请看下面的SQL：

```sql
SELECT    
    NAME AS 姓名,    
    SALARY AS 工资,    
    SUM(SALARY) OVER(ORDER BY SALARY NULLS FIRST ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS 小于本人工资的总额,    
    SUM(SALARY) OVER(ORDER BY SALARY NULLS FIRST ROWS BETWEEN  CURRENT ROW AND UNBOUNDED FOLLOWING) AS 大于本人工资的总额,    
    SUM(SALARY) OVER(ORDER BY SALARY NULLS FIRST ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS 工资总额1,    
    SUM(SALARY) OVER() AS 工资总额2    
FROM    
    (--姓名    部门  工资
     VALUES    
        ('张三','市场部',4000),
        ('赵红','技术部',2000),
        ('李四','市场部',5000),
        ('李白','技术部',5000),
        ('王五','市场部',NULL),
        ('王蓝','技术部',4000)
    ) AS EMPLOY(NAME,DEPT,SALARY)
;
```

查询结果如下：

```
姓名       工资       小于本人工资的总额    大于本人工资的总额    工资总额1     工资总额2    
王五       (null)     (null)             20000              20000            20000    
赵红       2000       2000               20000              20000            20000    
张三       4000       6000               18000              20000            20000    
王蓝       4000       10000              14000              20000            20000    
李四       5000       15000              10000              20000            20000    
李白       5000       20000              5000               20000            20000
```

上面SQL中的`OVER`部分出现了一个`ROWS`子句，我们先来看一下`ROWS`子句的结构：

```sql
ROWS BETWEEN <上限条件> AND <下限条件>
```

其中`<上限条件>`可以是如下关键字：

* `UNBOUNDED PRECEDING`
* `<number> PRECEDING`
* `CURRENT ROW`

`<下限条件>`可以是如下关键字：

* `CURRENT ROW`
* `<number> FOLLOWING`
* `UNBOUNDED FOLLOWING`

*注意，以上关键字都是相对当前行的。*

`UNBOUNDED PRECEDING`表示当前行前面的所有行，也就是说没有上限；  
`<number> PRECEDING`表示从当前行开始到它前面的<number>行为止，例如，number=2，表示的是当前行前面的2行；  
`CURRENT ROW`表示当前行。

*至于其它两个关键字，我想，不用我说，你也应该知道了吧。如果你还不明白，请仔细分析上面SQL的查询结果。*

`OVER`表达式还可以有个子句，那就是`RANGE`，它的使用方式和`ROWS`十分相似，或者说一模一样，作用也差多不，不过有点区别，如下所示：

```sql
RANGE BETWEEN <上限条件> AND <下限条件>
```

其中的`<上限条件>`、`<下限条件>`和`ROWS`一模一样，如下的SQL演示它们之间的区别：

```sql
SELECT    
    NAME AS 姓名,    
    DEPT AS 部门,    
    SALARY AS 工资,    
    FIRST_VALUE(SALARY, 'IGNORE NULLS') OVER(PARTITION BY DEPT) AS 部门最低工资,    
    LAST_VALUE(SALARY, 'RESPECT NULLS') OVER(PARTITION BY DEPT) AS 部门最高工资,    
    SUM(SALARY) OVER(ORDER BY SALARY ROWS BETWEEN 1 PRECEDING  AND 1 FOLLOWING) AS ROWS,    
    SUM(SALARY) OVER(ORDER BY SALARY RANGE BETWEEN 500 PRECEDING AND 500 FOLLOWING) AS RANGE    
FROM    
    (--姓名    部门  工资
     VALUES    
        ('张三','市场部',4000),
        ('赵红','技术部',2000),
        ('李四','市场部',5000),
        ('李白','技术部',5000),
        ('王五','市场部',NULL),
        ('王蓝','技术部',4000)
    ) AS EMPLOY(NAME,DEPT,SALARY)
;
```

查询结果如下：

```
姓名       部门       工资       部门最低工资       部门最高工资       ROWS    RANGE    
张三       市场部    2000       2000              4000             4400       4400    
赵红       技术部    2400       2400              5000             7400       4400    
李四       市场部    3000       2000              4000             8600       6200    
李白       技术部    3200       2400              5000             10200     6200    
王五       市场部    4000       2000              4000             12200     4000    
王蓝       技术部    5000       2400              5000             9000       5000
```

上面SQL的`RANGE`子句的作用是定义一个工资范围，这个范围的上限是当前行的工资-500，下限是当前行工资+500。

例如：李四的工资是3000，所以上限是3000-500=2500，下限是3000+500=3500，那么有谁的工资在2500-3500这个范围呢？

只有李四和李白，所以`RANGE`列的值就是3000(李四)+3200(李白)=6200。

以上就是`ROWS`和`RANGE`的区别。

上面的SQL还用到了`FIRST_VALUE`和`LAST_VALUE`两个函数，它们的作用也非常简单，用来求`OVER`定义集合的最小值和最大值。

值得注意的是这两个函数有个参数，`'IGNORE NULLS'`或`'RESPECT NULLS'`，它们的作用正如它们的名字一样，用来忽略`NULL`值和考虑`NULL`值。

还有两个函数我们没有介绍，`LAG`和`LEAD`，这两个函数的功能非常强大，请看下面SQL：

```sql
SELECT    
    NAME AS 姓名,    
    SALARY AS 工资,    
    LAG(SALARY,0) OVER(ORDER BY SALARY) AS LAG0,    
    LAG(SALARY) OVER(ORDER BY SALARY) AS LAG1,    
    LAG(SALARY,2) OVER(ORDER BY SALARY) AS LAG2,    
    LAG(SALARY,3,0,'IGNORE NULLS') OVER(ORDER BY SALARY) AS LAG3,    
    LAG(SALARY,4,-1,'RESPECT NULLS') OVER(ORDER BY SALARY) AS LAG4,    
    LEAD(SALARY) OVER(ORDER BY SALARY) AS LEAD    
FROM    
    (--姓名    部门  工资
     VALUES    
        ('张三','市场部',4000),
        ('赵红','技术部',2000),
        ('李四','市场部',5000),
        ('李白','技术部',5000),
        ('王五','市场部',NULL),
        ('王蓝','技术部',4000)
    ) AS EMPLOY(NAME,DEPT,SALARY)
;
```

查询结果如下：

```
姓名       工资       LAG0      LAG1      LAG2      LAG3      LAG4      LEAD    
张三       2000       2000      (null)   (null)       0       -1        2400    
赵红       2400       2400       2000    (null)       0       -1        3000    
李四       3000       3000       2400     2000       0        -1        3200    
李白       3200       3200       3000     2400       2000     -1        4000    
王五       4000       4000       3200     3000       2400     2000      5000    
王蓝       5000       5000       4000     3200       3000     2400      (null)
```

我们先来看一下`LAG`和`LEAD`函数的声明，如下：

```sql
LAG(表达式或字段, 偏移量, 默认值, IGNORE NULLS或RESPECT NULLS)

LEAD(表达式或字段, 偏移量, 默认值, IGNORE NULLS或RESPECT NULLS)
```

`LAG`是向下偏移，`LEAD`是向上偏移，大家看一下上面SQL的查询结果就一目了然了。

到此为止，有关DB2 OLAP函数的所有知识都介绍给大家了。

下面我们再次回顾一下DB2在线分析处理的组成部分，如下：

```sql
函数 OVER(PARTITION BY 子句 ORDER BY 子句 ROWS或RANGE子句)
```

关于DB2数据库中OLAP函数的使用的相关知识就介绍到这里了，希望本次的介绍能够对您有所收获！
