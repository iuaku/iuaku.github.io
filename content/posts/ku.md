---
title: "Python Data Pipeline Imports"
date: 2026-08-21
draft: false
---

这组 import 很典型，是一个**数据处理 / 文件处理 / SEC抓取 / Agent pipeline** 类项目常见的基础依赖。

我逐个解释它们的作用，以及在工程里通常怎么用。

---

# 1. hashlib

```python
import hashlib
```

作用：

> 生成哈希值（hash），用于唯一标识、校验、去重。

常见算法：

- MD5
- SHA1
- SHA256


例子：

```python
import hashlib


text = "hello"

hash_value = hashlib.sha256(
    text.encode()
).hexdigest()

print(hash_value)
```

输出：

```text
2cf24dba5fb0...
```

---

## 常见用途

### 文件去重

例如 SEC filing：

```python
file_hash = hashlib.sha256(
    file_bytes
).hexdigest()
```

得到：

```
abc123xxxx
```

判断：

两个文件 hash 一样：

```
文件A
 |
SHA256
 |
abc123


文件B
 |
SHA256
 |
abc123
```

说明内容一样。

---

### 生成 ID

例如：

```python
event_id = hashlib.md5(
    text.encode()
).hexdigest()
```

生成：

```
evt_a83f91...
```

你之前 SEC event pipeline 里面：

```
event_id
accession hash
document fingerprint
```

经常这么做。

---

# 2. html

```python
import html
```

处理 HTML 转义。

例如：

网页：

```html
Tom & Jerry
```

特殊字符：

```
&
<
>
"
```

需要转义。


---

## 转义

```python
html.escape(
    '<div>Hello</div>'
)
```

结果：

```text
&lt;div&gt;Hello&lt;/div&gt;
```

---

## 反转义

```python
html.unescape(
    "&lt;div&gt;"
)
```

得到：

```html
<div>
```

---

常用于：

- 网页抓取
- SEC HTML filing解析
- 文本清洗

---

# 3. json

```python
import json
```

处理 JSON。

Python:

```python
dict
```

和：

```json
{
"name":"Tom"
}
```

转换。


---

## dict → JSON

```python
data = {
    "name":"Tom"
}


json.dumps(data)
```

结果：

```json
{"name": "Tom"}
```

---

## JSON → dict

```python
text = '{"name":"Tom"}'


data = json.loads(text)
```

得到：

```python
{
"name":"Tom"
}
```

---

## 文件保存

```python
with open(
    "data.json",
    "w"
) as f:
    json.dump(
        data,
        f
    )
```

---

SEC pipeline 基本必用：

```text
filing
 |
metadata.json
 |
json.dump()
```

---

# 4. os

```python
import os
```

操作系统接口。

主要：

- 环境变量
- 文件路径
- 系统信息


---

## 环境变量

例如：

```python
os.environ["API_KEY"]
```

读取：

```bash
export API_KEY=xxx
```

---

## 判断文件

```python
os.path.exists(
    "file.txt"
)
```

---

## 创建目录

```python
os.makedirs(
    "data"
)
```

---

现在很多功能被：

```python
pathlib
```

替代。

---

# 5. re

```python
import re
```

正则表达式。

用于文本匹配。

---

例如：

提取 accession：

```python
text = """
0000123456-25-000001
"""


re.findall(
    r"\d{10}-\d{2}-\d{6}",
    text
)
```

结果：

```
0000123456-25-000001
```

---

常用于：

- SEC 文本解析
- HTML 清洗
- 文件名匹配
- 日期提取

---

# 6. shutil

```python
import shutil
```

高级文件操作。

类似：

Linux：

```
cp
mv
rm
```

---

复制：

```python
shutil.copy(
    "a.txt",
    "b.txt"
)
```

---

移动：

```python
shutil.move(
    "a.txt",
    "folder/"
)
```

---

删除目录：

```python
shutil.rmtree(
    "tmp"
)
```

相当于：

```bash
rm -rf tmp
```

---

# 7. tempfile

```python
import tempfile
```

创建临时文件/目录。


例如：

```python
with tempfile.TemporaryDirectory() as d:

    print(d)
```

自动：

创建：

```
/tmp/tmpxxxx
```

退出：

自动删除。

---

用途：

下载文件处理中间缓存：

例如：

```
SEC url
 |
下载
 |
temp目录
 |
解析
 |
删除
```

---

# 8. unicodedata

```python
import unicodedata
```

处理 Unicode 字符。


比如：

看起来一样：

```
é
```

可能有两种编码：

```
é

e + ́
```

Unicode标准化：

```python
unicodedata.normalize(
    "NFKC",
    text
)
```

---

用途：

文本清洗：

- 中文
- 英文
- 特殊符号
- PDF抽取文本

---

例如：

SEC 文本：

```
MICRON\u00a0TECHNOLOGY
```

里面可能有：

- 不可见空格
- 特殊字符

需要 normalize。

---

# 9. datetime相关

```python
from datetime import (
    date,
    datetime,
    time,
    timezone
)
```

处理日期时间。

---

## date

只有日期：

```python
d = date(2026,8,21)
```

表示：

```
2026-08-21
```

---

## datetime

日期+时间：

```python
dt = datetime.now()
```

例如：

```
2026-08-21 10:30:00
```

---

## time

时间：

```python
t = time(10,30)
```

表示：

```
10:30
```

---

## timezone

时区：

例如：

```python
timezone.utc
```

表示：

```
UTC+0
```

---

金融/SEC特别重要：

SEC：

```
accepted_datetime
filing_date
```

需要处理：

```
UTC
EST
北京时间
```

---

# 10. pathlib.Path

```python
from pathlib import Path
```

现代 Python 文件路径处理。


以前：

```python
os.path.join(
    "data",
    "file.json"
)
```

现在：

```python
Path("data") / "file.json"
```

---

例如：

```python
p = Path("data/file.json")


p.exists()

p.name

p.parent
```

---

遍历：

```python
for f in Path("data").glob("*.json"):
    print(f)
```

---

现代项目基本推荐：

```python
Path
```

替代：

```python
os.path
```

---

# 11. typing.Any

你前面问过：

```python
from typing import Any
```

表示：

任意类型。


例如：

```python
metadata: dict[str, Any]
```

表示：

```python
{
    "name": "MU",
    "facts": [],
    "config": {}
}
```

里面类型不固定。

---

# 总体分类

这几个库可以这样分：

```
Python标准库
│
├── 数据格式
│   ├── json
│   └── html
│
├── 文件系统
│   ├── os
│   ├── shutil
│   ├── tempfile
│   └── pathlib
│
├── 文本处理
│   ├── re
│   └── unicodedata
│
├── 时间
│   └── datetime
│
├── 数据安全/唯一标识
│   └── hashlib
│
└── 类型系统
    └── typing
```

---

结合你之前看的 SEC extraction pipeline，这一组 import 基本对应：

```
SEC Filing

 ↓

requests下载

 ↓

tempfile 临时保存

 ↓

html 清洗

 ↓

re 提取字段

 ↓

unicodedata 文本标准化

 ↓

json 保存metadata

 ↓

hashlib生成event_id

 ↓

Path管理目录

 ↓

datetime处理日期

 ↓

Any描述动态字段
```

属于非常典型的**数据工程 + Agent工具链**基础组合。
