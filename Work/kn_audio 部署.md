
首先我们需要离线把 docker 和 docker-compose 安装上去

[[Linux 环境离线安装 docker&docker-compose]]

```bash
docker save -o kn_audio_0.0.3.tar kn_audio:0.0.3
```

```shell
docker load -i kn_audio_0.0.3.tar
```

验证 md5 是否相同

首先我们在 linux 中验证：

```
md5sum kn_audio_0.0.1.tar
```

```
(kn-audio) (base) root@11110000:/data/services/kn_audio/docker/deploy_files# md5sum kn_audio_0.0.1.tar
50bd29fd71053a0012846955c3299dd8  kn_audio_0.0.1.tar
```

在本机下载之后：

```
CertUtil -hashfile kn_audio_0.0.1.tar MD5
```

```
C:\Desktop>CertUtil -hashfile kn_audio_0.0.1.tar MD5
MD5 的 kn_audio_0.0.1.tar 哈希:
50bd29fd71053a0012846955c3299dd8
CertUtil: -hashfile 命令成功完成。
```

使用 todesk 传输到其他 windows 中之后

b4ff6867e054c74145a8ed0a3a8e124

于是决定使用 minio 进行传输

---

```
curl -X 'GET' \
'http://192.168.167.232:8088/v1/health' \ 
-H 'accept: application/json'
```

---

### 中间件部署

```
scp docker-install.tar.gz root@192.168.167.233:/root/ntdata 
```

```
docker-compose --env-file middleware.env up -d
```

```bash
docker exec -it middleware-postgres-1 sh
```

```bash
psql -U kn_audio -d postgres_db
```

```sql
create database kn_audio;
```

```sql
\c kn_audio
```

```sql
-- 创建音频文件表
CREATE TABLE audio_file (
    id SERIAL PRIMARY KEY,
    person_id TEXT NOT NULL,
    conversation_id TEXT NOT NULL,
    file_url TEXT NOT NULL UNIQUE,
    transcription_content TEXT,
    created_at TIMESTAMP NOT NULL
);

-- 创建索引
CREATE INDEX ix_audio_file_conversation_id ON audio_file (conversation_id);
CREATE INDEX ix_audio_file_created_at ON audio_file (created_at);
CREATE INDEX ix_audio_file_person_id ON audio_file (person_id);
```

```
kn_audio=# \d audio_file
                                              Table "public.audio_file"
        Column         |            Type             | Collation | Nullable |                Default
-----------------------+-----------------------------+-----------+----------+----------------------------------------
 id                    | integer                     |           | not null | nextval('audio_file_id_seq'::regclass)
 person_id             | text                        |           | not null |
 conversation_id       | text                        |           | not null |
 file_url              | text                        |           | not null |
 transcription_content | text                        |           |          |
 created_at            | timestamp without time zone |           | not null |
Indexes:
    "audio_file_pkey" PRIMARY KEY, btree (id)
    "audio_file_file_url_key" UNIQUE CONSTRAINT, btree (file_url)
    "ix_audio_file_conversation_id" btree (conversation_id)
    "ix_audio_file_created_at" btree (created_at)
    "ix_audio_file_person_id" btree (person_id)
```

#### 修改 .env 文件

```
# ==============================================
# Uvicorn 服务配置
# ==============================================
# 服务监听的端口（默认：8088）
PORT=8088
# 用于发布文件的 IP 地址，需要可以给语音模型访问到
IP=192.168.167.232

# ==============================================
# STT（语音转文本）模型配置
# ==============================================
STT_MODEL=Qwen2-Audio-7B
STT_API_URL=http://10.156.1.20:9998/v1/chat/completions
STT_API_KEY=gpustack_3f56322609f2aafe_75bd70ebeaffa310b70c0db026cfcdfd


# ==============================================
# TTS（文本转语音）模型配置
# ==============================================
TTS_MODEL=CosyVoice2-0.5B
TTS_API_URL=http://10.156.1.20:9998/v1/audio/speech
TTS_API_KEY=gpustack_3f56322609f2aafe_75bd70ebeaffa310b70c0db026cfcdfd


# =======================
# MINIO 配置信息
# =======================
# 服务端点（必填）：格式为 "主机:端口"
MINIO_ENDPOINT=192.168.167.233:9000
# 访问密钥（必填）：对应 MinIO 的 access key
MINIO_ACCESS_KEY=minioadmin
# 密钥（必填）：对应 MinIO 的 secret key
MINIO_SECRET_KEY=minioadmin
# 是否使用 HTTPS（可选）：默认为 true
MINIO_SECURE=false
# 存储桶名称（必填）：文件上传的目标桶名（需提前在 MinIO 中创建）
MINIO_BUCKET_NAME=audio
# 本地下载目录（可选）：文件下载到本地的默认文件夹
MINIO_UPLOAD_FOLDER=/app/api/storage/uploads
# MinIO 中的路径（可选）：存储桶下的子路径，默认为空（直接存储在桶根目录）
MINIO_PATH=/tmp

# =======================
# CORS Configuration
# =======================
ENABLE_CORS=true
ALLOW_ORIGINS=["*"]
ALLOW_CREDENTIALS=false
ALLOW_METHODS=["*"]
ALLOW_HEADERS=["*"]

# =======================
# 日志配置信息
# =======================

# 是否开启 debug（会把 root logger 级别设为 DEBUG）
LOG_DEBUG=true

# 是否写入文件
LOG_TO_FILE=false

# 日志目录 & 文件名
LOG_DIR=/app/api/storage/logs
LOG_FILENAME=application.log

# 轮转参数（基于时间）
LOG_WHEN=midnight
LOG_INTERVAL=1
LOG_BACKUP_COUNT=7

# 日志级别（root 的默认级别，INFO / DEBUG / WARNING / ERROR / CRITICAL 皆可）
LOG_LEVEL=INFO

# 固定时区（IANA 时区名）
LOG_TIMEZONE=Asia/Shanghai

# 需要禁用的 noisy 第三方日志（逗号分隔）
LOG_DISABLE_LOGGERS=httpcore.connection,httpcore.http11,httpcore.proxy,httpx,asyncio,aiocache.base,aiosqlite,urllib3.connectionpool,multipart.multipart,apscheduler.scheduler,apscheduler.executors.default,tzlocal,alembic.runtime.migration,python_multipart.multipart,filelock,openai

# 在 debug=True 时，想要放开的（允许输出）的日志
LOG_DEBUG_LOGGERS=alembic.runtime.migration


# =======================
# Database Configuration
# =======================
DATABASE_URL=postgresql://kn_audio:1234qwer@192.168.167.233:5432/kn_audio
DATABASE_POOL_SIZE=20
DATABASE_MAX_OVERFLOW=10
DATABASE_POOL_TIMEOUT=30.0
DATABASE_ECHO=false
```

### 网络策略

```bash
curl -X GET http://10.66.2.1:8088/v1/health -H 'accept: application/json'
```

```bash
curl http://127.0.0.1:30015/apisix/admin/routes/file-download \
  -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" \
  -H "Content-Type: application/json" \
  -X PUT \
  -d '{
    "uri": "/v1/file/download/*",
    "upstream": {
      "type": "roundrobin",
      "nodes": {
        "10.66.2.1:8088": 1
      }
    }
  }'
```

```
{"key":"/apisix/routes/file-download","value":{"uri":"/v1/file/download/*","upstream":{"nodes":{"10.66.2.1:8088":1},"type":"roundrobin","scheme":"http","pass_host":"pass","hash_on":"vars"},"id":"file-download","create_time":1755673002,"status":1,"update_time":1755673002,"priority":0}}
```

```
curl -X GET http://127.0.0.1:40000/v1/file/download/tavilyAPIKEY.txt -H 'accept: application/json'
```

```
curl -X GET http://10.156.1.20:40000/v1/file/download/tavilyAPIKEY.txt -H 'accept: application/json'
```

#### 再度测试

```bash
curl -X POST \
  'http://192.168.167.232:8088/v1/audio/speech' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "text": "你好吗",
  "voice": "Chinese Female",
  "stream": false
}' -o output_1.mp3
```

```
curl -X POST \
  'http://127.0.0.1:8088/v1/audio/speech' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "text": "本论文通过提出Ships和Sahara方法，首次在多头注意力机制中识别出对安全性至关重要的注意力头，并通过大量实验验证了其有效性。研究发现，仅需消融极少数注意力头（0.006%参数），就能使模型的安全性大幅下降，同时不影响其帮助性。这些发现不仅深化了我们对LLM安全机制的理解，也为后续模型安全性设计与优化提供了重要指导。本论文通过提出Ships和Sahara方法，首次在多头注意力机制中识别出对安全性至关重要的注意力头，并通过大量实验验证了其有效性。研究发现，仅需消融极少数注意力头（0.006%参数），就能使模型的安全性大幅下降，同时不影响其帮助性。这些发现不仅深化了我们对LLM安全机制的理解，也为后续模型安全性设计与优化提供了重要指导。本论文通过提出Ships和Sahara方法，首次在多头注意力机制中识别出对安全性至关重要的注意力头，并通过大量实验验证了其有效性。研究发现，仅需消融极少数注意力头（0.006%参数），就能使模型的安全性大幅下降，同时不影响其帮助性。这些发现不仅深化了我们对LLM安全机制的理解，也为后续模型安全性设计与优化提供了重要指导。",
  "voice": "Chinese Female",
  "stream": true
}' -o output_1.mp3
```

