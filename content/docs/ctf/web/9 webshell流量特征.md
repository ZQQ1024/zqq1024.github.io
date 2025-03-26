---
title: "webshell流量特征" 
weight: 10
bookToc: true
---

## 中国菜刀

1. UA头
```
User-Agent: Mozilla/5.0 (compatible; Baiduspider/2.0; +http://www.baidu.com/search/spider.html)
```
2. payload base64编码，且存在固定的开头
```
QGluaV9zZXQoImRpc3BsYXlfZXJyb3JzIiwiMCIpO0BzZXRfdGltZV9saW1pdCgwKTtAc2V0X21hZ2ljX3F1b3Rlc19ydW50aW1lKDApO2
```
对应
```php
@ini_set("display_errors","0");@set_time_limit(0);@set_magic_quotes_runtime(0);
// 关闭错误显示、设置不限时运行、关闭魔术引号
```
![](/data/image/web//flow/image.png)
3. 请求存在 `z0/z1/...` 参数
![](/data/image/web//flow/image-1.png)
4. 响应使用 `->| xxxx |<-` 封装
![](/data/image/web//flow/image-2.png)

## 蚁剑

1. 参数名随机`d43729c7e26d7d`等
2. 存在 `@ini_set("display_errors","0");@set_time_limit(0);`
3. payload存在简单的混淆，执行的命令会被base64编码，且需要去除前2个字节，最终执行的命令是`$r = "{$p} {$c}";`类似`cmd.exe /c "xxx"`
![](/data/image/web//flow/image-3.png)
![](/data/image/web//flow/image-4.png)
![](/data/image/web//flow/image-5.png)
4. 因为执行命令时加了`echo xxx`的随机打印，所有返回会有对应随机的输出
![](/data/image/web//flow/image-6.png)

## 冰蝎

TODO

## 哥斯拉

TODO