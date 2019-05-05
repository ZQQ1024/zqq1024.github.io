---
title: "基于Session和Token认证"
categories:
  - security
tags:
  - authentication
classes: wide

excerpt: "Session和Token对比"
---

# 1 Introduction
HTTP是stateless的，即这一条请求不能感知到上一条请求的存在

举个例子：
比如我们网上在购物，
我们往购物车里添加了一堆商品(AirPods,Macbook Pro,Filco Keyboard等等)，然后我们点击支付，发现支付时商品全没了，是因为支付请求没法记住是具体那个用户购物车的添加状态，（而不是网站不想让你买，开个玩笑）

使用session和token可以为HTTP附加状态，而不用每个请求都包含用户的账户和密码进行认证，解决上面的问题

# 2 Session
Session是服务器端保存的key:value数据，常和Cookie结合

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190425111558.png)

基于Session的认证流程基本如下：
1. 用户输入username、pw登录
2. 服务器认证过后，生成session的key:value形式在服务器端保存
3. key为sessionID，value中包含用户信息，sessionID一般位数很长，被爆破撞到sessionID的可能性极低
4. 服务器返回sessionID，sessionID作为Cookie存在客户端浏览器当中
5. 后续请求都会含有sessionID，服务器端根据sessionID获取用户信息
6. 获取用户信息后，返回该用户相关的响应

# 3 Token
许多web应用使用Token如(JSON Web Token JWT)对用户进行认证，而不基于Session

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190425111940.png)

基于Token的认证流程基本如下：
1. 用户输入username、pw登录
2. 服务器端用服务器私钥创建（本质是使用私钥对相关数据进行签名）Token
3. 服务器返回Token给客户端，Token包含相关数据和签名
4. 后续请求相关Header，如"Authorization"如都会携带Token
5. 服务器对Token进行验签，验证通过后，提取用户信息
6. 获取用户信息后，返回该用户相关的响应

# 4 Comparison
对比下基于Session和Token进行的认证：

**扩展性**：
`Session base`: session数据要存在服务器的内存或其他存储空间中，当用户数量多的时候，扩展性越低
`Token base`: 服务器没有存储数据，而是客户端存，签名验签需要消耗时间，扩展性较高，Token有个短处，Token长度要比sessionID长度要大的多，所以每次请求的大小也要大点

**跨域名**：
`Session base`: sessionID需要和Cookie结合起来使用，而Cookie只能在同一域名或子域名下工作，当跨域名的时候，浏览器会禁掉Cookie

`Token base`: 跨域名也可以使用

个人认为基于Session的认证是以消耗空间为代价获得的，基于Token的认证是以消耗时间为代价获得的

现在的web应用推荐使用基于Token的认证

# 5 Security

Token中因为要包含信息，所以不能放入特别敏感的信息

不论是基于Session的还是基于Token的认证，若网站存在XSS漏洞，则安全性丢失（页面被植入XSS，用户执行这个脚本后，会将Token和sessionID上传给Hacker）

# 6 References
> [https://medium.com/@sherryhsu/session-vs-token-based-authentication-11a6c5ac45e4](https://medium.com/@sherryhsu/session-vs-token-based-authentication-11a6c5ac45e4)

主要是理解翻译