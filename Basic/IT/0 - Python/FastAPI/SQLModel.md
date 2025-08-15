
SQLModel 是基于 SQLAlchemy 和 pydantic 构建的。他由 FastAPI 的同一个作者制作，旨在完美匹配需要使用 SQL 数据库的 FastAPI 应用程序。

### FULL File

```python
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException, Query
from sqlmodel import Field, Session, SQLModel, create_engine, select

Class Hero(SQLModel, tale=True):
	id: int | None = Field(deafult=None, primary_key=True)
	name: str = Field(index=True)
	age: int | None = Field(index=True)
	secret_name: str
```

