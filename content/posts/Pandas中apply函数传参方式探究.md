---
title: pandas中apply函数传参
date: 2024-11-28
slug: blog-post-slug
tags:
  - "#pandas"
categories:
  - 分类
description: 描述
summary: 摘要
cover:
 image: 
draft: false
share: true
---

在使用pandas.apply函数时，遇到这样一个问题：
在`pandas.apply(func,axis=1)` 的函数`func`中，直接对传入的行进行操作，
例如：
```python
df = pd.DataFrame({
    'A': range(5),
    'B': range(5, 10)
})
buffer=[]
def func(row):
    if row['A'] > 2:
        row['A'] = 999
    else:
        row['A'] = 888
    buffer.append(row) # 
    # buffer.append(row.to_dict()) # 正常
    row_id = id(row)
    ids.append(row_id)

df.apply(func, axis=1)
```
会发生以下现象：
1. func中，直接对row进行修改时，执行速度相比 使用`row.to_dict()` 重新赋值慢10倍不止。
2. 当把修改后的row添加到全局变量buffer中时，输出的值都相同。


## apply传入参数方式及其问题
使用`id()`函数查看apply传入参数在python中的唯一id，发现每个传入的row的id都一样，尽管他们的值不相同。

```python
def non_parallel_modifyparam_function(row):
    """
    模拟不可并行化的函数，并且对传入参数进行修改
    使用time.sleep模拟耗时
    """
    # time.sleep(0.00001)  # 模拟I/O延迟
    global buffer
    print(f"modifyparam:{id(row)}")
```
发现输出的id都相同，但是row每次的值不同
为什么？
Pandas 的 `apply` 函数内部实现是这样的：

- 当 `apply` 函数迭代 DataFrame 的行时，它会创建一个视图（view）或副本（copy）来表示当前行。
- 由于每次迭代都使用同一个视图或副本，因此 `id` 值是相同的。
- 这种设计是为了避免不必要的内存分配和释放，从而提高性能。

```python
import pandas as pd

# 创建一个简单的 DataFrame
df = pd.DataFrame({
    'A': range(5),
    'B': range(5, 10)
})

# 定义一个函数来打印每行的 id 并记录下来
ids = []

def record_id(row):
    row_id = id(row)
    ids.append(row_id)
    print(f"id of row: {row_id}")

# 应用函数
df.apply(record_id, axis=1)

# 检查所有 id 是否相同
print(f"All ids are the same: {len(set(ids)) == 1}")
```

输出：
```
id of row: 139715457879392
id of row: 139715457879392
id of row: 139715457879392
id of row: 139715457879392
id of row: 139715457879392
All ids are the same: True
```

### python的id()
`id()` 函数返回的是对象的唯一标识符，通常称为对象的“身份”（identity）。这个标识符是一个整数，它在对象的生命周期内是唯一的且恒定的。对于 CPython 解释器而言，这个标识符实际上是对象在内存中的地址

### 为什么修改传入参数，记录到buffer中后，结果都一样？
```python
import pandas as pd

# 创建一个简单的 DataFrame
df = pd.DataFrame({
    'A': range(5),
    'B': range(5, 10)
})

# 定义一个函数来打印每行的 id 并记录下来
ids = []

# 添加到buffer中
buffer = []
def record_id(row):
    if row['A'] > 2:
        row['A'] = 999
    else:
        row['A'] = 888
    buffer.append(row) # 
    # buffer.append(row.to_dict()) # 正常
    row_id = id(row)
    ids.append(row_id)

# 应用函数
df.apply(record_id, axis=1)

# 检查所有 id 是否相同
print(f"All ids are the same: {len(set(ids)) == 1}")

print(df)

for i in buffer:
    print(str(i))
# print(buffer)
# 输出如下：
# A    999
# B      9
# Name: 4, dtype: int64
# A    999
# B      9
# Name: 4, dtype: int64
# A    999
# B      9
# Name: 4, dtype: int64
# A    999
# B      9
# Name: 4, dtype: int64
# A    999
# B      9
# Name: 4, dtype: int64

```

问题出在 `row` 对象上。当你在 `apply` 方法中使用 `axis=1` 参数时，Pandas 实际上是为每一行创建了一个新的 Series 对象，并将其传递给 `record_id` 函数。
但是，这些 Series 对象并不是原始 DataFrame 中行的独立副本；**它们实际上是对同一个内部对象的引用。（每行数据的一个视图）** 
**从始至终只用到一个对象，一个地址，但是每次迭代更新这个对象的值。**
这意味着，当 `record_id` 函数修改了 `row` 对象后，它实际上是在修改一个共享的对象。因此，当你将 `row` 添加到 `buffer` 列表中时，你实际上是在多次添加同一个对象的引用。

当循环结束后，`buffer` 列表中的所有元素都指向了最后一次修改后的 `row` 对象，也就是 'A' 列值被设置为 999 的那行数据。这就是为什么当你遍历 `buffer` 并打印其内容时，所有的输出都显示相同的值。

**所有加入buffer的row，他们指向的内存地址都是同一个，所以就会保留最后一次对改地址的值的修改，所有的row都一样。**

row的值每次传入都不一样，为什么他们是对同一个内部对象的引用？
- **同一个内部对象**：指的是每次迭代中使用的同一个 Series 对象的引用。
- **不同内容**：尽管引用相同，但对象的内容在每次迭代中被更新为当前行的数据。
- **解决方案**：使用 `.copy()` 方法创建新的 Series 对象，以避免对象引用问题。



