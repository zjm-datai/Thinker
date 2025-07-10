下面给出一个最小化的 FastAPI 示例项目，演示“多个协程并发使用同一个 `AsyncSession`”时极易复现的 `InterfaceError: another operation is in progress` 错误。你可以按此项目组织目录、安装依赖、运行后多次并发调用来验证。

---

### 项目结构

```
fastapi-async-session-demo/
├── requirements.txt
├── models.py
├── database.py
└── main.py
```

---

#### 1. `requirements.txt`

```txt
fastapi
uvicorn[standard]
sqlmodel
asyncpg
```

---

#### 2. `models.py`

```python
from sqlmodel import SQLModel, Field

class Item(SQLModel, table=True):
    id: int = Field(default=None, primary_key=True)
    name: str
```

---

#### 3. `database.py`

```python
from sqlmodel import SQLModel, create_engine
from sqlmodel.ext.asyncio.session import AsyncSession

DATABASE_URL = "postgresql+asyncpg://postgres:postgres@localhost/demo"

# 用于演示的单一 engine
engine = create_engine(DATABASE_URL, echo=True, future=True)

# 共享一个 AsyncSession 类
from sqlalchemy.ext.asyncio import async_sessionmaker
AsyncSessionLocal = async_sessionmaker(
    engine, expire_on_commit=False
)
```

> **注意**：`expire_on_commit=False` 只是为了简化重载演示，非核心。

---

#### 4. `main.py`

```python
import asyncio
from fastapi import FastAPI, HTTPException
from sqlmodel import select
from database import AsyncSessionLocal, engine
from models import Item

app = FastAPI()

@app.on_event("startup")
async def on_startup():
    # 初始化表
    async with engine.begin() as conn:
        await conn.run_sync(SQLModel.metadata.create_all)
    # 插入几条示例数据
    async with AsyncSessionLocal() as session:
        session.add_all([Item(name="foo"), Item(name="bar")])
        await session.commit()

@app.get("/test_shared_session")
async def test_shared_session():
    """
    演示：使用同一个 AsyncSession，给两个任务并发查询。
    会在高并发下抛出 InterfaceError。
    """
    session = AsyncSessionLocal()  # <— 共享给两个并发任务

    async def query_one():
        stmt = select(Item).where(Item.name == "foo")
        result = await session.execute(stmt)
        await asyncio.sleep(0.1)  # 延迟，制造并发重叠
        return result.scalars().first()

    async def query_two():
        stmt = select(Item).where(Item.name == "bar")
        result = await session.execute(stmt)
        await asyncio.sleep(0.1)
        return result.scalars().first()

    # 并发触发两个查询操作
    try:
        res1, res2 = await asyncio.gather(query_one(), query_two())
    except Exception as e:
        # 捕获并返回给客户端
        raise HTTPException(status_code=500, detail=str(e))
    finally:
        await session.close()

    return {"res1": res1, "res2": res2}
```

---

### 如何验证错误

1. **启动 PostgreSQL**，并确保 `postgres:postgres@localhost/demo` 可连通。
    
2. `cd fastapi-async-session-demo`，执行：
    
    ```bash
    pip install -r requirements.txt
    uvicorn main:app --reload
    ```
    
3. 并行多次调用 `/test_shared_session`，例如用 `ab`：
    
    ```bash
    ab -n 10 -c 5 http://127.0.0.1:8000/test_shared_session
    ```
    
4. 在控制台或客户端中你会看到类似：
    
    ```
    InterfaceError: another operation is in progress
    ```
    
    或者其他因并发排队导致的异步 I/O 错误。
    

---

### 正确的“每任务一 Session”写法

要避免此问题，只需将 `AsyncSessionLocal()` 的创建移动到每个协程内部，确保每个任务都有自己独立的连接和事务上下文：

```python
@app.get("/test_independent_sessions")
async def test_independent_sessions():
    async def query_one():
        async with AsyncSessionLocal() as session:
            stmt = select(Item).where(Item.name == "foo")
            result = await session.execute(stmt)
            await asyncio.sleep(0.1)
            return result.scalars().first()

    async def query_two():
        async with AsyncSessionLocal() as session:
            stmt = select(Item).where(Item.name == "bar")
            result = await session.execute(stmt)
            await asyncio.sleep(0.1)
            return result.scalars().first()

    res1, res2 = await asyncio.gather(query_one(), query_two())
    return {"res1": res1, "res2": res2}
```

在这种模式下，再做高并发测试，就不会出现 `InterfaceError` 之类的错误，因为每个协程拿到的都是自己专属的 `AsyncSession`。

---

通过这个实际项目，你就可以亲手验证“一个轮次共用同一个 `AsyncSession`” 的风险，以及如何用“每协程一个 Session”彻底解决。