---
title: "基本概念"
weight: 1
bookToc: false
---

## 余数

若整数m, n, k, r满足 \( m = kn + r，且 n ≠ 0，0 ≤ r < n \)，则称 r 是 m 对 n 的余数(Reminder)

举例来说，\( 13 = 2 * 5 + 3 \)，其中 3 就是 13 除以 5 的余数

## 质数

质数(Prime)也叫素数，定义为只能被1和其自身整数的正整数，如 2，3，5，7，11，13 等

## 分解质因数

每个合数都可以写成几个质数相乘的形式，其中每个质数都是这个合数的因数，把一个合数用质因数相乘的形式表示出来，叫做分解质因数。

并且如果把质因数按照由小到大的顺序排列在一起，相同的因数的积写成幂的形式，那么这种分解方法是**唯一**的

如 60 = 2 * 2 * 3 * 5 = 2^2 * 3 * 5 

## 最大公约数

GCD(greatest common divisor)表示不全为0的2个或多个整数的最大公有约数

12、16的公约数有1、2、4，其中最大的一个是4，记作

```
gcd(12, 16) = 4
```

{{< hint info >}}
**互质和公约数的关系**

两数互质(Relatively Prime)的充分必要条件是两数的最大公约数为 1，如3和10互斥，gcd(3, 10) = 1
{{< /hint >}}

## 同余

同余（`congruence modulo`，符号 \( \equiv \)）是数论中的一种等价关系。两个整数a, b，若它们除以正整数 m 所得的余数相等，则称a, b对于模 m 同余

记作
\[
a \equiv b \mod n
\]

即存在非零整数 k，使得a = b + nk

## 模逆元

对整数 a 和 b，若
\[
ab \equiv 1 \mod n
\]

则称 a 和 b 关于模 n 互为模倒数，也叫数论倒数、模反元素或模逆元（`modular multiplicative inverse`）还有如倒数、-1记法

{{< hint info >}}
**模逆元和互质的关系**

根据裴蜀定理，已知整数 a, b，gcd(a, b) = d 则一定存在整数x, y，使ax + by = d成立

对 ax + by = d 两边同模n

根据分配律：\( (ax + by) \mod b = [(ax \mod b) + (by \mod b)] \mod b = ax \mod b = d \mod b \) 即

\( ax \equiv d \mod b \)

当a, b互质时，d = 1，此时 x 就是 a 关于模 b 的逆元，也相当于扩展欧几里德可以用于计算模逆

a, b互质的充分必要条件是d = 1，也就是说，**a 关于模 b 的模逆元存在的充分必要条件是 a 和 b 互质**，\( a? \equiv 1 \mod b \)
{{< /hint >}}

## 模运算法则

模运算定义：
对于任意实数x, y，可以有
\[
x \mod y = x - y[x/y], y \ne 0。
\]

在一些场合，可以使用符号 `%` 表示，它是一个二元运算

\( x \mod y \)的值都介于 0 和模之间：
\[
0 <= x \mod y <= y, y > 0
\]

其中y = 0时，为了避免用零做除数，为了完整起见，我们定义 \( x \mod 0 = x \)

模运算类似基本四则可以进行加减乘等，但是除法例外。其规则如下：
\[
\begin{aligned}
(a + b) \mod p = (a \mod p + b \mod p) \mod p \\
(a - b) \mod p = (a \mod p - b \mod p) \mod p \\
(a * b) \mod p = (a \mod p * b \mod p) \mod p \\
a ^ b \mod p = [(a \mod p) ^ b] \mod p \\
\end{aligned}
\]

模运算满足结合律、交换律、分配律，具体如下：
{{< tabs "模运算规律" >}}
{{< tab "结合律" >}}
\[
\begin{aligned}
((a + b) \mod p + c) \mod p = (a + (b + c) \mod p) \mod p \\
((a * b) \mod p * c) \mod p = (a * (b * c) \mod p) \mod p
\end{aligned}
\]
{{< /tab >}}
{{< tab "交换律" >}}
\[
\begin{aligned}
(a + b) \mod p = (b + a) \mod p \\
(a * b) \mod p = (b * a) \mod p
\end{aligned}
\]
{{< /tab >}}
{{< tab "分配律" >}}
\[
\begin{aligned}
(a + b) \mod p = (a \mod p + b \mod p) \mod p \\
((a + b) \mod p * c) \mod p = ((a * c) \mod p + (b * c) \mod p ) \mod p
\end{aligned}
\]
{{< /tab >}}
{{< /tabs >}}