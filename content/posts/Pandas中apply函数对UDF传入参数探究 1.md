---
title: Pandas中apply函数对UDF传入参数探究
date: 2024-11-28
slug: blog-post-slug
tags:
  - "#pandas"
categories:
  - 分类
description: Pandas中apply函数对UDF（用户自定义函数）传入参数探究
summary: 阅读pandas源码，发现dataframe.apply()函数返回Series的实现逻辑，并解决遇到的问题。
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

pandas的官方文档提到：在使用UDF（用户自定函数）时，迭代容器时不应改变容器，否则可能导致要访问的数据提前遭到修改和删除。这也与普适的编程思想一致。

但当我对传入的row参数进行添加，并存储到buffer中，所产生的现象却不是前文所能解释的，我需要对apply的传参方式进行探究。

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
这说明，实际上虽然apply占有一整个dataframe，传入时并不是每行新建一个series对象传入，他们的唯一标识都是一样的。apply的行传入可能有某种重用机制。

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

问题出在 `row` 对象上。当 `apply` 方法中使用 `axis=1` 参数时，传入的Series对象**实际上是对同一个内部对象的引用。（每行数据的一个视图）** 
**从始至终只用到一个对象，一个地址，但是每次迭代更新这个对象的值。**

所以，当 `record_id` 函数修改了 `row` 对象后，它实际上是在修改一个共享的对象。因此，当你将 `row` 添加到 `buffer` 列表中时，**实际上是在多次添加同一个对象的引用，里面所有的引用地址都一样。**

当循环结束后，`buffer` 列表中的所有元素都指向了最后一次修改后的 `row` 对象，也就是 'A' 列值被设置为 999 的那行数据。

尽管引用相同，但对象的内容在每次迭代中被更新为当前行的数据。这就是传入的参数中，地址一样，值不一样的原因。


## 源码阅读

查看`df.apply` 的定义,跳转到pandas/core/frame.py：
```python
class DataFrame(NDFrame, OpsMixin):
	def apply(
        self,
        func: AggFuncType,
        axis: Axis = 0,
        raw: bool = False,
        result_type: Literal["expand", "reduce", "broadcast"] | None = None,
        args=(),
        by_row: Literal[False, "compat"] = "compat",
        engine: Literal["python", "numba"] = "python",
        engine_kwargs: dict[str, bool] | None = None,
        **kwargs,
    ):
	    from pandas.core.apply import frame_apply
	
		op = frame_apply(
			self,
			func=func,
			axis=axis,
			raw=raw,
			result_type=result_type,
			by_row=by_row,
			engine=engine,
			engine_kwargs=engine_kwargs,
			args=args,
			kwargs=kwargs,
		)
	    return op.apply().__finalize__(self, method="apply")
```

frame_apply方法实现
```python
def frame_apply(
    obj: DataFrame,
    func: AggFuncType,
    axis: Axis = 0,
    raw: bool = False,
    result_type: str | None = None,
    by_row: Literal[False, "compat"] = "compat",
    engine: str = "python",
    engine_kwargs: dict[str, bool] | None = None,
    args=None,
    kwargs=None,
) -> FrameApply:
    """construct and return a row or column based frame apply object"""
    axis = obj._get_axis_number(axis)
    klass: type[FrameApply]
    if axis == 0:
        klass = FrameRowApply
    elif axis == 1:
        klass = FrameColumnApply

    _, func, _, _ = reconstruct_func(func, **kwargs)
    assert func is not None

    return klass(
        obj,
        func,
        raw=raw,
        result_type=result_type,
        by_row=by_row,
        engine=engine,
        engine_kwargs=engine_kwargs,
        args=args,
        kwargs=kwargs,
    )


```

可以看到，当axis=1时，其返回了一个`FrameColumnApply`对象，看来答案就在这个类的实现中了：
同样在pandas/core/apply.py中，部分实现如下：
```python
class FrameColumnApply(FrameApply):
    axis: AxisInt = 1

    def apply_broadcast(self, target: DataFrame) -> DataFrame:
        result = super().apply_broadcast(target.T)
        return result.T

    @property
    def series_generator(self) -> Generator[Series, None, None]:
        values = self.values
        values = ensure_wrapped_if_datetimelike(values)
        assert len(values) > 0

        # We create one Series object, and will swap out the data inside
        #  of it.  Kids: don't do this at home.
        ser = self.obj._ixs(0, axis=0)
        mgr = ser._mgr

        is_view = mgr.blocks[0].refs.has_reference()  # type: ignore[union-attr]

        if isinstance(ser.dtype, ExtensionDtype):
            # values will be incorrect for this block
            # TODO(EA2D): special case would be unnecessary with 2D EAs
            obj = self.obj
            for i in range(len(obj)):
                yield obj._ixs(i, axis=0)

        else:
            for arr, name in zip(values, self.index):
                # GH#35462 re-pin mgr in case setitem changed it
                ser._mgr = mgr
                mgr.set_values(arr)
                object.__setattr__(ser, "_name", name)
                if not is_view:
                    # In apply_series_generator we store the a shallow copy of the
                    # result, which potentially increases the ref count of this reused
                    # `ser` object (depending on the result of the applied function)
                    # -> if that happened and `ser` is already a copy, then we reset
                    # the refs here to avoid triggering a unnecessary CoW inside the
                    # applied function (https://github.com/pandas-dev/pandas/pull/56212)
                    mgr.blocks[0].refs = BlockValuesRefs(mgr.blocks[0])  # type: ignore[union-attr]
                yield ser

```

这个类通过`series_generator` 方法生成一系列的series对象，在生成 `Series` 对象时，有一个重要的步骤是**重用同一个 `Series` 对象**，并在每次迭代中更新其内部数据

根据注释，他们创建了一个series对象`ser`，并在每次返回时交换这个对象的内部数据。`mgr` 是 `Series` 的数据管理器，通过 `mgr.set_values(arr)` 更新 `Series` 的数据。

**pandas的开发者希望返回的Series是一个视图**，并且，他们通过`is_view` 来确定，返回的`ser` 是否真的是一个引用。当`ser`不是一个引用时，会造成不必要的拷贝(CoW Copy-on-Write)

具体实现为：`is_view = mgr.blocks[0].refs.has_reference()` ，这一行代码检查 `mgr` 的第一个数据块是否有引用。如果有引用，说明 `Series` 对象是一个视图。




## Reference
- [pandas.DataFrame.apply — pandas 2.2.3 文档 --- pandas.DataFrame.apply — pandas 2.2.3 documentation](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.apply.html#pandas.DataFrame.apply)
- [Frequently Asked Questions (FAQ) — pandas 2.2.3 documentation](https://pandas.pydata.org/docs/user_guide/gotchas.html#mutating-with-user-defined-function-udf-methods)
- [CoW: Avoid warning in apply for mixed dtype frame by phofl · Pull Request #56212 · pandas-dev/pandas](https://github.com/pandas-dev/pandas/pull/56212)