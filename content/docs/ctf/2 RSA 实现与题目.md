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

### PKCS/OpenSSL

基于n e d p q等信息生成PKCS#1 RSA密钥（或使用OpenSSL），对文件进行解密
```
PyCryptodome RSA.construct 构建密钥

# 私钥生成公钥
openssl rsa -in prikey.pem -pubout -out pubkey.pem

# 提取公钥信息
openssl rsa -pubin -in pubkey.pem -text -noout  # pubin表示in的文件是一个公钥，默认是私钥，所以in不能少

Public-Key: (64 bit)
Modulus: 13826123222358393307 (0xbfe041d1197381db)
Exponent: 65537 (0x10001)

提取私钥信息
openssl rsa  -in prikey.pem -text -noout

Private-Key: (64 bit, 2 primes)
modulus: 13826123222358393307 (0xbfe041d1197381db)
publicExponent: 65537 (0x10001)
privateExponent: 9793706120266356337 (0x87ea3bd3bd0b9671)
prime1: 4184799299 (0xf96ef843)
prime2: 3303891593 (0xc4ed6289)
exponent1: 1771565249 (0x6997f0c1)
exponent2: 1637452169 (0x61998989)
coefficient: 568112984 (0x21dcb758)

公钥加密（需注意n的长度和数据长度）
openssl rsautl -encrypt -pubin -inkey pubkey.pem -in flag -out flag.enc

私钥解密
openssl rsautl -decrypt -inkey prikey.pem -in flag.enc -out flag
```

题目
```
p=135107209955704896198000222334245302654243745495559567671804820858012561141225874582790552130071789809406337174186164537655634738483338212007882091015146536797144814807921423249530783205417173226847805817505933320625728612032490509116441285849207880363824490795277589839619746571582256964496474979903709862693

q=160171769876064727845900448109595638308600010847305736067559931842453822615214982070630165865716840658372045734198096532958184768278400375005563834778185401064339526040432896616673824937436636299621536578531174689764824667158071306658990307745089364592739535140109411467625983684900575582794083374148143857419

e = 65537
```
和一个[oaep.txt](/data/oaep.txt)文件

思路
- PKCS#1解密，和 OAEP Padding使用

答案
```python
from Crypto.Cipher import PKCS1_OAEP
from Crypto.PublicKey import RSA

# RSA private key components (p, q, and exponent)
p = 135107209955704896198000222334245302654243745495559567671804820858012561141225874582790552130071789809406337174186164537655634738483338212007882091015146536797144814807921423249530783205417173226847805817505933320625728612032490509116441285849207880363824490795277589839619746571582256964496474979903709862693
q = 160171769876064727845900448109595638308600010847305736067559931842453822615214982070630165865716840658372045734198096532958184768278400375005563834778185401064339526040432896616673824937436636299621536578531174689764824667158071306658990307745089364592739535140109411467625983684900575582794083374148143857419
e = 65537

# Modulus calculation
n = p * q

# Function to calculate the private exponent d
def calculate_private_exponent(p, q, e):
    from Crypto.Util.number import inverse
    phi = (p - 1) * (q - 1)
    return inverse(e, phi)

# Calculate the private exponent
d = calculate_private_exponent(p, q, e)

# Generate an RSA private key using p, q, and exponent d
key = RSA.construct((n, e, d, p, q))

with open('oaep.txt', 'rb') as file:
    ciphertext = file.read()

# Decrypt the message using RSA with OAEP padding
cipher = PKCS1_OAEP.new(key)
plaintext = cipher.decrypt(ciphertext)

# plaintext: flag{Simple_OpenSSL_Operation}
print("plaintext:", plaintext.decode())
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
```
n = 0x888f5dc4bcc2384b948bbb07c8a4571c49d9f17dcf3f16f8801eec59580f10e264461753af699bc0d19a1311e50a1bfd892091751e005ed83038aa4ca8f74db64f7b033e6cc59e924c1c8f3b66208504d6fc7cc753060da5e56b1186e2e99332ee2c16775e33470fb49e80dfb821f2a6676360541bef23a1e8b2404d58a230605e360b3b
e = 0x10001
c = 0x2f3bc742059df8c4e1c951bf84a005d52a67300434c5b16d7e47f622831e2bce7402d2fc59ee16a73e0640f85ad93462948f66b0f3ed79dcf8403ad59433e7d838246d4fae570edf6feefaa34049072d51aa7e78b50846ffd8a79c613672534e9511990236cc566753ec0c4996cb0a4cc0f7429b8a3bf6312db5b8683a9ab4cbdc12fc40
```

思路
- 使用yafu工具分解n

答案
先试用 yafu 进行分解得到 p 和 q
```
root@ctf-crypto:~# yafu
YAFU Version 2.11

>> factor(0x888f5dc4bcc2384b948bbb07c8a4571c49d9f17dcf3f16f8801eec59580f10e264461753af699bc0d19a1311e50a1bfd892091751e005ed83038aa4ca8f74db64f7b033e6cc59e924c1c8f3b66208504d6fc7cc753060da5e56b1186e2e99332ee2c16775e33470fb49e80dfb821f2a6676360541bef23a1e8b2404d58a230605e360b3b)
fac: factoring 411868939986456363373990173141778791902817011526575418612339832876100462958316233209828010224992434354613342590841746178700615197354061668001966691103301956982861535695157774527412186140547807215622989999341143026241272142696297783130879727847037082845435612822592859982970300687091786386580932104341934289646232406843
fac: using pretesting plan: normal
fac: no tune info: using qs/gnfs crossover of 95 digits
fac: no tune info: using qs/snfs crossover of 95 digits
div: primes less than 10000
fmt: 1000000 iterations
rho: x^2 + 3, starting 1000 iterations on C318 
Total factoring time = 2.4719 seconds


***factors found***

P10 = 3056773583
P309 = 134739760339801511355180464192652894927487016144687553265975680501063942542687249207959414056802398882355606444287223539088212277760338675369259984573512966329132770346621310266476542675796805070604076206236025635740150317406187091852684245029683993721374558484135152898882632317976226892360228861720679915221

ans = 1
```
解题代码如下
```python
import libnum

p = 3056773583
q = 134739760339801511355180464192652894927487016144687553265975680501063942542687249207959414056802398882355606444287223539088212277760338675369259984573512966329132770346621310266476542675796805070604076206236025635740150317406187091852684245029683993721374558484135152898882632317976226892360228861720679915221

e = 0x10001

c = 0x2f3bc742059df8c4e1c951bf84a005d52a67300434c5b16d7e47f622831e2bce7402d2fc59ee16a73e0640f85ad93462948f66b0f3ed79dcf8403ad59433e7d838246d4fae570edf6feefaa34049072d51aa7e78b50846ffd8a79c613672534e9511990236cc566753ec0c4996cb0a4cc0f7429b8a3bf6312db5b8683a9ab4cbdc12fc40

n = p*q

phi_n = (p-1)*(q-1)

d = libnum.invmod(e, phi_n)

m = pow(c,d,n)

print(libnum.n2s(m))

# flag{w3_May_pRim3_f@ct0riZe}
```

### Rabbin

Rabin cryptosystem是一种基于模合数平方根困难性问题的非对称加密算法，该算法Rabin于1978年发布的Rabin signature scheme签名算法演变而来。有以下特点：
- 算法的安全性被证明了等价于大数的质数分解，在这点上RSA没有证明
- 相当于 e=2 的RSA
- trapdoor function的每个输出都可以由四个可能输入中的任何一个生成

为什么算法的安全性被证明了等价于大数的质数分解，可以参看以下链接：
> https://en.wikipedia.org/wiki/Quadratic_residuosity_problem  
> https://crypto.stackexchange.com/questions/27121/why-is-rabin-encryption-equivalent-to-factoring  
> https://crypto.stackexchange.com/questions/9332/quadratic-residuosity-problem-reduction-to-integer-factorization  
> https://crypto.stackexchange.com/questions/24720/is-computing-roots-moduli-a-composite-n-a-hard-problem-without-knowing-the-fac

#### 算法描述

**Key 生成**
1. 选择2个大的质数 p 和 q
2. 计算 n = p * q

n 是公钥，(p, q) 是私钥

**加密**

c = m^2 (mod n) (m < n)

**解密**

由 c = m^2 (mod n) 显然可得，
```
m^2 ≡ c mod p，同时模平方根记做 ±mp
m^2 ≡ c mod q，同时模平方根记做 ±mq
```
{{< hint info >}}
求解可以参看 Cipolla's algorithm。Cipolla's algorithm 可用于解二次剩余方程 x ≡ n mod p，p为奇素数时的通解

同时，如果找到了 m 满足 m^2 ≡ c mod p 和 m^2 ≡ c mod q，即也找到了 m 满足 m^2 ≡ c mod p*q
> https://math.stackexchange.com/questions/2005579/ccrt-constant-case-crt-if-p-q-are-coprime-then-x-equiv-a-pmod/2006919#2006919
{{< /hint >}}

根据等式构造出 r, s 满足：
```
r^2 ≡ m^2 ≡ c mod p
s^2 ≡ m^2 ≡ c mod q
```
解 r, s 的过程相当于求解二次剩余方程 m^2 ≡ c mod (p or q)

对mp和mq两两联立，即以下4组同余方程组分别使用中国剩余定理求解，可得 m 的4个解
```
1:
r ≡ mp mod p
s ≡ mq mod q

2:
r ≡ mp mod p
s ≡ -mq mod q

3:
r ≡ -mp mod p
s ≡ mq mod q

4:
r ≡ -mp mod p
s ≡ -mq mod q
```

{{< hint info >}}
使用中国剩余定理求解过程如下：
1. 根据裴蜀定理 p 和 q互质，存在 yp 和 yq 满足：
yp * p + yq * q = 1
2. 可以使用扩展欧几里得求得 yp 和 yq
3. 4个解分别为：
- r1 = (yp * p * mq + yq * q * mp) mod n
- r2 = n - r1
- r3 = (yp * p * mq - yq * q * mp) mod n
- r4 = n - r3

> https://en.wikipedia.org/wiki/Chinese_remainder_theorem#:~:text=In%20mathematics%2C%20the%20Chinese%20remainder,are%20pairwise%20coprime%20(no%20two
> https://crypto.stackexchange.com/questions/29949/how-to-use-crt-to-compute-4-square-roots-while-decryption-in-rabin-cryptosystem
{{< /hint >}}

**由于常规Rabin加密规定 p ≡ q ≡ 3 (mod 4)**，这个条件保证了mp和mq的唯一性，mp = c^(1/4(p+1)), mq = c^(1/4(q+1))
{{< hint info >}}
**为什么当p ≡ 3 (mod 4) 时，mp = c^(1/4(p+1)) mod p 是 m^2 ≡ c (mod p) 的模平方根？**

由于 m^2 ≡ c (mod p)，所以 c 是模 p 的二次剩余，记做勒让德符号(c/p) = 1

mp^2 ≡ c^1/2(p+1) ≡ c*c1/2(p-1) mod p，根据欧拉准则，得

c * c1/2(p-1) mod p ≡ c * 1 mod p

> https://en.wikipedia.org/wiki/Quadratic_residue
> https://en.wikipedia.org/wiki/Euler%27s_criterion
{{< /hint >}}


{{< tabs "例子和代码" >}}
{{< tab "例子" >}}
加密：
1. 取 p = 7, q = 11，则 n = 77
2. 取 m = 20 作为明文
3. 密文为 c = m^2 mod n，即 400 mod 77 = 15

解密：
1. 计算 mp = c^(1/4(p+1)) mod p，即 15^2 mod 7 = 1，同理 mq = 15^3 mod 11 = 9
2. 使用扩展欧几里得求得 yp 和 yq，满足 yp * p + yq * q = 1，得 yp = -3 和 yq = 2 (-3 * 7 + 2 * 11 = 1)
3. 计算4个明文候选：
- r1 = (-3 * 7 * 9 + 2 * 11 * 1) mod 77 = 64
- r2 = 77 - 64 = 13
- r3 = (-3 * 7 * 9 - 2 * 11 * 1) mod 77 = 20
- r4 = 77 - 20 = 57

r1-r4都是理论满足的，r3是实际想要的

{{< /tab >}}
{{< tab "代码" >}}
```python
import libnum

def rabin_encrypt(msg, p, q):
    n = p * q
    return pow(msg, 2, n)

def rabin_decrypt(ciphertext, p, q):
    n = p * q
    mp = pow(ciphertext, (p+1)//4, p)
    mq = pow(ciphertext, (q+1)//4, q)
    gcd, yp, yq = libnum.xgcd(p, q)
    r1 = (yp*q*mp + yq*p*mq) % n
    r2 = n - r1
    r3 = (yp*q*mp - yq*p*mq) % n
    r4 = n - r3
    return r1, r2, r3, r4

def generate_prime(bits):
    while True:
        p = libnum.generate_prime(bits)
        if p % 4 == 3:
            return p

# 生成两个满足条件的素数
p = generate_prime(512)
q = generate_prime(512)
while p == q:
    q = generate_prime(512)

# 加密和解密
msg = 123456789
ciphertext = rabin_encrypt(msg, p, q)
plaintext1, plaintext2, plaintext3, plaintext4 = rabin_decrypt(ciphertext, p, q)
print("明文：", msg)
print("密文：", ciphertext)
print("解密结果1：", plaintext1)
print("解密结果2：", plaintext2)
print("解密结果3：", plaintext3)
print("解密结果4：", plaintext4)
```
{{< /tab >}}
{{< /tabs >}}