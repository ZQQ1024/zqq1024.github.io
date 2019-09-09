---
title: "Maven Tomcat Webapp Project"
categories:
  - backend
tags:
  - mvn
  - tomcat
classes: wide

excerpt: "使用Maven构建部署webapp项目"
---

# 1 介绍
`Apache Maven`是主要用于Java项目的自动构建工具，`Apache Tomcat`是运行Servlet的容器，本文主要介绍使用`Maven`创建一个webapp项目，并运行于`Tomcat`容器中

# 2 依赖软件
依赖如下软件：
- jdk: [https://www.oracle.com/java/technologies/jdk8-downloads.html](https://www.oracle.com/java/technologies/jdk8-downloads.html)
- tomcat: [https://tomcat.apache.org/download-80.cgi](https://tomcat.apache.org/download-80.cgi)
- mvn: [https://maven.apache.org/download.cgi](https://maven.apache.org/download.cgi)

请自行下载上述依赖，设置PATH环境变量等

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190909132515.png)

# 3 创建项目

```
$ mvn archetype:generate -DgroupId=org.zqq.ppp -DartifactId=ppp-webapp -DarchetypeGroupId=org.apache.maven.archetypes -DarchetypeArtifactId=maven-archetype-webapp -DarchetypeVersion=1.4 -DinteractiveMode=false -X
```

生成的目录结构如下（development环境）：
```
$ pwd
/Users/zqq/Desktop/ppp
$ tree .
.
└── ppp-webapp
    ├── pom.xml
    └── src
        └── main
            └── webapp
                ├── WEB-INF
                │   └── web.xml
                └── index.jsp

5 directories, 3 files

```

# 4 部署
使用`tomcat7-maven-plugin`插件，将项目快速部署到`Tomcat`容器中

修改`pom.xml`，添加如下内容：
```
<plugin>
    <groupId>org.apache.tomcat.maven</groupId>
    <artifactId>tomcat7-maven-plugin</artifactId>
    <version>2.2</version>
</plugin>
```

上面没有显示提供Configuration，但隐含了以下配置：
- Tomcat manager的URL是`http://localhost:8080/manager`，认证的用户为`admin`，密码为空
- Context路径是`/${project.artifactId}`

所以修改tomcat的`conf/tomcat-users.xml`，添加用户：
```
<user username="admin" password="" roles="manager-script"/>
```
然后启动tomcat：
```
$ pwd
/Users/zqq/Downloads/apache-tomcat-8.5.45
$ bin/startup.sh 
Using CATALINA_BASE:   /Users/zqq/Downloads/apache-tomcat-8.5.45
Using CATALINA_HOME:   /Users/zqq/Downloads/apache-tomcat-8.5.45
Using CATALINA_TMPDIR: /Users/zqq/Downloads/apache-tomcat-8.5.45/temp
Using JRE_HOME:        /Library/Java/JavaVirtualMachines/openjdk-12.0.1.jdk/Contents/Home
Using CLASSPATH:       /Users/zqq/Downloads/apache-tomcat-8.5.45/bin/bootstrap.jar:/Users/zqq/Downloads/apache-tomcat-8.5.45/bin/tomcat-juli.jar
Tomcat started.
zqq-MacBook-Pro:apache-tomcat-8.5.45 zqq$
```
部署：
```
$ mvn tomcat7:deploy
xxxxxx
[INFO] tomcatManager status code:200, ReasonPhrase:
[INFO] OK - Deployed application at context path [/ppp-webapp]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  1.732 s
[INFO] Finished at: 2019-09-09T14:52:21+08:00
[INFO] ------------------------------------------------------------------------
```
tomcat/webapps的目录结构如下（deployment环境）：
```
$ tree ppp-webapp
ppp-webapp
├── META-INF
│   ├── MANIFEST.MF
│   ├── maven
│   │   └── org.zqq.ppp
│   │       └── ppp-webapp
│   │           ├── pom.properties
│   │           └── pom.xml
│   └── war-tracker
├── WEB-INF
│   ├── classes
│   └── web.xml
└── index.jsp

6 directories, 6 files
```

浏览器访问如下：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190909145944.png)

# 5 问题
- 从maven的仓库下载慢问题，自行找代理解决
- 关于单元测试和调试相关的内容，暂时略去

# 6 参考链接
> [https://blog.csdn.net/u010183402/article/details/51934269](https://blog.csdn.net/u010183402/article/details/51934269)  
[https://maven.apache.org/archetypes/maven-archetype-webapp/](https://maven.apache.org/archetypes/maven-archetype-webapp/)  
[https://www.mkyong.com/maven/how-to-create-a-web-application-project-with-maven/](https://www.mkyong.com/maven/how-to-create-a-web-application-project-with-maven/)  
[https://blog.csdn.net/bq1073100909/article/details/52490974](https://blog.csdn.net/bq1073100909/article/details/52490974)  
[https://tomcat.apache.org/maven-plugin-2.2/tomcat7-maven-plugin/plugin-info.html](https://tomcat.apache.org/maven-plugin-2.2/tomcat7-maven-plugin/plugin-info.html)