---
title: "Python Collections"
date: 2026-08-21
draft: false
---

`collections` 和 `collections.abc` 都是 Python 标准库的一部分，主要用于处理**集合类数据结构（container）和集合抽象接口（abstract interface）**。

简单理解：

- `collections`：提供**具体的数据结构实现**
- `collections.abc`：提供**描述这些结构行为的抽象接口**

---

# 1. collections 是什么？

Python 原生有：

```python
list
dict
tuple
set
```

但是很多场景需要更高级的数据结构。

`collections` 提供这些。

例如：

```python
from collections import Counter
```

---

## Counter：计数器

统计出现次数：

```python
from collections import Counter


words = [
    "apple",
    "banana",
    "apple"
]


c = Counter(words)

print(c)
```

输出：

```text
Counter({
    'apple': 2,
    'banana': 1
})
```

常用于：

- NLP词频
- 数据统计

---

## defaultdict：带默认值的字典

普通：

```python
d = {}

d["a"].append(1)
```

报错：

```text
KeyError
```

因为：

```python
"a"
```

不存在。

---

defaultdict：

```python
from collections import defaultdict


d = defaultdict(list)

d["a"].append(1)
```

自动：

```python
{
 "a":[1]
}
```

常用于：

分组：

```python
groups = defaultdict(list)

for user in users:
    groups[user.city].append(user)
```

---

## deque：双端队列

普通 list：

```python
list.pop(0)
```

慢。

deque：

```python
from collections import deque


q = deque()

q.append("task1")

q.popleft()
```

两端操作都是 O(1)。

常用于：

- 队列
- BFS
- 消息系统

---

## namedtuple

以前用于简单数据结构：

```python
from collections import namedtuple


User = namedtuple(
    "User",
    ["name", "age"]
)


u = User("Tom",20)
```

现在很多情况被：

```python
@dataclass
```

替代。

---

# 2. collections.abc 是什么？

abc = Abstract Base Class

中文：

> 抽象基类


它不是提供具体实现，而是定义：

> 一个对象应该具有什么行为。

---

例如：

什么叫“可调用”？

不是看它是不是函数。

而是：

```python
obj()
```

能不能执行。

所以：

```python
from collections.abc import Callable
```

定义：

```text
任何具有调用能力的对象
```

---

# 3. 一个简单理解

假设：

## collections

像：

> 具体产品

例如：

```text
ArrayList
HashMap
Queue
```

---

## collections.abc

像：

> 产品标准

例如：

```text
这个东西必须支持：
    __getitem__
    __iter__
    __call__
```

---

# 4. 常见 collections.abc

---

## Callable

你刚问的：

```python
from collections.abc import Callable
```

表示：

可以：

```python
obj()
```

---

例如：

```python
def hello():
    pass


isinstance(
    hello,
    Callable
)
```

True。

---

## Iterable

可迭代：

```python
from collections.abc import Iterable
```

例如：

```python
isinstance(
    [1,2,3],
    Iterable
)
```

True。


因为：

```python
for x in obj:
```

可以。

---

## Iterator

迭代器：

```python
from collections.abc import Iterator
```

例如：

```python
it = iter([1,2,3])
```

---

## Mapping

映射类型：

```python
from collections.abc import Mapping
```

例如：

```python
dict
```

属于 Mapping。


表示：

支持：

```python
obj[key]
```

---

## Sequence

序列：

```python
list
tuple
str
```

都属于。

支持：

```python
obj[0]
len(obj)
```

---

# 5. 为什么不用具体类型？

例如：

你写：

```python
def process(data: dict):
    ...
```

表示：

必须是 dict。


但是：

```python
collections.abc.Mapping
```

表示：

只要行为像字典即可。

例如：

- dict
- OrderedDict
- 自定义映射类

都可以。

---

这叫：

> 面向接口编程

不是看你是什么，而是看你能做什么。

---

# 6. 在现代 Python 代码中的意义

比如：

```python
from collections.abc import Mapping


def load_config(
    config: Mapping
):
    ...
```

比：

```python
config: dict
```

更灵活。


因为：

```python
dict
```

太具体。

但是：

```python
Mapping
```

表示：

只需要：

```text
key -> value
```

这个能力。

---

# 7. 和 typing 的关系

以前：

```python
from typing import Callable
```

现在：

```python
from collections.abc import Callable
```

因为：

这些东西本质是**运行时抽象接口**。

例如：

```python
collections.abc.Callable
```

可以：

```python
isinstance(obj, Callable)
```

进行检查。

---

而：

```python
typing.Callable
```

主要用于：

类型提示。

---

# 8. 放到你现在看的 Agent 代码里

例如：

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

Tool 不关心：

你传入的是：

```python
def search():
    ...
```

还是：

```python
class Search:
    def __call__(self):
        ...
```

只要：

```python
fn()
```

能执行。

---

# 9. 总结

一句话：

### `collections`

提供**现成的数据结构**：

```text
Counter
defaultdict
deque
OrderedDict
```

---

### `collections.abc`

提供**行为接口定义**：

```text
Callable     能调用
Iterable     能遍历
Iterator     是迭代器
Mapping      像字典
Sequence     像列表
```

---

现代 Python 工程中：

- `collections` → 需要一个高级容器时用
- `collections.abc` → 写函数接口、类型约束时用

你最近看的 Agent / FastAPI / SEC pipeline 里，`Callable`、`Mapping`、`Iterable` 这类基本都是为了让代码更模块化，不绑定具体实现。
