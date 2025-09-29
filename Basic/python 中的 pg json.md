
```python
from sqlalchemy.ext.mutable import MutableDict
from sqlalchemy.dialects.postgresql import JSONB


class ChatMessage(BaseModel, table=True):
    __tablename__ = "chat_message"

    id: str = Field(default_factory=lambda: str(uuid.uuid4()), primary_key=True)
    chat_session_id: str = Field(index=True)

    role: str = Field(index=True)  # "user" | "assistant" | "system"
    content: str = Field(default="")
    status: Optional[str] = Field(default=None, index=True)  # streaming | completed | error
	
	# 用这个方式的话修改不动字典
    # meta: Optional[Dict[str, Any]] = Field(
    #     sa_column=Column(JSON), default=None
    # )
  
    # 关键改动：MutableDict + default_factory=dict

    meta: Dict[str, Any] = Field(
        default_factory=dict,
        sa_column=Column(MutableDict.as_mutable(JSONB))  # 若是其他：MutableDict.as_mutable(JSON)
    )
```


### 如何直接删库

```
docker exec -it postgres sh
```

**切换到 postgres 用户**（PostgreSQL 通常以该用户运行）

```
su - postgres
```

```sh
psql
```

查看所有数据库

```
\l
```

```
postgres=# DROP DATABASE IF EXISTS digital_person;
ERROR:  database "digital_person" is being accessed by other users
DETAIL:  There are 74 other sessions using the database.
```

切换到 postgres 

```sql
\c postgres
```

终止所有连接到 digital_person 数据库的会话：

```sql
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE datname = 'digital_person';
```

这条命令会返回一个表示成功终止的会话数量的结果集。

再次尝试删除数据库：

```sql
DROP DATABASE IF EXISTS digital_person;
```

pg_terminate_backend(pid) 会强制终止指定进程 ID 的数据库连接

pg_stat_activity 是 PostgreSQL 系统视图，存储当前数据库活动会话信息

必须先切换到其他数据库（如 postgres）才能终止目标数据库的连接

重新创建数据库 

```sql
create database digital_person;
```

