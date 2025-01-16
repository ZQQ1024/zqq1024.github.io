---
weight: 3
bookToc: true
title: "深拷贝和浅拷贝"
---

## 深拷贝和浅拷贝

2者的主要区别在于是否会递归地拷贝子对象，浅拷贝虽然创建了一个新的容器，但容器内部元素的引用仍然是原来的。

{{< tabs "实现方式" >}}
{{< tab "浅拷贝" >}}
```python
import copy

old_list = [1, [2, 3], 4]
new_list = copy.copy(old_list)      # 浅拷贝

old_list[0] = 99
print(new_list)  # [1, [2, 3], 4]   -> 不受影响

old_list[1][0] = 99
print(new_list)  # [1, [99, 3], 4] -> 元素引用相同，受影响
```
{{< /tab >}}
{{< tab "深拷贝" >}}
```python
import copy

old_list = [1, [2, 3], 4]
new_list = copy.deepcopy(old_list)  # 深拷贝

old_list[0] = 99
old_list[1][0] = 99

print(old_list)  # [99, [99, 3], 4]
print(new_list)  # [1, [2, 3], 4]   -> 完全不受影响
```
{{< /tab >}}
{{< /tabs >}}

浅拷贝方式：
- 使用内置的 `copy.copy()` 函数
- 对于可变序列可以使用切片语法（例如 `new_list = old_list[:]`）等
- 使用某些内置方法进行拷贝，如 list()、dict() 等等（这些方法往往只拷贝了一层）

深拷贝方式：
- 主要使用 `copy.deepcopy()` 函数

{{< hint info >}}
copy.deepcopy() 本质为递归调用实现，之所以不会被循环引用困住，核心原因在于它使用了一个字典（memo）来管理已经拷贝过的对象
{{< /hint >}}