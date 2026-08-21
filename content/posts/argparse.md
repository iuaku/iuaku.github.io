---
title: "Python Argparse"
date: 2026-08-21
draft: false
---

这段代码使用的是 Python 标准库 `argparse`，作用是：

> **把命令行输入转换成 Python 对象（Namespace），方便脚本接收参数。**

比如你运行：

```bash
python test.py --expected-value success --max-turns 10
```

这些命令行参数会被解析成：

```python
arguments.expected_value == "success"
arguments.max_turns == 10
```

---

## 1. 创建 ArgumentParser

```python
parser = argparse.ArgumentParser(
    description="Test whether the local LLM can call tools."
)
```

创建一个参数解析器。

它负责：

- 定义有哪些参数
- 解析命令行输入
- 自动生成 `--help`

例如：

```bash
python test.py --help
```

输出类似：

```
usage: test.py [-h] [--expected-value EXPECTED_VALUE]
               [--max-turns MAX_TURNS]
               [--no-extra-body]

Test whether the local LLM can call tools.

options:
  -h, --help
      show this help message and exit

  --expected-value EXPECTED_VALUE

  --max-turns MAX_TURNS

  --no-extra-body
      Use default model settings instead of kimi_model_settings().
```

---

# 2. parser.add_argument()

基本格式：

```python
parser.add_argument(
    参数名,
    参数配置
)
```

例如：

```python
parser.add_argument("--name")
```

命令：

```bash
python test.py --name Bob
```

得到：

```python
args.name == "Bob"
```

---

# 3. 第一个参数

```python
parser.add_argument(
    "--expected-value",
    default="ok"
)
```

定义：

```bash
--expected-value
```

这个参数。


例如：

运行：

```bash
python test.py
```

没有提供：

那么：

```python
arguments.expected_value
```

默认：

```python
"ok"
```


---

如果：

```bash
python test.py --expected-value success
```

那么：

```python
arguments.expected_value
```

变成：

```python
"success"
```

---

注意：

命令行：

```bash
--expected-value
```

Python：

```python
expected_value
```

因为 argparse 自动把：

```text
-
```

转换成：

```text
_
```

---

# 4. type=int

第二个：

```python
parser.add_argument(
    "--max-turns",
    type=int,
    default=5
)
```

定义：

```bash
--max-turns
```

类型：

```python
int
```

默认：

```python
5
```

---

例如：

运行：

```bash
python test.py
```

结果：

```python
arguments.max_turns
```

是：

```python
5
```

类型：

```python
int
```

---

运行：

```bash
python test.py --max-turns 20
```

输入：

实际上是字符串：

```python
"20"
```

argparse转换：

```python
int("20")
```

得到：

```python
20
```

---

如果：

```bash
python test.py --max-turns abc
```

报错：

```
argument --max-turns: invalid int value: 'abc'
```

---

# 5. action="store_true"

第三个：

```python
parser.add_argument(
    "--no-extra-body",
    action="store_true",
    help="Use default model settings instead of kimi_model_settings()."
)
```

这是特殊参数。

它表示：

> 一个开关(flag)，出现就是 True，不出现就是 False。


例如：

不写：

```bash
python test.py
```

结果：

```python
arguments.no_extra_body
```

是：

```python
False
```


写：

```bash
python test.py --no-extra-body
```

结果：

```python
arguments.no_extra_body
```

是：

```python
True
```

---

类似：

Linux命令：

```bash
ls -a
```

里面：

```bash
-a
```

就是flag。

---

# 6. parse_args()

最后：

```python
arguments = parser.parse_args()
```

真正执行解析。

它读取：

```python
sys.argv
```

例如：

你运行：

```bash
python test.py \
    --expected-value hello \
    --max-turns 10 \
    --no-extra-body
```

Python内部：

```python
sys.argv
```

实际上：

```python
[
"test.py",
"--expected-value",
"hello",
"--max-turns",
"10",
"--no-extra-body"
]
```

---

argparse处理后：

```python
arguments
```

是：

```python
Namespace(
    expected_value='hello',
    max_turns=10,
    no_extra_body=True
)
```

---

可以打印：

```python
print(arguments)
```

输出：

```
Namespace(
 expected_value='hello',
 max_turns=10,
 no_extra_body=True
)
```

---

# 7. 完整例子

文件：

```python
# test.py

import argparse


parser = argparse.ArgumentParser(
    description="Demo"
)


parser.add_argument(
    "--name",
    default="Alice"
)


parser.add_argument(
    "--age",
    type=int,
    default=18
)


parser.add_argument(
    "--debug",
    action="store_true"
)


args = parser.parse_args()


print(args)
print(args.name)
print(args.age)
print(args.debug)
```

---

运行：

```bash
python test.py
```

输出：

```
Namespace(
 name='Alice',
 age=18,
 debug=False
)

Alice
18
False
```

---

运行：

```bash
python test.py \
--name Bob \
--age 30 \
--debug
```

输出：

```
Namespace(
 name='Bob',
 age=30,
 debug=True
)

Bob
30
True
```

---

# 8. 常见参数类型总结

|写法|作用|例子|
|-|-|-|
|`default`|默认值|`default=5`|
|`type`|类型转换|`type=int`|
|`action="store_true"`|开关|`--debug`|
|`action="store_false"`|反向开关|`--no-cache`|
|`choices`|限制范围|`choices=["a","b"]`|
|`required=True`|必须提供|必须输入|
|`help`|帮助说明|`--help`显示|

---

你这个代码属于典型的 **CLI 测试脚本设计**：

```text
命令行
 |
 | argparse
 v
Namespace对象
 |
 v
测试逻辑
```

在 Agent、LLM 测试、数据处理 pipeline 中非常常见，比如：

```bash
python eval.py \
  --model qwen \
  --max-turns 10 \
  --no-extra-body
```

然后代码里面：

```python
args.model
args.max_turns
args.no_extra_body
```

控制运行行为。


不完全是一个东西，但**开关（flag）通常最终表现为 bool 值**。

可以这样理解：

- **bool** 是一种数据类型：`True / False`
- **开关（flag）** 是命令行参数的一种设计方式：通过“有没有写这个参数”来控制一个 bool 值

---

## 1. 普通 bool

Python 里面：

```python
enabled = True
debug = False
```

这是变量：

```text
变量名        值
debug       False
enabled     True
```

你直接赋值。

---

## 2. 命令行开关 flag

例如：

```python
parser.add_argument(
    "--debug",
    action="store_true"
)
```

这里定义了一个开关：

运行：

```bash
python app.py
```

没有写：

```bash
--debug
```

得到：

```python
args.debug == False
```

---

运行：

```bash
python app.py --debug
```

得到：

```python
args.debug == True
```

也就是说：

```text
命令行是否出现 --debug

        ↓

转换成

        ↓

Python bool
```

---

## 3. 为什么不用普通 bool 参数？

比如：

```python
parser.add_argument(
    "--debug",
    type=bool
)
```

看起来合理，但实际上很坑。

例如：

```bash
python app.py --debug False
```

命令行所有输入本质都是字符串：

```python
"False"
```

Python：

```python
bool("False")
```

结果：

```python
True
```

因为：

```python
bool("任何非空字符串") == True
```

所以：

```python
bool("False") 
# True
```

这不是你想要的。

---

所以 argparse 提供：

```python
action="store_true"
```

专门解决：

> 有这个参数 = True，没有 = False

---

## 4. 还有一种反向开关

比如：

```python
parser.add_argument(
    "--no-cache",
    action="store_true"
)
```

意思：

默认：

```python
args.no_cache=False
```

运行：

```bash
python app.py --no-cache
```

变：

```python
args.no_cache=True
```

代码：

```python
if args.no_cache:
    disable_cache()
```

---

## 5. 常见设计模式

很多 Linux 命令都是这种：

### 启用某功能

```bash
python train.py --debug
```

对应：

```python
debug=True
```

---

### 禁用某功能

```bash
python train.py --no-cuda
```

对应：

```python
no_cuda=True
```

代码：

```python
if args.no_cuda:
    use_cpu()
```

---

## 6. 和配置文件里的 bool 区别

配置文件：

```yaml
debug: true
```

这里直接存 bool。


命令行：

```bash
--debug
```

这里是一个**动作**：

```text
出现参数
    ↓
设置变量
    ↓
True
```

---

所以一句话：

> bool 是值，flag 是产生 bool 值的一种输入方式。

在 Python CLI 中：

```python
action="store_true"
```

就是把：

```text
命令行有没有出现某个参数
```

映射成：

```python
True / False
```

这种设计在训练脚本、Agent 配置、部署脚本里非常常见。你之前看到的：

```bash
--no-extra-body
--enable-xbrl
--debug
--dry-run
```

基本都是这种 flag。

不是。**在 Python 代码内部，bool 类型非常常用；只是命令行参数里很少直接用 `type=bool`。**

要区分几个场景。

---

## 1. Python 内部：经常用 bool

例如：

```python
use_cache = True
enable_xbrl = False
debug = True
```

这是最自然的写法。

比如你的 SEC pipeline：

```python
ENABLE_XBRL_FACTS = True
DOWNLOAD_REPORT_EXHIBIT_TEXT = False
```

这些就是标准 bool。

---

函数参数也大量使用：

```python
def train(
    use_amp=True,
    shuffle=True,
    verbose=False
):
    ...
```

这里 bool 很常见。

---

# 2. 命令行参数：不推荐 `type=bool`

比如：

```python
parser.add_argument(
    "--debug",
    type=bool
)
```

不推荐。

原因：

命令：

```bash
python test.py --debug False
```

传入的是字符串：

```python
"False"
```

然后：

```python
bool("False")
```

结果：

```python
True
```

因为：

```python
bool("abc") == True
bool("False") == True
bool("0") == True
```

这很反直觉。

---

所以命令行通常：

## 开启型

```python
parser.add_argument(
    "--debug",
    action="store_true"
)
```

使用：

```bash
python test.py --debug
```

得到：

```python
True
```

---

## 禁止型

```python
parser.add_argument(
    "--no-cache",
    action="store_true"
)
```

使用：

```bash
python test.py --no-cache
```

得到：

```python
True
```

---

# 3. 会不会用 int 代替 bool？

有一些场景会。

因为 Python 里面：

```python
bool 是 int 的子类
```

例如：

```python
True == 1
```

结果：

```python
True
```

```python
False == 0
```

结果：

```python
True
```

因为：

```python
True
```

底层相当于：

```python
1
```

---

所以：

```python
x = 1

if x:
    print("yes")
```

可以。

---

但是：

**不建议用 int 表示 bool。**

不要写：

```python
use_cache = 1
```

表示：

```python
True
```

因为语义差。

---

# 4. 什么情况用 int？

如果不是二选一，而是数量，就用 int。

例如：

错误：

```python
enable_workers = True
```

如果实际意思是：

"开启几个 worker"

应该：

```python
num_workers = 8
```

---

比如训练：

```bash
python train.py --num-gpus 4
```

对应：

```python
args.num_gpus == 4
```

不是：

```python
args.use_gpu=True
```

---

# 5. 有些系统用 0/1 是为什么？

主要是：

## 数据库

例如：

SQL：

```sql
is_active INTEGER
```

存：

```
0
1
```

因为早期数据库没有 bool。

---

## 神经网络 tensor

PyTorch：

```python
mask = torch.tensor([1,0,1])
```

或者：

```python
mask = torch.bool
```

两种都有。

例如 attention mask：

```python
mask = torch.tensor(
    [True, False, True]
)
```

更推荐。

---

# 6. 配置文件区别

例如 YAML：

推荐：

```yaml
enable_xbrl: true
download_text: false
```

不要：

```yaml
enable_xbrl: 1
download_text: 0
```

---

# 7. 总结

|场景|推荐|
|-|-|
|Python变量|`bool` ✅|
|函数参数|`bool` ✅|
|配置文件 YAML/TOML|`true/false` ✅|
|argparse命令行开关|`action="store_true"` ✅|
|数量|`int` ✅|
|数据库老设计|可能 `0/1`|
|不要|`type=bool` ❌|

所以实际工程里：

- **业务逻辑大量使用 bool**
- **CLI 参数用 flag 表达 bool**
- **int 只表示数量，不应该代替 bool**

你现在看的 Agent/SEC/训练脚本里，像：

```bash
--debug
--dry-run
--disable-cache
--enable-xbrl
```

这些几乎都是 flag，而不是 `--debug True/False`。
