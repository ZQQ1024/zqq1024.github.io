---
title: "Rabin"
weight: 5
bookToc: true
---

## 介绍

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

def rabin_encrypt(msg, n):
    return pow(msg, 2, n)

def rabin_decrypt(ciphertext, p, q):
    n = p * q
    mp = pow(ciphertext, (p+1)//4, p)
    mq = pow(ciphertext, (q+1)//4, q)
    yp, yq, gcd = libnum.xgcd(p, q)
    r1 = (yp*p*mq + yq*q*mp) % n
    r2 = n - r1
    r3 = (yp*p*mq - yq*q*mp) % n
    r4 = n - r3
    return r1, r2, r3, r4

def generate_prime(bits):
    while True:
        p = libnum.generate_prime(bits)
        if p % 4 == 3:
            return p

# 生成两个满足条件的素数
p = generate_prime(32)
q = generate_prime(32)
while p == q:
    q = generate_prime(32)

print(p, q)

n = p*q
# 加密和解密
msg = 123456789
ciphertext = rabin_encrypt(msg, p*q)
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

## 题目

有以下2个文件：
- [flag](/data/rsa/rabbin/flag)
- [pubkey.pem](/data/rsa/rabbin/pubkey.pem)

## 思路

提取公钥信息，发现 e=2
```bash
openssl rsa -pubin -in pubkey.pem -text -noout
Public-Key: (256 bit)
Modulus:
    00:c2:63:6a:e5:c3:d8:e4:3f:fb:97:ab:09:02:8f:
    1a:ac:6c:0b:f6:cd:3d:70:eb:ca:28:1b:ff:e9:7f:
    be:30:dd
Exponent: 2 (0x2)
```
yafu分解
```bash
starting SIQS on c77: 87924348264132406875276140514499937145050893665602592992418171647042491658461

***factors found***

P39 = 275127860351348928173285174381581152299
P39 = 319576316814478949870590164193048041239

ans = 1
```
使用rabbin算法解密

答案
```python
import libnum
# from Crypto.Util.number import bytes_to_long, long_to_bytes

def rabin_decrypt(ciphertext, p, q):
    n = p * q
    mp = pow(ciphertext, (p+1)//4, p)
    mq = pow(ciphertext, (q+1)//4, q)
    yp, yq, gcd = libnum.xgcd(p, q)
    print(yp, yq)
    r1 = (yp*p*mq + yq*q*mp) % n
    r2 = n - r1
    r3 = (yp*p*mq - yq*q*mp) % n
    r4 = n - r3
    return r1, r2, r3, r4

p = 275127860351348928173285174381581152299
q = 319576316814478949870590164193048041239

with open('flag', 'rb') as f:
    ciphertext = libnum.s2n(f.read())

# 45617141162985597041928941111553655146539175146765096976546144304138540198644
print(ciphertext)

plaintext1, plaintext2, plaintext3, plaintext4 = rabin_decrypt(ciphertext, p, q)
print("解密结果1：", libnum.n2s(plaintext1))
print("解密结果2：", libnum.n2s(plaintext2))
print("解密结果3：", libnum.n2s(plaintext3))
print("解密结果4：", libnum.n2s(plaintext4))
# flag{R0bin_rsa_666666}
```