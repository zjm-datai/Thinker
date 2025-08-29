
进入 vastbase 服务器之后先进行：

```
su - vastbase
```

```
vsql -h 127.0.0.1 -p 5432 -U dify -d dify
```

```
scp -r dify.dump dify_plugin.dump root@192.168.120.69:/tmp
```

```
Unicom#123
```

```
pg_restore -U dify -d dify dify.dump
```

```
pg_restore -U dify -d dify_plugin dify_plugin.dump
```

```
[vastbase@ecs tmp]$ pg_restore -U dify -d dify dify.dump
could not open output file "/tmp/dify.dump": 权限不够
```

```
[vastbase@ecs tmp]$ ls -l /tmp/dify.dump
-rw-r--r-- 1 root root 7033828244  8月 28 16:13 /tmp/dify.dump
```

```bash
pg_restore -U dify -d dify /tmp/dify.dump
```

版本太低，傻逼

---

```
pg_dump -U postgres -d dify --format=plain --no-owner --no-privileges > dify.sql
```

```
scp -r dify.sql dify_plugin.sql root@192.168.120.69:/home/vastbase
```

---

```
-- 数据库/ schema 所有者改成 dify
ALTER DATABASE dify OWNER TO dify;
ALTER SCHEMA public OWNER TO dify;

-- 现有对象的使用权限
GRANT USAGE ON SCHEMA public TO dify;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES    IN SCHEMA public TO dify;
GRANT USAGE, SELECT, UPDATE          ON ALL SEQUENCES IN SCHEMA public TO dify;

-- 把管理员创建的对象统一过户给 dify（按你环境里可能的管理员名过户）
REASSIGN OWNED BY vastbase TO dify;
REASSIGN OWNED BY postgres TO dify;
```