---
title: "Django uWSGI Nginx"
categories:
  - backend
tags:
  - Django
classes: wide

excerpt: "Django和Nginx"
---

# 1 Introduction

Django部署方式多样，这篇介绍Nginx + uWSGI是其中的一种


首先大致了解一些概念。Web Server（如Nginx，Apache等）服务于静态文件，并不能直接和Python应用（如Django，Flask等）交互，WSGI是基于此诞生出来的一种Interface Specifiction，告诉你需要实现哪些方法用于在Web Server和Application之间传递请求和返回响应。而uWSGI是符合这一规范的的一种Implementation，uWSGI官方定义为`application server container`，即将Application包含在其后面，如下：

```
Web Client(Chrome) <-> Web Server(Nginx) <-> socket(web/unix socket) <-> uWSGI <-> Django
```

# 2 uWSGI

安装Django，创建一个项目（/root/mysite/）：
```
$ pip install django

$ django-admin startproject mysite
$ cd mysite
```

安装uWSGI：
```
$ pip install uwsgi
```

做一个基本测试，创建一个`test.py`文件，填入如下内容：
```
# test.py
def application(env, start_response):
    start_response('200 OK', [('Content-Type','text/html')])
    return [b"Hello World"] # python3
    #return ["Hello World"] # python2
```

启动uWSGI：
```
$ uwsgi --http :8000 --wsgi-file test.py
```

配置含义如下：
- `http: 8000`: 使用http协议，端口是8000
- `wsgi-file test.py`: 载入指定文件

浏览器访问如下：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190612204816.png)

此时的结构如下：
```
Web Client(Chrome) <-> uWSGI <-> Python
```

再使用Django site启动uWSGI：
```
$ uwsgi --http :8000 --module mysite.wsgi
```

浏览器访问如下：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190612205331.png)

此时的结构如下：
```
Web Client(Chrome) <-> uWSGI <-> Django
```

# 3 Nginx

安装Nginx：
```
$ apt-get update
$ apt-get install nginx
```

浏览器访问80端口如下：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190612232423.png)

此时的结构如下：
```
Web Client(Chrome) <-> Web Server(Nginx)
```

拷贝`/etc/nginx/uwsgi_params`文件到项目目录`/root/mysite/`下：
```
cp /etc/nginx/uwsgi_params /root/mysite/
```

在`/etc/nginx/sites-enabled/`目录下添加一个配置文件`mysite_nginx.conf`：
```
# mysite_nginx.conf

# the upstream component nginx needs to connect to
upstream django {
    # server unix:///path/to/your/mysite/mysite.sock; # for a file socket
    server 127.0.0.1:8001; # for a web port socket (we'll use this first)
}

# configuration of the server
server {
    # the port your site will be served on
    listen      8000;
    # the domain name it will serve for
    server_name example.com; # substitute your machine's IP address or FQDN
    charset     utf-8;

    # max upload size
    client_max_body_size 75M;   # adjust to taste

    # Django media
    location /media  {
        alias /root/mysite/media;  # your Django project's media files - amend as required
    }

    location /static {
        alias /root/mysite/static; # your Django project's static files - amend as required
    }

    # Finally, send all non-media requests to the Django server.
    location / {
        uwsgi_pass  django;
        include     /root/mysite/uwsgi_params; # the uwsgi_params file you installed
    }
}
```

项目静态文件配置`mysite/settings.py`：
```
STATIC_ROOT = os.path.join(BASE_DIR, "static/")
```

静态文件放到统一目录下：
```
$ python manage.py collectstatic
```

重启Nginx：
```
$ nginx -s reload
```

基本测试：
```
$ uwsgi --socket :8001 --wsgi-file test.py
```

浏览器访问略

此时的结构如下：
```
Web Client(Browser) <-> Web Server(Nginx) <-> socket(web socket) <-> uWSGI <-> Python
```

# 4 Django with uWSGI and Nginx

运行下面命令：
```
$ uwsgi --socket mysite.sock --module mysite.wsgi --chmod-socket=664
```

可以将相关配置写入文件：
```
# mysite_uwsgi.ini file
[uwsgi]

# Django-related settings
# the base directory (full path)
chdir           = /root/mysite
# Django's wsgi file
module          = mysite.wsgi
# the virtualenv (full path)
# home            = /path/to/virtualenv

# process-related settings
# master
master          = true
# maximum number of worker processes
processes       = 10
# the socket (use the full path to be safe
socket          = /root/mysite/mysite.sock
# ... with appropriate permissions - may be needed
chmod-socket    = 666
# clear environment on exit
vacuum          = true
```

再以下面的命令启动：
```
$ uwsgi --ini mysite_uwsgi.ini # the --ini option is used to specify a file
```

# 5 Problems and Improvments 

至此，Django使用uWSGI、Nginx基本结束

下面提供几个改进思路：
- 使用Unix Socket替代TCP Port Socket
- 运行uWSGI指明uid gid
- 开机自启
- 使用Emperor模式，管理所有vassal(uWSGI实例)

可能会出现Unix Socket的权限问题，配置证书在Nginx上配置就行

# 6 References

> [https://uwsgi-docs.readthedocs.io/en/latest/tutorials/Django_and_nginx.html](https://uwsgi-docs.readthedocs.io/en/latest/tutorials/Django_and_nginx.html)  
[https://www.cnblogs.com/alex3714/p/6538374.html](https://www.cnblogs.com/alex3714/p/6538374.html)  
[https://www.cnblogs.com/leohahah/p/9081541.html](https://www.cnblogs.com/leohahah/p/9081541.html)  
[https://github.com/unbit/uwsgi](https://github.com/unbit/uwsgi)  
[https://stackoverflow.com/questions/31970908/how-to-kill-all-instance-of-uwsgi](https://stackoverflow.com/questions/31970908/how-to-kill-all-instance-of-uwsgi)  
[https://stackoverflow.com/questions/38601440/what-is-the-point-of-uwsgi](https://stackoverflow.com/questions/38601440/what-is-the-point-of-uwsgi)