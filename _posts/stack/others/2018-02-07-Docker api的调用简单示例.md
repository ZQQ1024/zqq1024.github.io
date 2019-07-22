---
title: "Docker API"
categories:
  - others
tags:
  - docker
classes: wide

excerpt: "Docker api的调用简单示例"
---


# docker api的调用简单示例

## 1 环境
- curl版本7.40及以上（或者Postman好点）
- 操作系统CentOS 7
- docker server version：
  - 17.12.0-ce（API 1.35）
- 修改/etc/docker/daemon.json（这里使用的是Unix套接字）:
  - {"hosts": ["unix:///var/run/docker.sock"]}
- 说明：只是写了调用的例子，更详细的调用和返回参看官方api文档

## 2 Images
### 2.1 List images
```
curl --unix-socket /var/run/docker.sock -X GET http://v1.35/images/json
```
### 2.2 Build an image
先把Dockerfile打包：

```tar zcf Dockerfile.tar.gz Dockerfile```
```
curl -v --unix-socket /var/run/docker.sock -H "Content-Type: application/x-tar" -X POST \
--data-binary '@Dockerfile.tar.gz' 'http://v1.35/build?t=build:test'
```
Query Parameter：t指定tag

### 2.3 Create an image
通过pull或者import创建镜像的，没有指定tag则会pull该image的repo下面所有tag的镜像。
```
curl -v --unix-socket /var/run/docker.sock -H "Content-Type: application/tar" -X POST \
--data-binary '@hello.tar' 'http://v1.35/images/create?fromSrc=-&tag=hello:latest'
```
### 2.4 Tag an image
```
curl --unix-socket /var/run/docker.sock -X POST 'http://v1.35/images/c72c/tag?tag=zqq01&repo=test'
```
c72c换成image的name或者id

### 2.5 Push an image
暂略，指明X-Registry-Auth头就行。

### 2.6 Export an image
```
curl --unix-socket /var/run/docker.sock -o nginx.tar -X GET http://v1.35/images/nginx/get
```
就是以tarball形式获取一个镜像或者repo。

### 2.7 Import images
```
curl --unix-socket /var/run/docker.sock --data-binary '@nginx.tar' -H "Content-Type: application/tar" -X POST http://v1.35/images/load
```
和上面对应。```docker save```和```docker import```。

## 3 Containers
### 3.1 List containers
```
curl --unix-socket /var/run/docker.sock -X GET http://v1.35/containers/json

curl --unix-socket /var/run/docker.sock -X GET http://v1.35/containers/json?all=true

curl --unix-socket /var/run/docker.sock -X GET 'http://v1.35/containers/json?all=true&filters={"id":["123"]}'
```
分别展示运行中的、所有、和id=123XXXX的容器。

### 3.2 Create a container
```
curl --unix-socket /var/run/docker.sock -H "Content-Type: application/json" \
--data '{"Image": "nginx"}'-X POST http://v1.35/containers/create?name=curl
```

### 3.3 Inspect a container
```
curl --unix-socket /var/run/docker.sock -X GET http://v1.35/containers/77a/json
```
77a换成container的ID或这name

### 3.4 Start a container
```
curl --unix-socket /var/run/docker.sock -H "Content-Type: application/json" -X POST http://v1.35/containers/e33/start
```

### 3.5 Rename a container
```
curl --unix-socket /var/run/docker.sock -H "Content-Type: application/json" -X POST http://v1.35/containers/e33/rename?name="curl_new"
```

## 4 Exec
### 4.1 Create an exec instance
```
curl --unix-socket /var/run/docker.sock -H "Content-Type: application/json" \
--data '{"Cmd":["date"]}' -X POST http://v1.35/containers/6f9/exec
```
返回的是instance的ID。

### 4.2 Start an exec instance
```
curl --unix-socket /var/run/docker.sock -H "Content-Type: application/json" --data '{"Detach":false,"Tty":true}' \
-X POST http://v1.35/exec/a7f0c18c20b2cafbff56ea81b9502d2ce2d97167ee804a16d5383cb54fb7f584/start
```
ID写全。

## 5 Swarm
### 5.1 Inspect a swarm
```
curl --unix-socket /var/run/docker.sock -X GET http://v1.35/swarm
```
### 5.2 Init a swarm
```
curl --unix-socket /var/run/docker.sock -H "Content-Type: application/json" --data '{"AdvertiseAddr": "172.17.122.115:2377"}' -X POST http://v1.35/swarm/init
```
没成功。

### 5.3 Join a swarm

## 6 Service

## 7 Secrets

## 8 Configs

6、7、8同理，上面基本已经涉及所有的调用方式了，暂略。

> 参看 [https://docs.docker.com/engine/api/v1.35/](https://docs.docker.com/engine/api/v1.35/)