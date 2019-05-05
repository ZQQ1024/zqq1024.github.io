---
title: "安装WordPress"
classes: wide
toc: false
---

1. Create an empty project directory:
   ```
   This project directory contains a docker-compose.yml file which is complete in itself for a good starter wordpress project.

   Tip: You can use either a .yml or .yaml extension for this file. They both work.
   ```
2. Create a docker-compose.yml file:
   ```
   version: '3.3'

   services:
      db:
        image: mysql:5.7
        volumes:
          - db_data:/var/lib/mysql
        restart: always
        environment:
          MYSQL_ROOT_PASSWORD: somewordpress
          MYSQL_DATABASE: wordpress
          MYSQL_USER: wordpress
          MYSQL_PASSWORD: wordpress

        wordpress:
          depends_on:
            - db
          image: wordpress:latest
          ports:
            - "8000:80"
          restart: always
          environment:
            WORDPRESS_DB_HOST: db:3306
            WORDPRESS_DB_USER: wordpress
            WORDPRESS_DB_PASSWORD: wordpress
   volumes:
       db_data:
   ```
3. Build the project:
   ```
   $ docker-compose up -d
   ```

> 参考链接：  
> [https://docs.docker.com/compose/wordpress/](https://docs.docker.com/compose/wordpress/)