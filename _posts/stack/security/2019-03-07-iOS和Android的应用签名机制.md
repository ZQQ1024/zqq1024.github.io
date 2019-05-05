---
title: "iOS和Android的应用签名机制"
categories:
  - security
tags:
  - sign
classes: wide

excerpt: "iOS和Android签名机制对比"
---

# 1 Introduction

主要介绍以下两个方面：
1. `iOS App Store`和`Android Google Play`的应用签名机制
2. 对比一下2者的不同

# 2 iOS

## 2.1 作用
由于iOS系统的特性，苹果对应用签名，主要是控制用户可以在iOS上安装哪些应用，来控制软件恶意安装和盗版等问题

## 2.2 流程
满足上述需求，一个简单的实现就是，Apple生成一对公私钥对，私钥保存在Apple服务器上，公钥保留在iOS系统上：
1. 开发者上传应用到App Store上，Apple使用私钥对App进行签名
2. 用户从App Store上下载应用
3. iOS系统使用公钥对App进行验签，根据结果来决定是否安装

流程图如下：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190306195832.png)

上述满足了苹果需要最基本的一个需求1，即`iOS上安装的每个应用都是经过苹果官方认证的`

---

好的，新的需求来了，除了从App Store下载安装App，还有其他方式安装App，`开发人员可以直接将APP安装到iOS上，而且`

这里需要满足的需求2定义清楚就是：
1. App安装包不需要到苹果服务器进行签名
2. 同时，苹果仍然可以做到对App的安装进行控制

实现可以是，开发人员自己生成公私钥对，向苹果申请证书，苹果作为CA的存在：
1. 开发人员自己生成公私钥对，向苹果申请证书
2. 开发人员使用由自己生成的私钥，对App进行签名，将`App` + `签名` + `证书`打包在一起发布
3. iOS先使用苹果的公钥验证证书的合法性，如合法提取公钥
4. 然后使用提取出的公钥验证App的签名

流程图如下：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190307092231.png)

上述满足了需求2，即`iOS上安装的每个应用都是经过苹果官方认证的，同时苹果仍然对APP安装拥有控制权`

---

同时可以在证书中加入AppID和设备ID限制允许安装的APP和设备，基本思路和上面一致

# 3 Android

首先明确一点，Android对应用签名的作用`不是限制用户可以安装哪些软件`。而是通过证书识别Application作者的身份，并基于此建立多应用间可信的关系

## 3.1 说明

关于`Android APK Signing`的几点说明：
- 所有Application都是签了名的，Android系统不会运行没有签名的Application
- Debug时实际Android SDK使用debug key对Application签了名
- 不需要CA介入，自签名的证书就行
- 证书的时效性检查只是在应用安装的时候进行，应用安装后，证书过期，不会影响应用的功能

## 3.2 功能

关于Android应用签名的几个作用：
- `应用升级`：当应用（拥有相同的应用包名）升级时，系统会比对两个应用的证书，证书完全一样，系统才会升级应用（认为应用的作者为同一个作者）
- `应用模块化`：Android允许拥有相同证书的不同应用以相同的`process`运行，即每个应用相当于一个模块
- `数据/功能共享`：Android允许应用暴露特定的功能给指定证书/私钥签名的应用

## 3.3 Google Play
Google Play是Android官方的App Store，国内没法使用

开发者发布应用到Google Play的流程图如下：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190307100435.png)

可以看到的是，对App进行了2次签名，第一次是Google Play用于认证App作者的身份，第二次才是用户最终使用App时的签名，签名用的私钥是Google自己的而非应用者的

基于上述，可能会有一些问题存在，用户从Google Play下载的应用和其他地方下载的包名相同的应用不满足3.2所说的几个功能，比如出现应用不能升级等问题

# 4 References
> [http://blog.cnbang.net/tech/3386/](http://blog.cnbang.net/tech/3386/)  
[https://stackoverflow.com/questions/23906799/why-should-i-sign-my-application-apk-before-release](https://stackoverflow.com/questions/23906799/why-should-i-sign-my-application-apk-before-release)  
[https://blog.csdn.net/Dancen/article/details/81011669](https://blog.csdn.net/Dancen/article/details/81011669)
