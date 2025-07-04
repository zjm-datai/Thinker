

```bash
curl -X POST http://127.0.0.1:40000/qwen72/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-lKn0TBxjwG8InOpoTvipKAghtuMKT43XSRo9wNZJjmME7M0D" \
  -d '{
    "model": "Qwen2.5-72B-Instruct",
    "messages": [
      {"role": "user", "content": "你好"}
    ],
    "stream": true 
  }'
```

```bash
{"msg":"请求访问：/tool/apisix/massAuthentication，认证失败，无法访问系统资源","code":401}
```

![[Pasted image 20250703155103.png]]

下订单，获取令牌

```
1e7260abca766b05d5f93bf6d04b9ed8
```

```bash
curl -X POST http://10.17.105.16:40000/qwen72/v1/chat/completions \
  -H "Mass-Auth-Token: 1b741aa23f08b4ed43fb95d432a964af" \
  -H "Mass-Ai-Service: http://10.17.105.16:40000/qwen72" \
  -H "Authorization: Bearer sk-lKn0TBxjwG8InOpoTvipKAghtuMKT43XSRo9wNZJjmME7M0D" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{
    "model": "Qwen2.5-72B-Instruct",
    "messages": [
      {"role": "user", "content": "你好"}
    ],
    "stream": true 
  }'
```

```bash
curl -X POST http://10.17.105.16:40000/qwen72/v1/chat/completions \
  -H "Mass-Auth-Token: 1b741aa23f08b4ed43fb95d432a964af" \
  -H "Mass-Ai-Service: http://10.17.105.16:40000/qwen72" \
  -H "Authorization: Bearer sk-lKn0TBxjwG8InOpoTvipKAghtuMKT43XSRo9wNZJjmME7M0D" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen2.5-72B-Instruct",
    "messages": [
      {"role": "user", "content": "你好"}
    ] 
  }'
```

```bash
curl -X POST http://192.168.80.100:38080/v1/chat/completions \
  -H "Authorization: Bearer sk-lKn0TBxjwG8InOpoTvipKAghtuMKT43XSRo9wNZJjmME7M0D" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen2.5-72B-Instruct",
    "messages": [
      {"role": "user", "content": "你好"}
    ] 
  }'
```

响应正常

```
{"id":"endpoint_common_132634","object":"chat.completion","created":1751595682,"model":"Qwen2.5-72B-Instruct","choices":[{"index":0,"message":{"role":"assistant","content":"你好！有什么我可以帮助你的吗 ？","tool_calls":null},"finish_reason":"stop"}],"usage":{"prompt_tokens":30,"completion_tokens":9,"total_tokens":39},"prefill_time":62,"decode_time_arr":[40,40,40,40,40,40,40,41]}
```

```bash
curl -X POST http://192.168.80.100:38080/v1/chat/completions \
  -H "Authorization: Bearer sk-lKn0TBxjwG8InOpoTvipKAghtuMKT43XSRo9wNZJjmME7M0D" \
  -d '{
    "model": "Qwen2.5-72B-Instruct",
    "messages": [
      {"role": "user", "content": "你好"}
    ] 
  }'
```

响应失败

```
{"error":{"message":"当前分组 default 下对于模型  无可用渠道 (request id: 2025070410183680985070pKQP74zj)","type":"new_api_error"}}
```

```bash
curl -X POST http://192.168.80.100:38080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen2.5-72B-Instruct",
    "messages": [
      {"role": "user", "content": "你好"}
    ] 
  }'
```

查看插件配置

```bash
curl http://127.0.0.1:30015/apisix/admin/plugin_configs/9cbff318-043a-4507-8696-cc48340a0a82 \
  -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1"
```

```bash
curl -X POST http://192.168.120.44:9998/v1/chat/completions \
  -H "Authorization: Bearer gpustack_4f50a5c7bad4fd01_7c054bcb8d3cf921e9b1fa43517fa81b" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen2.5-32B-Instruct",
    "messages": [
      {"role": "user", "content": "你好"}
    ] 
  }'
```

```bash
curl -X POST http://10.17.105.16:40000/qwen25/v1/chat/completions \
  -H "Mass-Auth-Token: 1b741aa23f08b4ed43fb95d432a964af" \
  -H "Mass-Ai-Service: http://10.17.105.16:40000/qwen72" \
  -H "Authorization: Bearer gpustack_4f50a5c7bad4fd01_7c054bcb8d3cf921e9b1fa43517fa81b" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen2.5-32B-Instruct",
    "messages": [
      {"role": "user", "content": "你好"}
    ]
  }'
```


