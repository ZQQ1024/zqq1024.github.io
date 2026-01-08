---
weight: 3
bookToc: true
title: "推导式"
---

python中 `comprehension` 叫做推导式，可以使用很简短的语句遍历集合并创建一个新的集合。

和数学里面的 [set builder/set compression](https://en.wikipedia.org/wiki/Set-builder_notation) 概念类似，根据一个set附加特定条件构造一个新的set，以下从集合 S 里挑 x，满足条件的，把它变成 f(x)，构造成一个新集合：

\[
\{f(x) ∣ x \in S, 且满足条件\}
\]

### 列表推导式

```python
nums = [1,2,3,4,5,6]

square_even = [x * x for x in nums if x % 2 == 0] # [4, 16, 36]
```

### 集合推导式

```python
nums = [1,2,3,4,5,6]

mod = {x % 3 for x in nums} # {0, 1, 2} 去重了
```

### 字典推导式

```python
nums = [1,2,3,4,5,6]

square_map = {x: x * x for x in nums if x % 2 == 0} # {2: 4, 4: 16, 6: 36}
```

### 生成器表达式/生成器推导式

```python
nums = [1,2,3,4,5,6]

square_gen = (x*x for x in nums if x % 2 == 0)

next(square_gen) # 4

list(square_gen) # [16, 36]，4被消耗掉了
```