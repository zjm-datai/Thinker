
首先我们需要离线把 docker 和 docker-compose 安装上去

[[Linux 环境离线安装 docker&docker-compose]]

```bash
docker save -o kn_audio_0.0.1.tar kn_audio:0.0.1
```

```shell
docker load -i kn_audio_0.0.1.tar
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

