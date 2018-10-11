---
title: "安装APUE环境"
classes: wide
toc: false
---

1. Download apue3 source code:
   ```
   $ wget http://www.apuebook.com/src.3e.tar.gz
   ```
2. Extract and Ungzip:
   ```
   $ tar zxvf src.3e.tar.gz
   ```
3. Make:
   ```
   $ cd apue.3e/
   $ make
   # you may see can't find -lbsd, solve it by install libbsd.a

   $ sudo apt-get install libbsd-dev  
   $ make
   ```
4. Copy:
   ```
   # gcc will search lib.a in /usr/local/lib/ directory
   $ sudo cp ./include/apue.h /usr/include/
   $ sudo cp ./lib/libapue.a /usr/local/lib/
   ```
5. Compile and Run:
   ```
   $ gcc 1-3.c -o 1-3 -lapue
   $ ./1-3
   ```
   
> 参考链接：  
> [http://blog.sina.com.cn/s/blog_94977c890102vdms.html](http://blog.sina.com.cn/s/blog_94977c890102vdms.html)

`src.3e.tar.gz`[备份地址](/assets/files/src.3e.tar.gz)