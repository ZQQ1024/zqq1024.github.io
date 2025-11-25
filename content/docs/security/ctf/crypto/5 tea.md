---
title: "TEA"
weight: 5
bookToc: true
---

## TEA

"TEA" 的全称为"Tiny Encryption Algorithm" 是1994年由英国剑桥大学的David j.wheeler发明的。

TEA加密解密过程是：
- 原文以8字节（64位bit）为一组，密钥16字节（128位bit）为一组
- 该算法加密轮次可变，作者建议为32轮
- 加解密过程中内部会将key分成4部分
- 被加密的明文为64位，所以最终加密的结果也是64位

该算法使用了一个神秘常数 \(\delta\) 作为倍数，它来源于黄金比率 \(\delta = [\frac{1+\sqrt{5}}{2}]^{32} \)，对应的数指就是`0×9E3779B9`，以保证每一轮加密都不相同。但 \(\delta\) 的精确值似乎并不重要，可以修改。

{{< hint info >}}
严谨地说，黄金比率就是一个简单的无理数：
\[
    \varphi = \frac{1+\sqrt{5}}{2} ≈ 1.6180339887…
\]
它来自一个非常朴素的问题：把一段线分成两部分(a,b)，让整个长度(a+b)与较长部分(a)之比，与较长部分(a)和较短部分(b)之比相等。

换成方程：

\[
\frac{a+b}{a} = \frac{a}{b}，设\frac{a}{b} = x，就变成：
x = 1 + \frac{1}{x}
\]

这是一种很美妙的自递归结构，解出来就是前面的 \( \varphi \)。
{{< /hint >}}

```python
DELTA = 0x9E3779B9
MASK32 = 0xFFFFFFFF

# 明文： 64 bit
# ┌──────────────32 bit──────────────┐┌──────────────32 bit──────────────┐
# │                v0                ││                v1                │
# └──────────────────────────────────┘└──────────────────────────────────┘

def tea_encrypt_block(v0, v1, k):
    """Encrypt one 64-bit block (v0, v1) using TEA.
       k must be a list/tuple of 4 uint32."""
    sum_ = 0
    for _ in range(32):  # 32 rounds
        # sum += delta;
        # v0 += ((v1 << 4) + k0) ^ (v1 + sum) ^ ((v1 >> 5) + k1);
        # v1 += ((v0 << 4) + k2) ^ (v0 + sum) ^ ((v0 >> 5) + k3);
        sum_ = (sum_ + DELTA) & MASK32
        v0 = (v0 + (((v1 << 4) + k[0]) ^ (v1 + sum_) ^ ((v1 >> 5) + k[1]))) & MASK32
        v1 = (v1 + (((v0 << 4) + k[2]) ^ (v0 + sum_) ^ ((v0 >> 5) + k[3]))) & MASK32
    return v0, v1

def tea_decrypt_block(v0, v1, k):
    """Decrypt one 64-bit block."""
    sum_ = (DELTA * 32) & MASK32 # 经过32轮，固定值是 0xC6EF3720
    for _ in range(32):
        v1 = (v1 - (((v0 << 4) + k[2]) ^ (v0 + sum_) ^ ((v0 >> 5) + k[3]))) & MASK32
        v0 = (v0 - (((v1 << 4) + k[0]) ^ (v1 + sum_) ^ ((v1 >> 5) + k[1]))) & MASK32
        sum_ = (sum_ - DELTA) & MASK32
    return v0, v1

def tea_encrypt(data: bytes, key: bytes) -> bytes:
    assert len(data) == 8
    assert len(key) == 16

    k = struct.unpack(">4I", key)
    v0, v1 = bytes_to_block(data)
    v0, v1 = tea_encrypt_block(v0, v1, k)
    return block_to_bytes(v0, v1)

import struct

def bytes_to_block(b: bytes):
    v0, v1 = struct.unpack(">2I", b)  # big-endian
    return v0, v1

def block_to_bytes(v0, v1):
    return struct.pack(">2I", v0, v1)

def tea_decrypt(data: bytes, key: bytes) -> bytes:
    assert len(data) == 8
    assert len(key) == 16

    k = struct.unpack(">4I", key)
    v0, v1 = bytes_to_block(data)
    v0, v1 = tea_decrypt_block(v0, v1, k)
    return block_to_bytes(v0, v1)

key = b"0123456789abcdef"
plaintext = b"ABCDEFGH"

cipher = tea_encrypt(plaintext, key)
print("cipher:", cipher.hex())

plain2 = tea_decrypt(cipher, key)
print("decrypted:", plain2)
```

从逆向角度来看：
- TEA算法的特征是 `0x9E3779B9` 值以及32轮迭代（当然，这些能魔改）
- 在 32-bit 无符号整数（ \(mod 2^{32}\)） 的情况下，`x -= 0x61c88647`和 `x += 0x9e3779b9`，这两个值是等价的(\(\text{0x9e3779b9} + \text{0x61c88647} = 2^{32} \))

## XTEA

XTEA 是 TEA 的改进版（David Wheeler 和 Roger Needham 1997），它主要修补了TEA的一些结构缺陷，如TEA 存在 related-key attack。

---

TEA 使用 `k[0], k[1], k[2], k[3]` 固定顺序，容易被 related-key attack 利用。XTEA 改为：
```
k[sum & 3]
k[(sum >> 11) & 3]
```
sum每轮会变，使得key每轮也会变化。

---

TEA 的混合方式偏弱：
```
(v << 4) + k0 ^ (v >> 5) + k1
```
XTEA 的扩散性更好：
```
(v << 4 ^ v >> 5) + v
```

---

```python
DELTA = 0x9E3779B9
MASK32 = 0xFFFFFFFF

def xtea_encrypt_block(v0, v1, key):
    """Encrypt one 64-bit block using XTEA."""
    sum_ = 0
    for _ in range(32):
        v0 = (v0 + (((v1 << 4 ^ v1 >> 5) + v1) ^ (sum_ + key[sum_ & 3]))) & MASK32
        sum_ = (sum_ + DELTA) & MASK32
        v1 = (v1 + (((v0 << 4 ^ v0 >> 5) + v0) ^ (sum_ + key[(sum_ >> 11) & 3]))) & MASK32
    return v0, v1

def xtea_decrypt_block(v0, v1, key):
    """Decrypt one 64-bit block using XTEA."""
    sum_ = (DELTA * 32) & MASK32
    for _ in range(32):
        v1 = (v1 - (((v0 << 4 ^ v0 >> 5) + v0) ^ (sum_ + key[(sum_ >> 11) & 3]))) & MASK32
        sum_ = (sum_ - DELTA) & MASK32
        v0 = (v0 - (((v1 << 4 ^ v1 >> 5) + v1) ^ (sum_ + key[sum_ & 3]))) & MASK32
    return v0, v1

import struct

def bytes_to_block(b: bytes):
    v0, v1 = struct.unpack(">2I", b)  # big-endian
    return v0, v1

def block_to_bytes(v0, v1):
    return struct.pack(">2I", v0, v1)

def xtea_encrypt(data: bytes, key: bytes) -> bytes:
    assert len(data) == 8
    assert len(key) == 16
    k = struct.unpack(">4I", key)
    v0, v1 = bytes_to_block(data)
    v0, v1 = xtea_encrypt_block(v0, v1, k)
    return block_to_bytes(v0, v1)

def xtea_decrypt(data: bytes, key: bytes) -> bytes:
    assert len(data) == 8
    assert len(key) == 16
    k = struct.unpack(">4I", key)
    v0, v1 = bytes_to_block(data)
    v0, v1 = xtea_decrypt_block(v0, v1, k)
    return block_to_bytes(v0, v1)

key = b"0123456789abcdef"
plaintext = b"ABCDEFGH"

cipher = xtea_encrypt(plaintext, key)
print("cipher:", cipher.hex())

plain2 = xtea_decrypt(cipher, key)
print("decrypted:", plain2)
```

从逆向角度来看：
- 依然可以修改`0x9E3779B9` 值以及迭代轮数
- v的扩散包括key不固定，较TEA算法有变化

## XXTEA

XXTEA 相较于 TEA/XTEA 并不是改良版，而是相当于重新设计，是一个非平衡Feistel网络分组密码，**在可变长度块上运行，这些块是32位大小的任意倍数（最小64位），使用128位密钥**，且无专利保护。

```python
import struct

DELTA = 0x9E3779B9
MASK32 = 0xffffffff

def _mx(z, y, sum_, key, p, e):
    return (((z >> 5 ^ y << 2) + (y >> 3 ^ z << 4)) ^ 
            ((sum_ ^ y) + (key[(p & 3) ^ e] ^ z))) & MASK32

def xxtea_encrypt_uint32(v, key):
    n = len(v)
    rounds = 6 + 52 // n
    sum_ = 0
    z = v[-1]
    for _ in range(rounds):
        sum_ = (sum_ + DELTA) & MASK32
        e = (sum_ >> 2) & 3
        for p in range(n):
            y = v[(p + 1) % n]
            z = v[p - 1] if p > 0 else v[-1]
            v[p] = (v[p] + _mx(z, y, sum_, key, p, e)) & MASK32
    return v

def xxtea_decrypt_uint32(v, key):
    n = len(v)
    rounds = 6 + 52 // n
    sum_ = (rounds * DELTA) & MASK32
    y = v[0]
    for _ in range(rounds):
        e = (sum_ >> 2) & 3
        for p in reversed(range(n)):
            z = v[p - 1] if p > 0 else v[-1]
            y = v[(p + 1) % n]
            v[p] = (v[p] - _mx(z, y, sum_, key, p, e)) & MASK32
        sum_ = (sum_ - DELTA) & MASK32
    return v

def _to_uint32_list(data: bytes, include_length=False):
    n = (len(data) + 3) // 4
    result = list(struct.unpack('<%dI' % n, data.ljust(n * 4, b'\0')))
    if include_length:
        result.append(len(data))
    return result

def _to_bytes(v, include_length=False):
    if include_length:
        data = struct.pack('<%dI' % len(v), *v)
        real_len = v[-1]
        return data[:-4][:real_len]
    else:
        return struct.pack('<%dI' % len(v), *v)

def xxtea_encrypt(data: bytes, key: bytes) -> bytes:
    if not data:
        return data
    key = struct.unpack('<4I', key.ljust(16, b'\0')[:16])
    v = _to_uint32_list(data, include_length=True)
    e = xxtea_encrypt_uint32(v, key)
    return _to_bytes(e, include_length=False)


def xxtea_decrypt(data: bytes, key: bytes) -> bytes:
    if not data:
        return data
    key = struct.unpack('<4I', key.ljust(16, b'\0')[:16])
    v = _to_uint32_list(data, include_length=False)
    d = xxtea_decrypt_uint32(v, key)
    return _to_bytes(d, include_length=True)
```

从逆向角度来看：
- TEA/XTEA 是 一般是固定的32轮，而XXTEA数量是：6 + 52/n
- XXTEA 有一个独特的混合函数，涉及到左移右移，涉及数字`2` `5` `4` `3`