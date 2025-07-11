
下面我们将从原理到实操，系统地介绍 Git 中的 tag（标签）和分支，HEAD 及切换到 tag 的方法。

## 什么是 Tag

### 定义

标签是对特定提交 commit 的命名引用，用于标记重要的里程碑（如版本发布）。和分支不同，标签一旦创建之后通常不会移动。

### 类型

- **轻量标签（lightweight tag）**：只是对某个提交的简单引用，相当于一个不可变的分支指针。
  
- **附注标签（annotated tag）**：会存储创建者信息、日期、标签说明，并可以用 GPG 签名，适合用作正式发布版本。

### 常见用途

- 版本发布，如 `v1.0.0`、`v2.3.1`

- 标记里程碑、回滚点

## 什么是分支

### 定义

分支是对 commit 历史的可移动指针，代表着一条可以继续提交 commit 的开发线。

### 特点

分支指针会随着新的提交向前移动。

### 与 Tag 的区别

|特性|分支|标签|
|---|---|---|
|指针是否移动|会（随新提交前移）|不会|
|用途|并行开发、协作|标记版本、里程碑|
|可以提交吗|可以|不推荐（需新建分支）|

## HEAD 与 Detached HEAD

- **HEAD**：Git 中指向当前检出（checkout）的分支或提交的符号引用。

- **Detached HEAD**：当你检出一个标签或某个历史提交时，HEAD 不再指向分支名，而是直接指向某个提交对象，此时对仓库做的提交不会更新任何分支，除非手动新建分支来接住这些提交。

```bash
$ git checkout 1.1.3
Note: switching to '1.1.3'.
You are in 'detached HEAD' state.
...
HEAD is now at 1be0d26 fix metadata filter...
```

此时你不在任何分支上，再次 `git commit` 的改动会被“悬挂”在这个临时 HEAD 上，容易遗失。

### 什么是 HEAD

HEAD 是 Git 中的一个符号引用 symbolic reference，它指向我们当前检出的分支或是提交。通俗的说，HEAD 负责告诉 Git：

1. 我们现在站在哪一个分支上

- 绝大多数情况下，HEAD 指向一个分支名（如 `refs/heads/main`），表示你正处于该分支的最新提交上。

- 当你在该分支上执行 `git commit` 时，HEAD 所指的分支指针会自动前移到新提交。

2. 或者，你在“游离”状态（Detached HEAD）

- 如果你检出某个标签（tag）或直接检出一个具体的提交哈希（SHA-1），HEAD 就不再指向分支，而是直接指向那次提交本身。

- 这时做的任何新提交都不会更新任何分支，除非你先创建一个新分支来“接住”它们，否则提交会变成孤儿（难以找回）。

### HEAD 的存储位置

在你的仓库目录下，`.git/HEAD` 文件记录了它的取向。

- 若内容是 `ref: refs/heads/dev`，表示 HEAD 指向本地 `dev` 分支。
 
- 若内容是一串哈希，如 `1be0d26c1f8a2...`，则处于 Detached HEAD。

## 切换到 Tag 的方法

### 使用 git checkout 

```bash
git fetch --tags                     # 更新远程标签列表
git checkout <tagname>
```

结果：产生 **Detached HEAD**，适合仅作查看或临时调试。

### 创建分支并切换

如果想在 tag 基础上继续开发或提交，需要先创建分支来保留改动：

```bash
git checkout -b <new-branch> <tagname>
# 或者
git switch -c <new-branch> <tagname>
```

这样：

- `vast-base` 分支最初指向 tag `1.1.3` 所在提交；

- 后续 `git commit` 会更新 `vast-base` 分支，不会再处于 Detached HEAD。

## 完整示例

```bash
(base) root@11110000:/data/workspaces/zhujunmiao/dify-vectorbase/dify# git checkout 1.1.3
Note: switching to '1.1.3'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -c with the switch command. Example:

  git switch -c <new-branch-name>

Or undo this operation with:

  git switch -

Turn off this advice by setting config variable advice.detachedHead to false

HEAD is now at 1be0d26c1 fix metadata filter not affect in keyword-search and fulltext-search (#16644)
(base) root@11110000:/data/workspaces/zhujunmiao/dify-vectorbase/dify# git checkout -b vast-base 1.1.3
Switched to a new branch 'vast-base'
```


