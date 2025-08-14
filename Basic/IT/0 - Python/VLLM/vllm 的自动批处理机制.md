vLLM 的服务端会把同时到来的请求拼成批次，统一送进模型做推理，从而大幅提高吞吐。在我们的 FastAPI 进程里就不需要直接占着 GPU；

它只要把请求转给 vLLM 服务器即可。这样 **FastAPI 不会因为一次长推理而阻塞**。

vLLM 用 _PagedAttention_ 管理 KV cache，减少碎片与复制，让批处理更高效——这正是它能高吞吐还稳时延的底层原因。

```
# 单机单卡示例
vllm serve mistralai/Mistral-7B-Instruct-v0.2 \
  --host 0.0.0.0 --port 8001 \
  --gpu-memory-utilization 0.9 \
  --max-model-len 8192
```

--max-num-batched-tokens：

这是 **批次规模关键旋钮**，平衡吞吐与“每 token 延迟（ITL）”。

数值越大吞吐越高，但每个 token 的首字时间可能更慢；过大也会挤占 KV cache。

