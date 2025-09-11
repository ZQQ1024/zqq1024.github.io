---
title: "PKCS/OpenSSL"
weight: 2
bookToc: true
---

## 介绍

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

## 题目
```
p=135107209955704896198000222334245302654243745495559567671804820858012561141225874582790552130071789809406337174186164537655634738483338212007882091015146536797144814807921423249530783205417173226847805817505933320625728612032490509116441285849207880363824490795277589839619746571582256964496474979903709862693

q=160171769876064727845900448109595638308600010847305736067559931842453822615214982070630165865716840658372045734198096532958184768278400375005563834778185401064339526040432896616673824937436636299621536578531174689764824667158071306658990307745089364592739535140109411467625983684900575582794083374148143857419

e = 65537
```
和一个[oaep.txt](/data/rsa/oaep/oaep.txt)文件

## 思路
- PKCS#1解密，和 OAEP Padding使用

## 答案
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

# 也可尝试将key导出然后使用openssl解密
# with open("prikey.pem", "wb") as f:
#     data = key.export_key()
#     f.write(data)

with open('oaep.txt', 'rb') as file:
    ciphertext = file.read()

# Decrypt the message using RSA with OAEP padding
cipher = PKCS1_OAEP.new(key)
plaintext = cipher.decrypt(ciphertext)

# plaintext: flag{Simple_OpenSSL_Operation}
print("plaintext:", plaintext.decode())
```

{{< hint info >}}
当使用Crypto包或者openssl解密失败后，可以尝试使用`pow(c,m,d)`进行解密
{{< /hint >}}
