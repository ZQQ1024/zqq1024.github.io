---
title: "Database Relationships"
categories:
  - backend
tags:
  - MySQL
classes: wide

excerpt: "介绍关系型数据库表与表之间的关系"
---

# 1 Introduction

主要以Customer下Order，选择Item为例，介绍关系型数据库中的几种类型：
- Many-to-One Relationships
- Many-to-Many Relationships
- One-to-One Relationships
- Self Referencing Relationships

# 2 One-To-One

Customer和Address关系，Cutsomer只能对应唯一一个Address，这个Address也对应唯一一个Customer，相当与Address是Customer的额外补充

ER图如下：

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190307125107.png)

SQL语句特点：
Address的主键customer_id，又是外键

# 3 Many-To-One

实际场景：
一个Customer可以make多orders

ER图如下：


![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190307112205.png)

SQL语句特点：
Order表中的某个非主键外键customer_id引用Customer表

题外话：外键不一定reference另一表的主键，可以是能够被indexed的column（如UNIQUE约束，当然type要兼容），也可以结合Many-to-One中的One去理解


# 4 Many-To-Many

实际场景：
一个Order中可以包含多个items
一个Item能够出现在多个orders中

我们需要引入额外的一张表，ER图如下：

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190307114240.png)

SQL语句特点：
引入第三个表，包含2个分别指向2个表的外键，同时这两个外键有作为主键，记录一些信息，如特点Order中特定Item有没有做好

Many-to-Many的关系在Items_Orders中体现，Item-Items_Orders的关系是One-to-Many，Items_Orders中有多个Item。Order-Items_Orders的关系是One-to-Many，Items_Orders中有多个Order大致反映出了，Item和Order的关系是Many-to-Many

# 5 Self-Referencing

Customer可以是由另一名Customer介绍来的

ER图如下：

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190307130715.png)

SQL语句特点：
外键引用自己

最终的ER图如下：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190307130805.png)

# 6 References

> [https://code.tutsplus.com/articles/sql-for-beginners-part-3-database-relationships--net-8561](https://code.tutsplus.com/articles/sql-for-beginners-part-3-database-relationships--net-8561)  
[https://zhidao.baidu.com/question/193591009.html](https://zhidao.baidu.com/question/193591009.html)  
[https://stackoverflow.com/questions/10292355/how-do-i-create-a-real-one-to-one-relationship-in-sql-server](https://stackoverflow.com/questions/10292355/how-do-i-create-a-real-one-to-one-relationship-in-sql-server)
