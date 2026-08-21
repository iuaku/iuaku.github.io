---
title: "Python Dataclasses"
date: 2026-08-21
draft: false
---

`from dataclasses import asdict, dataclass` 来自 Python 标准库 `dataclasses`，作用是：

> **快速创建“主要用于存储数据”的类（Data Class），减少样板代码。**

它和你刚刚问的 `typing`、`Pydantic` 有一定关系，但定位不同。

---

# 1. 为什么需要 dataclass？

普通 Python 类：

```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age
```

如果只是存数据，会有很多重复代码。

比如你需要：

- `__init__`
- `__repr__`
- `__eq__`
- 默认值处理

这些都要手写。

---

`dataclass` 自动生成。

---

# 2. dataclass 基本使用

```python
from dataclasses import dataclass


@dataclass
class User:
    name: str
    age: int
```

等价于自动生成：

```python
class User:

    def __init__(
        self,
        name,
        age
    ):
        self.name = name
        self.age = age
```

---

现在：

```python
user = User(
    "Alice",
    20
)
```

得到：

```python
User(name='Alice', age=20)
```

---

# 3. dataclass 自动生成什么？

例如：

```python
@dataclass
class User:
    name: str
    age: int
```

它自动生成：

---

## __init__

不用写：

```python
user = User(
    "Tom",
    18
)
```

---

## __repr__

打印：

```python
print(user)
```

得到：

```text
User(name='Tom', age=18)
```

而不是：

```text
<__main__.User object at xxx>
```

---

## __eq__

可以比较：

```python
User("Tom",18) == User("Tom",18)
```

结果：

```python
True
```

---

# 4. asdict 是干嘛的？

`asdict()`：

> 把 dataclass 对象转换成字典。

例如：

```python
from dataclasses import asdict


user = User(
    "Alice",
    20
)


data = asdict(user)
```

结果：

```python
{
    "name": "Alice",
    "age": 20
}
```

---

所以：

```text
dataclass对象
      |
      | asdict()
      ↓
dict
```

---

# 5. 常见使用场景

## 场景1：配置对象

例如：

```python
@dataclass
class TrainConfig:
    batch_size: int = 32
    lr: float = 1e-4
    epochs: int = 10
```

使用：

```python
config = TrainConfig()
```

得到：

```python
config.batch_size
```

---

比：

```python
config = {
    "batch_size":32,
    "lr":1e-4
}
```

更安全。

---

## 场景2：内部数据结构

例如：

```python
@dataclass
class Event:

    ticker: str
    date: str
    score: float
```

然后：

```python
event = Event(
    "AAPL",
    "2025-01-01",
    0.9
)
```

---

## 场景3：机器学习训练参数

很多项目：

```python
@dataclass
class Args:

    model_name:str
    batch_size:int
    lr:float
```

类似 argparse：

```python
args = Args(...)
```

---

# 6. dataclass 和 Pydantic 区别

这个很重要。

它们都可以：

```python
class User:
    name:str
    age:int
```

但是目的不同。

---

## dataclass

定位：

> Python 内部数据容器


例如：

```python
@dataclass
class Config:
    lr:float
    batch:int
```

特点：

- 快
- 简单
- 无验证


例如：

```python
User(
    "Tom",
    "abc"
)
```

可以：

```python
User(
    name="Tom",
    age="abc"
)
```

不会报错。

---

## Pydantic

定位：

> 外部输入数据验证


例如：

```python
class User(BaseModel):
    name:str
    age:int
```

输入：

```python
age="20"
```

自动：

```python
age=20
```

输入：

```python
age="abc"
```

报错。

---

# 7. dataclass 和 typing 的关系

通常一起：

```python
from dataclasses import dataclass
from typing import Any


@dataclass
class Event:

    name: str

    metadata: dict[str, Any]
```

这里：

- dataclass 提供类结构
- typing 提供类型描述

---

# 8. 在你的项目里怎么判断用哪个？

结合你之前看的：

## SEC pipeline

例如：

```python
@dataclass
class Filing:
    ticker: str
    form: str
    date: str
```

内部处理：

✅ dataclass 很合适。


---

但是：

用户输入：

```json
{
 "ticker":"AAPL",
 "date":123
}
```

API入口：

✅ Pydantic。


---

## Agent

工具返回：

```json
{
 "tool":"search",
 "args":{}
}
```

外部：

✅ Pydantic


Agent内部状态：

```python
@dataclass
class AgentState:
    messages:list
    step:int
```

✅ dataclass

---

# 9. 总结

|工具|作用|
|-|-|
|`typing`|描述类型|
|`dataclass`|快速创建数据类|
|`asdict`|对象转字典|
|`Pydantic`|运行时数据验证|

关系：

```text
typing
  |
  | 提供类型信息
  ↓

dataclass
  |
  | 创建内部数据结构
  ↓

业务逻辑


外部输入
  |
  ↓
Pydantic
  |
  ↓
内部dataclass
```

现代 Python 工程里经常是：

- **Pydantic 管边界**
- **dataclass 管内部状态**
- **typing 管类型表达**

你最近看的 Agent harness、SEC extraction pipeline 这类代码里，这三个基本都会同时出现。
