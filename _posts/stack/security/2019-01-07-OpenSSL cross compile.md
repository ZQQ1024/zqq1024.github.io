---
title: "OpenSSL cross compile"
categories:
  - security
tags:
  - openssl
classes: wide

excerpt: "OpenSSL交叉编译"
---


介绍`OpenSSL`交叉编译，为`arm64`平台，`Linux`（相关）和`Android`系统

`OpenSSL`版本： 1.1.1 stable  
`NDK`版本：r18b

# 1 Linux

compiler: GNU C compiler

1. 安装`The GNU C compiler for arm64 architecture`：
```
$ sudo apt-get install gcc-aarch64-linux-gnu
```

2. 设置相关环境变量：
```
$ export CROSS_COMPILE=aarch64-linux-gnu-
$ export ARCH=arm64
```

3. 准备代码：
```
$ git clone https://github.com/openssl/openssl.git
$ cd openssl

# 下面命令默认track远程分支
$ git checkout OpenSSL_1_1_1-stable
Branch OpenSSL_1_1_1-stable set up to track remote branch OpenSSL_1_1_1-stable from origin.
Switched to a new branch 'OpenSSL_1_1_1-stable'
```

4. Configure：
```
$ ./Configure linux-aarch64 --cross-compile-prefix=${CROSS_COMPILE} --prefix=`pwd`/output --openssldir=`pwd`/output

# --cross-compile-prefix 设置其他环境变量的前缀
# --prefix是安装目录
```
5. make&make install：
```
$ make
$ make install

# file type bin/openssl
$ file openssl  
openssl: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, for GNU/Linux 3.7.0, BuildID[sha1]=9e2550ea73e9dc838bf71713987300daea8cc6cd, not stripped
```

# 2 Android
compiler: Clang/LLVM

1. 下载NDK
[https://developer.android.com/ndk/downloads/](https://developer.android.com/ndk/downloads/)，这里下载的版本是`r18b`

2. 设置相关环境变量：
```
$ export ANDROID_NDK=/home/zqq/Desktop/android-ndk-r18b
$ export PATH=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64:$PATH
```

3. 准备代码：同上
4. Configure：
```
$ ./Configure android-arm -D__ANDROID_API__=21 --prefix=`pwd`/output --openssldir=`pwd`/output
```
5. make&make install：
```
$ make
$ make install

# file type bin/openssl
$ file openssl  
openssl: ELF 32-bit LSB shared object, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /system/bin/linker, not stripped
```

ps：ndk-r18b移除了gcc，使用clang/llvm替换了gcc。这里使用的是arm而不是arm64

# 3 Test
```
$ openssl speed -evp aes-256-cbc -elapsed -multi 1

# mutli-core
$ openssl speed -evp aes-256-cbc -elapsed -multi 2
```

# 4 References
> [http://wiki.macchiatobin.net/tiki-index.php?page=OpenSSL+Installation+Guide](http://wiki.macchiatobin.net/tiki-index.php?page=OpenSSL+Installation+Guide)  
[https://github.com/openssl/openssl/issues/7578](https://github.com/openssl/openssl/issues/7578)  
[https://github.com/openssl/openssl/blob/OpenSSL_1_1_1-stable/INSTALL](https://github.com/openssl/openssl/blob/OpenSSL_1_1_1-stable/INSTALL)  
[https://github.com/openssl/openssl/blob/master/NOTES.ANDROID](https://github.com/openssl/openssl/blob/master/NOTES.ANDROID)  
[https://github.com/projectNe10/Ne10/issues/131](https://github.com/projectNe10/Ne10/issues/131)  
[https://developer.android.com/ndk/downloads/revision_history](https://developer.android.com/ndk/downloads/revision_history)  