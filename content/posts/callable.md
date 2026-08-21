---
title: "Python Callable"
date: 2026-08-21
draft: false
---

`from collections.abc import Callable` 的作用是：

> **导入一个“可调用对象”的抽象类型，用来表示某个变量/参数应该是一个函数（或者像函数一样可以调用的对象）。**

简单说：

```python
Callable = 可以被调用的东西
```

---

## 1. 什么叫“可调用对象”？

Python 中，不只是函数可以调用：

```python
func()
```

能这样执行的对象，都叫 callable。

例如：

## 普通函数

```python
def add(x, y):
    return x + y


add(1, 2)
```

`add` 是 Callable。

---

## lambda

```python
f = lambda x: x * 2

f(3)
```

也是 Callable。

---

## 类

```python
class User:
    pass


User()
```

类本身也可以调用。

因为：

```python
User()
```

会创建对象。

---

## 实现 `__call__` 的对象

例如：

```python
class Model:

    def __call__(self, x):
        return x * 2


model = Model()

model(10)
```

这里：

```python
model()
```

可以运行。

所以：

```python
model
```

也是 Callable。

---

# 2. Callable 用在哪里？

主要用于**类型提示**。

例如：

```python
from collections.abc import Callable


def run(
    func: Callable
):
    return func()
```

意思：

`run` 接受一个可以调用的对象。

例如：

```python
def hello():
    print("hello")


run(hello)
```

合法。

---

# 3. 指定参数和返回值

更常见：

```python
from collections.abc import Callable


def execute(
    func: Callable[[int], str],
    value: int
):
    return func(value)
```

这里：

```python
Callable[[int], str]
```

表示：

输入：

```text
int
```

输出：

```text
str
```

也就是：

等价于：

```python
def func(x: int) -> str:
    ...
```

---

例如：

正确：

```python
def convert(x: int) -> str:
    return str(x)


execute(convert, 10)
```

---

错误：

```python
def bad(x: str) -> int:
    return 1
```

类型检查器会提示：

```text
参数类型不匹配
```

---

# 4. 为什么不是 typing.Callable？

以前很多代码：

```python
from typing import Callable
```

现在：

```python
from collections.abc import Callable
```

更推荐。

原因：

Python 3.9+：

很多类型已经移动到：

```python
collections.abc
```

例如：

以前：

```python
from typing import List
from typing import Dict
from typing import Callable
```

现在：

```python
list
dict
collections.abc.Callable
```

更符合 Python 标准。

---

# 5. Callable 和普通函数类型有什么区别？

比如：

```python
def f(x):
    return x
```

你不能写：

```python
func: function
```

因为 Python 没有一个通用 `function` 类型。

而：

```python
func: Callable
```

表示：

任何可以：

```python
func()
```

的对象。

包括：

- 函数
- lambda
- 类
- 方法
- `__call__`对象

---

# 6. 在 Agent 框架里非常常见

你最近研究 OpenAI Agents / harness，会经常看到：

例如工具注册：

```python
from collections.abc import Callable


class Tool:

    def __init__(
        self,
        fn: Callable
    ):
        self.fn = fn
```

意思：

工具本质上是一个函数：

```python
def search(query):
    ...
```

注册：

```python
Tool(search)
```

---

或者：

Agent loop：

```python
async def run_tool(
    tool: Callable,
    args: dict
):
    return await tool(**args)
```

---

# 7. Callable 和 async Callable

异步里还有：

```python
Callable
```

和：

```python
Awaitable
```

区别：

普通函数：

```python
def f():
    return 1
```

类型：

```python
Callable[[], int]
```

---

异步函数：

```python
async def f():
    return 1
```

调用：

```python
f()
```

返回：

```python
Coroutine
```

所以：

```python
Callable[[], Awaitable[int]]
```

表示：

函数调用后返回一个 awaitable。

例如：

```python
async def fetch() -> int:
    return 1
```

---

# 8. 和你前面问的几个东西放一起

现在你看到的这些：

```python
from __future__ import annotations

from typing import Any

from collections.abc import Callable

from dataclasses import dataclass

from pydantic import BaseModel
```

分别负责：

|工具|作用|
|-|-|
|`__future__.annotations`|延迟解析类型|
|`typing.Any`|任意类型|
|`Callable`|函数类型|
|`dataclass`|定义数据对象|
|`Pydantic`|运行时数据验证|

在 Agent、FastAPI、数据处理 pipeline 里面，这几个组合非常典型。

尤其 `Callable`，在 Agent 代码里通常意味着：

> 这个位置不是传数据，而是传“能力”（函数/工具）。
