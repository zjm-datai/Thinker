## 路由函数

### 数据模型和依赖注入

```python
@router.get("", response_model=WorkerPublic)
async def get_workers(
	session: SessionDep, params: ListParamsDep, name: str = None, search: str = None
):
	...
```

[[Worker 表结构解析]] 里面有 WorkerPublic 的详细信息

```python
# gpustack/server/deps.py
from sqlmodel.ext.asyncio.session import AsyncSession
from gpustack.schemas.common import ListParams
from gpustack.server.db import get_session

SessionDep = Annotated[AsyncSession, Depends(get_session)]
ListParamsDep = Annotated[ListParams, Depends[ListParams]]

# gpustack/schemas/common.py
class ListParams(BaseModel):
	page: int = Query(default=1, ge=1)
	perPage: int = Query(default=100, ge=1, le=100)
	watch: bool = Query(default=False)
```

```python
# gpustack/server/db.py

from sqlalchemy.ext.asyncio import (
    AsyncEngine,
    create_async_engine,
)

_engine = None

def get_engine():
	return _engine

async def get_session():
	async with AsyncSession(_engine) as session:
		yield session

async def init_db(db_url: str):
	global _engine, _session_maker
	if _engine is None:
		connect_args = {}
		if db_url.startswith("sqlite://")
			connect_args = {"check_same_thread": False}
			# use async driver
			db_url = re.sub(r'^sqlite://', 'sqlite+aiosqlite://', db_url)
		elif db_url.startswith("postgresql://"):
			db_url = re.sub(r"^postgresql://", 'postgresql+asyncpg://', db_url)
		else:
			raise Exception(f"Unsupported database URL: {db_url}")

		_engine = create_async_engine(
			db_url,
			echo=DB_ECHO,
			pool_size=DB_POOL_SIZE,
			max_overflow=DB_MAX_OVERFLOW,
			pool_timeout=DB_POOL_TIMEOUT,
			connect_args=connect_args,
		)
		listen_events(_engine)

    await create_db_and_tables(_engine)
```

要注意这里的 global 关键字，所以不要担心我们调用 `get_session` 这个依赖函数的时候得不到引擎。我们在应用初始化的时候就 `init_db` 了，所以到时候调依赖函数的时候，得到的引擎是已经 ok 了的。

```python
# gpustack/server/server.py

class Server:
	...

	async def start(self):
		logger.info("Starting GPUStack server.")
		...
		self._run_migrations()
		await self._prepare_data()
		...
		# 后续由对 fastapi 的启动，路由启动的时候早就有 engine 了所以
	
	async def _prepare_data(self):
		self._setup_data_dir(self.config.data_dir)

		await init_db(self._config.database_url)
		engine = get_engine()
		async with AsyncSession(engine) as session:
			await self._init_data(session)

		logger.debug("Data initialization completed.")
```

好像讲完依赖和数据模型就差不多了。。。

### 分页查询

```python

```




