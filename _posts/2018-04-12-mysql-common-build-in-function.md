---
layout: post
author: "杨小定定"
title:  "MySQL 常用的內建函数"
description: "记录MySQL常用的一些內建函数，包括字符串的处理、数值的运算、日期的运算等等。"
date:   2018-04-12 00:00:00 +0800
categories: technology
---

> 记录MySQL常用的一些內建函数，包括字符串的处理、数值的运算、日期的运算等等。

在MySQL数据库中，函数可以用在SELECT语句及其子句（例如WHERE、ORDER BY、HAVING等）中，也可以用在UPDATE、DELETE语句及其子句中。

*Ps.本文仅做记录，不做示例，可根据功能描述自行尝试。*

*如遇某函数不会用，使用`help xxxx`来查看帮助。*

## 字符串函数

| 函数 | 功能 |
|:--- |:--- |
| `CONCAT(S1, S2, ...Sn)` | 连接S1、S2...Sn为一个字符串 |
| `INSERT(str, x, y,instr)` | 将字符串str从第x位置开始，y个字符长的子串替换为字符串instr |
| `LOWER(str)` | 将字符串str中所有字符变为小写 |
| `UPPER(str)` | 将字符串str中左右字符变为大写 |
| `LEFT(str, x)` | 返回字符串str最左边的x个字符 |
| `RIGHT(str, x)` | 返回字符串str最右边的x个字符 |
| `LPAD(str, n, pad)` | 用字符串pad对str最左边进行填充，直到长度为n个字符长度；<br/>*若n小于str长度，则结果为从左往右截取str字符串n位的子串* |
| `RPAD(str, n, pad)` | 用字符串pad对str最右边进行填充，直到长度为n个字符长度；<br/>*若n小于str长度，则结果为从左往右截取str字符串n位的子串* |
| `TRIM(str)` | 去掉字符串str行尾和行首的空格 |
| `LTRIM(str)` | 去掉字符串str左侧的空格 |
| `RTRIM(str)` | 去掉字符串str右侧的空格 |
| `REPEAT(str, x)` | 返回str重复x次的结果 |
| `REPLACE(str, a, b)` | 用字符串b替换字符串str中所有出现的字符串a |
| `STRCMP(s1, s2)` | 比较字符串s1和s2<br/>若两者相等，返回0<br/>若s1小于s2，返回-1，否则返回1 |
| `SUBSTRING(str, x, y)` | 返回从字符串str中x位置开始，y个字符长度的子串 |

## 数值函数

| 函数 | 功能 |
|:--- |:--- |
| `ABS(x)` | 返回x的绝对值 |
| `CEIL(x)` | 返回大于x的最小整数值 |
| `FLOOR(x)` | 返回小于x的最大整数值 |
| `MOD(x, y)` | 返回x/y的模 |
| `RAND()` | 返回0~1之间的随机数 |
| `ROUND(x, y)` | 返回参数x的四舍五入的有y位小数的值 |
| `TRUNCATE(x, y)` | 返回数字x截断为y位小数的结果<br/>*直接截断，不进行四舍五入* |

## 日期和时间函数

| 函数 | 功能 |
|:--- |:--- |
| `CURDATE()` | 返回当前日期 |
| `CURTIME()` | 返回当前时间 |
| `NOW()` | 返回当前的日期和时间 |
| `UNIX_TIMESTAMP(date)` | 返回日期date的UNIX时间戳<br/>*即距1970年01月01日8时0分0秒的秒数* |
| `FROM_UNIXTIME(t)` | 返回UNIX时间戳t的日期值<br/>即同上函数相反，为秒数转日期 |
| `WEEK(date)` | 返回日期date为一年中的第几周 |
| `YEAR(date)` | 返回日期date的年份 |
| `HOUR(time)` | 返回time的小时值 |
| `MINUTE(time)` | 返回time的分钟值 |
| `MONTHNAME(date)` | 返回date的月份名，*英文名* |
| `DATE_FORMAT(date, fmt)` | 返回按字符串fnt格式化日期date值<br/>fmt详见下附一： |
| `DATE_ADD(date, INTERVAL expr type)` | 返回一个日期或时间值加上一个时间间隔的时间值<br/>其中INTERVAL是关键字，expr是表达式，type是间隔类型<br/>详见下附二： |
| `DATEDIFF(expr1, expr2)` | 返回起始时间expr1和结束时间expr2之间的天数<br/>即用第一个时间减去第二个时间，的天数差 |

附一：
*`DATE_FORMAT(date, fmt)`中`fmt`的格式符说明*

| 格式符 | 格式说明 |
|:---:|:--- |
| `%S`和`%s` | 两位数字形式的秒（00,01...59） |
| `%i` | 两位数字形式的分（00,01...59） |
| `%H` | 两位数字形式的小时，24小时（00,01...23） |
| `%h`和`%I` | 两位数字形式的小时，12小时（01,02...12） |
| `%k` | 数字形式的小时，24小时（0,1...23） |
| `%l` | 数字形式的小时，12小时（1,2...12） |
| `%T` | 24小时的时间形式（hh:mm:ss） |
| `%r` | 12小时的时间形式（hh:mm:ss AM或hh:mm:ss PM） |
| `%p` | AM 或 PM |
| `%W` | 一周中哪一天的名称（Sunday,Monday...Saturday） |
| `%a` | 一周中哪一天的名称的缩写（Sun,Mon...Sat） |
| `%d` | 两位数字表示月中的天数（01,02...31） |
| `%e` | 数字形式表示月中的天数（1,2...31） |
| `%D` | 英文后缀表示月中的天数（1st,2nd,3rd...） |
| `%w` | 以数字形式表示周中的天数（0=Sunday,1=Monday...6=Saturday） |
| `%j` | 以3位数字表示年中的天数（001,002...366） |
| `%U` | 年中第几周（00,01...52），其中Sunday为周中的第一天 |
| `%u` | 年中第几周（00,01...52），其中Monday为周中的第一天 |
| `%M` | 年中月名（January,February...December） |
| `%b` | 缩写的月名（Jan,Feb...Dec） |
| `%m` | 两位数表示的月份（01,02...12） |
| `%c` | 数字表示的月份（1,2...12） |
| `%Y` | 4位数字表示的年份 |
| `%y` | 2位数字表示的年份 |
| `%%` | 直接值"%" |

附二：
*`DATE_ADD(date, INTERVAL expr type)` 中`type`间隔类型*

| 表达式类型（`type`） | 描述 | 格式（`expr`） |
|:--- |:--- |:--- |
| `HOUR` | 小时 | `hh` |
| `MINUTE` | 分 | `mm` |
| `SECOND` | 秒 | `ss` |
| `YEAR` | 年 | `YY` |
| `MONTH` | 月 | `MM` |
| `DAY` | 日 | `DD` |
| `YEAR_MONTH` | 年和月 | `YY_MM` |
| `DAY_HOUR` | 日和小时 | `DD hh` |
| `DAY_MINUTE` | 日和分钟 | `DD hh:mm` |
| `DAY_SECOND` | 日和秒 | `DD hh:mm:ss` |
| `HOUR_MINUTE` | 小时和分 | `hh:mm` |
| `HOUR_SECOND` | 小时和秒 | `hh:ss` |
| `MINUTE_SECOND` | 分钟和秒 | `mm:ss` |

## 流程函数

| 函数 | 功能 |
|:--- |:--- |
| `IF(value, t, f)` | 如果value为真，则返回t；否则返回f |
| `IFNULL(value1, value2)` | 如果value1不为NULL，则返回value1；否则返回value2 |
| `CASE WHEN [value1] THEN [result1] ... ELSE [default] END` | 如果value1为真，则返回result1，...，否则返回default |
| `CASE [expr] WHEN [value1] THEN [result1] ... ELSE [default] END` | 如果expr等于value1，则返回result1，...，否则返回default |

## 其它常用函数

| 函数 | 功能 |
|:--- |:--- |
| `DATABASE()` | 返回当前数据库名 |
| `VERSION()` | 返回当前数据库版本 |
| `USER()` | 返回当前登录用户名 |
| `INET_ATON(IP)` | 返回IP地址的数字表示 |
| `INET_NTOA(num)` | 返回数字代表的IP地址 |
| `PASSWORD(str)` | 返回字符串str的加密版本 |
| `MD5(str)` | 返回字符串str的MD5值 |

