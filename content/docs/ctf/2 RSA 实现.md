---
title: "RSA 实现与题目"
weight: 2
bookToc: true
---

## RSA实现
Python实现如下：

{{< tabs "RSA Python实现" >}}
{{< tab "基于libnum的实现" >}}
```python
# encoding: utf-8
# pip install libnum
import libnum
from uuid import uuid1

#生成随机素数
p=libnum.generate_prime(1024)
q=libnum.generate_prime(1024)
e=65537
n=p*q
phi_n=(p-1)*(q-1)

#求逆元
d=libnum.invmod(e,phi_n)

m="flag{"+str(uuid1())+"}"

print(m)

#字符串转数字
m=libnum.s2n(m)

c=pow(m,e,n)
m=pow(c,d,n)

print(c, libnum.n2s(m))
```
{{< /tab >}}
{{< tab "基于gmpy2的实现" >}}
```python
# encoding: utf-8
# pip install gmpy2 pycryptodome        
import gmpy2
from Crypto.Util.number import * #getPrime, bytes_to_long, long_to_bytes
from uuid import uuid1

#生成随机素数
p = getPrime(512)
q = getPrime(512)
e=65537
n=p*q
phi_n=(p-1)*(q-1)

#求逆元
d=gmpy2.invert(e,phi_n)

m="flag{"+str(uuid1())+"}"

print(m)

#字符串转数字
m=bytes_to_long(m)

c=pow(m,e,n)
m=pow(c,d,n)

print(c, long_to_bytes(m))
```
{{< /tab >}}
{{< /tabs >}}

## 素数判定

素数判定或素性检测参看博客中的其他文章

## 题型扩充

### 已知p、q

题目
```
p = 0xb6ce21ea137206f9ac1a0e9a004457b090a7fd1745026ba230bd6e33d134eebf8498d98e061be16bc5cb3dfa1e24adc9fbf4214af86aaf6100a064dd6bb0d737ff3a38ac274487a1bf2a6a736fd03fa783b84399d65c79d64c19090b9adf629e55b40e735320535e09684d830432c53e740ba62da8498022820b89aec87b8941L
q = 0xf77307cf5c1bc4f5e45b6515d2de47ce64e9bfc1b3362391a8a791063dcd6c7711c5f1397780abcfb8eaa1201ab82f87e5b314c57fefbfb4f7b2b99a42f93918fd12cb039b50e1ba38bf86f8efc40fe60cc2e57eafce3f3597d6ed8b939d988a34dcadac6b394bf447ce5024d2083dc12b7f1ccd73073f0af70943cf2f133defL
e = 0x10001
c = 0x9aa88a7c5cd1ada3f7ba0d28161dab5f9f27f4b2320e8308b7205f43654ed01f55a7f463b21193fb89f6e97328f68bbf14656d0f3c4aeafdaf90f29f0e1410adddb71573ae164587cc613754467f49ca714c11489a7b70fe7b8382f9d44e5702edab0482d15d8f6b788125fc3f14d63e9f6ce71da93c4de00b2b918201c45adef2dc207fc264057885754f426e9a26072bd9d3ae41ed4f2ba7ac2105a80bd6b783e413fe8004c0ad75bbf9409dddb2ea005eec76f6106b1884b3ed0d16bf794926e32d872d29786135edd4ecc0ac8d490f79a922e3b910ab3d573e248a05763afa778d8a5ad9a418133e20a76cb54f6b864e0d88c3c772e3b12dd035cc372feaL
```

思路
- p、q题目已经提供，就是简单考查RSA的代码实现

答案
```python
import libnum

p = 0xb6ce21ea137206f9ac1a0e9a004457b090a7fd1745026ba230bd6e33d134eebf8498d98e061be16bc5cb3dfa1e24adc9fbf4214af86aaf6100a064dd6bb0d737ff3a38ac274487a1bf2a6a736fd03fa783b84399d65c79d64c19090b9adf629e55b40e735320535e09684d830432c53e740ba62da8498022820b89aec87b8941
q = 0xf77307cf5c1bc4f5e45b6515d2de47ce64e9bfc1b3362391a8a791063dcd6c7711c5f1397780abcfb8eaa1201ab82f87e5b314c57fefbfb4f7b2b99a42f93918fd12cb039b50e1ba38bf86f8efc40fe60cc2e57eafce3f3597d6ed8b939d988a34dcadac6b394bf447ce5024d2083dc12b7f1ccd73073f0af70943cf2f133def
e = 0x10001
c = 0x9aa88a7c5cd1ada3f7ba0d28161dab5f9f27f4b2320e8308b7205f43654ed01f55a7f463b21193fb89f6e97328f68bbf14656d0f3c4aeafdaf90f29f0e1410adddb71573ae164587cc613754467f49ca714c11489a7b70fe7b8382f9d44e5702edab0482d15d8f6b788125fc3f14d63e9f6ce71da93c4de00b2b918201c45adef2dc207fc264057885754f426e9a26072bd9d3ae41ed4f2ba7ac2105a80bd6b783e413fe8004c0ad75bbf9409dddb2ea005eec76f6106b1884b3ed0d16bf794926e32d872d29786135edd4ecc0ac8d490f79a922e3b910ab3d573e248a05763afa778d8a5ad9a418133e20a76cb54f6b864e0d88c3c772e3b12dd035cc372fea

n = p*q
phi_n = (p-1)*(q-1)
d = libnum.invmod(e, phi_n)

m = pow(c,d,n)
print(libnum.n2s(m))
# flag{we1c0me_t0_Chainer_s_training_r00m}
```

### 分解n

素数分解问题是困难的，但是可以通过计算机进行暴力分解。通常意义上来说，一般认为 2048bit 以上的 n 是安全的

满足以下条件可以快速分解 n:
- 如果 n 的大小小于 256bit，通过本地工具即可爆破成功，可以在几分钟内完成 256bit 的 n 的分解
- 如果 n 在 768bit 或者更高，可以尝试使用一些在线的 n 分解网站([factordb.com](http://www.factordb.com/))，这些网站会存储一些已经分解成功的 n
- 如果在两次公钥的加密过程中使用的 n1 和 n2 具有相同的素因子，那么可以利用欧几里得算法直接将 n1 和 n2 分解。通过欧几里得算法可以直接求出 n1 和 n2 的最大公约数 p

- 在 p，q 的取值差异过大，或者 p，q 的取值过于相近的时候，Fermat 方法与 Pollard rho 方法都可以很快将 n 分解成功。此类分解方法有一个开源项目 [yafu](https://github.com/bbuhrow/yafu) 将其自动化实现了，不论 n 的大小，只要 p 和 q 存在相差过大或者过近时，都可以通过 yafu 很快地分解成功

#### Fermat's factorization method

此方法适用于 |p - q| 相差不大的情况下，p 或 q 近似等于 N 的平方根

N 为奇数，可以用2个 square 的差表示，当找到了这2个 square即找到了 对应的 N 质因数分解
N = a^2 - b^2 = (a + b)(a - b)

p = a + b, q = a - b，p 和 q很接近，python实现代码如下
```python
import math

N = 55

a = math.ceil(math.sqrt(N))

while a < N:
    b = math.sqrt(a*a - N)
    if math.floor(b) == b:
        if (a+b) != 1 or (a-b) != 1:
            print("p is:", (a+b), "q is:", (a-b))
            break
    a += 1
```

> References  
> https://fermatattack.secvuln.info/  
> https://en.wikipedia.org/wiki/Fermat%27s_factorization_method  
> https://math.stackexchange.com/questions/263101/prove-every-odd-integer-is-the-difference-of-two-squares

#### Pollard's rho algorithm

Pollard's rho 算法是一种整数分解算法，分解计算的时间复杂度与被分解合数的最小质因数的平方根成正比，所以此方法适用于 |p - q| 相差较大即存在一个很小的质因数的情况下

理解该算法需要先理解以下前置知识：
- 伪随机数数列
- 生日悖论，Birthday problem
- Floyd判圈算法，又称龟兔赛跑算法

##### 伪随机数数列

使用一个多项式 mod n，`g(x) = (x^2 + 1) mod n` 来生成伪随机数数列，如开始值为 g(2)
```
x1 = g(2), x2 = g(g(2)), x3 = g(g(g(2))), ...
```
因为后一个数的取值取决于前一个数，所以中间一旦某个数 xi 出现重复(repeated)，则后续的数列会出现循环(cycle)

##### 生日悖论

生日悖论是指在不少于 23 个人中至少有两人生日相同的概率大于 50%
```
1 - 两两不相同的概率
1 - 365/365 x 364/365 x 363/365 x (365-22)/365 > 0.5
```
这和我们的直观感觉不符合，因为我们一般会将自己带入，但是50%的概率是针对整体而言

对于算法的作用就是：如果我们不断在某个范围内[1,N]生成随机整数，很快便会生成到重复的数，期望大约在根号N级别

##### Floyd判圈算法

可以在有限状态机、迭代函数或者链表上判断是否存在环，求出该环的起点与长度的算法

可以用我们跑步的例子来解释：
- 判断是否有环：如果两个人同时出发，如果赛道有环，那么快的一方总能追上慢的一方，追上时快的一方肯定比慢的一方多跑了几圈，多跑的长度是圈的长度的倍数
- 求环的长度：设相遇点为B点，让其中一个人停在B不动，另一个一步一步向前走并记录步数，再次相遇时步数即为环的长度
- 确定环的起点：方法是将其中一个指针移到链表起点，另一个指针为B点，两者同时移动，每次移动一步，那么两者相遇的地方就是环的起点

在算法中的用途是判断上述伪随机数数列是否存在环

##### 算法思想

算法用于分解N，N = pq

使用`g(x) = (x^2 + 1) mod n` 来生成伪随机数数列 {xk}

2个数列 {xk} 和 {xk mod p}，2个数列分别在期望根号 N 和根号 p 时出现重复进而开始循环

使用序列中的2个数 xi 和 xj，xi 每次走一步，xj 每次走两步，判断 gcd(xi - xj, n) 是否等于 1，如果不等于1，则说明 {xk mod p} 数列出现了循环，

xi - xj 的差是 p 的倍数，如果 gcd(xi - xj, n) 大于1小于n，则找到了p。如果 gcd(xi - xj, n) 如果是n本身，则换一个伪随机数生成器重新执行算法

python实现代码如下
```python
import math
import random

def pollard_rho(n):
    if n % 2 == 0:
        return 2
    
    x = y = 2
    c = 1

    d = 1
    
    def gcd(a, b):
        while b != 0:
            a, b = b, a % b
        return a
    
    def f(x):
        return (x**2 + c) % n
    
    while d == 1:
        x = f(x)
        y = f(f(y))
        d = gcd(abs(x-y), n)
    
    return d

def factorize(n):
    factors = []
    
    while n > 1:
        factor = pollard_rho(n)
        factors.append(factor)
        n //= factor
        
    return factors

# 示例用法
n = 987654321
factors = factorize(n)
print(f"Factors of {n}: {factors}")
```

题目

思路

答案