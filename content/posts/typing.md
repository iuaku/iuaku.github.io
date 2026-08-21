---
title: "Python Typing"
date: 2026-08-21
draft: false
---

`typing` 不是一个普通的第三方库，它是 **Python 标准库**，主要作用是：

> 提供**类型提示（Type Hints）相关的工具**，帮助开发者、IDE、静态检查工具理解代码结构。

`Any` 只是其中一个工具。

---

## 1. typing 解决什么问题？

Python 本身：

```python
def add(a, b):
    return a + b
```

别人不知道：

- a 是 int？
- b 是 float？
- 返回什么？

typing 可以写：

```python
def add(a: int, b: int) -> int:
    return a + b
```

注意：

**Python 运行时不会强制检查。**

它主要给：

- IDE（PyCharm、VSCode）
- 类型检查器（mypy、pyright）
- 开发者

使用。

---

# 2. typing 常用功能

## (1) 基础类型

Python 自带：

```python
str
int
float
bool
```

例如：

```python
name: str
age: int
```

---

# 3. Any

你已经看到：

```python
from typing import Any
```

表示：

```python
任何类型
```

例如：

```python
data: Any
```

---

# 4. Optional

表示：

> 这个值可以为空 None

例如：

```python
from typing import Optional


def get_user(
    id: int
) -> Optional[str]:
    ...
```

表示：

返回：

```text
str
或者
None
```

等价于：

Python 3.10：

```python
def get_user(id:int) -> str | None:
```

---

# 5. Union

多个可能类型：

```python
from typing import Union


value: Union[int, str]
```

表示：

```python
value = 123
```

或者：

```python
value = "hello"
```

现在推荐：

```python
value: int | str
```

---

# 6. List / Dict / Tuple

以前：

```python
from typing import List, Dict


names: List[str]

config: Dict[str, Any]
```

现在：

```python
names: list[str]

config: dict[str, Any]
```

---

例如：

```python
users: list[str]
```

表示：

```python
[
"Alice",
"Bob"
]
```

而不是：

```python
[
1,
True
]
```

---

# 7. Type

表示类型本身。

例如：

```python
from typing import Type


class Model:
    pass


def create(
    cls: Type[Model]
):
    return cls()
```

表示：

传入的是：

```text
Model这个类
```

不是：

```text
Model对象
```

---

# 8. Callable

表示函数类型。

例如：

```python
from typing import Callable


def run(
    func: Callable[[int], str]
):
    ...
```

表示：

传入函数：

输入：

```text
int
```

输出：

```text
str
```

例如：

```python
def f(x:int)->str:
    return str(x)
```

合法。

---

# 9. Literal

限制固定值。

例如：

```python
from typing import Literal


mode: Literal[
    "train",
    "test"
]
```

表示：

只能：

```python
mode="train"
```

或者：

```python
mode="test"
```

不能：

```python
mode="abc"
```

---

# 10. TypedDict

定义字典结构。

例如：

普通：

```python
data: dict[str, Any]
```

太宽泛。


TypedDict：

```python
from typing import TypedDict


class UserDict(TypedDict):
    name: str
    age: int
```

要求：

```python
{
"name":"Tom",
"age":20
}
```

---

# 11. Generic 泛型

比如：

```python
from typing import Generic, TypeVar


T = TypeVar("T")


class Box(Generic[T]):
    value:T
```

可以：

```python
Box[int]

Box[str]
```

类似 C++：

```cpp
template<typename T>
```

---

# 12. Protocol

定义接口。

类似：

Go interface。

例如：

```python
from typing import Protocol


class Database(Protocol):

    def save(self, data):
        ...
```

任何实现：

```python
save()
```

的对象都满足。

---

# 13. 常见 typing 组成图

```
typing

├── Any
│
├── Optional
│
├── Union
│
├── Literal
│
├── Callable
│
├── Type
│
├── Generic
│
├── TypeVar
│
├── Protocol
│
├── TypedDict
│
├── Final
│
└── ClassVar
```

---

# 14. 在工程里的典型用途

你最近看的项目基本都会用：

## Agent

```python
class ToolResult(BaseModel):
    output: Any
    metadata: dict[str, Any]
```

---

## SEC pipeline

```python
facts: list[dict[str, Any]]
```

---

## ML代码

例如：

```python
def forward(
    x: torch.Tensor
) -> torch.Tensor:
```

---

## 配置

```python
Config = dict[str, Any]
```

---

# 15. 一个重要点

很多初学者误解：

```python
x: int
```

是不是限制：

```python
x="hello"
```

不能？

不是。

Python：

```python
x: int = "hello"
```

运行仍然可以。

真正检查的是：

- mypy
- pyright
- IDE

例如：

```bash
pyright main.py
```

才会提示错误。

---

所以总结：

`typing` 是 Python 的**类型描述系统**。

它提供：

- `Any`：任意类型
- `Optional`：可能为空
- `Union`：多类型
- `List/Dict`：容器类型
- `Callable`：函数类型
- `TypedDict`：结构化字典
- `Generic`：泛型
- `Protocol`：接口

现代大型 Python 项目（FastAPI、Pydantic、Agent 框架、ML 工程）几乎都会大量使用。你现在看到的 `BaseModel + typing` 组合，基本就是现代 Python 后端和 AI 工程的标准写法。

这个问题非常关键。很多人刚接触现代 Python 工程时都会觉得：

> `typing` 已经能定义类型了，为什么还需要 Pydantic？

答案：

**有交集，但职责完全不同。**

简单说：

- `typing`：描述“应该是什么类型”（静态提示）
- `Pydantic`：检查“实际来的数据是不是符合要求”（运行时验证）

---

## 1. typing 是说明书

例如：

```python
from typing import Any


def process(data: dict[str, int]):
    ...
```

意思：

> 我希望 data 是一个字典，key 是 str，value 是 int。


但是 Python 不会阻止：

```python
process({
    "age": "hello"
})
```

程序仍然运行。

因为：

```python
dict[str, int]
```

只是一个**类型注解**。

---

## 2. Pydantic 是检查员

例如：

```python
from pydantic import BaseModel


class User(BaseModel):
    name: str
    age: int
```

然后：

```python
user = User(
    name="Tom",
    age="20"
)
```

Pydantic：

发现：

```text
age 是字符串
```

尝试转换：

```python
age = 20
```

得到：

```python
User(
    name="Tom",
    age=20
)
```

---

如果：

```python
User(
    name="Tom",
    age="abc"
)
```

运行时报：

```
ValidationError
```

---

# 3. 对比一下

| |typing|Pydantic|
|-|-|-|
|作用|类型描述|数据验证|
|运行时执行|❌|✅|
|修改/转换数据|❌|✅|
|生成JSON Schema|❌|✅|
|API输入校验|❌|✅|
|IDE提示|✅|部分支持|
|mypy检查|✅|配合使用|

---

# 4. 一个实际例子

假设 API 收到：

```json
{
    "name": "Alice",
    "age": "18"
}
```

---

## 只有 typing

```python
def create_user(
    user: dict[str, int]
):
    ...
```

Python：

```python
user["age"]
```

还是：

```python
"18"
```

不会转换。

---

## Pydantic

```python
class User(BaseModel):
    name: str
    age: int


user = User(**json_data)
```

结果：

```python
user.age
```

是：

```python
18
```

---

# 5. 为什么 Pydantic 里面也大量使用 typing？

因为 Pydantic 的模型定义依赖类型：

例如：

```python
class Event(BaseModel):
    ticker: str
    date: str
    metadata: dict[str, Any]
```

这里：

```python
str
dict[str, Any]
```

来自 typing/Python 类型系统。

Pydantic读取这些类型：

```text
类型注解
    |
    v
Pydantic
    |
    v
生成验证规则
```

---

# 6. 类比

可以类比：

## typing

像设计图：

```
User:
    name: string
    age: int
```

告诉工程师：

应该这样造。

---

## Pydantic

像质检：

```
收到一个User数据

检查:
name是不是字符串?
age是不是数字?

不符合:
拒绝
```

---

# 7. 在 FastAPI 里面两者一起用

典型：

```python
from pydantic import BaseModel


class User(BaseModel):
    name: str
    age: int


@app.post("/user")
async def create_user(
    user: User
):
    return user
```

这里：

```python
name: str
age: int
```

看起来像 typing。

但实际上：

FastAPI调用：

```
HTTP JSON
    |
    v
Pydantic验证
    |
    v
User对象
    |
    v
函数
```

---

# 8. 在 Agent 项目里的关系

比如：

LLM 返回：

```json
{
  "tool": "search",
  "args": {
     "query": "SEC filing"
  }
}
```

---

typing：

```python
class ToolCall:
    tool: str
    args: dict[str, Any]
```

只是描述。

---

Pydantic：

```python
class ToolCall(BaseModel):
    tool: str
    args: dict[str, Any]
```

负责：

- LLM 输出是否符合格式
- 缺字段报错
- 自动转换

---

所以现代 AI 工程常见组合：

```
typing
  ↓
定义数据结构

Pydantic
  ↓
验证外部输入

业务逻辑
```

---

一句话：

> `typing` 是给人和工具看的“类型声明”；Pydantic 是给程序运行时使用的“数据验证框架”。两者不是替代关系，而是经常配合使用。你看到的 `BaseModel + typing.Any` 正是现在 Python AI 工程最常见的数据建模方式。
