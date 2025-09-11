---
title: "RSA 实现"
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