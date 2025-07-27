FastAPI support dependencies that do some extra steps after finishing. 

Sometimes also called "exit code", "cleanup code", "teardown code", "closing code", "context manager exit code", etc

To do this, use `yield` instead of `return` , and write the extra steps (code) after.

>[!tips]
>Make sure to use `yield` one single time per dependency.

### 带有 yield 的数据库依赖项



### A dependency with `yield` and `try`

If you use a try block in a dependency with `yield` , you will receive any exception that exception that was thrown when using the dependency.

For example, if some code at some point in the middle, in another dependency or in a path operation, made a database transaction "rollback" or create any other error, you will receive the exception in your dependency.

FastAPI 允许通过依赖项（`Depends`）管理资源（如数据库连接、认证逻辑等）。当程序在**依赖项、路径操作函数或其他依赖链中**发生异常（例如数据库事务回滚）时，异常会向上传播到依赖项中。

- **异常来源**：可能来自数据库操作（如事务回滚）、第三方服务调用、权限验证失败等。
- **异常传播路径**：例如，路径操作函数 → 子依赖项 → 主依赖项，最终异常会被依赖项捕获。

So, you can look for that specfic exception inside the dependency with `except SomeException` .

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.exc import SQLAlchemyError

app = FastAPI()

def get_db():
	engine = create_engine("sqlite:///test.db")
	SessionLocal = sessionmake(autocommit=False, autoflush=False, bind=engine)
	db = SessionLocal()

	try:
		yield db
	except SQLAlchemyError as e:
		# 捕获数据库操作异常
		db.rollback() # 手动回滚
		raise HTTPException(status_code=500, detail="数据库操作失败")
	finally:
		db.close()

# 路径操作函数，依赖 get_db
@app.post("/items/")
def create_item(item: dict, db: Session = Depends(get_db)):
    try:
        # 假设这里执行数据库操作时发生异常
        db.add(item)
        db.commit()
    except Exception as e:
        # 此处异常会传播到 get_db 依赖项中
        raise e
```

In the same way, you can use `finally` to make sure the exit steps are executed, no matter if there was an exception or not.

### Sub-dependencies with yield

You can have sub-dependencies and "trees" of sub-dependencies of any size and shape, and any of all of them can use yield.

FastAPI will make sure that the exit code in each dependency with yield is run in the correct order.

For example, `dependency_c` can have a dependency on `dependency_b` , and `dependency_b` on `dependency_a` :

```python
from typing import Annotaed
from fastapi import FastAPI

async def dependency_a():
	dep_a = generate_dep_a()
	try:
		yield dep_a
	finally:
		dep_a.close()

async def dependency_b(dep_a: Annotated[DepA, Depends(dependency_a)]):
	dep_b = generate_dep_b()
	try:
		yield dep_b
	finally:
		dep_b.close(dep_a)

async def dependency_c(dep_b: Annotated[DepB, Depends(dependency_b)]):
	dep_c = generate_dep_c()
	try:
		yield dep_c
	finally:
		dep_c.close(dep_b)
```

The same way, you could have some dependencies with yield and some other dependencies with return, and have some of those depend on some of the others.

And you could have a single dependency that requires several dependencies with `yield` , etc.

You can have any combinations of dependencies that you want. FastAPI wll make sure everything is run in the correct order.

>[!Technical Details]
>This works thanks to Python's [[Context Managers]]
> 
>FastAPI uses them internally to achieve this.

### Dependencies with yield and HTTPException

You saw that you can dependencies with yield and have try blocks that catch exceptions.

The same way, you could raise an `HTTPException` or similar in the exit code, after the `yield` .

>[!Tip]
>
>This is a somewhat advanced technique, and in most of the cases you will not really need it, as you can raise exceptions (including `HTTPException`) from inside of the rest of your application code, for example, in the path operation function.

```python
from typing import Annotated

from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

data = {
	"plumbus": {"descrption": "a...", "owner": "a"},
	"portal-gun": {"descrption": "b...", "owner": "b"}
}

class OwnerError(Exception):
	pass

def get_username():
	try:
		yield "Rick"
	except OwnerError as e:
		raise HTTPException(status_code=400, detail=f"Owner error: {e}")

@app.route("/items/{item_id}")
def get_item(item_id: str, username: Annotated[str, Depends(get_username)]):
	if item_id not in data:
		raise HTTPException(status_code=404, detail="Item not found")
	item = data[item_id]
	if item["owner"] != username:
		raise OwnerError(username)

	return item
```

An alternative you could use to catch exceptions (and possibly also raise another HTTPException) is create a [[Custom Exception Handler]] .

### Dependencies with yield and except

If you can catch an exception using except in a dependency with yield and you donnot raise it again (or rasie a new exception), FastAPI will not be able to notice there was an exception, the same way that would happen with regular python:

```python
from typing import Annotated
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

class InternalError(Exception):
	pass 

def get_username():
	try:
		yield "Rick"
	except InternalError:
		print("Oops, ...")

@app.get("/items/{item_id}")
def get_item(item_id: str, username: Annotated[str, Depends(get_username)]):
	if item_id == "portal-gun":
		raise InternalError(
			f"The ..."
		)
	if item_id != "plumbus":
		raise HTTPException(
			status_code=404, detail="Item ..."
		)
	return item_id
```

In this case, the client will see an HTTP 500 internal Server Error response as it should, given that we are not raising an HTTPException or similar, but the server will not have any logs or any other indication of what was the error.







