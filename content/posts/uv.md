---
title: "uv 配置清华 PyPI 镜像源"
date: 2026-08-21
draft: false
---

# uv 配置清华 PyPI 镜像源

`uv` 默认使用官方 PyPI 源：

```
https://pypi.org/simple
```

在国内网络环境下，可以配置清华 PyPI 镜像：

```
https://pypi.tuna.tsinghua.edu.cn/simple
```

---

## 方法 1：项目级配置（推荐）

适用于单个 uv 项目。

进入项目目录：

```bash
cd /Users/agent
```

修改：

```
pyproject.toml
```

添加：

```toml
[tool.uv]
default-index = "https://pypi.tuna.tsinghua.edu.cn/simple"
```

```
[[tool.uv.index]]
url = "https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple"
default = true
```

例如：

```toml
[project]
name = "agent"
version = "0.1.0"

dependencies = [
    "openai-agents",
    "pydantic"
]

[tool.uv]
default-index = "https://pypi.tuna.tsinghua.edu.cn/simple"
```

之后：

```bash
uv sync
```

或者：

```bash
uv add requests
```

都会默认使用清华源。

---

## 方法 2：全局配置（推荐个人电脑）

如果希望所有 uv 项目默认使用清华源，可以配置环境变量。

### macOS / Linux

编辑：

```bash
vim ~/.zshrc
```

加入：

```bash
export UV_DEFAULT_INDEX="https://pypi.tuna.tsinghua.edu.cn/simple"
```

生效：

```bash
source ~/.zshrc
```

检查：

```bash
echo $UV_DEFAULT_INDEX
```

输出：

```
https://pypi.tuna.tsinghua.edu.cn/simple
```

之后所有 uv 项目都会使用清华源。

---

## 方法 3：临时指定镜像源

只对当前命令生效。

安装依赖：

```bash
uv add requests \
  --default-index https://pypi.tuna.tsinghua.edu.cn/simple
```

同步环境：

```bash
uv sync \
  --default-index https://pypi.tuna.tsinghua.edu.cn/simple
```

---

## 方法 4：使用 uv.toml 配置

创建：

```
uv.toml
```

内容：

```toml
[[index]]
url = "https://pypi.tuna.tsinghua.edu.cn/simple"
default = true
```

---

# 验证配置是否生效

运行：

```bash
uv add requests -v
```

查看日志：

如果看到：

```
https://pypi.tuna.tsinghua.edu.cn/simple
```

说明已经使用清华源。

---

# 常用 uv 工作流

## 第一次拉取项目

```bash
git clone <project>
cd project

uv sync
```

---

## 添加依赖

```bash
uv add package_name
```

例如：

```bash
uv add openai
uv add fastapi
```

会自动：

```
修改 pyproject.toml
        ↓
更新 uv.lock
        ↓
安装依赖
```

---

## 删除依赖

```bash
uv remove package_name
```

例如：

```bash
uv remove requests
```

---

## 更新锁文件

```bash
uv lock
```

---

## 启动项目

```bash
uv run python main.py
```

`uv run` 会自动：

```
读取 pyproject.toml
        ↓
检查 uv.lock
        ↓
同步 .venv
        ↓
执行 Python 程序
```

---

# 注意事项

## 1. uv.lock 不需要手动修改

正常流程：

```
pyproject.toml
        ↓
uv add / uv lock
        ↓
uv.lock
```

不要直接编辑 `uv.lock`。

---

## 2. PyTorch CUDA 特殊情况

普通 Python 包：

```
openai
pydantic
fastapi
numpy
```

可以直接使用清华源。

但是 PyTorch CUDA 版本通常需要官方 wheel 源：

例如：

```bash
uv pip install torch \
--index-url https://download.pytorch.org/whl/cu121
```

否则可能安装 CPU 版本或者找不到对应 CUDA 包。

---

# 推荐配置

个人开发机器：

```
~/.zshrc
```

配置：

```bash
export UV_DEFAULT_INDEX="https://pypi.tuna.tsinghua.edu.cn/simple"
```

项目中：

```
pyproject.toml
```

保持：

```toml
[tool.uv]
default-index = "https://pypi.tuna.tsinghua.edu.cn/simple"
```

这样：

- 本机开发速度快
- 项目可复现
- uv.lock 保持稳定
```


1. export UV_CACHE_DIR=... 是干嘛的？
告诉 uv：
把下载过的 Python 包、wheel、源码包等缓存到这个目录。

默认情况下 uv 会放：
Linux：
~/.cache/uv

2. .venv 是什么？
这里才是真正：
Python 运行时 import 的包

比如：
import requests
import pandas
实际找的是：
.venv/lib/python3.10/site-packages/

3. venv 包含全部依赖吗？
对于 uv 项目：
是的。
正常流程：
uv sync
会：
1. 读取：
pyproject.toml
2. 根据：
uv.lock
确定版本
3. 安装到：
.venv/
最终：
.venv
 └── site-packages
      ├── requests
      ├── pandas
      ├── edgartools
      └── ...
所以：
如果你复制：
项目目录
+
.venv
理论上可以运行。
4. 那为什么还需要 cache？
因为 .venv 和 cache 作用不同。
举例：
你有两个项目：
project_A/.venv
project_B/.venv
两个都需要：
requests==2.32
pandas==2.2
没有 cache：
pip/uv
 ↓
下载两次
 ↓
安装两次
有 cache：
            .uv-cache
               |
      -------------------
      |                 |
 project_A/.venv   project_B/.venv
下载一次，多处复用。
