
```
docker stats
```

列出所有正在运行的容器及其各项资源指标，包括容器 ID、容器名称、CPU 百分比、内存使用量 / 限制、内存百分比、网络 I/O、磁盘 I/O 等。

若只想关注某个或某些特定容器，可在 `docker stats` 命令后指定容器名或容器 ID，如 `docker stats my - container`。

- **自定义输出格式**：通过 `--format` 选项可按照需要展示信息，如 `docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"`，仅显示容器 ID、CPU 使用率和内存使用量。

---

**使用 docker top 命令**：`docker top` 命令与 Linux 系统中的 `ps` 命令基本一致，用于查看容器内正在运行的进程信息。例如 `docker top <container_name_or_id>`，还可添加 `ps` 命令的参数来指定显示的列，如 `docker top <container_name_or_id> -o pid,stat,cmd`，查看容器内进程的 PID、状态和命令。

**查看容器内进程资源占用**：进入容器内部，使用 Linux 系统命令查看容器内进程的资源占用情况。如`docker exec -it <container_id> /bin/bash` 进入容器，再执行 `top` 或 `htop` 命令查看容器内进程的 CPU 和内存占用率，识别异常进程。

**查看磁盘占用情况**：使用 `docker system df` 命令可查看镜像、容器、卷、构建缓存的总占用， `docker system df -v` 可列出每个镜像、容器、数据卷的具体体积和关联关系。若想查看容器内某个挂载路径的磁盘占用，可进入容器执行 `du -sh /挂载路径`。

---

底层原理可以参考：

[[Docker 容器对底层资源的访问]]

---

### 实操

```bash
docker top 31be6ba2818e
```

```bash
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD

root                3768701             3768681             0                   15:18               ?                   00:00:00            uv run uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4

root                3769156             3768701             0                   15:19               ?                   00:00:11            /app/.venv/bin/python /app/.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4

root                3769158             3769156             0                   15:19               ?                   00:00:00            /app/.venv/bin/python -c from multiprocessing.resource_tracker import main;main(6)

root                3769159             3769156             0                   15:19               ?                   00:00:16            /app/.venv/bin/python -c from multiprocessing.spawn import spawn_main; spawn_main(tracker_fd=7, pipe_handle=9) --multiprocessing-fork

root                3769160             3769156             0                   15:19               ?                   00:00:16            /app/.venv/bin/python -c from multiprocessing.spawn import spawn_main; spawn_main(tracker_fd=7, pipe_handle=13) --multiprocessing-fork

root                3769161             3769156             0                   15:19               ?                   00:00:16            /app/.venv/bin/python -c from multiprocessing.spawn import spawn_main; spawn_main(tracker_fd=7, pipe_handle=17) --multiprocessing-fork

root                3769162             3769156             0                   15:19               ?                   00:00:16            /app/.venv/bin/python -c from multiprocessing.spawn import spawn_main; spawn_main(tracker_fd=7, pipe_handle=21) --multiprocessing-fork
```


`docker top` 命令的输出格式类似于 Linux 系统的 `ps` 命令，各列含义如下：

1. **UID**  
    运行进程的用户 ID（这里均为 `root`，表示以超级用户权限运行）。

2. **PID**  

    进程在宿主机上的实际进程 ID（不是容器内的 PID）。例如：

    - `3768701` 是主进程（UVicorn 启动命令）。
    - `3769156` 是主 Python 进程（`uvicorn main:app`）。
    - `3769158`、`3769159` 等是子进程（由主进程派生的工作进程）。

3. **PPID**  

    父进程的 PID。例如：

    - `3769156` 的父进程是 `3768701`。
    - `3769158`、`3769159` 等的父进程是 `3769156`。

4. **C**  

    CPU 使用率（这里均为 `0`，表示采样瞬间 CPU 使用率较低）。

5. **STIME**

    进程启动时间（如 `15:18`、`15:19`）。

6. **TTY**  

    终端设备（`?` 表示没有关联终端）。

7. **TIME**  

    进程累计使用的 CPU 时间（如 `00:00:16` 表示某个工作进程累计使用了 16 秒 CPU 时间）。

8. **CMD**  

    启动进程的命令。例如：

    - `uv run uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4`：启动 Uvicorn 服务器，指定 4 个工作进程。
    - `/app/.venv/bin/python -c from multiprocessing.spawn import spawn_main...`：多进程工作进程的启动命令。

#### 容器进程结构分析

从输出可以看出，这个容器运行的是一个基于 **Uvicorn** 的 FastAPI 应用（或其他 ASGI 框架），且配置了 `--workers 4`（4 个工作进程）。具体结构如下：

1. **主进程（PID 3768701）**  
    启动命令为 `uv run uvicorn main:app`，负责管理整个应用生命周期。

2. **Python 主进程（PID 3769156）**  
    执行 `uvicorn main:app` 命令，是实际的 Uvicorn 服务器进程。

3. **工作进程（PID 3769159~3769162）**  
    由主进程派生的 4 个子进程，每个进程处理一部分客户端请求，实现并行处理。

#### 常见问题排查提示

1. **进程数量异常**

    - 如果发现工作进程数量不足 4 个，可能有进程崩溃，需检查应用日志。
    - 如果进程数量过多（远超 4 个），可能存在进程泄漏或配置错误。

2. **CPU 使用率过高**

    - 若某个工作进程的 CPU 使用率持续接近 100%，可能是代码中存在死循环或高计算负载的逻辑。

3. **内存异常**

    - 单独使用 `docker top` 无法查看内存信息，需结合 `docker stats` 或 `docker exec` 进入容器执行 `free -h`、`ps aux` 等命令。

#### 建议组合命令

##### 查看实时资源占用

```bash
docker stats 31be6ba2818e
```

##### 进入容器查看详细信息

```bash
docker exec -it 31be6ba2818e bash # 进入后执行： 
top # 查看实时进程状态 
free -h # 查看内存使用 
ps aux # 查看所有进程（含内存、CPU 详细信息）
```

##### 查看应用日志

```bash
docker logs 31be6ba2818e
```


#### LSOF

当我们使用 `lsof -i :30017` 查看端口情况的时候发现：

```
docker-pr 3769002 root    7u  IPv4 862294455      0t0  TCP *:30017 (LISTEN)
docker-pr 3769010 root    7u  IPv6 862294456      0t0  TCP *:30017 (LISTEN)
```

这是因为在宿主机上执行 `lsof -i :30017` 时，你看到的并不是容器内的 Uvicorn 进程，而是两个 **docker-proxy** 进程在监听 30017（分别对应 IPv4 和 IPv6）。这是因为：

**端口映射由 docker-proxy 实现**  

当你使用 `-p 30017:8000` 发布端口时，Docker（在默认启用了用户态代理的情况下）会为每个映射的端口启动一个 **docker-proxy** 进程，在宿主上监听 30017，并将流量转发到容器内的 8000 端口。

[Docker Userland Proxy](https://blog.ipspace.net/kb/DockerSvc/40-userland-proxy/)

**容器进程在独立的网络命名空间**  

Uvicorn 等容器内部进程运行于各自的网络命名空间，并在容器内监听 8000 端口。宿主机上的 `lsof`/`netstat` 默认只展示宿主网络命名空间的套接字，不会列出容器命名空间内的监听端口

https://unix.stackexchange.com/questions/539797/docker-mapped-port-doesnt-show-up-on-netstat-ss-and-lsof

**为什么会有两个 docker-proxy 进程？**  

docker-proxy 会为 IPv4 和 IPv6 分别创建监听套接字，故 `lsof -i :30017` 会列出两条记录：

```bash
docker-pr … TCP *:30017 (LISTEN)   # IPv4
docker-pr … TCP *:30017 (LISTEN)   # IPv6
```

如果我们要查看容器内的监听的话，可以进去看，但是要看的是 8000 而且需要安装了 lsof 这样的工具。

