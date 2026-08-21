---
title: "Python Error Tracking Imports"
date: 2026-08-21
draft: false
---

这几个也是 Python 标准库，主要涉及：

- **错误追踪**
- **多线程并发**
- **超时控制**
- **路径处理**
- **URL 拼接**

在你之前看的 SEC 抓取 pipeline 里，这些非常常见。

---

# 1. `traceback`

```python
import traceback
```

作用：

> 获取和打印完整异常调用栈（stack trace）。

---

普通异常：

```python
try:
    1 / 0
except Exception as e:
    print(e)
```

输出：

```
division by zero
```

问题：

你不知道：

- 哪个文件？
- 哪一行？
- 调用链是什么？

---

使用 traceback：

```python
import traceback


try:
    1 / 0

except Exception:
    traceback.print_exc()
```

输出：

```
Traceback (most recent call last):
  File "test.py", line 5, in <module>
    1 / 0
ZeroDivisionError: division by zero
```

包含：

```
文件
 ↓
函数
 ↓
行号
 ↓
错误类型
```

---

## 常见用途

工程日志：

```python
try:
    collect_sec()
except Exception:
    logger.error(
        traceback.format_exc()
    )
```

保存完整错误。

---

例如你的 SEC pipeline：

```
下载 filing

解析 XBRL

生成 json
```

如果失败：

```text
Traceback:
collect.py line 100
parser.py line 50
json error
```

方便定位。

---

# 2. ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor
```

作用：

> 创建线程池，执行多个任务。

---

比如：

顺序下载：

```python
download(url1)
download(url2)
download(url3)
```

假设：

每个 2 秒：

```
2+2+2=6秒
```

---

线程池：

```python
from concurrent.futures import ThreadPoolExecutor


def download(url):
    ...


with ThreadPoolExecutor(max_workers=3) as pool:

    pool.map(
        download,
        urls
    )
```

执行：

```
线程1 download(url1)
线程2 download(url2)
线程3 download(url3)
```

大约：

```
2秒
```

---

## 为什么叫 Executor？

因为：

你提交任务：

```python
submit()
```

它负责：

- 创建线程
- 调度任务
- 返回结果

---

例如：

```python
executor.submit(func, arg)
```

返回：

```
Future对象
```

代表：

未来会产生结果。

---

# 3. FuturesTimeoutError

```python
from concurrent.futures import TimeoutError as FuturesTimeoutError
```

这里有两个知识点。

---

## TimeoutError

表示：

> 等待任务超过时间。

例如：

```python
future.result(timeout=5)
```

如果：

5秒没完成：

抛异常：

```
TimeoutError
```

---

为什么改名字？

因为 Python 有多个 TimeoutError：

例如：

```python
asyncio.TimeoutError
```

系统：

```python
TimeoutError
```

所以：

```python
as FuturesTimeoutError
```

避免冲突。


例如：

```python
try:
    result = future.result(timeout=10)

except FuturesTimeoutError:
    print("任务超时")
```

---

# 4. Path

```python
from pathlib import Path
```

前面讲过。

作用：

> 面向对象地操作文件路径。


例如：

以前：

```python
os.path.join(
    "data",
    "a.json"
)
```

现在：

```python
Path("data") / "a.json"
```

---

常用：

## 判断存在

```python
p.exists()
```

---

## 创建目录

```python
p.mkdir(
    parents=True,
    exist_ok=True
)
```

---

## 遍历文件

```python
for f in Path("data").glob("*.json"):
    print(f)
```

---

# 5. urljoin

```python
from urllib.parse import urljoin
```

作用：

> 拼接 URL。

---

例如：

基础地址：

```python
base = "https://www.sec.gov/"
```

文件：

```python
doc = "Archives/file.htm"
```

直接：

```python
base + doc
```

得到：

```
https://www.sec.gov/Archives/file.htm
```

简单情况可以。

---

但是 URL 有复杂情况：

```python
base = "https://example.com/a/b/"
```

相对路径：

```
../file.html
```

应该：

```
https://example.com/a/file.html
```

自己拼很麻烦。

---

用：

```python
urljoin(
    base,
    "../file.html"
)
```

自动处理：

```
https://example.com/a/file.html
```

---

## SEC中特别常见

SEC filing：

index:

```
https://www.sec.gov/Archives/edgar/data/123/
```

里面：

```html
<a href="doc.htm">
```

需要：

```python
urljoin(
    index_url,
    "doc.htm"
)
```

得到：

```
https://www.sec.gov/Archives/edgar/data/123/doc.htm
```

---

# 综合看这个 import 组合

```python
import traceback

from concurrent.futures import ThreadPoolExecutor
from concurrent.futures import TimeoutError as FuturesTimeoutError

from pathlib import Path

from urllib.parse import urljoin
```

基本说明这个模块负责：

```
网络任务

    |
    |
ThreadPoolExecutor
    |
    | 多线程下载
    |
Future
    |
    | 超时
    |
FuturesTimeoutError


文件保存

    |
    Path


URL处理

    |
    urljoin


异常记录

    |
    traceback
```

---

放到 SEC 抓取场景：

```text
获取100家公司filing

        |
        v

ThreadPoolExecutor
并发请求


        |
        v

urljoin
拼完整SEC URL


        |
        v

Path
保存文件


        |
        v

异常

        |
        v

traceback记录详细错误


        |
        v

TimeoutError
处理慢请求
```

这是一个典型的**网络爬虫 / 数据采集系统基础组件组合**。
