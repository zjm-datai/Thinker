
首先查看一下生产环境中有无 java 环境，大概率是没有的 ......... 

如果有 Java 环境的话，直接 jar 包打过去运行了，但是大概率版本也是匹配不上的 ........

所以还是使用 docker 镜像打包，然后把压缩包传过去，进行部署合适一点。

这就涉及到两边的 Linux 环境主要是系统架构一致性的问题了，这个要看一下。然后决定是在 240 上打包还是在本机打包，240 上面吧。

然后还要全局设置 rocketmq 的插件给到所有路由（暂时还是只给我们那个统一的大模型路由放吧~）主要是解析 json 的逻辑并不健壮。

还要去到 rocketmq 里面新建消息组，麻烦死了，马的

很幸运，我们的 240 和 生产环境服务器都是 x86_64 

## 构建镜像

[docker.io/eclipse-temurin:21-jdk-alpine - 镜像下载 | docker.io](https://docker.aityp.com/image/docker.io/eclipse-temurin:21-jdk-alpine)

先打包

```bash
mvn clean package
```

使用这个加速源下载基础镜像

```bash
docker build -t mq2data-app .
```

```bash
docker save -o mq2data.tar mq2data-app
```

### 传递给生产环境

然后传入生产环境跳板机中

```bash
scp mq2data.tar root@10.17.105.16:/data
```

跳板机在传入服务器中：

```bash
scp mq2data.tar root@192.168.120.43:/ntdata/mq2data
```

```bash
ssh 192.168.120.43
```

加载镜像：

```bash
docker load -i mq2data.tar
```

## 创建数据库和环境变量配置

```
ssh 192.168.120.42
```

mysql 数据库的信息

```
MYSQL_PASSWORD=1234qwer
MYSQL_HOST=192.168.120.42
MYSQL_PORT=30004
```

连接数据库

```
docker exec -it mysql bash
```

```bash
mysql -u root -p1234qwer -h 127.0.0.1 -P 3306
```

### 建表语句

```sql
create database mqdb;
```

```sql
use mqdb;
```

```sql
CREATE TABLE `mq_consume_log` (
  `id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键，自增',
  `msg_id` VARCHAR(64) NOT NULL COMMENT '消息唯一 ID',
  `topic` VARCHAR(128) NOT NULL COMMENT 'RocketMQ Topic',
  `tags` VARCHAR(128) DEFAULT NULL COMMENT '消息标签',
  `message_keys` VARCHAR(128) DEFAULT NULL COMMENT '消息 Key',
  `body` TEXT NOT NULL COMMENT '原始日志 JSON 字符串',
  `route_id` VARCHAR(64) NOT NULL COMMENT 'APISIX 路由 ID',
  `prompt_tokens` INT NOT NULL DEFAULT 0 COMMENT 'OpenAI Prompt Tokens',
  `completion_tokens` INT NOT NULL DEFAULT 0 COMMENT 'OpenAI Completion Tokens',
  `total_tokens` INT NOT NULL DEFAULT 0 COMMENT 'OpenAI Total Tokens',
  `mass_auth_token` VARCHAR(255) DEFAULT NULL COMMENT '从 APISIX 日志中提取的认证 Token',
  `create_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '记录创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_msg_id` (`msg_id`),
  KEY `idx_route_id` (`route_id`),
  KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci
  COMMENT='消费日志表，含 APISIX 路由和 OpenAI Token 明细';
```

```sql
ALTER TABLE `mq_consume_log`
  MODIFY COLUMN `prompt_tokens` INT DEFAULT 0 COMMENT 'OpenAI Prompt Tokens',
  MODIFY COLUMN `completion_tokens` INT DEFAULT 0 COMMENT 'OpenAI Completion Tokens',
  MODIFY COLUMN `total_tokens` INT DEFAULT 0 COMMENT 'OpenAI Total Tokens';
```

## 消息队列

生产环境中的：[RocketMQ-Dashboard](http://10.17.105.16:30049/)

这坑爹项目搞的，消息队列的配置没有组成配置传入，操了。

现在改掉了。不过还是有点不符合 spring 项目的最佳实践，不熟，真不熟。

在这次更改中，我们使用了：

```
# ——————————————————————————————————————————————————————————————————
# 应用基础配置
# ——————————————————————————————————————————————————————————————————
spring.application.name=mq2data
server.port=${SERVER_PORT:8081}

# ——————————————————————————————————————————————————————————————————
# 数据源配置（MySQL）
# ——————————————————————————————————————————————————————————————————
spring.datasource.url=${MYSQL_URL:jdbc:mysql://211.90.240.240:30004/mqdb?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai}
spring.datasource.username=${MYSQL_USERNAME:root}
spring.datasource.password=${MYSQL_PASSWORD:1234qwer}
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# ——————————————————————————————————————————————————————————————————
# MyBatis 配置
# ——————————————————————————————————————————————————————————————————
mybatis-plus.type-aliases-package=com.jmz.mq2data.data
mybatis-plus.mapper-locations=classpath*:mapper/*.xml
mybatis-plus.configuration.map-underscore-to-camel-case=true
```

```java
package com.jmz.mq2data.config;

public class RocketMQConfig {

    public static final String NAME_SERVER;
    public static final String TOPIC;
    public static final String CONSUMER_GROUP;

    static {
        // 从系统环境变量或 JVM 启动参数中读取，如果没有则使用默认值
        NAME_SERVER = System.getenv().getOrDefault("ROCKETMQ_NAME_SERVER", "211.90.240.240:30040");
        TOPIC = System.getenv().getOrDefault("ROCKETMQ_TOPIC", "test-apisix");
        CONSUMER_GROUP = System.getenv().getOrDefault("ROCKETMQ_CONSUMER_GROUP", "apisix_group");
    }
}
```

这样我们就可以在容器启动的时候传入环境变量，被检测到了

TODO: 好像没有生效，先他妈写死吧

## APISIX 插件注册

首先需要在 apisix_conf/config.yaml 中写入 :

```
plugins:
  - serverless-pre-function
  - prometheus
  - model_router
  - forward-auth
  - proxy-rewrite
  - rocketmq-logger
```

重启容器

```bash
docker-compose restart
```

获取插件列表

```
curl http://127.0.0.1:30015/apisix/admin/plugins/list -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1'
```

### APISIX 连接信息

192.168.120.42:30040

### 给统一大模型入口路由添加该插件

查看当前该路由的插件信息：

```bash
curl http://127.0.0.1:30015/apisix/admin/routes/llm \
  -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1"
```

我们需要为他添加这个插件，而不改变之前的插件配置：

```bash
{
  "plugins": {
    "rocketmq-logger": {
      "nameserver_list": ["192.168.120.42:30040"],
      "topic": "apisix",
      "include_req_body": true,
      "include_resp_body": true
    }
  }
}
```

**创建一个 Plugin Config**，只包含 `rocketmq-logger` 插件配置。

```bash
curl http://127.0.0.1:30015/apisix/admin/plugin_configs/rocketmq-log \
  -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" \
  -X PUT -d '{
    "desc": "添加的 rocketmq 日志",
    "plugins": {
      "rocketmq-logger": {
        "nameserver_list": ["192.168.120.42:30040"],
        "topic": "apisix",
        "include_req_body": true,
        "include_resp_body": true
      }
    }
  }'
```

**在原 route 里添加 `plugin_config_id` 字段**，即便 route 本身已有 inline plugins，也会自动“合并” Plugin Config 插件 ✨

```bash
curl http://127.0.0.1:30015/apisix/admin/routes/llm \
  -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" \
  -X PATCH -d '{
    "plugin_config_id": "rocketmq-log"
  }'
```

### 测试

```bash
curl -X POST http://127.0.0.1:40000/v1/chat/completions \
  -H "Mass-Auth-Token: 4b7c991149a08ee51e75be0459e2449a" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen2.5-72B-Instruct",
    "messages": [
      {"role": "user", "content": "你好"}
    ]
  }'
```

```bash
curl -X POST http://127.0.0.1:40000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen2.5-72B-Instruct",
    "messages": [
      {"role": "user", "content": "你好"}
    ]
  }'
```

![[Pasted image 20250728162551.png]]

运作正常，说明不是覆盖，没有遮住之前的插件，现在我们去看有没有写入日志，我们去 dashborad 发现这个主题并没有自动创建，come on 很明显出问题了，于是我们去 

```bash
tail apisix_logs/error.log
```

```
2025/07/28 08:21:58 [error] 54#54: *51714 [lua] batch-processor.lua:104: Batch Processor[rocketmq logger] exceeded the max_retry_count[1] dropping the entries, context: ngx.timer, client: 172.18.0.1, server: 0.0.0.0:9080
2025/07/28 08:22:16 [warn] 56#56: *52202 [lua] model_router.lua:93: phase_func(): model_router [rewrite] uri: /v1/chat/completions method: POST, client: 172.18.0.1, server: _, request: "POST /v1/chat/completions HTTP/1.1", host: "127.0.0.1:40000"
2025/07/28 08:22:16 [warn] 56#56: *52202 [lua] plugin.lua:1210: run_plugin(): forward-auth exits with http status code 401, client: 172.18.0.1, server: _, request: "POST /v1/chat/completions HTTP/1.1", host: "127.0.0.1:40000"
2025/07/28 08:22:21 [error] 56#56: *52335 [lua] batch-processor.lua:95: Batch Processor[rocketmq logger] failed to process entries: failed to send data to rocketmq topic: getTopicRouteInfoFromNameserver return TOPIC_NOT_EXIST, No topic route info in name server for the topic: apisix
See https://rocketmq.apache.org/docs/bestPractice/06FAQ for further details., nameserver_list: ["192.168.120.42:30040"], context: ngx.timer, client: 172.18.0.1, server: 0.0.0.0:9080
2025/07/28 08:22:21 [error] 56#56: *52335 [lua] batch-processor.lua:104: Batch Processor[rocketmq logger] exceeded the max_retry_count[1] dropping the entries, context: ngx.timer, client: 172.18.0.1, server: 0.0.0.0:9080
```

可以看到 

```
failed to send data to rocketmq topic: getTopicRouteInfoFromNames server return TOPIC_NOT_EXIST, No topic route info in name server for the topic: apisix
```

**错误原因**：  

APISIX 的 `rocketmq-logger` 插件尝试向 RocketMQ 的 `apisix` 主题发送日志，但 RocketMQ 服务器中**不存在该主题（Topic）**，导致发送失败。  
同时，插件重试达到最大次数（`max_retry_count=1`）后，放弃并丢弃了日志条目。

#### 创建 apisix TOPIC

**进入某个正在运行的 RocketMQ 容器**（通常进入 namesrv 容器或 broker 容器）：

```
docker exec -it 6fe65b2284b8 /bin/bash
```

**创建 Topic**（使用 `mqadmin` 工具）：

RocketMQ 的官方镜像里自带了 `mqadmin` 工具，使用如下命令创建主题：

```
export NAMESRV_ADDR=rmqnamesrv:9876
```

```
./mqadmin updateTopic -n $NAMESRV_ADDR -c DefaultCluster -t apisix
```

参数解释：

- `-n`：NameServer 地址（可以用容器名或 IP:端口）
    
- `-c`：集群名称，默认是 `DefaultCluster`
    
- `-t`：你想创建的 Topic 名称，例如 `MyTopic`

---

![[Pasted image 20250728163550.png]]

用这种方式得了 ~

这个时候我们再请求就看到了日志消息：

![[Pasted image 20250728163739.png]]

## 运行镜像 

```
SERVER_PORT=8081
MYSQL_URL=jdbc:mysql://192.168.120.42:30004/mqdb?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
MYSQL_USERNAME=root
MYSQL_PASSWORD=1234qwer
ROCKETMQ_NAME_SERVER=192.168.120.42:30040
ROCKETMQ_TOPIC=apisix
ROCKETMQ_CONSUMER_GROUP=apisix_group
```

```bash
docker run -d \
  --name mq2data-app \
  --network=host \
  -e SERVER_PORT=8081 \
  -e MYSQL_URL="jdbc:mysql://192.168.120.42:30004/mqdb?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai" \
  -e MYSQL_USERNAME=root \
  -e MYSQL_PASSWORD=1234qwer \
  -e ROCKETMQ_NAME_SERVER=192.168.120.42:30040 \
  -e ROCKETMQ_TOPIC=apisix \
  -e ROCKETMQ_CONSUMER_GROUP=apisix_group \
  mq2data-app
```

```
docker logs -f --tail 100 mq2data-app
```

```
docker rm -f mq2data-app
```

```
docker rmi -f mq2data-app:latest
```


```sql
-- 切换到 mqdb 数据库（若未切换）
use mqdb;

-- 修改 body 字段为 longtext 类型（保留原有数据，无风险）
ALTER TABLE mq_consume_log MODIFY COLUMN body LONGTEXT NOT NULL;

-- 验证修改结果（查看 Type 列是否变为 longtext）
desc mq_consume_log;
```


