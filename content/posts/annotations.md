---
title: "Python Future Annotations"
date: 2026-08-21
draft: false
---

`from __future__ import annotations` 是 Python 的一个**未来特性开关**，作用是：

> **让类型注解（type hints）延迟求值（postponed evaluation），把类型保存成字符串，而不是立即解析。**

它主要解决**类型引用顺序、循环引用、性能**等问题。

---

## 1. 先看没有它的问题

例如：

```python
class User:
    pass


class Group:
    users: list[User]
```

这个没问题，因为 `User` 已经定义。

但是如果：

```python
class Group:
    users: list[User]


class User:
    pass
```

运行：

```text
NameError: name 'User' is not defined
```

为什么？

因为 Python 执行到：

```python
users: list[User]
```

时，会马上尝试寻找：

```python
User
```

但是此时还没有定义。

---

# 2. 加上 future annotations

```python
from __future__ import annotations


class Group:
    users: list[User]


class User:
    pass
```

现在可以。

因为：

```python
list[User]
```

不会立即执行。

Python 保存成：

```python
"list[User]"
```

等以后需要时再解析。

---

# 3. 它改变了什么？

没有：

```python
x: int
```

类似：

```python
x.__annotations__
```

可能：

```python
{
    "x": int
}
```

也就是：

```python
int对象
```

---

有：

```python
from __future__ import annotations
```

可能：

```python
{
    "x": "int"
}
```

字符串。

---

所以叫：

> postponed evaluation of annotations

类型注解延迟解析。

---

# 4. 最常见用途：类自身引用

例如链表：

```python
from __future__ import annotations


class Node:

    value: int

    next: Node | None
```

这里：

```python
next: Node | None
```

引用自己。

如果没有：

```text
Node还没创建
```

会报错。

---

# 5. 解决循环引用

例如两个类：

file:

```
user.py
group.py
```

---

user.py:

```python
from group import Group


class User:
    group: Group
```

---

group.py:

```python
from user import User


class Group:
    users: list[User]
```

形成：

```text
user.py
  ↓
group.py
  ↓
user.py
```

循环导入。

使用：

```python
from __future__ import annotations
```

可以减少很多问题。

---

# 6. 为什么现代项目大量使用？

你最近看的代码：

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Any
```

很常见。

原因：

## (1) 类型引用更自由

例如：

```python
class Event:

    parent: Event | None
```

不用考虑定义顺序。

---

## (2) 减少 import

比如：

```python
def process(
    model: HugeModel
):
    ...
```

如果只是为了类型：

以前：

```python
from models import HugeModel
```

可能导致循环依赖。


现在：

```python
from __future__ import annotations


def process(
    model: HugeModel
):
    ...
```

可以后处理。

---

## (3) Python版本兼容

例如：

Python 3.9：

写：

```python
list[int]
```

可以。

但是：

```python
User | None
```

需要 Python 3.10。

future 可以帮助一些兼容。

---

# 7. 和 typing.TYPE_CHECKING 配合

非常常见：

```python
from __future__ import annotations

from typing import TYPE_CHECKING


if TYPE_CHECKING:
    from models import User


class Event:

    user: User
```

运行时：

```python
models.User
```

不会真的导入。

但是：

IDE / mypy：

知道：

```python
user 是 User
```

---

# 8. 它不是异步

注意：

虽然名字里面有：

```python
future
```

但是：

```python
from __future__ import annotations
```

和：

```python
asyncio.Future
```

没有关系。


这里：

future = Python未来版本特性。

---

# 9. 它是不是必须？

不是。

比如：

```python
class User:
    name: str
```

完全不用。

但是大型项目推荐：

```python
from __future__ import annotations
```

尤其：

- FastAPI
- Pydantic
- Agent框架
- 大型ML项目
- 数据模型很多的项目

---

# 10. 总结一句话

```python
from __future__ import annotations
```

就是：

> 告诉 Python：类型提示先不要急着计算，把它当成字符串保存，之后需要时再解析。

它主要解决：

- 类提前引用
- 自引用类型
- 循环导入
- 类型提示性能

所以你最近看到很多文件开头：

```python
from __future__ import annotations
```

通常意味着：

这个文件大量使用类型注解，并希望类型系统更灵活。
