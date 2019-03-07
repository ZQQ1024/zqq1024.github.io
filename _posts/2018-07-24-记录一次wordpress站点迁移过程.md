---
title: "记录一次wordpress站点迁移过程"
categories:
  - linux
tags:
  - wordpress
---



迁移和备份还原的区别是针对不同的`install`而言的，使用上的区别可能是访问的`IP`会变

几乎所有系统的备份还原都主要涉及下面两个方面，wordpress也不例外：

- 数据库：mysqldump，或者应用自身带的备份生成xml（与具体数据库无关）
- 文件系统（插件、附件等）：直接拷贝`/var/www/html`或者只拷贝关键目录

# 1 docker部署
上一此直接在宿主机上安装`apache`、`mysql`和`wordpress`部署的，这次使用`docker`部署（`docker`和`docker-compose`安装略）

`docker-compose.yml`文件如下：
```
version: '3.3'

services:
   db:
     image: mysql:5.7
     volumes:
       - ./db_data:/var/lib/mysql
     restart: always
     ports:
       - "6033:3306"
     environment:
       MYSQL_ROOT_PASSWORD: xxxxx
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress

   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     ports:
       - "80:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress
```
`wordpress`关键的是文章，这种数据是存在数据库的，所以数据库持久性的挂载出来，`wordpress`的文件系统不重要没挂载出来

# 2 原站点备份
这里直接使用的是`UpdraftPlus Backup/Restore`进行数据库、插件和主题等的备份

![image](http://pic.yupoo.com/840486874/HvNOJhzi/medish.jpg)

也可以：
1. `mysqldump`使用的wordpress数据库
2. 拷贝整个`/var/www/html`

# 3 新站点还原
同样，直接使用`UpdraftPlus Backup/Restore`上传`Upload backup files`刚刚备份的文件，进行还原，还原过程中会提示尼这是站点迁移，忽略继续，完成后站点就不能访问了（如果访问地址变了），则连接数据库，更新如下表：

```
UPDATE wp_posts SET guid = replace(guid, 'http://old ip/', 'http://new ip/');
UPDATE wp_options SET option_value = replace(option_value, 'http://old ip/', 'http://new ip/');
UPDATE wp_postmeta SET meta_value = replace(meta_value, 'http://old ip/', 'http://new ip/');
```
在数据库中把之前的访问地址，换成现在的地址

也可以：
1. 使用之前的`/var/www/html`
2. 还原数据库（dump生成的是sql文件）
3. 修改`wp-config.php`中的数据库配置信息
4. 若访问地址修改了，则同样按上述更新相关表

# 4 出现的问题

问题一：Workbench使用
```
update wp_posts set guid=replace(guid,"http://been.ltd","http://114.116.85.188");
```
更新时报错：
```
You are using safe update mode and you tried to update a table 
without a WHERE that uses a KEY column To disable safe mode, 
toggle the option in Preferences -> SQL Editor and reconnect.
```
解决：不让批量更新的意思，关闭safe mode
![image](http://pic.yupoo.com/840486874/HvNW140A/medish.jpg)


问题二：update过后，站点可以访问了，但是之前customer的menu里面的连接没变

解决：第三句update漏掉了，匹配的是`http://been.ltd/`而不是`http://been.ltd`

![image](http://pic.yupoo.com/840486874/HvNY8Tqh/medish.jpg)
# 5 参看链接

> 
docker-compose部署wordpress：  
[https://docs.docker.com/compose/wordpress/#define-the-project](https://docs.docker.com/compose/wordpress/#define-the-project)  
关闭safe mode：  
[https://stackoverflow.com/questions/11448068/mysql-error-code-1175-during-update-in-mysql-workbench](https://stackoverflow.com/questions/11448068/mysql-error-code-1175-during-update-in-mysql-workbench)  
customer的menu在哪：  
[https://stackoverflow.com/questions/3767725/where-are-custom-menus-saved-in-the-database-for-wordpress-3-0](https://stackoverflow.com/questions/3767725/where-are-custom-menus-saved-in-the-database-for-wordpress-3-0)  
参看的一篇博客：  
[https://blog.csdn.net/h8178/article/details/78451987](https://blog.csdn.net/h8178/article/details/78451987)
