
在 VSCode 中，你可以通过设置工作区的 **排除规则** 来隐藏 `__pycache__` 文件夹，使其在资源管理器（Explorer）中不显示。以下是具体做法：

---

### 方法一：通过 `settings.json` 设置隐藏 `__pycache__`

1. 打开 VSCode。
    
2. 在左侧资源管理器中点击齿轮 → 选择“**设置**”。
    
3. 点击右上角的 **“打开设置（JSON）”图标**（或使用快捷键 `Ctrl+Shift+P`，输入 `Preferences: Open Settings (JSON)`）。
    
4. 在 `settings.json` 中添加以下内容：
    

```json
"files.exclude": {
  "**/__pycache__/": true
}
```

这将使所有项目中名为 `__pycache__` 的文件夹都不在 VSCode 文件浏览器中显示，但实际并未删除它。

---

### 方法二：仅对当前工作区隐藏

如果你只想对当前项目隐藏它，而不影响全局设置：

1. 打开 `.vscode/settings.json`（如果没有这个文件夹，就新建一个）。
    
2. 添加：

```json
{
  "files.exclude": {
    "**/__pycache__/": true
  }
}
```

---

### 可选：同时排除 `.pyc` 文件

你也可以顺便隐藏 `.pyc` 等缓存文件：

```json
"files.exclude": {
  "**/__pycache__/": true,
  "**/*.pyc": true
}
```

---

如有需要，我也可以帮你写一个 `.vscode/settings.json` 的模板文件。需要吗？