---
title: "Http Post Methods"
categories:
  - backend
tags:
  - http
classes: wide

excerpt: "4种http post方法"
---

# 1 Introduction
使用`python requests module`编写Client，django编写Server，来理解、总结4种HTTP POST methods
-  x-www-form-urlencoded
-  form-data
-  raw
-  binary

下面将定义一个场景将HTTP POST不同methods串联起来：
> `ZQQ`的大学老师向`ZQQ`同学布置了一个任务  
老师给了`ZQQ`10张图片  
让`ZQQ`调用阿里的内容安全检测接口，辨别10张图片是否涉及暴力、色情  
并输出结果报告，报告名称为`内容检测报告.doc`  
输出结果报告发给老师

同时我们需要安装下面的包：
- `requests`: 构造请求，模拟Client
- `requests_toolbelt`: 输出上述请求内容
```
(venv) # pip install requests requests_toolbelt
```

# 2 HTTP format
先复习下，HTTP请求的格式：
```
format:

<http_method> <url> <http_version>\r\n
<http_headers> \r\n
\r\n
<http_body>

eg:

POST /seafhttp/repo/head-commits-multi/ HTTP/1.1
Host: xx.xx.xx.xx
Accept: */*
User-Agent: Seafile/6.1.8 (Linux)
Content-Length: 196
Content-Type: application/x-www-form-urlencoded

name=zqq&desc=sbzqq
```
响应格式略

# 3 x-www-form-urlencoded
这种POST方法可能是最常见的了，在`ZQQ`调用阿里云内容检测接口之前，需要在阿里云注册帐号，注册帐号时需要提供以下内容：
- `userName`: sbzqq
- `fullName`: 周倩倩
- `password`: test my password
- `telphone`: 13312341234
- `age`: 18

这时候可以使用`x-www-form-urlencoded`发送数据

Client代码：
```
import requests
from requests_toolbelt.utils import dump
import json


def formURLEncode():
    url = 'http://127.0.0.1:8000/postMethod/formURLEncode'

    enrollment = {
        'userName': 'sbzqq',
        'fullName': '周倩倩',
        'password': 'test my password',
        'telphone': '13312341234',
        'age': 18
    }

    resp = requests.post(url, data=enrollment)

    data = dump.dump_all(resp)
    print(data.decode('utf-8'))
```
Server代码：
```
# 简单的打印
def formURLEncode(request):
    print(request.POST.get('userName'))
    print(request.POST.get('fullName'))
    print(request.POST.get('password'))
    print(request.POST.get('age'))
    return HttpResponse("zqq shi sb")
```
Client输出：
```
< POST /postMethod/formURLEncode HTTP/1.1
< Host: 127.0.0.1:8000
< Connection: keep-alive
< Accept-Encoding: gzip, deflate
< Accept: */*
< User-Agent: python-requests/2.21.0
< Content-Length: 105
< Content-Type: application/x-www-form-urlencoded
< 
< fullName=%E5%91%A8%E5%80%A9%E5%80%A9&telphone=13312341234&userName=sbzqq&age=18&password=test+my+password

> HTTP/1.1 200 OK
> Date: Sun, 28 Apr 2019 05:10:53 GMT
> Server: WSGIServer/0.2 CPython/3.5.2
> Content-Length: 10
> X-Frame-Options: SAMEORIGIN
> Content-Type: text/html; charset=utf-8
> 
zqq shi sb
```
Server输出：
```
sbzqq
周倩倩
test my password
18
```

GET和POST`x-www-form-urlencoded`都用到了URLEncode，都是对printable-text进行操作

我们故意让`password=test my password`包含了空格，`fullName=周倩倩`包含了中文，可以看出空格变成了`+`，中文等非ASCII字符会被`utf-8`编码
```
utf-8
周倩倩 = \xE5\x91\xA8\xE5\x80\xA9\xE5\x80\xA9

urlencode
周倩倩 = %E5%91%A8%E5%80%A9%E5%80%A9
test my password = test+my+password
```

# 4 form-data
`ZQQ`同学在阿里云注册的帐号注册成功了，这是他想上传下个人信息的照片，需要传输以下内容：
- `userName`: sbzqq
- 一张图片

这时候可以使用`multipart/form-data`发送数据

Client代码：
```
def formData():
    url = 'http://127.0.0.1:8000/postMethod/formData'

    photo_form_data = {
        'userName': (None, 'sbzqq'),
        'avator': ('avator.jpg', open('avator.jpg', 'rb'), 'image/jpeg'),
    }

    resp = requests.post(url, files=photo_form_data)
    data = dump.dump_all(resp)
    print(data)
```
Server代码：
```
def formData(request):
    print(request.POST.get('userName'))
    avator = request.FILES.get('avator')
    print(avator.name)
    f = open('zqq_avator.jpg', 'wb')
    f.write(avator.read())
    f.close()
    return HttpResponse("zqq shi sb")
```
Client输出：
```
略
一堆bytes
```
Server输出：
```
sbzqq
avator.jpg
```

photo_form_data中文件部分包含`form-data name`，`filename`，`file-like object`，`content-type`等信息，非文件传个`None`的placeholder

# 5 raw
raw表示原生的意思，就是自定义请求体的传输内容，当然目前最常见的是JSON格式

`ZQQ`同学想正式调用阿里云内容检测接口了，但是接口提供的形式是`application/json`的

需要传输以下内容：
- `userName`: sbzqq
- `password`: test my password
- `url`: https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png
- `policy`: 严格

Client代码：
```
def raw():
    url = 'http://127.0.0.1:8000/postMethod/raw'

    json_dic = {
        'userName': 'sbzqq',
        'password': 'test my password',
        'url': 'https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png',
        'policy': '严格'
    }

    headers = {
        'Content-Type': 'application/json'
    }

    resp = requests.post(url, data=json.dumps(json_dic), headers=headers)

    data = dump.dump_all(resp)
    print(data.decode('utf-8'))
```
Server代码：
```
def raw(request):
    data_bytes = request.body
    data_json = data_bytes.decode('utf-8')
    data_dic = json.loads(data_json)

    print(data_dic['userName'])
    print(data_dic['password'])
    print(data_dic['url'])
    print(data_dic['policy'])
    return HttpResponse("zqq shi sb")
```
Client输出：
```
< POST /postMethod/raw HTTP/1.1
< Host: 127.0.0.1:8000
< Accept-Encoding: gzip, deflate
< Connection: keep-alive
< User-Agent: python-requests/2.21.0
< Accept: */*
< Content-Type: application/json
< Content-Length: 172
< 
< {"policy": "\u4e25\u683c", "url": "https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png", "password": "test my password", "userName": "sbzqq"}

> HTTP/1.1 200 OK
> Date: Sun, 28 Apr 2019 07:03:43 GMT
> Server: WSGIServer/0.2 CPython/3.5.2
> Content-Type: text/html; charset=utf-8
> Content-Length: 10
> X-Frame-Options: SAMEORIGIN
> 
zqq shi sb
```
Server输出：
```
sbzqq
test my password
https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png
严格
```

可以看到json.dumps在处理空格和非ASCII字符时与URLEncode的不同之处，`严格`中文字符变成了Unicode Code Point`\u4e25\u683c`，表明在Unicode字符集中的位置，而不是`utf-8`编码，空格仍然保留

# 6 binary
好的，图片已经检测完了，`ZQQ`同学需要将结果汇总，报告名称为`内容检测报告.doc`，将报告传给老师

这时候需要传送的内容：
- 内容检测报告.doc

`binary`其实适合上面的`raw`一样，都是自定义body的内容，只不过上面传递的是`text`，这里传的是`binary`文件，本质还是都是一堆`bytes`

Client代码：
```
def binary():
    url = 'http://127.0.0.1:8000/postMethod/binary'

    f = open('内容检测报告.doc', 'rb')
    headers = {
        'Content-Type': 'application/msword'
    }

    resp = requests.post(url, data=f.read(), headers=headers)
    data = dump.dump_all(resp)
    print(data)
```
Server代码：
```
def binary(request):
    f = open('zqq_内容检测报告.doc', 'wb')
    f.write(request.body)
    f.close()
    return HttpResponse("zqq shi sb")
```
Client输出：
```
略
一堆bytes
```
Server输出：
```
无

# zqq_内容检测报告和之前的文件内容一样
```

# 7 Others

还可以使用`postman` + `wireshark`，发送请求和抓包理解，而不使用上述编写代码的方式
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190428155455.png)

postman collection：  
[https://www.getpostman.com/collections/3596bbf2359474d79ca0](https://www.getpostman.com/collections/3596bbf2359474d79ca0)

# 8 References
> [https://2.python-requests.org//en/v0.10.7/user/quickstart/#make-a-post-request](https://2.python-requests.org//en/v0.10.7/user/quickstart/#make-a-post-request)  
[https://www.imooc.com/article/33641](https://www.imooc.com/article/33641)  
[https://stackoverflow.com/questions/20658572/python-requests-print-entire-http-request-raw](https://stackoverflow.com/questions/20658572/python-requests-print-entire-http-request-raw)  
[https://stackoverflow.com/questions/12385179/how-to-send-a-multipart-form-data-with-requests-in-python](https://stackoverflow.com/questions/12385179/how-to-send-a-multipart-form-data-with-requests-in-python)  
[https://www.urlencoder.io/](https://www.urlencoder.io/)  
[https://mothereff.in/utf-8](https://mothereff.in/utf-8)  
[https://stackoverflow.com/questions/4706255/how-to-get-value-from-form-field-in-django-framework](https://stackoverflow.com/questions/4706255/how-to-get-value-from-form-field-in-django-framework)  
[https://stackoverflow.com/questions/3702465/how-to-copy-inmemoryuploadedfile-object-to-disk](https://stackoverflow.com/questions/3702465/how-to-copy-inmemoryuploadedfile-object-to-disk)  
[https://stackoverflow.com/questions/10729397/how-to-get-name-of-file-in-request-post](https://stackoverflow.com/questions/10729397/how-to-get-name-of-file-in-request-post)  
[https://unicode-table.com/en/search/?q=%E4%B8%A5](https://unicode-table.com/en/search/?q=%E4%B8%A5)    
[https://stackoverflow.com/questions/4212861/what-is-a-correct-mime-type-for-docx-pptx-etc](https://stackoverflow.com/questions/4212861/what-is-a-correct-mime-type-for-docx-pptx-etc)  