---
title: "Docker Nginx uWSGI Django SSL"
categories:
  - backend
tags:
  - Django
classes: wide

excerpt: "Docker Django Nginx SSL"
---

# 1 Introduction

使用Docker部署Nginx uWSGI Django等应用，仅仅提供配置文件样例供参考

准备条件：
- 安装Docker
- 安装Docker-compose

使用的目录大致如下：
```
├── Dockerfile
├── __init__.py
├── data
│   └── db
├── db.sqlite3
├── doc
├── docker-compose.yaml
├── manage.py
├── nginx
│   ├── cert
│   ├── nginx.conf
│   └── pay_nginx.conf
├── pay
│   ├── __init__.py
│   ├── admin.py
│   ├── migrations
│   ├── models.py
│   ├── urls.py
│   ├── utils.py
│   └── views.py
├── payserver
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── requirements.txt
├── static
│   └── admin
├── uwsgi.ini
```

# 2 uWSGI + Django

将Django项目文件打包成镜像，`Dockerfile`类似如下：
```
FROM python:3.7
MAINTAINER zqq840486874@gmail.com

ENV PYTHONUNBUFFERED 1
RUN mkdir /pay
WORKDIR /pay
COPY requirements.txt /pay/
RUN pip install -r requirements.txt
# EXPOSE 8000 # link不需要expose的
COPY . /pay/
```

需要将uWSGI放入requirements.txt中：
```
$ pip freeze > requirements.txt
```

wsgi配置文件如下：
```
# mysite_uwsgi.ini file
[uwsgi]

# Django-related settings
# the base directory (full path)
chdir           = /pay
# Django's wsgi file
module          = payserver.wsgi
# the virtualenv (full path)
# home            = /path/to/virtualenv

# process-related settings
# master
master          = true
# maximum number of worker processes
processes       = 10
# the socket (use the full path to be safe
socket          = :8000 # /pay/pay.sock
# ... with appropriate permissions - may be needed
# chmod-socket    = 666
# clear environment on exit
vacuum          = true
```

# 3 Nginx + SSL

`nginx.conf`配置文件如下：
```
user nginx;
worker_processes auto;
pid /run/nginx.pid;

events {
        worker_connections 768;
        # multi_accept on;
}

http {

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        ##
        # Gzip Settings
        ##

        gzip on;
        gzip_disable "msie6";

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}
```

站点配置文件`pay_nginx.conf`内容如下：
```
# the upstream component nginx needs to connect to
upstream django {
    # server unix:///etc/nginx/pay.sock; # for a file socket
    server pay_server:8000; # for a web port socket (we'll use this first)
}

# configuration of the server
server {
    # the port your site will be served on
    listen      443 ssl;
    # the domain name it will serve for
    server_name _; # substitute your machine's IP address or FQDN

    ssl on;

    ssl_certificate      /etc/nginx/pay_cert/certificatepay.crt;     #pem 位置
    ssl_certificate_key  /etc/nginx/pay_cert/privatepay.key;     #key 位置
    ssl_session_timeout  5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2; 

    charset     utf-8;

    # max upload size
    client_max_body_size 75M;   # adjust to taste

    # Django media
    # location /media  {
    #    alias /root/mysite/media;  # your Django project's media files - amend as required
    # }

    location /static {
        alias /etc/nginx/static; # your Django project's static files - amend as required
    }

    # Finally, send all non-media requests to the Django server.
    location / {
        # return 200 "hello";
        uwsgi_pass  django;
        include     /etc/nginx/uwsgi_params; # the uwsgi_params file you installed
    }
}
```

准备证书、私钥

# 4 Docker-compose

docker-compose用于启动多个服务，`docker-compose.ymal`文件如下，注意挂载信息：
```
version: '3'

services:
  db:
    image: mongo:latest
    container_name: mongo_db
    volumes:
          - ./data/db:/data/db
  web:
    build: .
    command: uwsgi --ini /pay/uwsgi.ini
    container_name: pay_server
    volumes:
      - .:/pay
    # ports:
    #  - "8000:8000"
    depends_on:
      - db
    environment:
      - MONGODB_HOST=mongo_db
      - MONGODB_DB=paydb

  nginx:
    image: nginx:latest
    container_name: web_server
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      # - ./pay.sock:/etc/nginx/pay.sock
      - ./nginx/pay_nginx.conf:/etc/nginx/sites-enabled/pay_nginx.conf
      - ./nginx/cert:/etc/nginx/pay_cert
      - ./static:/etc/nginx/static
    depends_on:
      - web
    links:
      - web
    ports:
      - "80:80"
      - "443:443"
```


启动：
```
$ docker-compose up --build
```

# 5 References
> [https://www.jerrycoding.com/article/site_building_14/](https://www.jerrycoding.com/article/site_building_14/)  
[https://blog.csdn.net/benimang/article/details/80515330](https://blog.csdn.net/benimang/article/details/80515330)