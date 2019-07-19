---
title: "docker-compose 启动JIRA和MySQL"
categories:
  - others
tags:
  - Jira
  - Docker
classes: wide

excerpt: "docker-compose 启动JIRA和MySQL"
---


# 1 安装docker-compose


Compose is a tool for defining and running multi-container Docker applications.

```
sudo curl -L https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

$ docker-compose --version
docker-compose version 1.22.0, build 1719ceb
```
详细参看：[https://docs.docker.com/compose/](https://docs.docker.com/compose/)

# 2 YAML文件
通过写`docker-compose.yaml`文件：
```
version: '2'
services:
  mysql:
    image: mysql:5.7
    volumes:
      - ./conf/:/etc/mysql/conf.d/
      - ./db_data:/var/lib/mysql
    restart: always
    ports:
      - '3309:3306'
    environment:
      MYSQL_ROOT_PASSWORD: <password>
      MYSQL_DATABASE: jira

  jira:
    image: dockerzqq/crack:jira-base
    volumes:
      - ./jira_home:/var/atlassian/application-data/jira
    restart: always
    ports:
      - '8060:8080'
    depends_on:
      - mysql
```
其中挂载出来的目录有：
- ./db_data：数据库目录用于还原数据库
- ./conf：数据库的相关配置
- ./jira_home：jira的home目录

# 3 运行
通过`sudo docker-compose up -d`启动jira和mysql容器
```
zqq@zqq-t480s:~/Desktop/jira + confluence + bitbucket/jira$ sudo docker-compose up -d
Creating network "jira_default" with the default driver
Creating jira_mysql_1 ... done
Creating jira_jira_1  ... done
zqq@zqq-t480s:~/Desktop/jira + confluence + bitbucket/jira$ sudo docker ps
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                              NAMES
56617a5b518c        dockerzqq/crack:jira-base   "/sbin/tini -- /entr..."   6 minutes ago       Up 6 minutes        8443/tcp, 0.0.0.0:8060->8080/tcp   jira_jira_1
f9c3ae25f908        mysql:5.7                   "docker-entrypoint.s..."   6 minutes ago       Up 6 minutes        0.0.0.0:3309->3306/tcp             jira_mysql_1
```
数据库链接如下设置：

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/97754432-2018-08-04%2011-17-28%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

同样填写一个评估码就行了，~~可以更新下安装镜像，方便以后迁移~~

git地址：[https://git.chislab.com/projects/BUILD/repos/jira_build/browse](https://git.chislab.com/projects/BUILD/repos/jira_build/browse)