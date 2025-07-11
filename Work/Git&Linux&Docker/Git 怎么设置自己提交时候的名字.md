
### 全局设置

如果我们希望在所有的 git 仓库中都使用相同的用户名和邮箱，可以执行如下的命令：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

### 针对单个仓库进行设置

要是为特定仓库单独设置用户名和邮箱，只需去掉 `--global` 参数，在仓库目录下执行：

```bash
git config user.name "你的名字" 
git config user.email "你的邮箱@example.com"
```

### 查看已有的配置

```bash
git config --list
```