---
title: "file open mode"
categories:
  - others
tags:
  - file

classes: wide

excerpt: "介绍r wb+等打开模式的区别"
---


# 1 Introduction
经常看到`r wb+`等各种文件的打开模式，没有总结过他们的区别，这篇使用python实验了解一下各种file open mode的区别（和读取权限是2个东西）

先交代下用到的两个文件：
- `test.wav`是一个二进制音频文件
- `test.txt`是一个文本文件（文本文件也是二进制文件，只不过涉及了编码），内容如下，大小为14bytes：
```
abc
zqq
帅比
```

# 2 r/w/a
**r**: read-only，返回的是string，文件头打开

验证代码如下：
```
# binary file
f1 = open('test.wav', 'r')
# text file
f2 = open('test.txt', 'r')

try:
    f1.write("123")
except Exception as e:
    print("write str:")
    print(e)
    print()

try:
    f1.write(b'\x5a\x51\x51')
except Exception as e:
    print("write bytes:")
    print(e)
    print()

try:
    binary_data = f1.read()
    print(type(binary_data))
except Exception as e:
    print("read binary file:")
    print(e)
    print()

try:
    print("Current Position (byte):")
    print(f2.tell())

    text_data = f2.read()

    print("Return type")
    print(type(text_data))

    print("Return text:")
    print(text_data)

    print("Current Position (byte):")
    print(f2.tell())
except Exception as e:
    print("read text file:")
    print(e)
    print()
```

验证结果如下：
- 只读，不能写入
- 最终返回的结果是str
- 文件打开位置在文件头


---


**w**: write-only，写入的是str，文件头打开，文件不存在则创建，文件存在覆盖写

验证代码如下：
```
f1 = open('test.txt', 'r')
# text file
f2 = open('test.txt', 'w') # 这里文件已经被清空了
# new text file
f3 = open('test_new.txt', 'w')

try:
    text_data = f2.read()
except Exception as e:
    print("read str:")
    print(e)
    print()

try:
    f1.write(b'\x5a\x51\x51')
except Exception as e:
    print("write bytes:")
    print(e)
    print()

try:
    print("Current Position (byte):")
    print(f2.tell())

    f2.write("zqq shi sb")

    print("Current Position (byte):")
    print(f2.tell())

    text_data = f1.read()
    print("Return text:")
    print(text_data)
except Exception as e:
    print("write text file:")
    print(e)
    print()

try:
    f3.write("zqq shi sb")
except Exception as e:
    print("write new text file:")
    print(e)
    print()
```

验证结果如下：
- 只写，不能读取
- 只能写入str
- 如果文件存在，之前文件内容会被完全覆盖；若文件不存在，则创建文件，就是一个brand new的文件

如下所有时间都会更新，所以就相当于一个brand new的文件：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190419181033.png)

---

**a**: 和**w**类似，不过是追加写，而不是覆盖，若文件不存在则创建

验证代码如下：
```
f2 = open('test.txt', 'w')
f1 = open('test.txt', 'a')
f3 = open('test.txt', 'r')

try:
    f2.write("zqq shi sb")
    f2.close()

    text_data = f3.read()
    print(text_data)

    f1.write("zqq shi sb")
    f1.close()

    f3.seek(0)
    text_data = f3.read()
    print(text_data)
except Exception as e:
    print(e)
```

验证结果如下：
- **a**是移动文件尾+写入，即不论文件尾是否变化，每次写入都保证了是在文件尾追加写

# 3 +

**r+**: read-write，侧重read

验证代码如下：
```
f1 = open('test.txt', 'r+')

try:
    text_data = f1.read()
    print(text_data)
except Exception as e:
    print(e)

try:
    f1.write("zqq shi sb")
    text_data = f1.read()
    print("f1 read nothing:" + text_data)

    f2 = open('test.txt', 'r')
    text_data = f2.read()
    print("f2 read all:" + text_data)
except Exception as e:
    print(e)
```

验证结果如下：
- r/w操作共享position，就read几个字节，write时的偏移也变了，即read/write操作都会改变文件的当前偏移

---

**w+**: read-write，侧重write

验证代码如下：
```
f1 = open('test.txt', 'w+')

try:
    f1.write("zqq shi sb")


    text_data = f1.read()
    print("read before seek:" + text_data)

    f1.seek(0)
    text_data = f1.read()
    print("read after seek:" + text_data)
except Exception as e:
    print(e)
```

验证结果如下：
- open的时候文件已经被覆盖了，r/w操作共享position

---

**a+**: 相比**a**可以读

# 4 b

r/w都是针对str，加上**b**就是针对二进制文件，读写bytes

验证代码如下，这里仅以**rb**为例：
```
f1 = open('test.wav', 'rb')

try:
    f1.write("123")
except Exception as e:
    print("write str error")

try:
    f1.write(b'\x5a\x51\x51')
except Exception as e:
    print("write bytes error")

try:
    binary_data = f1.read()
    print(type(binary_data))
except Exception as e:
    print("read binary file error")
```

验证结果如下：
- 和r/w相比，操作的对象是bytes，而不是str，没有其他的区别

# 5 mode
总结python open file的各种mode：
- **r、rb**: 只读str/bytes
- **r+、rb+**: 读写str/bytes，不会truncate
- **w、wb**: 只写str/bytes
- **w+、wb+**: 读写str/bytes，会truncate
- **a、ab**: 只写str/bytes，append方式写入
- **a+、ab+**: 读写str/bytes，append方式写入

没有rw 这种mode

# 6 others

C standard library`fopen`主要有是以下6种（因为读写都是针对bytes，是python等其他语言又加入了**b**用以区分str和bytes）：`r、r+、w、w+、a、a+`

每种mode可以结合下面的图记忆：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190420173104.png)

python等语言使用buffer减少实际的文件IO交互次数来提高效率，
所以f.write()可能并不会写入到文件中，需要调用f.flush()（f.close释放fd，包含了flush)

# 7 references

> [https://superuser.com/questions/387042/how-to-check-all-timestamps-of-a-file](https://superuser.com/questions/387042/how-to-check-all-timestamps-of-a-file)  
[https://blog.csdn.net/HeatDeath/article/details/79526065](https://blog.csdn.net/HeatDeath/article/details/79526065)  
[https://stackoverflow.com/questions/1466000/python-open-built-in-function-difference-between-modes-a-a-w-w-and-r](https://stackoverflow.com/questions/1466000/python-open-built-in-function-difference-between-modes-a-a-w-w-and-r)