---
title: "安装MySQL"
classes: wide
toc: false
---

1. Install the three following packages:
```
$ sudo apt-get install mysql-server
$ sudo apt install mysql-client
$ sudo apt install libmysqlclient-dev
```

2. Set for allowing remote access:
```
$ sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf

# comment bind-address = 127.0.0.1

$ mysql -u root -p
>  grant all on *.* to root@'%' identified by 'your password' with grant option;
```

3. Restart mysql service:
```
$ service mysql restart
```

References:
> [https://blog.csdn.net/xiangwanpeng/article/details/54562362](https://blog.csdn.net/xiangwanpeng/article/details/54562362)