---
layout: post
author: "杨小定定"
title:  "MySQL 支持的数据类型"
description: "简单介绍MySQL中所支持的数据类型，包括数值类型、字符串类型、日期和时间类型。"
date:   2018-04-11
categories: technology
---


> 记录MySQL数据库支持的数据类型，以及同DB2部分简单的对比分析。

MySQL提供了多种数据类型，主要包括数值类型、字符串类型、日期和时间类型。

下面我们一个一个来看一下。

## 数值类型

| 整数类型 | 字节 | 最小值 | 最大值 |
|:--- |:---:|:--- |:--- |
| `TINYINT` | 1 | 有符号 -128<br/>无符号 0 | 有符号 127<br/>无符号 255 |
| `SMALLINT` | 2 | 有符号 -32768<br/>无符号 0 | 有符号 32767<br/>无符号 65535 |
| `MEDIUMINT` | 3 | 有符号 -8388608<br/>无符号 0 | 有符号 8388607<br/>无符号 1677215 |
| `INT、INTEGER` | 4 | 有符号 -2147483648<br/>无符号 0 | 有符号 2147483647<br/>无符号 4294967295 |
| `BIGINT` | 8 | 有符号 -9223372036854775808<br/>无符号 0 | 有符号 9223372036854775807<br/>无符号 18446744073709551615 |
| **浮点数类型** | **字节** | **最小值** | **最大值** |
| `FLOAT` | 4 | ±1.17549435E-38 | ±3.402823466E+38  |
| `DOUBLE` | 8 | ±2.2250738585072014E-308  | ±1.7976931348623157E+308  |
| **定点数类型** | **字节** | **最小值** | **最大值** |
| `DEC(M,D)`<br/>`DECIMAL(M,D)` | M+2 | 最大取值范围与`DOUBLE`相同 | 给定`DECIMAL`的有效取值范围有`M`和`D`决定 |
| **位类型** | **字节** | **最小值** | **最大值** |
| `BIT(M)` | 1~8 | BIT(1) | BIT(64) |

*如果超出对应类型范围的操作，会发生“Out of range”错误提示。*

#### 整数类型

对于整数类型，MySQL还支持在类型名称后面的小括号内指定显示宽度，比如`int(5)`表示当数值宽度小于5位的时候在数字前面填满宽度；如果不指定，直接写`int`的话，就默认为`int(11)`。

*DB2不支持这样指定。*

这个一般配合`zerofill`来使用，即用`0`填充。可以如下这么指定：

```bash
mysql> create table test(a int(5) zerofill);
或
mysql> alter table test modify a int(5) zerofill;
```

然后我们执行`desc`查看表结构的时候，发现：

```bash
mysql> desc test;
+-------+--------------------------+------+-----+---------+-------+
| Field | Type                     | Null | Key | Default | Extra |
+-------+--------------------------+------+-----+---------+-------+
| a     | int(5) unsigned zerofill | YES  |     | NULL    |       |
+-------+--------------------------+------+-----+---------+-------+
1 row in set (0.00 sec)

mysql>
```

多了一个`unsigned`，这是因为如果一个列指定为`zerofill`，MySQL会自动为该列添加`unsigned`属性。

再一点就是，我们上面是指定了`5`的显示宽度（`int(5)`），这个是不影响实际存储的，对插入数据没有任何影响（也就是说可以插入大于5位的数据），仅仅只是不会填充`0`了而已。

另外一个整数类型的属性是：`AUTO_INCREMENT`。自动增长，在需要产生唯一标识符或顺序值时，可利用此属性，这个属性**只**用于整数类型。

指定任何使用`AUTO_INCREMENT`的列，应该定义为`NOT NULL`，以及定义为`PRIMARY KEY`或`UNIQUE`键；如不指定主键或唯一键的话，会报如下错误提示：

```bash
mysql> create table test(id int auto_increment);
ERROR 1075 (42000): Incorrect table definition; there can be only one auto column and it must be defined as a key
```

`AUTO_INCREMENT`一般从1开始，每行增加1，在插入`NULL`到一个`AUTO_INCREMENT`列时，MySQL插入一个比该列中当前最大值大1的值。同时，插入时，也可以指定该列的值，只要比现有的最大值大就行了，即如下插入都可以：

```bash
mysql> create table test(id int not null auto_increment, name char(2), primary key(id));
Query OK, 0 rows affected (0.08 sec)

mysql> desc test;
+-------+---------+------+-----+---------+----------------+
| Field | Type    | Null | Key | Default | Extra          |
+-------+---------+------+-----+---------+----------------+
| id    | int(11) | NO   | PRI | NULL    | auto_increment |
| name  | char(2) | YES  |     | NULL    |                |
+-------+---------+------+-----+---------+----------------+
2 rows in set (0.00 sec)

mysql> insert into test values (null, '01');
Query OK, 1 row affected (0.03 sec)

mysql> insert into test(name) values('02');
Query OK, 1 row affected (0.02 sec)

mysql> insert into test values(5, '03');
Query OK, 1 row affected (0.00 sec)

mysql> select * from test;
+----+------+
| id | name |
+----+------+
|  1 | 01   |
|  2 | 02   |
|  5 | 03   |
+----+------+
3 rows in set (0.00 sec)

mysql>
```


#### 小数类型

MySQL主要分为两种方式来表示小数：**浮点数**和**定点数**。

浮点数包括`float`（单精度）和`double`（双精度），而定点数只有`decimal`这一种。

*Ps.定点数在MySQL内部是以字符串形式存放的，比浮点数更精确，适合用来表示货币等精度高的数据，这一点和DB2是很像的。*

对于浮点数和定点数，MySQL都支持类型名后面加`(M, D)`的方式来进行表示，其中`M`表示精度，`D`表示标度（通俗一点讲就是`M`表示一共显示多少位数字（包括整数和小数），`D`表示小数位数）。

*在DB2中`float`和`double`是不支持加的。*

如果精度和标度不写的话，`float`和`double`是按照实际精度值显示（由实际的硬件和操作系统决定），`decimal`则会默认`decimal(10,0)`，如下所示：

```bash
mysql> create table test(f float, d double, dc decimal);
Query OK, 0 rows affected (0.03 sec)

mysql> desc test;
+-------+---------------+------+-----+---------+-------+
| Field | Type          | Null | Key | Default | Extra |
+-------+---------------+------+-----+---------+-------+
| f     | float         | YES  |     | NULL    |       |
| d     | double        | YES  |     | NULL    |       |
| dc    | decimal(10,0) | YES  |     | NULL    |       |
+-------+---------------+------+-----+---------+-------+
3 rows in set (0.00 sec)

mysql>
```

对于指定了精度和标度的`float`和`double`类型，在进行`insert`插入时，数据长度超过精度，会报错，但是小数位超过标度的话，不会报错，但是会按照标度进行四舍五入。

对于`decimal`类型，同样，超过精度会报错，小数位超过标度同样会按照四舍五入来插入，但同时会报出警告信息。

比如我们如下示例：

```bash
mysql> create table test(f float(5,2), d double(5,2), dc decimal(5,2));
Query OK, 0 rows affected (0.07 sec)

mysql> desc test;
+-------+--------------+------+-----+---------+-------+
| Field | Type         | Null | Key | Default | Extra |
+-------+--------------+------+-----+---------+-------+
| f     | float(5,2)   | YES  |     | NULL    |       |
| d     | double(5,2)  | YES  |     | NULL    |       |
| dc    | decimal(5,2) | YES  |     | NULL    |       |
+-------+--------------+------+-----+---------+-------+
3 rows in set (0.00 sec)

mysql> insert into test values(1.23, 1.23, 1.23);
Query OK, 1 row affected (0.00 sec)

mysql> insert into test values(1.234, 1.234, 1.23);
Query OK, 1 row affected (0.04 sec)

mysql> insert into test values(1.23, 1.23, 1.234);
Query OK, 1 row affected, 1 warning (0.00 sec)

mysql> show warnings;
+-------+------+-----------------------------------------+
| Level | Code | Message                                 |
+-------+------+-----------------------------------------+
| Note  | 1265 | Data truncated for column 'dc' at row 1 |
+-------+------+-----------------------------------------+
1 row in set (0.00 sec)

mysql> insert into test values(1.236, 1.236, 1.236);
Query OK, 1 row affected, 1 warning (0.00 sec)

mysql> select * from test;
+------+------+------+
| f    | d    | dc   |
+------+------+------+
| 1.23 | 1.23 | 1.23 |
| 1.23 | 1.23 | 1.23 |
| 1.23 | 1.23 | 1.23 |
| 1.24 | 1.24 | 1.24 |
+------+------+------+
4 rows in set (0.00 sec)

mysql>
```

*在DB2中，对于decimal类型，是直接截取指定位数的，并不是四舍五入，这点要注意。*

#### 位类型

用来存放位字段值的，可以用`BIT`（位）类型。

`BIT(M)`可以用来存放多位二进制数，其中`M`的范围可以是1~64，如果不写的话，默认1位。

我们需要查看位类型的字段值的话，直接`SELECT`将不会看到结果，如下所示：

```bash
mysql> create table test(a1 bit, a2 bit(5));
Query OK, 0 rows affected (0.05 sec)

mysql> insert into test values(1, 1);
Query OK, 1 row affected (0.00 sec)

mysql> select * from test;
+------+------+
| a1   | a2   |
+------+------+
|      |      |
+------+------+
1 row in set (0.00 sec)

mysql>
```
 
但是我们可以使用`bin()`（显示为二进制格式）或者`hex()`（显示为十六进制格式）函数来进行读取，如：

```bash
mysql> select bin(a1), hex(a2) from test;
+---------+---------+
| bin(a1) | hex(a2) |
+---------+---------+
| 1       | 1       |
+---------+---------+
1 row in set (0.03 sec)

mysql>
```
 
往位类型字段插入数据时，会首先转换为二进制，如果位数允许，将成功插入，否则插入报错。

如我们刚刚创建的`test`表，`a1`字段未指定位数，默认为1位；而`a2`指定了5位。分析可得，`a1`字段只有可能值为0或1（十进制），如果为2（十进制），那么对应二进制值为10，是两位了，超出了`a1`的位数，应当插入失败；而`a2`的值范围可以为0~31（十进制），所以插入2（十进制）应该成功：

```bash
mysql> insert into test(a1) values(2);
ERROR 1406 (22001): Data too long for column 'a1' at row 1
mysql> insert into test(a2) values(2);
Query OK, 1 row affected (0.00 sec)

mysql>
```

## 日期时间类型

| 日期和时间类型 | 字节 | 最小值 | 最大值 |
|:--- |:---:|:--- |:--- |
| `DATE` | 4 | 1000-01-01 | 9999-12-31 |
| `DATETIME` | 8 | 1000-01-01 00:00:00 | 9999-12-31 23:59:59 |
| `TIMESTAMP` | 4 | 19700101080001 | 2038年的某个时刻 |
| `TIME` | 3 | -838:59:59 | 838:59:59 |
| `YEAR` | 1 | 1901 | 2155 |

如插入的时间范围超出对应的类型的范围，则会有错误提示，并将以零值来进行存储（零值就是各个都为0）。

其它的都很简单，我们主要来看看`TIMESTAMP`类型的一些特性。

我们先创建一个测试表，字段为timestamp类型，可以看到：

```bash
mysql> create table test(ts timestamp);
Query OK, 0 rows affected (0.31 sec)

mysql> desc test;
+-------+-----------+------+-----+-------------------+-----------------------------+
| Field | Type      | Null | Key | Default           | Extra                       |
+-------+-----------+------+-----+-------------------+-----------------------------+
| ts    | timestamp | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+-------+-----------+------+-----+-------------------+-----------------------------+
1 row in set (0.00 sec)

mysql>
```

我们并没有指定`not null`以及`default`的值，但是系统默认给添加了这两个属性，且默认值是`CURRENT_TIMESTAMP`，这表示默认的是当前系统日期，比如我们插入一个`NULL`值试试：

```bash
mysql> insert into test values(null);
Query OK, 1 row affected (0.01 sec)

mysql> select * from test;
+---------------------+
| ts                  |
+---------------------+
| 2018-04-11 11:08:29 |
+---------------------+
1 row in set (0.00 sec)

mysql>
```
 
发现它给了默认系统的日期时间。

`TIMESTAMP`还有一个重要特点，就是和*时区*相关。当插入日期时，会先转换为本地时区后存放；而从数据库中取出时，也同样需要将日期转换为本地时区后显示。

所以，两个不同时区的用户看到的同一个日期可能是不一样的，如下所示：

> Ps. `show variables like 'par'`可以查看数据库配置变量
> `set`关键字可以设置配置变量
> 如遇到某个命令不会使用，可以敲`help xxx`来查看帮助

```bash
mysql> show variables like 'time_zone';
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| time_zone     | SYSTEM |
+---------------+--------+
1 row in set (0.00 sec)

mysql> create table test(ts timestamp);
Query OK, 0 rows affected (0.02 sec)

mysql> insert into test values(now());
Query OK, 1 row affected (0.03 sec)

mysql> select * from test;
+---------------------+
| ts                  |
+---------------------+
| 2018-04-11 11:34:28 |
+---------------------+
1 row in set (0.00 sec)

mysql> set time_zone='+9:00';
Query OK, 0 rows affected (0.03 sec)

mysql> select * from test;
+---------------------+
| ts                  |
+---------------------+
| 2018-04-11 12:34:28 |
+---------------------+
1 row in set (0.00 sec)

mysql>
```
 
最后，对于日期和时间类型的插入，是允许“不严格”语法，即任何标点符都可以用作日期部分或时间部分之间的间隔符；甚至，如果日和月的值小于10，还有时、分、秒的值小于10，都可以不需要指定两位数。但是必须要保证每个部分是合理的（比如不能出现分钟值61等）。

## 字符串类型

| 字符串类型 | 字节 | 描述及存储需求 |
|:--- |:---:|:--- |
| `CHAR(M)` | `M` | `M`为0~255之间的整数 |
| `VARCHAR(M)` |  | `M`为0~65535之间的整数，值的长度+1个字节 |
| `TINYBLOB` |  | 允许长度0~255字节，值的长度+1个字节 |
| `BLOB` |  | 允许长度0~65535字节，值的长度+2个字节 |
| `MEDIUMBLOB` |  | 允许长度0~167772150字节，值的长度+3个字节 |
| `LONGBLOB` |  | 允许长度0~4294967295字节，值的长度+4个字节 |
| `TINYTEXT` |  | 允许长度0~255字节，值的长度+2个字节 |
| `TEXT` |  | 允许长度0~65535字节，值的长度+2个字节 |
| `MEDIUMTEXT` |  | 允许长度0~167772150字节，值的长度+3个字节 |
| `LONGTEXT` |  | 允许长度0~4294967295字节，值的长度+4个字节 |
| `VARBINARY(M)` |  | 允许长度0~M字节的变长字节字符串，值的长度+1个字节 |
| `BINARY(M)` | `M` | 允许长度0~M字节的定长字节字符串 |

- `CHAR`和`VARCHAR`
  两者很类似，均用来保存MySQL中较短的字符串。主要区别在于存储方式的不同：`CHAR`列的长度固定为创建表时申明的长度，而`VARCHAR`列中的值为可变长字符串。
  在检索的时候，`CHAR`列删除了尾部的空格，而`VARCHAR`则保留这些空格。（头部的空格均不删除）
- `BINARY`和`VARBINARY`
  这两个类似于`CHAR`和`VARCHAR`，只是他们包含的是二进制字符串而不包含非二进制字符串。
  当保存`BINARY`值时，是通过在值的最后填充`0x00`（零字节）以达到指定的字段定义长度。

#### `ENUM`类型

枚举类型，也是字符串类型的一种，其值只能指定字符串。

它的值范围需要在创建表时通过枚举方式显式指定，最多允许有65535个成员。

如下示例：

```bash
mysql> create table test(gender enum('M', 'F'));
Query OK, 0 rows affected (0.12 sec)

mysql> desc test;
+--------+---------------+------+-----+---------+-------+
| Field  | Type          | Null | Key | Default | Extra |
+--------+---------------+------+-----+---------+-------+
| gender | enum('M','F') | YES  |     | NULL    |       |
+--------+---------------+------+-----+---------+-------+
1 row in set (0.03 sec)

mysql> insert into test values('M'),('1'),('f'),(null);
Query OK, 4 rows affected (0.07 sec)
Records: 4  Duplicates: 0  Warnings: 0

mysql> select * from test;
+--------+
| gender |
+--------+
| M      |
| M      |
| F      |
| NULL   |
+--------+
4 rows in set (0.00 sec)

mysql>
```

从上述示例我们可以看出：

- `ENUM`类型是忽略大小写的
- 对于插入不在指定范围内的值，并没有返回警告或报错，而是插入了第一个枚举值
- `ENUM`类型只允许从值集合中选取单个值，而不能一次取多个值

#### `SET`类型

集合类型，和`ENUM`非常类似，也是一个字符串对象，里面可以包含0~64个成员。

其与`ENUM`类型最主要的区别在于`SET`类型一次可以选取多个成员，下面示例：

```bash
mysql> create table test(col set('a','b','c','d'));
Query OK, 0 rows affected (0.06 sec)

mysql> desc test;
+-------+----------------------+------+-----+---------+-------+
| Field | Type                 | Null | Key | Default | Extra |
+-------+----------------------+------+-----+---------+-------+
| col   | set('a','b','c','d') | YES  |     | NULL    |       |
+-------+----------------------+------+-----+---------+-------+
1 row in set (0.00 sec)

mysql> insert into test values('a,b'),('a,d,a'),('a,b'),('a,c'),('a');
Query OK, 5 rows affected (0.01 sec)
Records: 5  Duplicates: 0  Warnings: 0

mysql> select * from test;
+------+
| col  |
+------+
| a,b  |
| a,d  |
| a,b  |
| a,c  |
| a    |
+------+
5 rows in set (0.01 sec)

mysql> insert into test values('a,d,f');
ERROR 1265 (01000): Data truncated for column 'col' at row 1
mysql>
```

从上述示例可以看出：

- `SET`类型可以从允许的值集合中选择任意1个或多个元素进行组合，都可以正确插入
- 对于超出允许值范围的（如上例`('a,d,f')`）将不允许插入
- 对于包含重复成员的（如上例`('a,d,a')`）集合将只取一次，写入结果为`a,d`

Done.

