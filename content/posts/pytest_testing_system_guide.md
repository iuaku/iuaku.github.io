---
title: "Pytest 使用与 Python 测试系统构建"
date: 2026-08-21
draft: false
---

# Pytest 使用与 Python 测试系统构建

## 1. 测试系统是什么

在 Python 项目中，测试系统的目标不是“证明程序绝对正确”，而是通过一组明确的测试用例，持续验证代码是否满足预期行为。

可以把测试系统理解为：

```text
项目代码
   ↓
准备测试输入
   ↓
调用业务代码
   ↓
检查输出和状态
   ↓
PASS / FAIL
```

测试的核心价值主要包括：

- 验证函数是否满足预期行为。
- 防止修改代码后破坏原有功能。
- 快速定位错误发生在哪一层。
- 为重构提供安全保障。
- 将“代码应该满足什么规则”显式记录下来。

例如：

```python
def add(a, b):
    return a + b


def test_add():
    result = add(1, 2)
    assert result == 3
```

这里的测试实际上定义了一条行为契约：

```text
add(1, 2) 必须得到 3
```

如果以后有人修改 `add()` 导致这个行为发生变化，测试就会立即失败。

---

# 2. pytest 是什么

`pytest` 是 Python 中非常常用的测试框架。

安装：

```bash
pip install pytest
```

或者在项目依赖中加入：

```toml
pytest>=8,<9
```

运行：

```bash
pytest
```

pytest 会自动：

```text
寻找测试文件
   ↓
寻找测试函数
   ↓
准备 fixture
   ↓
执行测试函数
   ↓
检查 assert
   ↓
输出 PASS / FAIL
```

因此，pytest 本质上是一个：

> 测试发现 + 测试运行 + fixture 管理 + 失败报告框架。

---

# 3. pytest 如何发现测试

pytest 主要依赖命名约定。

通常测试目录为：

```text
tests/
```

测试文件通常命名为：

```text
test_*.py
```

例如：

```text
test_registry.py
test_scan.py
test_agent.py
test_schemas.py
```

测试函数通常命名为：

```python
def test_xxx():
    ...
```

例如：

```python
def test_add():
    ...
```

pytest 执行：

```bash
pytest
```

以后，会自动扫描这些测试。

一个典型项目结构如下：

```text
project/
├── event_agent/
│   ├── mining/
│   │   ├── registry.py
│   │   ├── scan.py
│   │   └── schemas.py
│   └── agent.py
│
├── tests/
│   ├── test_registry.py
│   ├── test_scan.py
│   ├── test_schemas.py
│   └── test_agent.py
│
└── pyproject.toml
```

---

# 4. pytest 最基本的测试逻辑

pytest 测试通常遵循：

```text
Arrange
准备测试环境和输入

Act
执行被测试代码

Assert
检查结果
```

也可以写成：

```text
Given
给定什么条件

When
执行什么操作

Then
应该得到什么结果
```

例如：

```python
def test_add():
    # Arrange
    a = 1
    b = 2

    # Act
    result = add(a, b)

    # Assert
    assert result == 3
```

对于 Agent 项目：

```python
def test_make_type_example_preserves_event_fields():
    # Arrange
    event = _scan().events[0]

    # Act
    example = make_type_example(event)

    # Assert
    assert example.event_id == event.event_id
    assert example.form == "8-K"
```

逻辑就是：

```text
Given
一个 EventSource

When
执行 make_type_example()

Then
关键字段必须被正确保留
```

---

# 5. assert 的作用

`assert` 本身是 Python 语法。

例如：

```python
x = 10

assert x == 10
```

条件成立时，什么都不会发生。

如果：

```python
assert x == 20
```

Python 会抛出：

```text
AssertionError
```

pytest 会捕获这个错误，并提供更清晰的错误报告。

例如：

```python
def test_value():
    a = 10
    b = 20

    assert a == b
```

pytest 会输出类似：

```text
E       assert 10 == 20
```

因此，测试最重要的部分就是：

```python
assert 实际结果 == 期望结果
```

---

# 6. Fixture 是什么

fixture 是 pytest 非常重要的机制。

它可以理解为：

> 测试依赖注入。

例如很多测试都需要一个事件：

```python
import pytest


@pytest.fixture
def event():
    return EventSource(
        event_id="evt_1",
        form="8-K",
        ...
    )
```

然后测试可以直接声明：

```python
def test_event_id(event):
    assert event.event_id == "evt_1"
```

pytest 会自动：

```text
发现 test_event_id(event)
        ↓
发现它需要 event
        ↓
找到 @pytest.fixture def event()
        ↓
调用 fixture
        ↓
把结果传给测试函数
```

相当于 pytest 自动完成了：

```python
event_obj = event()
test_event_id(event_obj)
```

---

# 7. tmp_path

`tmp_path` 是 pytest 内置 fixture。

例如：

```python
def test_write_file(tmp_path):
    output = tmp_path / "result.txt"

    output.write_text("hello")

    assert output.read_text() == "hello"
```

pytest 会自动创建临时目录。

你不需要手动：

```text
mkdir
删除测试文件
清理目录
```

在  Event Agent 测试中：

```python
def test_build_initial_state_tracks_pending_events(
    tmp_path: Path,
) -> None:
    scan = _scan()

    state = build_initial_state(
        tmp_path,
        tmp_path / "out",
        scan.snapshot,
        scan.events,
    )

    assert state.pending_event_ids == ["evt_1"]
```

这里：

```text
tmp_path
```

模拟项目或 corpus 路径。

而：

```text
tmp_path / "out"
```

模拟 Agent 输出目录。

这样不会污染真实项目目录。

---

# 8. Helper Function 和 Fixture 的区别

例如：

```python
def _scan():
    ...
```

这是普通 helper function。

测试需要显式调用：

```python
def test_xxx():
    scan = _scan()
```

如果改为 fixture：

```python
@pytest.fixture
def scan():
    ...
```

那么可以：

```python
def test_xxx(scan):
    ...
```

pytest 自动传入。

通常：

- 少量测试共用：helper function 就够了。
- 很多测试共用：推荐 fixture。
- 需要复杂初始化和清理：推荐 fixture。

---

# 9. 测试异常

测试不只是测试“正确结果”。

有时候正确行为就是抛异常。

例如：

```python
def divide(a, b):
    return a / b
```

可以测试：

```python
import pytest


def test_divide_by_zero():
    with pytest.raises(ZeroDivisionError):
        divide(1, 0)
```

逻辑：

```text
调用 divide(1, 0)

如果抛 ZeroDivisionError
    PASS

如果没有抛
    FAIL
```

这对 Agent 项目尤其有用，例如测试：

- 非法 JSON。
- 不存在的文件。
- event_id 缺失。
- schema 校验失败。
- 非法状态转换。
- LLM 返回非法结构。

---

# 10. pytest 常用命令

运行所有测试：

```bash
pytest
```

显示详细信息：

```bash
pytest -v
```

显示 `print()`：

```bash
pytest -s
```

详细模式并显示输出：

```bash
pytest -v -s
```

只测试一个文件：

```bash
pytest tests/test_registry.py
```

只运行某一个测试：

```bash
pytest tests/test_registry.py::test_make_type_example_preserves_event_fields
```

根据名字过滤：

```bash
pytest -k registry
```

遇到第一个失败立即停止：

```bash
pytest -x
```

只显示简洁 traceback：

```bash
pytest --tb=short
```

常见开发组合：

```bash
pytest -v --tb=short
```

---

# 11. pytest-asyncio

Agent 项目里经常存在异步函数：

```python
async def call_llm():
    ...
```

或者：

```python
async def run_agent():
    ...
```

这时候通常使用：

```toml
pytest-asyncio>=1,<2
```

测试可以写：

```python
import pytest


@pytest.mark.asyncio
async def test_agent_run():
    result = await run_agent()

    assert result is not None
```

`pytest-asyncio` 主要负责：

```text
pytest
  ↓
创建 asyncio event loop
  ↓
执行 async def 测试
  ↓
await Agent / API / Tool
```

因此：

```text
pytest
```

负责测试框架本身，

而：

```text
pytest-asyncio
```

是 pytest 的异步能力扩展。

---

# 12. Ruff

项目依赖：

```toml
ruff>=0.9.2
```

需要注意：

> Ruff 不是测试框架。

Ruff 主要负责：

```text
静态代码检查
+
代码格式检查
```

例如：

```python
def test():
    x = 123
    return True
```

pytest 可能认为没有问题。

但 Ruff 可能发现：

```text
变量 x 定义后没有使用
```

运行：

```bash
ruff check .
```

自动修复部分问题：

```bash
ruff check . --fix
```

格式化：

```bash
ruff format .
```

检查格式但不修改：

```bash
ruff format --check .
```

因此：

```text
pytest
    动态测试代码运行行为

ruff
    静态检查代码质量
```

两者是互补关系。

---

# 13. pytest + pytest-asyncio + ruff

下面三个库：

```toml
"pytest>=8,<9",
"pytest-asyncio>=1,<2",
"ruff>=0.9.2",
```

已经足够构建一套基础而完整的 Python 测试系统。

整体结构：

```text
                   Python 项目
                       │
          ┌────────────┴────────────┐
          ↓                         ↓
        Ruff                      pytest
          │                         │
   静态代码检查             动态执行测试
                                    │
                      ┌─────────────┴─────────────┐
                      ↓                           ↓
                  普通测试                     异步测试
                                                   │
                                            pytest-asyncio
```

可以把它们理解成：

```text
pytest
=
测试发动机

pytest-asyncio
=
异步测试插件

ruff
=
代码质量检查器
```

---

# 14. 推荐 pyproject.toml

一个基础配置可以是：

```toml
[project.optional-dependencies]
dev = [
    "pytest>=8,<9",
    "pytest-asyncio>=1,<2",
    "ruff>=0.9.2",
]
```

pytest 配置：

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "-v --tb=short"
asyncio_mode = "auto"
```

Ruff：

```toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = [
    "E",
    "F",
    "I",
    "B",
]
```

其中常见规则：

```text
E
pycodestyle error

F
Pyflakes

I
import 排序

B
常见 Python bug / bad practice
```

---

# 15. 推荐测试目录

 Event Agent 可以设计成：

```text
tests/
├── unit/
│   ├── test_schemas.py
│   ├── test_registry.py
│   ├── test_scan.py
│   ├── test_state.py
│   └── test_tools.py
│
├── integration/
│   ├── test_mining_pipeline.py
│   ├── test_agent_workflow.py
│   └── test_registry_flow.py
│
├── fixtures/
│   ├── sample_event.json
│   ├── metadata.json
│   └── sample_filing.txt
│
└── conftest.py
```

其中：

```text
unit/
```

主要测试单个函数或模块。

```text
integration/
```

测试多个模块组合。

```text
fixtures/
```

存放测试数据。

```text
conftest.py
```

存放多个测试文件共享的 fixture。

---

# 16. conftest.py

pytest 会自动发现 `conftest.py`。

例如：

```python
import pytest

from event_agent.mining.schemas import EventSource


@pytest.fixture
def sample_event():
    return EventSource(
        event_id="evt_1",
        ...
    )
```

然后其他测试文件可以直接：

```python
def test_make_type_example(sample_event):
    example = make_type_example(sample_event)

    assert example.event_id == "evt_1"
```

不需要：

```python
from tests.conftest import sample_event
```

pytest 会自动完成 fixture 注册。

---

# 17. 单元测试

单元测试测试一个尽可能小的功能。

例如：

```python
def test_make_type_example(sample_event):
    example = make_type_example(sample_event)

    assert example.event_id == sample_event.event_id
    assert example.form == sample_event.form
```

它只关心：

```text
EventSource
    ↓
make_type_example
    ↓
TypeExample
```

并不关心：

- 文件扫描。
- LLM。
- Agent workflow。
- 网络请求。
- database。

优点是：

```text
快
稳定
错误容易定位
```

---

# 18. 集成测试

集成测试检查多个组件组合之后是否仍然工作。

例如：

```text
测试 filing
   ↓
scan()
   ↓
EventSource
   ↓
build_initial_state()
   ↓
MiningState
```

这时候不只是测试一个函数，而是在验证：

> scan 和 registry 能不能正确连接。

例如：

```python
def test_scan_to_initial_state(tmp_path):
    scan = scan_corpus(tmp_path)

    state = build_initial_state(
        tmp_path,
        tmp_path / "out",
        scan.snapshot,
        scan.events,
    )

    assert len(state.pending_event_ids) == len(scan.events)
```

---

# 19. Agent 项目的测试分层

对于  Event Agent，可以按照下面几层设计。

## 19.1 Schema Test

测试：

```text
EventSource
CorpusSnapshot
TypeExample
MiningState
```

例如：

```python
def test_event_source_requires_event_id():
    ...
```

目标：

> 数据结构本身必须合法。

---

## 19.2 Scan Test

测试：

```text
文件 / JSON
    ↓
scan
    ↓
EventSource
```

重点检查：

- event_count。
- filing_count。
- ticker。
- form。
- accession。
- event_id。
- statement。
- evidence_refs。

---

## 19.3 Registry / State Test

测试：

```text
CorpusScanResult
    ↓
build_initial_state()
    ↓
MiningState
```

例如：

```python
assert state.pending_event_ids == ["evt_1"]
assert state.round_count == 0
assert state.audit_count == 0
```

这里是在测试 Agent 的状态不变量。

---

## 19.4 State Transition Test

Agent 往往不是只初始化一次。

状态可能：

```text
pending
   ↓
processing
   ↓
processed
   ↓
audited
```

应该测试：

```text
pending event 是否正确移除
processed event 是否正确加入
round_count 是否增加
audit_count 是否增加
失败以后是否能够恢复
```

---

## 19.5 Tool Test

如果 Agent 有：

```text
search
merge
cluster
audit
register_type
```

等工具，可以单独测试工具。

例如：

```python
def test_register_event_type():
    ...
```

重点测试：

```text
输入是否合法
输出是否合法
状态是否被正确修改
错误是否被正确处理
```

---

## 19.6 Async Agent Test

对于：

```python
async def agent.run():
```

使用：

```python
@pytest.mark.asyncio
async def test_agent_run():
    ...
```

重点不一定要求自然语言文本完全一致。

更加适合测试：

```text
返回 schema 是否正确
字段是否存在
状态是否正确
工具调用是否发生
结果是否满足约束
```

---

## 19.7 Integration Test

例如：

```text
小型  corpus
    ↓
scan
    ↓
initial state
    ↓
Agent
    ↓
type mining
    ↓
registry
```

检查整个最小流程是否跑通。

---

## 19.8 Error Handling Test

必须专门测试错误输入。

例如：

```text
不存在的 filing
损坏的 JSON
没有 event_id
空 statement
重复 event_id
非法 form
LLM 返回非法 JSON
Tool 调用失败
```

一个健壮系统不能只测试正常路径。

---

# 20. 为什么不要直接测试 LLM 文本完全一致

例如不要过度依赖：

```python
assert result == "Executive Departure"
```

因为 LLM 可能输出：

```text
Executive Departure
Executive Leadership Change
Leadership Transition
```

这些可能语义接近。

Agent 测试更适合验证：

```python
assert result.event_type is not None
assert result.confidence >= 0
assert result.confidence <= 1
```

或者：

```text
schema 是否满足
工具调用是否合法
状态转换是否正确
最终结果是否满足业务约束
```

因此 Agent 项目应该尽量：

> 对确定性部分做严格测试，对 LLM 部分测试结构和约束。

---

# 21. Mock

测试 Agent 时，不应该每个测试都真实调用 LLM API。

否则会产生：

```text
测试慢
测试收费
结果不稳定
依赖网络
API 失败导致测试失败
```

Python 自带：

```python
from unittest.mock import Mock, patch
```

例如：

```python
from unittest.mock import AsyncMock


async def test_agent():
    llm = AsyncMock()

    llm.generate.return_value = {
        "event_type": "Executive Departure"
    }

    result = await run_agent(llm)

    assert result["event_type"] == "Executive Departure"
```

这样实际上是在测试：

```text
Agent 收到某个 LLM 返回值之后
业务逻辑是否正确
```

而不是测试 OpenAI、Anthropic 等外部服务。

---

# 22. 回归测试 Regression Test

测试系统还有一个非常重要的作用：

> 防止已经修好的功能再次被改坏。

例如某次发现：

```text
build_initial_state()
没有把 evt_1 放进 pending_event_ids
```

修复以后加入测试：

```python
def test_build_initial_state_tracks_pending_events():
    ...
```

以后无论怎么修改代码，只要：

```bash
pytest
```

这个 bug 再出现，就会立刻检测出来。

因此可以把测试集合理解为：

```text
项目历史上所有重要行为
+
曾经出现过的 bug
+
关键业务规则
```

---

# 23. 可选扩展库

当前：

```text
pytest
pytest-asyncio
ruff
```

已经够构建基础测试系统。

随着项目变大，可以增加下面工具。

## pytest-cov

测试覆盖率：

```bash
pytest --cov=event_agent
```

可以看到：

```text
哪些文件被测试
哪些代码行没有执行
```

---

## pytest-mock

更方便使用 mock。

例如：

```python
def test_xxx(mocker):
    mocker.patch(...)
```

但不是必须，因为 Python 已经有：

```python
unittest.mock
```

---

## respx

如果大量使用 `httpx`：

```text
Agent
  ↓
HTTP API
```

可以用 respx mock HTTP 请求。

---

## hypothesis

自动生成大量测试输入。

例如：

```text
不同长度字符串
不同数字
边界值
异常组合
```

特别适合：

```text
parser
schema
纯函数
数据转换
```

---

# 24. 测试覆盖率不等于测试质量

例如：

```text
Coverage = 100%
```

不代表项目一定可靠。

测试：

```python
def test_x():
    func()
```

虽然执行了代码，但没有：

```python
assert ...
```

这种覆盖率没有太大意义。

真正高质量的测试应该检查：

```text
业务行为
边界条件
状态变化
错误处理
关键不变量
```

所以：

```text
测试质量
≠
测试数量

测试质量
≠
覆盖率

测试质量
=
关键行为是否被正确约束
```

---

# 25. 推荐的本地开发流程

修改代码以后：

```bash
ruff check .
```

然后：

```bash
ruff format --check .
```

再：

```bash
pytest
```

完整：

```bash
ruff check .
ruff format --check .
pytest -v --tb=short
```

可以理解为：

```text
第一关
Ruff
检查静态问题

第二关
pytest
检查运行行为

第三关
integration tests
检查整体流程
```

---

# 26. CI 测试流程

GitHub Actions / GitLab CI 等 CI 系统里也可以执行同样流程：

```text
git push
   ↓
CI
   ↓
安装 Python
   ↓
安装依赖
   ↓
ruff check
   ↓
ruff format --check
   ↓
pytest
   ↓
全部 PASS
   ↓
允许 merge
```

最核心的 CI 命令仍然可以非常简单：

```bash
ruff check .
ruff format --check .
pytest
```

---

# 27.  Event Agent 推荐测试架构

整体可以设计成：

```text
                         Event Agent
                              │
         ┌────────────────────┼────────────────────┐
         ↓                    ↓                    ↓
       Ruff                 pytest              pytest-asyncio
         │                    │                    │
    静态检查             确定性测试            async Agent
                              │
             ┌────────────────┼──────────────────┐
             ↓                ↓                  ↓
          Unit Test     Integration Test      Error Test
             │                │                  │
       ┌─────┼─────┐     ┌────┴────┐       ┌────┴────┐
       ↓     ↓     ↓     ↓         ↓       ↓         ↓
    Schema  Scan Registry Agent Workflow  bad JSON  LLM error
```

推荐重点测试：

```text
1. schemas
2. scan
3. EventSource → TypeExample
4. build_initial_state
5. pending / processed 状态
6. round / audit 状态
7. Tool 输入输出
8. async Agent workflow
9. LLM mock
10. 异常和边界输入
11. 最小 end-to-end pipeline
```

---

# 28. 最终理解

对于：

```toml
"pytest>=8,<9",
"pytest-asyncio>=1,<2",
"ruff>=0.9.2",
```

可以这样理解：

```text
pytest
    ↓
测试框架核心
    ↓
unit / integration / fixture / assert / error test

pytest-asyncio
    ↓
pytest 的 asyncio 扩展
    ↓
测试 async Agent / async Tool / async API

ruff
    ↓
静态代码质量检查
    ↓
lint + format
```

因此，一个基础测试系统就是：

```text
                  测试系统
                     │
          ┌──────────┴───────────┐
          ↓                      ↓
     Static Check            Runtime Test
          │                      │
        Ruff                  pytest
                                 │
                         pytest-asyncio
```

真正决定测试体系质量的并不是安装了多少测试库，而是：

```text
测试框架
    ×
测试用例设计
    ×
关键行为覆盖
    ×
错误场景覆盖
```

其中最重要的是：

> 把项目中“必须始终成立的规则”写成测试。

这也是 pytest 在 Agent 项目中最大的价值。
