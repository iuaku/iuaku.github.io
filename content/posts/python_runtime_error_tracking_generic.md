# Python 运行时错误追踪设计

## 1. 错误追踪的目标

代码运行中的错误追踪，不应该只依赖：

```python
print("error")
```

或者在最外层写一个：

```python
try:
    ...
except Exception:
    ...
```

对于一个包含多个模块、异步任务、外部调用和状态管理的 Python 项目，错误追踪系统至少应该能够回答下面 4 个问题：

```text
1. 哪里出错了？
2. 执行什么任务时出错了？
3. 为什么出错？
4. 出错以后系统应该怎么办？
```

一个比较合理的错误追踪链路是：

```text
Exception
    ↓
Context
    ↓
Logging
    ↓
Traceback
    ↓
run_id / task_id / stage
    ↓
Error Classification
    ↓
Retry / Skip / Fail
```

---

# 2. 不要吞掉异常

不推荐：

```python
try:
    process_task(task)
except Exception:
    pass
```

这样虽然程序可能继续运行，但错误信息完全丢失。

至少应该：

```python
import logging

logger = logging.getLogger(__name__)

try:
    process_task(task)
except Exception:
    logger.exception("Failed to process task")
    raise
```

推荐使用：

```python
logger.exception(...)
```

因为它会自动保留完整 traceback。

例如：

```text
Traceback (most recent call last):
  File "processor.py", line 120, in process_task
    ...
ValueError: invalid input
```

---

# 3. 日志必须包含上下文

仅仅记录：

```text
Failed to process task
```

通常是不够的。

建议至少记录：

```text
run_id
task_id
stage
operation
retry_count
```

例如：

```python
try:
    process_task(task)
except Exception:
    logger.exception(
        "Failed to process task",
        extra={
            "task_id": task.id,
            "stage": "processing",
        },
    )
    raise
```

理想情况下可以快速看到：

```text
run_id=8ac294b12f31
task_id=task_1382
stage=processing
```

这样错误出现时，可以快速定位到具体任务。

---

# 4. 为每次运行创建 run_id

推荐每次启动一次完整任务时生成：

```python
from uuid import uuid4

run_id = uuid4().hex[:12]
```

例如：

```text
run_id=8ac294b12f31
```

同一次运行中的所有日志，都携带相同的 `run_id`。

整体可以理解为：

```text
run_id
  │
  ├── step 1
  │
  ├── task 1
  │     ├── stage A
  │     ├── stage B
  │     └── stage C
  │
  └── task 2
        ├── stage A
        └── error
```

这相当于一个轻量级 tracing 系统。

---

# 5. 错误应该分类

不推荐系统中所有错误都只是：

```python
Exception
```

可以定义统一错误基类：

```python
class ApplicationError(Exception):
    pass
```

再划分业务错误：

```python
class InputError(ApplicationError):
    pass


class ValidationError(ApplicationError):
    pass


class ExternalServiceError(ApplicationError):
    pass


class OperationError(ApplicationError):
    pass
```

形成：

```text
ApplicationError
├── InputError
├── ValidationError
├── ExternalServiceError
└── OperationError
```

例如：

```python
if not task.id:
    raise InputError("task id is empty")
```

数据解析失败：

```python
raise ValidationError("Invalid response format")
```

外部调用失败：

```python
raise ExternalServiceError("External service failed")
```

---

# 6. 不同错误应该有不同处理策略

不能简单写成：

```python
except Exception:
    retry()
```

因为很多错误重试没有意义。

## 6.1 临时错误

例如：

```text
网络超时
HTTP 429
连接断开
服务暂时不可用
```

通常适合：

```text
retry
```

---

## 6.2 数据错误

例如：

```text
JSON 损坏
必要字段缺失
格式不合法
```

这种问题重新执行一次通常不会恢复。

比较合适的是：

```text
记录错误
    ↓
标记 failed
    ↓
skip
```

---

## 6.3 编程错误

例如：

```text
AttributeError
TypeError
IndexError
KeyError
```

很多时候意味着程序本身存在 bug。

比较合理的是：

```text
记录完整 traceback
    ↓
停止当前流程
    ↓
抛出异常
```

不要静默跳过。

---

# 7. 推荐的异常处理结构

例如：

```python
try:
    process_task(task)

except ExternalServiceError:
    retry_task(task)

except InputError as exc:
    mark_failed(task, exc)

except Exception:
    logger.exception(
        "Unexpected error",
        extra={"task_id": task.id},
    )
    raise
```

基本原则：

```text
可恢复错误
    → retry

数据错误
    → failed + skip

未知程序错误
    → traceback + raise
```

---

# 8. 状态中保存失败任务

如果系统维护任务状态，建议显式保存失败项：

```python
failed_task_ids: list[str]
```

或者：

```python
task_errors: dict[str, str]
```

例如：

```python
state.failed_task_ids.append(task.id)
state.task_errors[task.id] = str(exc)
```

这样状态会非常明确：

```text
task_1 → processed
task_2 → failed
task_3 → pending
```

更规范一些，可以设计：

```python
class TaskFailure(BaseModel):
    task_id: str
    stage: str
    error_type: str
    message: str
```

然后保存：

```python
failures: list[TaskFailure]
```

---

# 9. 记录执行阶段 stage

仅仅知道：

```text
task_1 出错
```

通常还不够。

一个任务可能经历：

```text
load
  ↓
validate
  ↓
transform
  ↓
process
  ↓
persist
```

因此错误日志应该带：

```text
stage
```

例如：

```python
logger.info(
    "Processing task",
    extra={
        "task_id": task.id,
        "stage": "validation",
    },
)
```

错误时：

```python
logger.exception(
    "Task processing failed",
    extra={
        "task_id": task.id,
        "stage": "validation",
    },
)
```

最终日志：

```text
run_id=abc123
task_id=task_1
stage=validation
ValidationError: invalid input
```

---

# 10. 使用结构化日志

开发初期普通日志已经够用：

```text
2026-08-17 11:00:00 ERROR failed processing task
```

项目变复杂以后，可以使用结构化日志。

例如：

```json
{
  "level": "ERROR",
  "run_id": "abc123",
  "task_id": "task_1",
  "stage": "validation",
  "error_type": "ValidationError",
  "message": "Invalid input"
}
```

结构化日志的好处是可以方便查询：

```text
run_id = abc123
```

或者：

```text
task_id = task_1
```

或者：

```text
error_type = ValidationError
```

---

# 11. Python logging 基础配置

没有必要一开始就引入复杂日志框架。

Python 标准库 `logging` 就可以完成基础能力。

例如：

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format=(
        "%(asctime)s "
        "%(levelname)s "
        "%(name)s "
        "%(message)s"
    ),
)

logger = logging.getLogger(__name__)
```

然后：

```python
logger.debug("Detailed internal information")
```

```python
logger.info("Task started")
```

```python
logger.warning("Unexpected but recoverable condition")
```

```python
logger.error("Task failed")
```

异常时：

```python
logger.exception("Processing failed")
```

---

# 12. 日志级别

推荐理解为：

```text
DEBUG
    ↓
详细内部调试信息

INFO
    ↓
正常运行中的关键节点

WARNING
    ↓
出现异常情况，但程序还能继续

ERROR
    ↓
某个任务已经失败

CRITICAL
    ↓
整个系统无法继续
```

---

# 13. 不要在所有函数中都写 try / except

一种常见问题是：

```python
def a():
    try:
        ...
    except Exception:
        ...

def b():
    try:
        ...
    except Exception:
        ...

def c():
    try:
        ...
    except Exception:
        ...
```

这样容易导致：

```text
异常重复捕获
日志重复记录
根因被隐藏
异常链混乱
```

更推荐的原则是：

> 在真正知道“这个错误应该如何处理”的层级捕获异常。

例如：

```text
底层函数
parse_data()
    ↓
发现问题并 raise

业务层
process_task()
    ↓
决定 retry / skip

最外层
run()
    ↓
记录最终失败 + 保存状态
```

---

# 14. 使用异常链 raise ... from ...

如果底层错误需要转换成业务错误，建议：

```python
try:
    result = parse_response(response)
except ValueError as exc:
    raise ValidationError(
        "Failed to parse response"
    ) from exc
```

这样会保留原始异常。

最终 traceback 会类似：

```text
ValueError:
invalid JSON

The above exception was the direct cause of:

ValidationError:
Failed to parse response
```

相比：

```python
raise ValidationError(...)
```

更容易找到真正根因。

---

# 15. 外部调用中的错误追踪

如果系统会调用外部服务，推荐记录：

```text
run_id
task_id
service_name
operation
stage
retry_count
response_status
```

例如：

```python
try:
    result = call_service(data)

except Exception as exc:
    logger.exception(
        "External service call failed",
        extra={
            "task_id": task.id,
            "stage": "external_call",
        },
    )

    raise ExternalServiceError(
        "External service failed"
    ) from exc
```

---

# 16. Retry 设计

对于网络和外部服务错误，可以设计重试：

```text
第一次失败
    ↓
等待
    ↓
第二次
    ↓
等待更久
    ↓
第三次
    ↓
最终失败
```

例如指数退避：

```text
1 秒
2 秒
4 秒
8 秒
```

但重试应该只用于可能恢复的问题：

```text
网络错误
timeout
429
503
```

不要对下面问题盲目 retry：

```text
代码 bug
格式错误
必要字段缺失
本地数据损坏
```

---

# 17. 使用 pytest 测试错误处理

错误处理系统本身也应该测试。

例如：

```python
import pytest


def test_invalid_input_raises():
    with pytest.raises(InputError):
        process_task(invalid_task)
```

测试失败任务是否被记录：

```python
def test_failed_task_is_recorded():
    ...
    assert "task_1" in state.failed_task_ids
```

测试异常链：

```python
def test_validation_error_preserves_cause():
    ...
```

测试 retry：

```python
def test_temporary_error_is_retried():
    ...
```

测试不可恢复异常：

```python
def test_programming_error_is_not_silently_skipped():
    ...
```

---

# 18. 推荐项目结构

可以设计为：

```text
project/
├── errors.py
├── logging_config.py
├── main.py
│
├── core/
│   ├── processor.py
│   ├── state.py
│   └── schemas.py
│
└── tests/
    ├── test_errors.py
    ├── test_processor.py
    └── test_state.py
```

`errors.py`：

```python
class ApplicationError(Exception):
    pass


class InputError(ApplicationError):
    pass


class ValidationError(ApplicationError):
    pass


class ExternalServiceError(ApplicationError):
    pass


class OperationError(ApplicationError):
    pass
```

---

# 19. 推荐的完整运行流程

```text
main()
  │
  ├── 创建 run_id
  │
  ├── 初始化 logging
  │
  ├── 加载任务
  │
  └── 遍历 task
        │
        ├── INFO: task start
        │
        ├── load
        │
        ├── validate
        │
        ├── process
        │
        └── persist
              │
              ├── success
              │     ↓
              │  processed
              │
              └── failure
                    │
                    ├── logger.exception
                    ├── 保存 failure
                    └── 根据类型：
                          retry
                          skip
                          raise
```

---

# 20. 推荐错误上下文字段

建议至少统一：

```text
run_id
task_id
stage
operation
error_type
error_message
retry_count
```

如果后面需要性能追踪，还可以加入：

```text
duration_ms
start_time
end_time
```

---

# 21. 推荐错误记录模型

例如：

```python
from datetime import datetime

from pydantic import BaseModel


class TaskFailure(BaseModel):
    run_id: str
    task_id: str
    stage: str
    error_type: str
    message: str
    created_at: datetime
```

进一步可以加入：

```python
class TaskFailure(BaseModel):
    run_id: str
    task_id: str
    stage: str
    operation: str | None = None
    error_type: str
    message: str
    retry_count: int = 0
    created_at: datetime
```

---

# 22. 核心设计原则

整个错误追踪系统可以归纳为 6 点：

```text
1. 不吞异常

2. logger.exception 保留 traceback

3. 每次运行都有 run_id

4. 每个错误携带 task_id + stage

5. 对错误分类，并分别执行
   retry / skip / fail

6. 系统状态中显式保存失败任务
```

最终错误信息应该从：

```text
程序报错了
```

提升为：

```text
run_id=abc123
task_id=task_291
stage=validation
error_type=ValidationError

原因：
输入无法通过校验

处理：
任务标记为 failed
继续处理剩余任务
```

---

# 23. 最终架构

可以把整个运行时错误追踪体系理解为：

```text
                    Runtime
                       │
                       ↓
                    run_id
                       │
            ┌──────────┼──────────┐
            ↓          ↓          ↓
          Task      Operation   External Call
            │          │          │
            └──────────┼──────────┘
                       ↓
                    Exception
                       ↓
               Error Classification
                       ↓
                  Logging
                       ↓
                  Traceback
                       ↓
                Failure Record
                       ↓
            ┌──────────┼──────────┐
            ↓          ↓          ↓
          Retry       Skip       Fail
```

核心目标不是简单地“把错误打印出来”，而是：

> 让每一个错误都能够被定位、解释、记录，并有明确的后续处理策略。
