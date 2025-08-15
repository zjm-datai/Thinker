
```bash
uv venv --python 3.12 --seed
source .venv/bin/activate
uv pip install vllm --torch-backend=auto
```

uv 可以通过 `--torch-backend=auto` （或 `UV_TORCH_BACKEND=auto`）在运行时检查已安装的 CUDA 驱动版本，从而选择合适的 pytorch 后端。要选择特定的后端（例如 `cu126`），请设置 `--torch-backend=cu126`（或 `UV_TORCH_BACKEND=cu126`）。

另一种便捷的方法是使用 `uv run` 配合 `--with [dependency]` 选项，这允许您运行诸如 `vllm serve` 这样的命令而无需创建任何永久环境。

```bash
uv run --with vllm vllm --help
```

### 离线批量推理



### 兼容 OpenAI 的服务器

VLLM 可以部署为实现 OpenAI API 协议的服务器。这使得 vLLM 可以作为使用 OpenAI API 的应用程序的即插即用替代品。默认情况下，它在 `https://:8000` 启动服务器。您可以使用 `--host` 和 `--port` 参数指定地址。服务器目前一次托管一个模型，并实现了诸如列出模型，创建聊天补全和创建补全等端点。

[OpenAI 接口文档](https://platform.openai.com/docs/api-reference)

运行以下命令，使用 Qwen2.5-1.5B-Instruct 模型启动 vLLM 服务器

```bash
vllm server Qwen/Qwen2.5-1.5B-Instruct
```

>[!注意]
>默认情况下，服务器使用存储在分词器中预定义的聊天模板。可以参阅这里了解如何覆盖它。

