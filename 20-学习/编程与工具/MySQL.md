---
title: MySQL 基础
tags:
  - mysql
  - 数据库
  - sql
status: draft
updated: 2026-10-06
---

# MySQL 基础

## MySQL创建数据库

使用`CREATE`命令来创建数据库
```MYSQL
CREATE DATABASE 数据库名;
```

数据库名字不能重复,为了防止报错可以使用
```MySQL
CREATE DATABASE IF NOT EXISTS 数据库名;
```
上面这句表示如果输入的名字不存在,创建这个数据库

在实际操作中还要添加字符集和排序规则
```MySQL
CREATE DATABASE IF NOT EXISTS 数据库名;
DEFAULT CHARACTER SET utf8mb4
DEFAULT COLLATE utf8mb4_unicode_ci;
```
* `utf8mb4` 完整utf-8
* `utf8mb4_unicode_ci` 通用排序规则

***
## MySQL删除数据库

使用`DROP`来删除数据库
```MySQL
DROP DATABASE IF EXISTS 数据库名;
```

***
## MySQL选择数据库

在MySQL中,使用`USE`来选择数据库,语法为
```MySQL
USE 数据库名;
```

*查看所有数据库*
```MySQL
SHOW DATABASES;
```

***
## MySQL创建数据表

创建数据表需要以下信息
* 表名
* 列名
* 字符数据类型

以下为创建MySQL数据表的语法
使用 `CREATE TABLE` 创建
```MySQL
CREATE TABLE 数据表名(
列名1 数据类型1 NOT NULL, # 不为空值
列名2 数据类型2 DEFAULT 1, # 如果没指定数据,默认为1
列名3 数据类型3 NOT NULL PRIMARY KEY # 定义主键
);
```
还能在创建表时定义主键和是否能是空值`NULL`

***
## MySQL删除数据表

删除数据表和删除数据库的方法一样
只不过把DATABASE 改为 TABLE
[[MySQL#MySQL删除数据库|删除数据库的方法]]

***
## MySQL插入数据
在MySQL中,可以使用`INSERT INTO`来向表中插入数据

以下为向数据表中插入数据的语法
```MySQL
INSERT INTO 数据表名(
列名1,
列名2,
列名3
)
VALUES (
列值1,
列值2,
列值3
);
```
列名和列值必须全部对应
如果要插入所有列的数据,可以省略列名
```MySQL
INSERT INTO 数据表名 VALUES (
列值1,
列值2,
列值3
)
```
也可以同时插入多行
```MySQL
INSERT INTO 数据表名(
列名1,
列名2,
列名3
)
VALUES (列值1, 列值2, 列值3),
       (列值4, 列值5, 列值6);
```

***
## MySQL 查询数据

MySQL数据库使用`SELECT`来查询数据
语法为
```MySQL
		SELECT 列名1, 列名2 FROM 表名
		WHERE 条件 # 筛选满足条件的行
		ORDER BY 列值 # 按这个列值排序
```
该语句输出列值1和列值2 的表
可以把表中的所有列值输出,相当于查了整个表
```MySQL
SELECT * FROM 表名
```

***
## MySQL WHERE 子句

如需有条件地从表中选取数据，可将 WHERE 子句添加到 SELECT 语句中。
WHERE 子句用于在 MySQL 中过滤查询结果，只返回满足特定条件的行。
```MySQL
SELECT 列名1, 列名2 FROM 表名
WHERE 条件
```
`WHERE`操作可以使用许多操作符来操作
```MySQL
SELECT * FROM users WHERE username = 'test';
SELECT * FROM users WHERE username != 'test';
SELECT * FROM users WHERE price > 50;
SELECT * FROM users WHERE price >= 50;
SELECT * FROM users WHERE price < 50;
SELECT * FROM users WHERE price <= 50;
SELECT * FROM users WHERE category = 'Electronics' AND price > 100;
SELECT * FROM users WHERE category = 'Electronics' OR price > 100;
SELECT * FROM users WHERE first_name LIKE 'J%';
SELECT * FROM users WHERE country_code IN ('US','CN','JP');
SELECT * FROM users WHERE NOT category = 'Clothing';
SELECT * FROM users WHERE order_date BETWEEN '2004' AND '2005';
SELECT * FROM users WHERE username IS NULL;
SELECT * FROM users WHERE username IS NOT NULL;
```

***
## MySQL排序数据

使用SELECT语句的`ORDER BY` 子句来对结果进行排序
```MySQL
SELECT * FROM users
ORDER BY user_id;
```
上面的语句按照user_id的大小进行排序
>**ORDER BY 语句必须放在语句最后**

同时也可以逆序输出,只要在后面加上`DESC`
```MySQL
SELECT * FROM users
ORDER BY user_id DESC;
```
其实有两个命令`DESC`和`ASC`但默认的是`ASC`

***
## 使用通配符进行过滤

 如果要过滤模糊值的话可以使用`LIKE`命令进行过滤
 比如你只知道一部分,或者你要过滤含有一部分内容的值
 **要注意通配符只适用于字符串类型的值**
 1. 使用`%`进行任何长度字符的过滤
 ```MySQL
 SELECT * FROM users
 WHERE name LIKE '%A%'
 ```
 这串代码表示筛选出所有名字里含有大写字母A的人名的行
 *是否区分大小写取决于列的字符集和排序规则；以 `_ci` 结尾的排序规则通常不区分大小写。*
 前面的%表示在A之前有字符,后面的%表示后面也有字符
 
 2. 使用`_`进行精准过滤
一个_只能匹配一个字符
 ```MySQL
 SELECT * FROM users
 WHERE name LIKE '__'
 ```
 上面的代码表示筛选出名字只有两个字符的行
***
## MySQL中的函数

1. 文本处理函数
	1. `LEFT()` 返回字符串左边的字符
	2. `CHAR_LENGTH()` 返回字符串的字符数
	3. `LOWER()` 将字符串转换为小写
	4. `LTRIM()` 去掉字符串左边的空格
	5. `RIGHT()` 返回字符串右边的字符
	6. `RTRIM()` 去掉字符串右边的空格
	7. `SUBSTR()` 提取字符串的组成部分
	8. `SOUNDEX()` 返回和字符串听起来像的值
	9. `UPPER()` 将字符串全转换为大写
2. 日期和时间处理函数
	1. `EXTRACT()` 提取日期的成分
	2. `NOW()` 返回当前的年月日时分秒
	3. `CURDATE()` 只获取日期
	4. `CURTIME()` 只获取时间
	5. `SYSDATE()` 动态实时时间
	6. `YEAR()` 提取年份
	7. `MONTH()` 提取月份
	8. `DAY()` 提取日期
	9. `HOUR()` 提取小时
	10. `MINUTE()` 提取分钟
	11. `SECOND()` 提取秒
	12. `WEEKDAY()` 提取星期
	13. `DAYOFYEAR()` 一年第几天
**日期加减**
`DATE_ADD`向后加时间
```MySQL
SELECT DATE_ADD(NOW(), INTERVAL 1 DAY);# 加一天
```
`DATE_SUB` 向前减时间
```MySQL
SELECT DATE_SUB(NOW(), INTERVAL 7 DAY);# 加一天
```
**计算日期间隔**
`DATEDIFF(日期1,日期2)`
计算天数差值,结果 = 日期1-日期2
```MySQL
SELECT DATEDIFF(日期1,日期2)
```
`TIMESTAMPDIFF(单位,开始日期,结束日期)`
可以指定计算的单位,比如年份或月份
```MySQL
SELECT TIMESTAMPDIFF(YEAR, '2012-01-01', '2026-01-01'); -- 返回 14
```
**日期格式化**
`DATE_FORMAT(时间,格式符)`
常用格式符
* `%Y` 4 位年，`%y` 2 位年
* `%m` 月份, `%d`日期
* `%H` 24小时, `%i`分钟,`%s`秒
```MySQL
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');
```

***
## 汇总函数
### 聚集函数
对某些行运行的函数,计算并返回一个值
1. `AVG()` 返回某列的平均值
2. `COUNT()` 返回某列的计数
3. `MAX()` 返回某列的最大值
4. `MIN()` 返回某列的最小值
5. `SUM()` 返回某列值的和
这些函数都使用在SELECT的后方
```MySQL
SELECT COUNT(某列) AS total FROM 表名;
```
使用只需用其他函数代替`COUNT()`即可
### 聚集不同值
如果不想要数据有重复的值,可以使用`DISTINCT`参数
*当不使用聚集函数时*
```MySQL
SELECT DISTINCT 某列 FROM 表名;
```
*当使用聚集函数时*
```MySQL
SELECT COUNT(DISTINCT 某列) FROM 表名;
```

***
## 分组数据

当SELECT 后面有两个列值,其中一个是上一节的聚集函数时
后面必须加上`GROUP BY` 来进行分组
```MySQL
SELECT vend_id, COUNT(*) AS num_pords FROM Products
GROUP BY vend_id
```
*标准 SQL 通常要求 `GROUP BY` 使用表达式；MySQL 也支持在部分场景引用 SELECT 别名，但为兼容性建议优先写表达式。*
*GROUP BY 子句必须出现在WHERE子句之后,ORDER BY 子句之前

**过滤分组**
除了使用`GROUP BY `分组外,还可以对分组进行过滤
但不使用`WHERE`,而是使用`HAVING`
```MySQL
SELECT cust_id,COUNT(*) AS orders
FROM Orders
GROUP BY cust_id 
HAVING COUNT(*) >= 2;
```
*`WHERE` 在分组前过滤普通行，`HAVING` 在分组后过滤分组结果；二者可以同时使用。*

***
## 使用子查询

子查询就是嵌套
```MySQL
SELECT cust_id FROM Orders
WHERE order_num IN (
SELECT order_num FROM OrderItems
WHERE prod_id = 'RGAN01'
)
```
也可以在SELECT后面的列值就嵌套,然后输出别名
*作为子查询的SELECT语句只能查询单个列*

***
## 联结表


### INNER JOIN 示例

```sql
SELECT customers.cust_id, orders.order_num
FROM customers
INNER JOIN orders
    ON customers.cust_id = orders.cust_id;
```

`INNER JOIN` 只返回两张表中连接条件匹配的行。

### LEFT JOIN 示例

```sql
SELECT customers.cust_id, orders.order_num
FROM customers
LEFT JOIN orders
    ON customers.cust_id = orders.cust_id;
```

`LEFT JOIN` 会保留左表的所有行；右表没有匹配时，右表列返回 `NULL`。
