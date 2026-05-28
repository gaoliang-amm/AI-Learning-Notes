# FastAPI & SQLAlchemy 课程笔记

> [!info] 课程信息
> - **出处**：尚硅谷大模型技术系列
> - **版本**：V1.0
> - **技术栈**：FastAPI（Web框架）+ SQLAlchemy（ORM）+ Uvicorn（ASGI服务器）+ asyncio（异步编程）

---

## 目录
- [[#一、协程（Coroutine）]]
- [[#二、WSGI 与 ASGI]]
- [[#三、FastAPI]]
- [[#四、SQLAlchemy ORM]]
- [[#五、FastAPI + SQLAlchemy 整合]]

---

## 一、协程（Coroutine）

### 1.1 什么是协程

**协程（Coroutine）** 是 Python 处理高并发场景的强大工具，尤其在 **I/O 密集型**任务中表现出色。

> [!quote] 定义
> 协程是一种**用户态的轻量级线程**，不像线程那样由操作系统调度，而是由**程序自身控制**，也叫**微线程**。

### 1.2 进程 / 线程 / 协程 对比

| 机制 | 调度者 | 切换开销 | 资源共享 | 适用场景 | 核心优势 | 局限性 |
|------|--------|----------|----------|----------|----------|--------|
| 进程 | 操作系统 | 极大（内核级） | 完全隔离 | CPU 密集型 | 利用多核 | 资源消耗大，数量有限 |
| 线程 | 操作系统 | 较大（内核级） | 共享内存（需加锁） | I/O 密集型（低并发） | 共享地址空间 | 受 GIL 限制，数量过多时调度开销大 |
| 协程 | 程序自身 Event Loop | 极小（函数调用级） | 共享内存（无需加锁） | I/O 密集型（极高并发） | 极高并发，高性能 I/O | 需要配套异步库，或线程池处理阻塞 |

### 1.3 线程处理 I/O 的不足

> [!warning] 线程模型的瓶颈
> - **切换开销大**：操作系统在数千个线程间切换耗费大量 CPU 时间
> - **同步复杂**：需要用锁（Lock）保护共享数据，容易死锁
> - **阻塞浪费**：线程等待 I/O 时被挂起，线程本身切换成本仍然高

### 1.4 协程的优势

- **调度者**：由程序自身调度，通过 `await` 主动让出 CPU
- **轻量级**：一个线程可创建**数万个协程**，创建成本极低
- **无锁机制**：同一时间只有一个协程运行，无需锁保护共享资源

---

### 1.5 `async` 和 `await` 关键字

> [!note] Python 3.5+ 支持，通过 `async`/`await` 实现协程

#### 定义协程函数：`async def`

```python
import asyncio

# 使用 async def 定义协程函数，返回一个协程对象
async def hello():
    print("Hello")
    await asyncio.sleep(1)  # 模拟 I/O 操作，让出 CPU
    print("World")

# 调用 hello() 不会立即执行，只返回协程对象
coro_obj = hello()
print(type(coro_obj))  # <class 'coroutine'>
```

#### `await` 关键字的作用

> [!important] `await` 是协程的灵魂
> 1. **遇到 `await`**：当前协程**让出 CPU 控制权**，进入"挂起"状态
> 2. **Event Loop 接管**：调度其他"就绪"状态的协程执行
> 3. **等待完成后**：Event Loop 将控制权**交还**给被挂起的协程，从 `await` 之后继续执行

---

### 1.6 事件循环（Event Loop）

**事件循环**是管理和调度所有协程的**核心机制**。Python 3.7+ 使用 `asyncio.run()` 启动。

#### 串行 vs 并发示例

```python
import asyncio, time

async def my_func(name, delay):
    print(f"任务 {name} 开始")
    await asyncio.sleep(delay)
    print(f"任务 {name} 完成")
    return f"任务 {name} 结果"

# 串行执行（总耗时 = 1s + 2s = 3s）
async def main_sync():
    await my_func("A", 1)   # 等 A 完成再执行 B
    await my_func("B", 2)

asyncio.run(main_sync())
```

#### 三种任务注册方式

| 方式 | 目的 | 机制 |
|------|------|------|
| `asyncio.run(main_coro)` | 启动整个异步程序（顶级注册） | 创建 EventLoop，将顶级协程包装为 Task 并运行 |
| `asyncio.create_task(coro)` | **实现并发**（最常用） | 将协程立即包装为 Task，加入队列后台运行 |
| `await coro` | 实现串行等待 | 隐式注册，必须等待该协程完成才能继续 |

#### 事件循环工作流程

```
注册任务 → 启动协程 → 遇到 await → 切换任务 → I/O 完成通知 → 恢复协程 → 终止与清理
```

事件循环伪代码：
```python
任务列表 = [Task1, Task2, ...]
while True:
    已完成的IO = monitor_io_events()    # 检查 I/O 事件，更新任务状态
    for task in 就绪任务队列:
        task.run_until_await()           # 运行直到遇到下一个 await
    if not 任务列表 and not 正在等待IO:
        break
```

---

### 1.7 实现并发：Task 与 `asyncio.gather()`

#### Task 的概念

`Task` 是对协程的**封装**，自动将协程注册到事件循环，使其具备被调度和执行的能力。

```python
task1 = asyncio.create_task(my_func("C", 1))  # 立即在后台运行
task2 = asyncio.create_task(my_func("D", 2))
```

#### `asyncio.gather()` 批量并发

```python
async def main_concurrent():
    task1 = asyncio.create_task(my_func("C", 1))
    task2 = asyncio.create_task(my_func("D", 2))
    # gather 等待所有任务完成，返回结果列表
    results = await asyncio.gather(task1, task2)
    print(f"所有结果: {results}")

# 总耗时 = max(1s, 2s) = 2s（而非 3s）
asyncio.run(main_concurrent())
```

> [!tip] 并发执行原理
> - task C 打印"开始"，遇到 `await sleep(1)`，让出控制权
> - 事件循环切换到 task D，打印"开始"，遇到 `await sleep(2)`，让出
> - 1 秒后 task C 恢复，打印"完成"
> - 2 秒后 task D 恢复，打印"完成"
> - 总耗时 ≈ 2s

---

### 1.8 高级控制

#### 处理阻塞 I/O

> [!warning] 协程的限制
> 协程在单线程运行。若包含 `time.sleep()`、`requests.get()` 等**阻塞 I/O**，整个事件循环会被卡住！

**解决方案**：使用 `asyncio.to_thread()`（Python 3.9+）隔离阻塞操作

```python
import requests

async def fetch_blocking_url(url):
    # 将同步 requests.get() 放到另一个线程执行，不阻塞事件循环
    response = await asyncio.to_thread(requests.get, url)
    return response.status_code
```

#### 超时处理：`asyncio.wait_for()`

```python
async def main_timeout():
    try:
        await asyncio.wait_for(slow_task(), timeout=2.0)  # 2 秒超时
    except asyncio.TimeoutError:
        print("任务超时！")
```

#### 任务取消：`Task.cancel()`

用于请求取消正在运行的 Task，被取消的 Task 会抛出 `asyncio.CancelledError`。

#### 实战：异步网络请求（aiohttp）

```python
import asyncio, aiohttp, time

async def fetch_url(session, url):
    async with session.get(url) as response:
        await response.text()  # 等待响应体时让出 CPU
        return f"{url} 请求完成"

async def main():
    urls = ["https://www.baidu.com", "https://www.bing.com", "https://www.github.com"]
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks)  # 并发请求
        for result in results:
            print(result)

asyncio.run(main())
```

---

## 二、WSGI 与 ASGI

### 2.1 核心对比

| 特性 | WSGI | ASGI |
|------|------|------|
| 全称 | Web Server Gateway Interface | Asynchronous Server Gateway Interface |
| 提出时间 | 2003 年 | 2018 年 |
| 异步支持 | ❌ 仅支持同步 | ✅ 原生支持异步，兼容 WSGI |
| 适配框架 | Flask、Django（旧版） | FastAPI、Starlette、Django 3.1+ |
| 服务器 | Gunicorn、uWSGI | **Uvicorn** |
| WebSocket | ❌ 不支持 | ✅ 支持 |

### 2.2 工作流程差异

**WSGI 工作流程**（同步阻塞）：
```
客户端请求 → WSGI服务器 → 同步调用 application(environ, start_response) → 返回响应
```
> 一个请求未处理完，对应线程无法处理其他请求。

**ASGI 工作流程**（异步非阻塞）：
```
客户端请求 → ASGI服务器 → 封装为事件(http.request) → 事件循环异步传递给应用 → 返回事件(http.response)
```
> 等待 I/O 时，事件循环切换到其他请求，实现非阻塞处理。

### 2.3 如何选择

> [!tip] 选择建议
> - 同步框架（Flask、Django 旧版）→ **WSGI 服务器**（Gunicorn）
> - 异步框架（FastAPI、Starlette）→ **ASGI 服务器**（Uvicorn）
> - 需要 WebSocket / 实时通信 → **必须使用 ASGI**

---

## 三、FastAPI

### 3.1 FastAPI 介绍

> [!quote] 官方定义
> FastAPI 是一个**现代、快速（高性能）**的 Web 框架，用于构建 API。建立在 **Starlette** 和 **Pydantic** 之上。

**核心特性**：
- 基于 Python 3.7+ **类型提示**和**异步编程（asyncio）**
- **自动交互式文档**（基于 OpenAPI 规范和 JSON Schema）
- 内置**数据验证**（Pydantic）
- **依赖注入**（Dependency Injection）

| 资源 | 地址 |
|------|------|
| 文档 | https://fastapi.tiangolo.com |
| 源码 | https://github.com/fastapi/fastapi |

---

### 3.2 第一个 FastAPI 程序

#### 安装依赖

```bash
pip install fastapi
pip install uvicorn
```

#### 创建 `main.py`

```python
from fastapi import FastAPI
import uvicorn

app = FastAPI()

@app.get("/")
async def read_root():
    return {"Hello": "World"}

@app.get("/items/{item_id}")
async def read_item(item_id: int, q: str | None = None):
    return {"item_id": item_id, "q": q}

if __name__ == "__main__":
    uvicorn.run(
        app="main:app",   # 模块:应用实例
        host="0.0.0.0",
        port=8000,
        reload=True       # 开发模式，代码修改自动重启（生产环境去掉）
    )
```

#### 启动服务器

```bash
uvicorn main:app --reload
```

#### 访问地址

| 地址 | 说明 |
|------|------|
| `http://127.0.0.1:8000/` | 主页 |
| `http://127.0.0.1:8000/items/5?q=iphone17` | 带路径参数和查询参数 |
| `http://127.0.0.1:8000/docs` | 自动生成的 Swagger UI 文档 |

---

### 3.3 同步函数 vs 异步函数

| 对比项 | `def`（同步） | `async def`（异步） |
|--------|--------------|---------------------|
| 内部使用 `await` | ❌ 不能 | ✅ 可以 |
| 执行机制 | FastAPI 自动放入**线程池**执行 | 直接在**事件循环**中运行 |
| 适用场景 | 纯计算逻辑 / 无异步库可用 | I/O 密集型操作（数据库、网络请求） |
| 并发能力 | 依赖线程池，线程耗尽会排队 | 真正的异步并发，遇到 `await` 切换任务 |

> [!tip] 选择原则
> - 涉及 **I/O 操作**（数据库、网络）→ 优先用 `async def` + 异步库
> - **纯计算逻辑**或无异步库 → 用 `def`（FastAPI 自动管理线程池）

---

### 3.4 路径参数

FastAPI 使用 Python 字符串格式化语法声明路径参数。

```python
@app.get("/items/{item_id}")
async def read_item(item_id: int):  # 类型注解自动转换 + 校验
    return {"item_id": item_id}
```

> [!important] 关键特性
> - **类型转换**：URL 中的字符串 `"3"` 自动转为 `int` 的 `3`
> - **类型校验**：传入 `"foo"` 会返回 HTTP 422 错误
> - **参数顺序**：固定路径（如 `/users/me`）必须在动态路径（`/users/{user_id}`）**之前**声明

```python
# 正确顺序：固定路径在前
@app.get("/users/me")      # ✅ 先声明
async def read_user_me():
    return {"user_id": "the current user"}

@app.get("/users/{user_id}")  # ✅ 后声明
async def read_user(user_id: str):
    return {"user_id": user_id}
```

---

### 3.5 查询参数

**不在路径中的参数**，FastAPI 自动识别为查询参数（`?key=value&...`）。

```python
items_list = [{"item1": "Foo"}, {"item2": "Bar"}, {"item3": "Baz"}]

@app.get("/items/")
async def read_item(start: int = 0, limit: int = 10):  # 有默认值
    return items_list[start: start + limit]
```

| URL | 实际效果 |
|-----|----------|
| `http://127.0.0.1:8000/items/` | start=0, limit=10（默认值） |
| `http://127.0.0.1:8000/items/?start=20` | start=20, limit=10 |
| `http://127.0.0.1:8000/items/?start=0&limit=2` | start=0, limit=2 |

#### 可选参数

```python
@app.get("/items/{item_id}")
async def read_item(item_id: str, q: str | None = None):  # None 表示可选
    if q:
        return {"item_id": item_id, "q": q}
    return {"item_id": item_id}
```

#### bool 类型自动转换

```python
@app.get("/items/{item_id}")
async def read_item(item_id: str, short: bool = False):
    ...
# 以下 URL 的 short 均被识别为 True：
# ?short=1  ?short=True  ?short=true  ?short=on  ?short=yes
```

#### 必选查询参数

```python
# 不声明默认值 → 该参数为必选
@app.get("/items/{item_id}")
async def read_item(item_id: str, needy: str):  # needy 是必选的
    return {"item_id": item_id, "needy": needy}
```

---

### 3.6 请求体传参（Pydantic）

使用 `POST`/`PUT`/`DELETE` 等方法发送请求体数据，通过 **Pydantic 模型**声明。

```python
from fastapi import FastAPI
from pydantic import BaseModel

# 定义数据模型，继承 BaseModel
class Item(BaseModel):
    name: str
    desc: str | None = None   # 可选字段
    price: float

app = FastAPI()

@app.post("/items/")
async def create_item(item: Item):
    return item
```

> [!note] Pydantic 的功能
> - 自动**数据验证**（类型不匹配返回 422）
> - 自动**文档生成**
> - 自动**类型转换**

---

### 3.7 路由分发（APIRouter）

当项目规模扩大，使用 **APIRouter** 将路由拆分到不同模块，再通过 `app.include_router()` 整合。

#### 项目结构

```
myproject/
├── main.py          # 主应用
└── routers/
    ├── user.py      # 用户相关路由
    └── item.py      # 商品相关路由
```

#### 子模块路由定义（`routers/user.py`）

```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/users",     # 所有路由自动加上 /users 前缀
    tags=["用户管理"]    # 文档中分类标签
)

@router.get("/")
def get_all_users():
    return {"message": "获取所有用户列表"}

@router.get("/{user_id}")
def get_user(user_id: int):
    return {"message": f"获取 ID 为 {user_id} 的用户信息"}
```

#### 主应用整合（`main.py`）

```python
from fastapi import FastAPI
from routers import user, item

app = FastAPI(title="路由分发示例")

app.include_router(user.router)  # 挂载用户路由
app.include_router(item.router)  # 挂载商品路由

@app.get("/")
def root():
    return {"message": "欢迎访问主页面"}
```

> [!tip] 访问路径
> - 主页：`http://127.0.0.1:8000/`
> - 用户列表：`http://127.0.0.1:8000/users/`
> - 商品详情：`http://127.0.0.1:8000/items/100`
> - 文档（按 tags 分类）：`http://127.0.0.1:8000/docs`

---

## 四、SQLAlchemy ORM

### 4.1 ORM 介绍

> [!quote] 什么是 ORM
> **ORM（Object-Relational Mapping，对象关系映射）**：将数据库中的**表**映射为编程语言中的**类**，将表中的**行**映射为类中的**对象**，让开发者可以用面向对象方式操作数据库，无需直接编写 SQL。

**核心价值**：
- **屏蔽数据库差异**：同一套代码适配 MySQL、PostgreSQL 等
- **简化开发流程**：用类、对象、方法替代 SQL 语句
- **提高可读性**：代码更符合面向对象思维
- **自动类型转换**：无需手动转换数据库字段与 Python 类型

#### 常见 ORM 产品对比

| ORM 工具 | 特点 | 适用场景 |
|----------|------|----------|
| **SQLAlchemy** | 功能全面，支持 ORM 和原生 SQL，灵活度极高，生态完善 | 中大型项目、复杂查询、跨数据库兼容 |
| Django ORM | 与 Django 深度绑定，开箱即用，灵活性较低 | Django Web 应用 |
| Tortoise-ORM | 异步 ORM，支持 async/await | FastAPI + 异步数据库 |
| SQLModel | 基于 SQLAlchemy + Pydantic，简化模型定义 | FastAPI 项目 |

---

### 4.2 SQLAlchemy 基本架构

```
SQLAlchemy ORM（对象关系映射层）
         ↕
SQLAlchemy Core（核心层）
    ├── Schema / Types     # 表结构和数据类型
    ├── SQL Expression Language  # Python 生成 SQL
    ├── Engine（引擎）    # 连接入口
    │   ├── Connection Pooling（连接池）
    │   └── Dialect（方言适配器）
         ↕
DBAPI（数据库通信层）
    ├── pymysql（MySQL）
    ├── psycopg2（PostgreSQL）
    └── ...
```

**核心组件说明**：

| 组件 | 作用 |
|------|------|
| `Engine` | 连接数据库的"引擎"，管理连接池和 SQL 执行 |
| `Base` | 所有 ORM 模型的父类，统一管理表结构 |
| `Session` | 执行数据库操作的会话，负责暂存、提交、回滚 |
| `Dialect` | 适配不同数据库 SQL 语法差异 |
| Connection Pool | 预先创建连接并复用，提升高并发性能 |

---

### 4.3 环境准备

```bash
pip install sqlalchemy
pip install pymysql   # MySQL 驱动
```

确保 MySQL 服务已启动（默认端口 3306），并创建数据库：
```sql
CREATE DATABASE fastapi_db;
```

---

### 4.4 核心文件结构

```
project/
├── base.py       # 定义 Base 基类
├── models.py     # 定义 ORM 模型（映射数据库表）
├── database.py   # 创建引擎和会话
└── main.py       # 主程序（建表、CRUD 操作）
```

#### `base.py`：定义基类

```python
from sqlalchemy.ext.declarative import declarative_base

# 生成基类，所有模型需继承该类
Base = declarative_base()
```

#### `models.py`：定义 ORM 模型

```python
from sqlalchemy import Column, Integer, String, ForeignKey, Date
from sqlalchemy.orm import relationship
from demo.base import Base

class Department(Base):
    """部门模型（一对多：一个部门包含多个员工）"""
    __tablename__ = "departments"

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(50), nullable=False, unique=True)  # 部门名称（唯一）
    location = Column(String(100))                           # 部门位置

    # 一对多关联员工
    employees = relationship(
        "Employee",
        back_populates="department",
        lazy="selectin"   # 查询部门时同时加载员工
    )

class Employee(Base):
    """员工模型（多对一：多个员工属于一个部门）"""
    __tablename__ = "employees"

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(50), nullable=False)
    age = Column(Integer)
    hire_date = Column(Date)
    department_id = Column(
        Integer,
        ForeignKey("departments.id", ondelete="RESTRICT"),  # 禁止删除有员工的部门
        nullable=False
    )

    # 多对一关联部门
    department = relationship(
        "Department",
        back_populates="employees",
        lazy="joined"   # 立即加载部门数据
    )
```

**字段约束说明**：

| 约束 | 说明 |
|------|------|
| `primary_key=True` | 设为主键 |
| `autoincrement=True` | 整数类型自动自增 |
| `unique=True` | 唯一索引，避免重复 |
| `nullable=False` | 非空约束 |

#### `database.py`：创建引擎和会话

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# MySQL 连接字符串格式
DATABASE_URL = "mysql+pymysql://root:123456@localhost:3306/fastapi_db"

# 创建引擎
engine = create_engine(
    DATABASE_URL,
    echo=True,           # 打印执行的 SQL（开发用，生产关闭）
    pool_pre_ping=True   # 连接前检查有效性
)

# 创建会话工厂
SessionLocal = sessionmaker(
    autocommit=False,    # 关闭自动提交，需手动 commit()
    bind=engine
)
```

#### `main.py`：建表

```python
from demo.base import Base
from demo.database import engine
from demo.models import Employee, Department  # 必须导入，Base 才能识别模型

def create_table():
    Base.metadata.create_all(bind=engine)  # 创建所有模型对应的表
    print("表创建成功")

if __name__ == '__main__':
    create_table()
```

---

### 4.5 CRUD 操作

#### Create（新增）

```python
def insert_data():
    db = SessionLocal()
    try:
        # 新增部门
        new_dept = Department(name="研发部", location="北京总部")
        db.add(new_dept)
        db.commit()
        db.refresh(new_dept)  # 刷新，获取自增 ID 等字段

        # 新增关联员工
        emp1 = Employee(name="张三", age=30, hire_date=date(2023,1,1), department_id=new_dept.id)
        emp2 = Employee(name="李四", age=28, hire_date=date(2023,3,15), department_id=new_dept.id)
        db.add_all([emp1, emp2])   # 批量添加
        db.commit()
        db.refresh(emp1); db.refresh(emp2)

    except Exception as e:
        db.rollback()   # 出错时回滚
        print(f"新增失败：{e}")
    finally:
        db.close()      # 关闭会话
```

#### Delete（删除）

```python
def delete_data():
    session = SessionLocal()
    try:
        emp = session.query(Employee).filter(Employee.name == "李四").first()
        if emp:
            session.delete(emp)
            session.commit()
    except Exception as e:
        session.rollback()
    finally:
        session.close()
```

#### Update（修改）

```python
def update_data():
    session = SessionLocal()
    try:
        emp = session.query(Employee).filter(Employee.name == "张三").first()
        if emp:
            emp.age = 31        # 直接修改属性
            session.commit()    # 提交更新
    except Exception as e:
        session.rollback()
    finally:
        session.close()
```

#### Read（查询）

```python
def read_data():
    session = SessionLocal()
    try:
        # 按主键查询
        dept = session.get(Department, 1)

        # 过滤查询
        employees = session.query(Employee).filter(Employee.department_id == 1).all()
        old_employees = session.query(Employee).filter(Employee.age > 30).all()

        # 逻辑运算
        from sqlalchemy import and_, or_
        emp = session.query(Employee).filter(
            and_(Employee.age.between(30, 40), Employee.department_id == 1)
        ).first()

        emps = session.query(Employee).filter(
            or_(Employee.department_id == 2, Employee.age > 32)
        ).all()

        # 表连接（JOIN）
        result = session.query(Employee, Department).join(
            Department, Employee.department_id == Department.id
        ).all()

        # 预加载关联数据（避免 N+1 问题）
        from sqlalchemy.orm import joinedload
        employees = session.query(Employee).options(
            joinedload(Employee.department)
        ).all()

        # 子查询
        from sqlalchemy import func
        dept_emp_count = session.query(
            Employee.department_id,
            func.count(Employee.id).label("count")
        ).group_by(Employee.department_id).subquery()

        depts = session.query(Department).join(
            dept_emp_count, Department.id == dept_emp_count.c.department_id
        ).filter(dept_emp_count.c.count > 0).all()

        # 去重
        locations = session.query(Department.location).join(Employee).distinct().all()

        # first() / all()
        first_emp = session.query(Employee).first()
        all_depts = session.query(Department).all()

    except Exception as e:
        session.rollback()
    finally:
        session.close()
```

---

### 4.6 关联关系（Relationship）

#### 四种关联类型

| 关联类型 | 场景示例 | 数据库实现 |
|----------|----------|------------|
| 一对多（One-to-Many） | 一个用户拥有多个商品 | 子表通过外键关联主表 |
| 多对一（Many-to-One） | 多个商品属于一个用户 | 同上（一对多的反向视角） |
| 一对一（One-to-One） | 一个用户对应一个个人资料 | 子表外键设为 `unique=True` |
| 多对多（Many-to-Many） | 多个学生选修多个课程 | 通过**中间表**关联两个表 |

#### `relationship` 核心参数

| 参数 | 作用 | 示例 |
|------|------|------|
| `argument` | 关联的目标模型（必选） | `relationship("Item")` |
| `back_populates` | 双向关联，指定反向字段名（显式定义） | `back_populates="owner"` |
| `backref` | 简化写法，自动生成反向字段（隐式） | `backref="owner"` |
| `foreign_keys` | 显式指定外键（多外键场景必用） | `foreign_keys=[Item.owner_id]` |
| `cascade` | 级联操作规则 | `cascade="all, delete-orphan"` |
| `lazy` | 关联数据加载方式 | `lazy="selectin"` |
| `uselist` | 是否为集合（False 表示一对一） | `uselist=False` |
| `secondary` | 多对多中间表 | `secondary=user_course` |

#### 一对多 / 多对一

```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    username = Column(String(50))
    items = relationship("Item", back_populates="owner", lazy="selectin", cascade="save-update")

class Item(Base):
    __tablename__ = "items"
    id = Column(Integer, primary_key=True)
    title = Column(String(100))
    owner_id = Column(Integer, ForeignKey("users.id"))
    owner = relationship("User", back_populates="items")
```

#### 一对一

```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    profile = relationship("Profile", back_populates="user",
                           uselist=False,               # 关键：非集合
                           cascade="all, delete-orphan")

class Profile(Base):
    __tablename__ = "profiles"
    id = Column(Integer, primary_key=True)
    bio = Column(String(200))
    user_id = Column(Integer, ForeignKey("users.id"), unique=True)  # 唯一约束
    user = relationship("User", back_populates="profile")
```

#### 多对多

```python
from sqlalchemy import Table

# 中间表（无需定义模型类）
student_course = Table(
    "student_course", Base.metadata,
    Column("student_id", Integer, ForeignKey("students.id"), primary_key=True),
    Column("course_id", Integer, ForeignKey("courses.id"), primary_key=True)
)

class Student(Base):
    __tablename__ = "students"
    id = Column(Integer, primary_key=True)
    name = Column(String(50))
    courses = relationship("Course", secondary=student_course,
                           back_populates="students", lazy="selectin")

class Course(Base):
    __tablename__ = "courses"
    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    students = relationship("Student", secondary=student_course, back_populates="courses")
```

#### 双向关联：`back_populates` vs `backref`

> [!note] 区别
> - **`back_populates`（推荐）**：两个模型中**显式各自定义**，逻辑清晰
> - **`backref`**：简化写法，在一个模型定义即可，自动为另一个模型生成反向字段

#### 级联操作（`cascade`）

| 规则 | 说明 |
|------|------|
| `save-update` | 保存主对象时自动保存关联对象（**默认**） |
| `delete` | 删除主对象时自动删除关联对象 |
| `delete-orphan` | 关联对象与主对象解除关联时自动删除（仅一对多） |
| `all` | 包含 save-update、merge、refresh-expire、expunge、delete |

常用组合：`cascade="all, delete-orphan"`

#### 加载方式（`lazy`）

| 值 | 说明 | 适用场景 |
|----|------|----------|
| `select`（默认） | 访问关联属性时才查询（可能 N+1 问题） | 偶尔访问 |
| `selectin` | 通过 IN 语句一次性加载（**推荐**） | 一对多，批量加载 |
| `joined` | 通过 JOIN 与主对象一起加载 | 一对一 |
| `subquery` | 通过子查询加载 | 特殊场景 |
| `dynamic` | 返回查询对象（可继续过滤） | 大数据量 |

---

### 4.7 sqlacodegen：从数据库表生成模型类

`sqlacodegen` 根据**现有数据库表结构**自动生成 SQLAlchemy 模型类，适合已有数据库的项目迁移。

```bash
pip install sqlacodegen
pip install pymysql
```

**使用方式**（在代码中调用）：

```python
import subprocess, sys

def table_2_model(run=False):
    if not run:
        return
    url = "mysql+pymysql://root:123456@localhost:3306/fastapi_db"
    output_path = "table_2_models.py"
    venv_python = sys.executable  # 使用虚拟环境的 Python
    cmd = [venv_python, "-m", "sqlacodegen", url]
    result = subprocess.run(cmd, capture_output=True, text=True, encoding="utf-8")
    with open(output_path, "w", encoding="utf-8") as f:
        f.write(result.stdout)
```

---

## 五、FastAPI + SQLAlchemy 整合

### 5.1 项目结构

```
myproject/
├── fastapi_sqlalchemy.py   # FastAPI 主应用（接口定义）
├── base.py                 # 模型基类
├── database.py             # 数据库配置（引擎、会话）
└── models.py               # SQLAlchemy 模型
```

### 5.2 依赖注入：`get_db()`

使用 FastAPI 的 **Depends** 机制，在每次请求时自动获取数据库会话，请求结束后自动关闭。

```python
from fastapi import Depends
from sqlalchemy.orm import Session
from database import SessionLocal

def get_db():
    db = SessionLocal()
    try:
        yield db    # 提供会话给路由函数
    finally:
        db.close()  # 请求结束后自动关闭
```

### 5.3 完整 API 示例

```python
import uvicorn
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from table_2_models import Departments, Employees
from database import SessionLocal, engine

app = FastAPI()

# 依赖项：获取数据库会话
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# 创建部门
@app.post("/departments/")
def create_department(name: str, location: str, db: Session = Depends(get_db)):
    db_department = Departments(name=name, location=location)
    db.add(db_department)
    db.commit()
    db.refresh(db_department)
    return db_department

# 查询所有部门
@app.get("/departments/")
def read_departments(db: Session = Depends(get_db)):
    return db.query(Departments).all()

# 查询单个部门
@app.get("/departments/{department_id}")
def read_department(department_id: int, db: Session = Depends(get_db)):
    department = db.query(Departments).filter(Departments.id == department_id).first()
    if not department:
        raise HTTPException(status_code=404, detail="Department not found")
    return department

if __name__ == "__main__":
    uvicorn.run(app="fastapi_sqlalchemy:app", host="0.0.0.0", port=8000, reload=True)
```

> [!tip] 测试接口
> 启动后访问 `http://127.0.0.1:8000/docs` 查看自动生成的交互式文档，可直接在页面上测试接口。

---

## 总结：核心概念速查

```
协程 → async def / await / asyncio.run() / asyncio.gather()
         ↓
ASGI服务器（Uvicorn）
         ↓
FastAPI
  ├── 路由：@app.get/post/put/delete("/path/{param}")
  ├── 参数：路径参数 / 查询参数 / 请求体（Pydantic）
  ├── 路由分发：APIRouter + include_router()
  └── 依赖注入：Depends(get_db)
         ↓
SQLAlchemy ORM
  ├── Base → 模型基类
  ├── Column → 字段定义（类型 + 约束）
  ├── relationship → 关联关系（一对多/一对一/多对多）
  ├── Engine → 数据库连接引擎
  └── Session → CRUD 操作（add/commit/query/delete）
```
