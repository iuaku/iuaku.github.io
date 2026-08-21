---
title: "Python Async Await"
date: 2026-08-21
draft: false
---

这是一个很好的问题。理解 `async/await` 的关键，不是记住语法，而是理解它背后的**协作式调度系统**。

先给结论：

- `async def`、`await` 是 **Python 语言关键字（语法层面）**
- `asyncio` 是 **Python 标准库，实现异步运行框架**
- **事件循环（Event Loop）** 是调度中心
- **协程（Coroutine）** 是可以被事件循环暂停和恢复的小任务

整体关系：

```
Python语法
   |
   |  async def / await
   |
   v

协程对象 Coroutine

   |
   | asyncio负责管理

   v

Event Loop（事件循环）

   |
   | 调度多个协程

   v

IO事件完成 -> 恢复协程
```

---

# 1. 什么是协程 Coroutine？

先看普通函数：

```python
def download():
    print("start")
    time.sleep(3)
    print("finish")
```

调用：

```python
download()
```

流程：

```
进入函数
 |
执行
 |
等待3秒
 |
结束
```

函数不能暂停。

---

协程：

```python
async def download():
    print("start")
    await asyncio.sleep(3)
    print("finish")
```

它可以：

```
进入函数

执行start

遇到 await

暂停

保存当前状态

以后恢复

继续执行finish
```

也就是说：

> 协程是一种可以暂停执行，并且之后恢复执行的函数。

---

例如：

```python
async def test():
    print(1)
    await asyncio.sleep(1)
    print(2)
```

执行：

```python
coro = test()
```

得到：

```
Coroutine对象
```

它类似一个"暂停的任务":

```
状态:

[print(1)]
       |
       |
     await
       |
       |
[print(2)]
```

---

# 2. 为什么需要协程？

假设：

1000个HTTP请求：

```
请求A
等待服务器返回

请求B
等待服务器返回

请求C
等待服务器返回
```

如果同步：

```
线程
 |
请求A
 |
阻塞10秒
 |
请求B
 |
阻塞10秒
```

效率很低。


协程：

```
线程

协程A
 |
等待网络


协程B
 |
等待网络


协程C
 |
等待网络


网络返回

恢复对应协程
```

一个线程可以管理几千个等待任务。

---

# 3. 什么是事件循环 Event Loop？

事件循环就是：

> 一个不断检查任务状态，并决定下一步运行哪个任务的调度器。


类似操作系统 CPU 调度：

操作系统：

```
CPU

进程A
运行

进程B
运行

进程C
运行
```

asyncio：

```
Event Loop

协程A
运行

协程B
运行

协程C
运行
```

区别：

操作系统：

```
抢占式
```

CPU 可以强制切换。

asyncio：

```
协作式
```

任务自己说：

"我现在要等待了，可以切走"


通过：

```python
await
```

完成。

---

# 4. asyncio到底做了什么？

假设：

```python
async def main():

    await task1()

    await task2()
```

执行：

```
asyncio.run(main())

          |
          v

创建Event Loop

          |
          v

运行main协程

          |
          v

遇到task1

          |
          v

task1执行

          |
          v

遇到await IO

          |
          v

暂停task1

          |
          v

Event Loop寻找其他任务

          |
          v

IO完成

          |
          v

恢复task1
```

---

# 5. asyncio里面核心组件

大概结构：

```
asyncio

├── Event Loop
│
├── Task
│     |
│     把Coroutine包装成可调度任务
│
├── Future
│     |
│     表示未来会完成的结果
│
├── IO Selector
│     |
│     监听网络/文件事件
│
└── Queue/Lock/Semaphore
      |
      异步同步工具
```

---

## Coroutine

来源：

Python语法：

```python
async def
```

产生：

```python
coro = func()
```

---

## Task

asyncio创建：

```python
task = asyncio.create_task(coro)
```

区别：

Coroutine:

```
只是代码
```

Task:

```
交给事件循环管理的任务
```

例如：

```python
async def job():
    await asyncio.sleep(1)


task = asyncio.create_task(job())
```

现在事件循环知道：

"我要执行这个任务"

---

## Future

表示：

"未来某个时间会有结果"

类似：

快递单：

```
现在:
包裹没到

未来:
收到包裹
```

---

# 6. async/await是谁实现的？

分层看：

## Python语言层

Python 3.5加入：

```python
async
await
```

属于语法。

例如：

```python
async def foo():
    await bar()
```

Python解释器认识。


---

## asyncio库

标准库：

```
Lib/asyncio
```

负责：

- event loop
- task调度
- future
- socket IO


例如：

```python
asyncio.run()
asyncio.gather()
asyncio.create_task()
```

这些都是 asyncio 提供。


---

## 底层操作系统

真正等待网络：

Linux：

```
epoll
```

Mac：

```
kqueue
```

Windows：

```
IOCP
```

asyncio调用这些。

---

整体：

```
你的代码

async def
await

    ↓

Python解释器

    ↓

asyncio

    ↓

Event Loop

    ↓

操作系统IO机制

    ↓

网络/磁盘
```

---

# 7. 为什么 Agent 框架大量使用 async？

例如：

一个 LLM Agent：

```
用户输入

    |
    v

调用LLM API
    |
    | 等待5秒
    |
    v

调用搜索API
    |
    | 等待3秒
    |
    v

调用数据库
```

如果同步：

```
一个agent占一个线程
```

如果1000用户：

```
1000线程
```

成本高。


异步：

```
一个event loop

agent1 等LLM
agent2 搜索
agent3 数据库

同时等待
```

所以：

- FastAPI
- LangChain
- OpenAI Agents SDK
- 爬虫框架

都会大量使用：

```python
async def
await
asyncio
```

---

你可以把它理解成：

> `async def` 创建“可暂停任务”，`await` 是“主动让出CPU”，`asyncio` 提供“调度系统”，event loop 是“总调度员”。

这四个组合起来，就是 Python 的异步编程模型。
