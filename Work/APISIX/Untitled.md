
```lua
--
--  APISIX Plugin: model_router
--  功能说明：
--    • 普通推理请求：解析 body 中的 `model` 字段 → 动态路由 + 注入 Token。
--    • GET /models（含 /v1/models）：聚合所有上游的模型列表后直接返回。
--      ⚙️ 规则：如果对应 entry.token 中包含子串 "gpustack"，
--               则对子请求路径使用  `/v1-openai/models`，
--               否则使用 `/v1/models`。
--

local core = require("apisix.core")
local http = require("resty.http")      -- HTTP 子请求客户端  :contentReference[oaicite:0]{index=0}

local name = "model_router"

----------------------------------------------------------------------
-- 插件配置 Schema
----------------------------------------------------------------------
local schema = {
    type = "object",
    properties = {
        body_key = { type = "string", default = "model" },
        mapping  = {
            type              = "object",
            minProperties     = 1,
            additionalProperties = {
                type       = "object",
                required   = { "target" },   -- host:port
                properties = {
                    target  = { type = "string" },
                    token   = { type = "string" },
                    timeout = { type = "integer", minimum = 100, default = 2000 }
                }
            }
        }
    },
    required = { "mapping" }
}

----------------------------------------------------------------------
local _M = {
    version  = 0.5,
    priority = 500,
    name     = name,
    schema   = schema,
}

----------------------------------------------------------------------
-- 辅助：按 token 内容决定拉取路径并返回 list 数据
----------------------------------------------------------------------
local function fetch_models(entry)
    local host, port = core.utils.parse_addr(entry.target)       -- 解析 host:port :contentReference[oaicite:1]{index=1}
    if not host or not port then
        return nil, "invalid target: " .. tostring(entry.target)
    end

    local path = (entry.token and entry.token:find("gpustack", 1, true))
                  and "/v1-openai/models"
                  or "/v1/models"

    local httpc = http.new()
    httpc:set_timeout(entry.timeout or 2000)

    local res, err = httpc:request_uri(
        "http://" .. host .. ":" .. port .. path,
        {
            method  = "GET",
            headers = entry.token and { Authorization = entry.token } or nil
        }
    )

    if not res or res.status >= 300 then
        return nil, err or ("status: " .. (res and res.status or "nil"))
    end

    local body = core.json.decode(res.body)
    if body and body.data then
        return body.data
    end

    return nil, "no data[] in body"
end

----------------------------------------------------------------------
-- rewrite 阶段
----------------------------------------------------------------------
function _M.rewrite(conf, ctx)
    ------------------------------------------------------------------
    -- A. 处理 GET /models | /v1/models
    ------------------------------------------------------------------
    local uri, method = ngx.var.uri, ngx.req.get_method()

    core.log.warn(name, " [rewrite] uri: ", uri, " method: ", method)

    if method == "GET" and (uri == "/models" or uri == "/v1/models") then
        core.log.warn(name, " [rewrite] aggregating /models list")

        local visited, merged = {}, {}

        for _, entry in pairs(conf.mapping) do
            local sig = entry.target .. "|" .. (entry.token or "")
            if not visited[sig] then
                visited[sig] = true

                local list, err = fetch_models(entry)
                if not list then
                    core.log.error(name, " [models] fetch failed from ", entry.target, ": ", err)    
                else
                    for _, m in ipairs(list) do
                        merged[m.id or (#merged + 1)] = m
                    end
                end
            end
        end

        local data, i = {}, 0
        for _, m in pairs(merged) do
            i = i + 1
            data[i] = m
        end

        core.log.warn(name, " [models] final size=", #data)
        -- 直接返回聚合结果（rewrite 阶段可 return status + body） :contentReference[oaicite:2]{index=2}
        return 200, { object = "list", data = data }
    end

    ------------------------------------------------------------------
    -- B. 解析普通推理请求中的 model 字段
    ------------------------------------------------------------------
    local body, err = core.request.get_body(nil, ctx)
    if not body then
        core.log.error(name, " [rewrite] failed to get body: ", err)
        return
    end

    local decoded, jerr = core.json.decode(body)
    if not decoded then
        core.log.error(name, " [rewrite] json decode error: ", jerr)
        return
    end

    local model = tostring(decoded[conf.body_key] or "")
    local entry = conf.mapping[model]
    if not entry then
        core.log.warn(name, " [rewrite] no entry for model: ", model)
        return
    end

    ctx.model_router_entry = entry
end

----------------------------------------------------------------------
-- before_proxy 阶段：动态设置上游并注入 Authorization
----------------------------------------------------------------------
function _M.before_proxy(conf, ctx)
    local entry = ctx.model_router_entry

    core.log.warn("[before_proxy] entry = ", core.json.encode(entry))

    if not entry then
        return   -- 默认上游
    end

    local host, port = core.utils.parse_addr(entry.target)
    if not host or not port then
        core.log.error(name, " [before_proxy] invalid target: ", entry.target)
        return
    end

    core.log.warn(name, " [before_proxy] setting upstream to ", host, ":", port)

    ----------------------------------------------------------------------
    -- ✅ 1. 真正设置 ctx.upstream，让 APISIX 转发到新目标
    ----------------------------------------------------------------------
    ctx.upstream = {
        type = "roundrobin",
        scheme = "http",
        pass_host = "pass",
        nodes = {
            [host .. ":" .. port] = 1
        }
    }

    ----------------------------------------------------------------------
    -- ✅ 2. 更新 matched_route.value.upstream.nodes 中的字段
    -- 注意：这是修改原始路由配置的副本，对后续插件和日志可见
    ----------------------------------------------------------------------
    local matched_upstream = ctx.matched_route
        and ctx.matched_route.value
        and ctx.matched_route.value.upstream

    if matched_upstream and matched_upstream.nodes then
        for _, node in ipairs(matched_upstream.nodes) do
            node.host = host
            node.port = port
            core.log.warn(name, " [before_proxy] updated matched_route upstream node to ", host, ":", port)
        end
    end

    ----------------------------------------------------------------------
    -- ✅ 3. 注入 Authorization Header（如配置了 token）
    ----------------------------------------------------------------------
    if entry.token then
        core.request.set_header(ctx, "Authorization", entry.token)
        core.log.warn(name, " [before_proxy] injected Authorization header")
    end

    ----------------------------------------------------------------------
    -- 🔍 最后打印 ctx，确保设置生效
    ----------------------------------------------------------------------
    core.log.warn(" [before_proxy] final ctx = ", core.json.encode(ctx, true))
end

return _M
```

```bash
curl -X PUT http://127.0.0.1:30015/apisix/admin/routes/llm \
  -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -d '
{
  "uri": "/v1/*",
  "name": "统一大模型入口",
  "plugins": {
    "model_router": {
      "body_key": "model",
      "mapping": {
        "Qwen2.5-72B-Instruct": {
          "target": "192.168.80.100:38080",
          "token": "Bearer sk-lKn0TBxjwG8InOpoTvipKAghtuMKT43XSRo9wNZJjmME7M0D"
        },
        "DeepSeek-R1-Llama-70B": {
          "target": "192.168.80.100:38080",
          "token": "Bearer sk-lKn0TBxjwG8InOpoTvipKAghtuMKT43XSRo9wNZJjmME7M0D"
        },
        "QwQ-32B": {
          "target": "192.168.80.100:38080",
          "token": "Bearer sk-lKn0TBxjwG8InOpoTvipKAghtuMKT43XSRo9wNZJjmME7M0D"
        },
        "Qwen2.5-32B": {
          "target": "192.168.120.44:9998",
          "token": "Bearer gpustack_4f50a5c7bad4fd01_7c054bcb8d3cf921e9b1fa43517fa81b"
        },
        "bge-m3": {
          "target": "192.168.120.44:9998",
          "token": "Bearer gpustack_4f50a5c7bad4fd01_7c054bcb8d3cf921e9b1fa43517fa81b"
        },
        "bge-reranker": {
          "target": "192.168.120.44:9998",
          "token": "Bearer gpustack_4f50a5c7bad4fd01_7c054bcb8d3cf921e9b1fa43517fa81b"
        },
        "Qwen3-30B-A3B-GPTQ-Int4": {
          "target": "192.168.120.44:9998",
          "token": "Bearer gpustack_4f50a5c7bad4fd01_7c054bcb8d3cf921e9b1fa43517fa81b"
        },
        "Qwen3-32B": {
          "target": "192.168.120.44:9998",
          "token": "Bearer gpustack_4f50a5c7bad4fd01_7c054bcb8d3cf921e9b1fa43517fa81b"
        },
        "DeepSeek-R1-0528-Qwen3-8B": {
          "target": "192.168.120.44:9998",
          "token": "Bearer gpustack_4f50a5c7bad4fd01_7c054bcb8d3cf921e9b1fa43517fa81b"
        },
        "Qwen3-Embedding-8B": {
          "target": "192.168.120.44:9998",
          "token": "Bearer gpustack_4f50a5c7bad4fd01_7c054bcb8d3cf921e9b1fa43517fa81b"
        }
      }
    },
    "forward-auth": {
      "_meta": { "priority": 3000 },
      "uri": "http://192.168.120.40:6060/tool/apisix/massAuthentication",
      "request_method": "GET",
      "request_headers": ["Authorization", "Mass-Auth-Token", "Mass-Ai-Service"],
      "client_headers": ["Location"],
      "upstream_headers": ["X-User-ID"],
      "ssl_verify": false,
      "timeout": 3000,
      "keepalive": true,
      "keepalive_timeout": 60000,
      "keepalive_pool": 5,
      "allow_degradation": false,
      "status_on_error": 401
    }
  },
  "upstream": {
    "type": "roundrobin",
    "nodes": {
      "192.168.120.44:9998": 1
    }
  }
}'
```

