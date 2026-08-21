---
title: "Python Pydantic BaseModel"
date: 2026-08-21
draft: false
---

`from pydantic import BaseModel, ConfigDict` 来自 **Pydantic** 库，它主要用于：

> **数据校验（validation） + 数据解析（parsing） + 数据结构定义（schema）**

简单理解：

Python 原生：

```python
class User:
    name: str
    age: int
```

只是告诉别人：

"我希望 name 是字符串，age 是整数"

但是 Python 不会检查：

```python
user.age = "abc"
```

也不会自动转换。

Pydantic 会。

---

# 1. BaseModel 是什么？

`BaseModel` 是 Pydantic 最核心的类。

你继承它定义数据模型：

```python
from pydantic import BaseModel


class User(BaseModel):
    name: str
    age: int
```

现在：

```python
user = User(
    name="Alice",
    age=20
)

print(user)
```

结果：

```
name='Alice' age=20
```

---

## 自动校验

例如：

```python
User(
    name="Alice",
    age="20"
)
```

Pydantic 会尝试转换：

```python
age = 20
```

因为：

```python
"20" -> int
```

可以转换。

---

但是：

```python
User(
    name="Alice",
    age="hello"
)
```

报错：

```
ValidationError
```

---

# 2. 它解决什么问题？

主要解决：

## 外部数据不可信

例如：

HTTP API：

```json
{
    "name": "Tom",
    "age": 20
}
```

进入 Python：

```python
data = request.json()
```

你不知道：

```json
{
    "name":123,
    "age":"abc"
}
```

怎么办？

Pydantic：

```python
user = User(**data)
```

自动：

- 检查字段
- 类型转换
- 报错

---

# 3. ConfigDict 是什么？

`ConfigDict` 是 Pydantic v2 的配置方式。

以前：

Pydantic v1：

```python
class User(BaseModel):

    class Config:
        orm_mode=True
```

现在 v2：

```python
class User(BaseModel):

    model_config = ConfigDict(
        from_attributes=True
    )
```

作用：

> 控制 BaseModel 的行为。

---

# 4. 常见 ConfigDict 配置

## (1) 是否允许额外字段

默认：

```python
class User(BaseModel):
    name:str
```

输入：

```python
User(
    name="Tom",
    age=20
)
```

报错或者忽略（根据版本配置）。


允许：

```python
class User(BaseModel):

    model_config = ConfigDict(
        extra="allow"
    )

    name:str
```

现在：

```python
user.age
```

存在。

---

## (2) ORM对象转换

比如数据库：

```python
class UserTable:
    name="Tom"
    age=20
```

转Pydantic：

```python
User.model_validate(
    UserTable()
)
```

需要：

```python
ConfigDict(
    from_attributes=True
)
```

以前叫：

```python
orm_mode=True
```

---

## (3) 字段别名

例如：

API返回：

```json
{
    "user_name":"Tom"
}
```

但是你的变量：

```python
username
```

可以：

```python
class User(BaseModel):

    username:str = Field(
        alias="user_name"
    )

    model_config = ConfigDict(
        populate_by_name=True
    )
```

---

# 5. 在 FastAPI 中非常常见

FastAPI 基本就是：

```
HTTP JSON
      |
      v
 Pydantic Model
      |
      v
 Python对象
```

例如：

```python
from fastapi import FastAPI
from pydantic import BaseModel


class User(BaseModel):
    name:str
    age:int


app=FastAPI()


@app.post("/user")
async def create_user(
    user:User
):
    return user
```

请求：

```json
{
"name":"Bob",
"age":30
}
```

FastAPI自动：

1. JSON解析
2. Pydantic校验
3. 创建User对象
4. 调用函数

---

# 6. 为什么 Agent 框架大量使用 Pydantic？

你最近看的：

- OpenAI Agents
- tool calling
- harness
- SEC event pipeline

里面大量需要定义结构：

例如工具输出：

```python
class SearchResult(BaseModel):

    title:str

    url:str

    score:float
```

LLM输出：

```json
{
"title":"SEC Filing",
"url":"xxx",
"score":0.98
}
```

进入系统：

```python
result = SearchResult(**json)
```

保证：

```text
LLM
 |
JSON
 |
Pydantic
 |
可靠对象
 |
业务逻辑
```

---

# 7. 和 dataclass 的区别

Python 自带：

```python
from dataclasses import dataclass


@dataclass
class User:
    name:str
    age:int
```

也能定义结构。

但是：

| |dataclass|Pydantic|
|-|-|-|
|类型提示|✅|✅|
|运行时校验|❌|✅|
|JSON转换|一般|强|
|API Schema|❌|✅|
|FastAPI支持|一般|原生|
|配置复杂对象|弱|强|

---

# 8. 一句话总结

```python
from pydantic import BaseModel, ConfigDict
```

表示：

- `BaseModel`：定义一个**带校验的数据结构**
- `ConfigDict`：控制这个数据结构的行为

在现代 Python 项目里，它经常作为：

```
外部输入
   ↓
Pydantic Model
   ↓
内部可靠对象
   ↓
业务逻辑
```

的一层安全边界。

你最近看的 Agent / SEC pipeline 里，`schemas.py` 一类文件基本就是干这个事情的：定义系统内部的数据契约。
