在 linux 系统中，程序运行会产生三种输出流：

- 标准输入 stdin 编号为 0 用于接收程序的输入数据
- 标准输出 stdout 编号为 1 程序正常运行的输出内容
- 标准错误 stderr 编号为 2 程序运行的错误信息

### 为什么需要重定向标准错误？

默认情况下，标准输出和标准错误都会显示在终端上。但是当我们使用 > 符号重定向输出的时候，默认只重定向标准输出（stdout），标准错误（stderr）仍然会显示在终端上。

例如，下面的命令只会将标准输出写入 `output.log`，错误信息还是会显示在屏幕上：

```bash
python3 your_script.py > output.log &
```

如果你希望错误信息也保存到同一个日志文件中，就需要额外重定向标准错误。

### 2>&1 的含义

`2>&1` 是 shell 中重定向标准错误的语法，具体解释：

- `2` 代表标准错误（stderr）
- `>` 表示重定向
- `&1` 中的 `&` 表示引用，这里引用的是标准输出（stdout）的目标位置

所以 `2>&1` 的意思是：**将标准错误（2）重定向到与标准输出（1）相同的目标位置**。

### 常见的组合方式

#### 将标准输出和标准错误分别重定向到不同文件

```bash
python3 your_script.py > output.log 2> error.log &
```

#### 将标准输出和标准错误合并到同一个文件

```bash
python3 your_script.py > output.log 2>&1 &
```

