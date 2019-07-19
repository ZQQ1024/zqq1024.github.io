---
title: "构建JIRA docker镜像"
categories:
  - others
tags:
  - Jira
  - Docker
classes: wide

excerpt: "构建Jira docker镜像"
---

破解的JIRA docker镜像主要分为以下几层：
1. 基础系统镜像
2. 在1基础上的JDK镜像
3. 在2基础上的JIRA镜像


# 1 基础系统镜像

这里使用的是`alpine:3.8`，docker hub上面的`Dockerfile`如下：
```
FROM scratch
ADD rootfs.tar.xz /
CMD ["bash"]
```
`scratch`是docker是预留的、最小的镜像，也可以使用自己的linux发行版构建`base image`

> 参考  
https://hub.docker.com/_/alpine/  
https://github.com/gliderlabs/docker-alpine/blob/2bfe6510ee31d86cfeb2f37587f4cf866f28ffbc/versions/library-3.8/x86_64/Dockerfile


官方镜像，使用`docker pull alpine:3.8`拉取该镜像

# 2 JDK镜像
这里使用的是`openjdk:8-jdk-alpine`，docker hub上面的`Dokcerfile`如下：
```
#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "update.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

FROM openjdk:8-jdk-alpine

# A few reasons for installing distribution-provided OpenJDK:
#
#  1. Oracle.  Licensing prevents us from redistributing the official JDK.
#
#  2. Compiling OpenJDK also requires the JDK to be installed, and it gets
#     really hairy.
#
#     For some sample build times, see Debian's buildd logs:
#       https://buildd.debian.org/status/logs.php?pkg=openjdk-8

# Default to UTF-8 file.encoding
ENV LANG C.UTF-8

# add a simple script that can auto-detect the appropriate JAVA_HOME value
# based on whether the JDK or only the JRE is installed
RUN { \
        echo '#!/bin/sh'; \
        echo 'set -e'; \
        echo; \
        echo 'dirname "$(dirname "$(readlink -f "$(which javac || which java)")")"'; \
    } > /usr/local/bin/docker-java-home \
    && chmod +x /usr/local/bin/docker-java-home
ENV JAVA_HOME /usr/lib/jvm/java-1.8-openjdk
ENV PATH $PATH:/usr/lib/jvm/java-1.8-openjdk/jre/bin:/usr/lib/jvm/java-1.8-openjdk/bin

ENV JAVA_VERSION 8u171
ENV JAVA_ALPINE_VERSION 8.171.11-r0

RUN set -x \
    && apk add --no-cache \
        openjdk8="$JAVA_ALPINE_VERSION" \
    && [ "$JAVA_HOME" = "$(docker-java-home)" ]

# If you're reading this and have any feedback on how this image could be
# improved, please open an issue or a pull request so we can discuss it!
#
#   https://github.com/docker-library/openjdk/issues
```

官方镜像，使用`docker pull openjdk:8-jdk-alpine`拉取该镜像




> 参考  
https://hub.docker.com/r/library/openjdk/  
https://github.com/docker-library/openjdk/blob/1778c73b834d04d5b5c61baee4cce8c127031f9c/8/jdk/alpine/Dockerfile

# 3 JIRA镜像

这里在上述的基础上构建我们自己的破解的镜像和更改时区等（截止写的时候，docker hub上并没有官方`atlassian/jira`镜像）

Dockerfile:
```
FROM openjdk:8-alpine
MAINTAINER ZQQ 840486874@qq.com

ENV RUN_USER            daemon
ENV RUN_GROUP           daemon

ENV JIRA_HOME          /var/atlassian/application-data/jira
ENV JIRA_INSTALL_DIR   /opt/atlassian/jira

VOLUME ["${JIRA_HOME}"]

EXPOSE 8080
EXPOSE 8443

WORKDIR $JIRA_HOME
CMD ["/entrypoint.sh", "-fg"]
ENTRYPOINT ["/sbin/tini", "--"]

# 这里设置了代理，方便下载
ARG HTTP_PROXY=http://172.17.0.1:8118
ARG HTTPS_PROXY=http://172.17.0.1:8118

RUN apk update \
    && update-ca-certificates \
    && apk add ca-certificates wget curl openssh bash procps openssl perl ttf-dejavu tini libc6-compat \
    && rm -rf /var/lib/{apt,dpkg,cache,log}/ /tmp/* /var/tmp/*

ENV TZ=Asia/Shanghai
RUN apk add tzdata

COPY entrypoint.sh              /entrypoint.sh

ARG JIRA_VERSION=7.11.1
ARG DOWNLOAD_URL=https://www.atlassian.com/software/jira/downloads/binary/atlassian-jira-software-${JIRA_VERSION}.tar.gz
ARG EXTRAS_CRACK_FILE=atlassian-extras-3.2.jar
ARG PLUGIN_CRACK_FILE=atlassian-universal-plugin-manager-plugin-2.22.9.jar


COPY . /tmp

RUN mkdir -p                             ${JIRA_INSTALL_DIR} \
    && curl -L                           ${DOWNLOAD_URL} | tar -xz --strip-components=1 -C "$JIRA_INSTALL_DIR" \
    && curl -L                           "https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.45.tar.gz" \
    | tar -xz --directory                "${JIRA_INSTALL_DIR}/lib" --strip-components=1 --no-same-owner "mysql-connector-java-5.1.45/mysql-connector-java-5.1.45-bin.jar" \
    && sed --in-place                    "s/java version/openjdk version/g" "${JIRA_INSTALL_DIR}/bin/check-java.sh" \
    && mv /tmp/${EXTRAS_CRACK_FILE}      ${JIRA_INSTALL_DIR}/atlassian-jira/WEB-INF/lib/ \
    && mv /tmp/${PLUGIN_CRACK_FILE}      ${JIRA_INSTALL_DIR}/atlassian-jira/WEB-INF/atlassian-bundled-plugins/ \
    && chown -R ${RUN_USER}:${RUN_GROUP} ${JIRA_INSTALL_DIR}/
```
entrypoint.sh:
```
#!/bin/bash

# Start Jira as the correct user
if [ "${UID}" -eq 0 ]; then
    echo "User is currently root. Will change directory ownership to ${RUN_USER}:${RUN_GROUP}, then downgrade permission to ${RUN_USER}"
    PERMISSIONS_SIGNATURE=$(stat -c "%u:%U:%a" "${JIRA_HOME}")
    EXPECTED_PERMISSIONS=$(id -u ${RUN_USER}):${RUN_USER}:700
    if [ "${PERMISSIONS_SIGNATURE}" != "${EXPECTED_PERMISSIONS}" ]; then
        chmod -R 700 "${JIRA_HOME}" &&
            chown -R "${RUN_USER}:${RUN_GROUP}" "${JIRA_HOME}"
    fi
    # Now drop privileges
    exec su -s /bin/bash "${RUN_USER}" -c "$JIRA_INSTALL_DIR/bin/start-jira.sh $@"
else
    exec "$JIRA_INSTALL_DIR/bin/start-jira.sh" "$@"
fi
```

构建镜像：
```
sudo docker build -t jira:crack .
```
git地址：[https://git.chislab.com/projects/BUILD/repos/jira_crack/browse](https://git.chislab.com/projects/BUILD/repos/jira_crack/browse)

> 参考  
https://bitbucket.org/atlassian/docker-atlassian-confluence-server/src  
https://bitbucket.org/atlassian/docker-atlassian-bitbucket-server/src

# 4 额外操作

这部分主要进行下面两点：
- 将构建成功的初始镜像上传到docker hub仓库上（还需要安装）
- docker run进行一次安装过程，然后将容器commit成镜像，并上传到docker hub仓库上（之后就不需要安装了，之后再用docker-compose方式的再更新下）

## 4.1 初始镜像上传到docker hub
```
zqq@zqq-t480s:~/jira_crack$ sudo docker tag jira:crack dockerzqq/crack:jira-base
zqq@zqq-t480s:~/jira_crack$ sudo docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username (dockerzqq): ^C
zqq@zqq-t480s:~/jira_crack$ sudo docker push dockerzqq/crack:jira-base
The push refers to repository [docker.io/dockerzqq/crack]
d50c5179c813: Pushed 
51afbf5de568: Pushed 
da980f5d579a: Pushed 
c19a86f454c2: Pushed 
25312b71aa48: Pushed 
1c223f8dcff9: Pushed 
93351e248e6e: Mounted from library/openjdk 
298c3bb2664f: Mounted from library/openjdk 
73046094a9b8: Mounted from library/openjdk 
jira-base: digest: sha256:fed3602728cca83eb91226e48aa64fd81c06802a05daadf3618adfeb05ce227f size: 2208
```

~~## 4.2 安装镜像上传到docker hub~~

**以下方法不对，应该采用一个volume，因为后面docker-compose部署，数据全没了**

这里这样做的主要目的是用的同一个installation，不用以后再去官网配置评估版的许可证了
启动容器：
```
zqq@zqq-t480s:~/jira_crack$ sudo docker run -d -p 8090:8080 jira:crack
646081c7d19b0099cb3cc35f8dc73dd3f521f59cd62d4ce243366572046dae78
```
浏览器访问`localhost:8090`进行安装配置：


![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/97753665-2018-08-03%2016-15-11%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

详细过程略


安装后commit当前镜像，并上传到docker hub上：
```
zqq@zqq-t480s:~/jira_crack$ sudo docker commit 646 dockerzqq/crack:jira-after-install
sha256:5736daa8edb57ca69ccda953d286486caa94f4e94b6ef361e5f55014f4818e92
zqq@zqq-t480s:~/jira_crack$ sudo docker push dockerzqq/crack:jira-after-install
The push refers to repository [docker.io/dockerzqq/crack]
3ec945b80cf3: Pushed 
d50c5179c813: Layer already exists 
51afbf5de568: Layer already exists 
da980f5d579a: Layer already exists 
c19a86f454c2: Layer already exists 
25312b71aa48: Layer already exists 
1c223f8dcff9: Layer already exists 
93351e248e6e: Layer already exists 
298c3bb2664f: Layer already exists 
73046094a9b8: Layer already exists 
jira-after-install: digest: sha256:33a6e3acfe3341379191fd836d5ea24f12ecf70c8aa00d99483698f6270463a8 size: 2416
```

## 4.3 docker hub截图
docker hub上面的私有仓库

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/97753666-2018-08-03%2016-28-45%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

