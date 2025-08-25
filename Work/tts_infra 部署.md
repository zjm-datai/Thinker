
```
docker build -t tts_infra:0.0.1 .
```

```
docker save -o tts_infra_0.0.1.tar tts_infra:0.0.1
```

```
scp tts_infra_0.0.1.tar star@183.136.129.102:/mnt/data1/tts_infra
```

```
scp docker-compose.yaml .env star@183.136.129.102:/mnt/data1/tts_infra
```

```
docker load -i tts_infra_0.0.1.tar
```

```bash
sudo apt-get install git-lfs
```

在 pretrained_models 文件夹下面

```
git clone https://www.modelscope.cn/iic/CosyVoice2-0.5B.git
```

在网关服务器上

```bash
curl -s -X POST "http://10.199.14.102:40088/v1/audio/speech" \
  -H "Content-Type: application/json" \
  -d '{
        "input":"我真操了",
        "voice":"Chinese Female",
        "response_format":"mp3",
        "speed":1.0,
        "stream":true
      }'\
  -o out_stream.mp3
```

```bash
curl -s -X POST "http://183.136.129.102:40088/v1/audio/speech" \
  -H "Content-Type: application/json" \
  -d '{
        "input":"我真操了",
        "voice":"Chinese Female",
        "response_format":"mp3",
        "speed":1.0,
        "stream":true
      }'\
  -o out_stream.mp3
```

```bash
curl -s -X POST "http://183.136.129.102:40088/v1/audio/speech" \
  -H "Content-Type: application/json" \
  -d '{
        "input":"我真操了",
        "voice":"Chinese Female",
        "response_format":"mp3",
        "speed":1.0,
        "stream":false
      }'\
  -o out_nonstream.mp3
```

```
curl http://10.199.14.102:40088/health
```

### 注册到 APISIX 上

```bash
curl http://127.0.0.1:30015/apisix/admin/routes/tts-proxy \
  -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" \
  -H "Content-Type: application/json" \
  -X PUT \
  -d '{
    "name": "tts-proxy",
    "desc": "External /api/tts -> internal /v1/audio/speech",
    "uri": "/api/tts",
    "methods": ["POST"],
    "plugins": {
      "proxy-rewrite": {
        "regex_uri": ["/api/tts", "/v1/audio/speech"]
      }
    },
    "upstream": {
      "type": "roundrobin",
      "scheme": "http",
      "nodes": { "10.199.14.102:40088": 1 },
      "timeout": { "connect": 5, "send": 60, "read": 3600 }
    }
  }'
```

```
curl -s -X POST "http://127.0.0.1:40000/api/tts" \
  -H "Content-Type: application/json" \
  -d '{
        "input":"我真操了",
        "voice":"Chinese Female",
        "response_format":"mp3",
        "speed":1.0,
        "stream":true
      }'
```
