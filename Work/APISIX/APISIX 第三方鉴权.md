

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
  -H "Mass-Auth-Token: c5ba51be875286e93a567d64c1ccc95b" \
  -H "Mass-Ai-Service: http://10.17.105.16:40000/qwen72" \
  -H "Authorization: Bearer sk-lKn0TBxjwG8InOpoTvipKAghtuMKT43XSRo9wNZJjmME7M0D" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen2.5-72B-Instruct",
    "messages": [
      {"role": "user", "content": "你好"}
    ],
    "stream": true 
  }'
```

