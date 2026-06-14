# agentmemory MCP HTTP Bridge（mcp-proxy）NAS 部署指南

> **目标**：把 agentmemory 的 stdio MCP server 桥接成 Streamable HTTP endpoint，给只支持 HTTP transport 的 MCP 客户端（如 Hermes）用。
> **服务地址**：`http://192.168.123.104:8014/mcp`
> **底层链路**：`Hermes → mcp-proxy:8014 (stdio↔HTTP) → agentmemory-mcp 子进程 → agentmemory:3111 REST`

---

## 1. 为什么需要这层桥

agentmemory v0.9.27 的 MCP server 是 **stdio-only**（`npx -y @agentmemory/mcp`）。3111 上只有 REST `/agentmemory/*`，没有原生 MCP HTTP endpoint。

某些 MCP 客户端（Hermes 是典型）只支持 HTTP transport，无法 spawn 子进程跑 stdio MCP。这时需要在中间放一个翻译层：[sparfenyuk/mcp-proxy](https://github.com/sparfenyuk/mcp-proxy) 干这事——它 spawn stdio 子进程，对外暴露 Streamable HTTP + SSE 两种 transport。

```
┌──────────────────────────────────────────────────────────┐
│ Synology NAS 192.168.123.104                             │
│                                                           │
│ ┌─────────────────────┐         ┌──────────────────────┐ │
│ │ agentmemory-mcp-    │ spawn   │ agentmemory-          │ │
│ │ bridge (mcp-proxy)  │────────▶│ agentmemory-1        │ │
│ │ :8014               │ stdio   │ :3111                │ │
│ │  - Python+uvicorn   │         │  - iii-engine        │ │
│ │  - spawn child:     │ ◀──────│  - REST + stdio MCP  │ │
│ │    agentmemory-mcp  │  HTTPS  │    + LLM (GLM-4-     │ │
│ └─────────┬───────────┘  REST   │    Flash)            │ │
│           │                        └──────────────────────┘ │
└───────────┼────────────────────────────────────────────────┘
            │ HTTP POST /mcp
            │ (Streamable HTTP, MCP 2025-03-26)
       ┌────▼────┐
       │ Hermes  │
       └─────────┘
```

| 端口 | 协议 | 用途 | 对外 |
|---|---|---|---|
| 8014 | HTTP | Streamable HTTP `/mcp` + SSE `/sse` | ✅ 0.0.0.0 |

端口选择：8011-8015 段在 NAS 上已用 8011（graphiti）/ 8012（eltdx）/ 8015（mempalace）；8013、8014 空闲，选 8014 与 MemPalace 挨着便于记忆。

---

## 2. 镜像构建

镜像基于 `sparfenyuk/mcp-proxy:latest`（Alpine + Python + uvicorn），加 `apk add nodejs npm` 和 `npm install -g @agentmemory/mcp@0.9.27`。

源文件：`deploy/mcp-proxy-bridge/`，结构：

```
deploy/mcp-proxy-bridge/
├── Dockerfile          # 基于 sparfenyuk/mcp-proxy:latest + node + @agentmemory/mcp
└── docker-compose.yml  # 本地开发用（带 build:）
```

`Dockerfile` 内容：
```dockerfile
FROM sparfenyuk/mcp-proxy:latest

RUN apk add --no-cache nodejs npm \
 && npm config set registry https://registry.npmmirror.com \
 && npm install -g @agentmemory/mcp@0.9.27 \
 && npm cache clean --force

ENTRYPOINT ["catatonit", "--", "mcp-proxy"]
```

### 本地 build

```bash
cd /opt/claude/agentmemory/deploy/mcp-proxy-bridge

# 国内拉 sparfenyuk/mcp-proxy 走 1ms 镜像
docker pull docker.1ms.run/sparfenyuk/mcp-proxy:latest
docker tag docker.1ms.run/sparfenyuk/mcp-proxy:latest sparfenyuk/mcp-proxy:latest

docker compose build
# 期望：agentmemory-mcp-bridge:0.1-nas，~641MB
# build 主要耗时在 npm install @agentmemory/mcp（~1-2 分钟）
```

### 本地烟雾测试

```bash
docker compose up -d
sleep 5
docker logs mcp-proxy-bridge-agentmemory-mcp-bridge-1
# 期望看到：
#   Configured default server: agentmemory-mcp
#   [@agentmemory/mcp] Standalone MCP server v0.9.27 starting...
#   Uvicorn running on http://0.0.0.0:8014

# MCP initialize 握手
curl -i -X POST -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}' \
  http://127.0.0.1:8014/mcp
# 期望：HTTP 200，Content-Type: application/json，body 含 "serverInfo":{"name":"agentmemory","version":"1.27.1"}
#       响应头里返回 mcp-session-id（后续请求带上）

docker compose down
```

---

## 3. 传输到 NAS

```bash
docker save agentmemory-mcp-bridge:0.1-nas | gzip > /tmp/mcp-bridge.tar.gz   # ~174MB

sshpass -p 'c790414J' scp -O \
  -o StrictHostKeyChecking=no \
  -o PreferredAuthentications=password \
  -o PubkeyAuthentication=no \
  -P 223 /tmp/mcp-bridge.tar.gz \
  john@192.168.123.104:/tmp/

NAS() { sshpass -p 'c790414J' ssh \
  -o StrictHostKeyChecking=no \
  -o PreferredAuthentications=password \
  -o PubkeyAuthentication=no \
  -p 223 john@192.168.123.104 "$@"; }

NAS "echo 'c790414J' | sudo -S /usr/local/bin/docker load -i /tmp/mcp-bridge.tar.gz"
# 期望：Loaded image: agentmemory-mcp-bridge:0.1-nas

NAS "rm /tmp/mcp-bridge.tar.gz"
rm /tmp/mcp-bridge.tar.gz
```

---

## 4. compose 部署

NAS 上的 compose 文件（**只用 image，不 build**——避免 NAS 上 build 拖延）放在 `/volume5/docker5/agentmemory-mcp-bridge/docker-compose.yml`：

```yaml
services:
  agentmemory-mcp-bridge:
    image: agentmemory-mcp-bridge:0.1-nas
    container_name: agentmemory-mcp-bridge
    restart: unless-stopped
    command:
      - --pass-environment
      - --host=0.0.0.0
      - --port=8014
      - --log-level=INFO
      - --
      - agentmemory-mcp
    environment:
      AGENTMEMORY_URL: http://192.168.123.104:3111
      AGENTMEMORY_SECRET: <NAS data/.hmac 里的值>
      AGENTMEMORY_TOOLS: "all"
    ports:
      - "0.0.0.0:8014:8014"
    healthcheck:
      test: ["CMD-SHELL", "nc -z 127.0.0.1 8014 || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

**关键字段**：

| 字段 | 作用 |
|---|---|
| `--pass-environment` | **必须**。否则 spawned 子进程拿不到 `AGENTMEMORY_URL` / `AGENTMEMORY_SECRET` |
| `--host 0.0.0.0` | 让端口对容器外可见（默认 127.0.0.1 只容器内可访问） |
| `--port 8014` | mcp-proxy 监听端口 |
| `agentmemory-mcp` | `--` 后面是子进程命令；这是 `npm install -g @agentmemory/mcp` 装出来的 bin 名 |
| `AGENTMEMORY_URL` | 指向 NAS 上 agentmemory 本体的 REST 端点 |
| `AGENTMEMORY_TOOLS=all` | 暴露全部 53 个工具（默认只暴露 7 个 core 工具） |

启动：

```bash
NAS "echo 'c790414J' | sudo -S mkdir -p /volume5/docker5/agentmemory-mcp-bridge"
# 把上面的 compose 内容写到 /volume5/docker5/agentmemory-mcp-bridge/docker-compose.yml
NAS "cd /volume5/docker5/agentmemory-mcp-bridge && \
  echo 'c790414J' | sudo -S /usr/local/bin/docker compose up -d"
```

---

## 5. 验证（跨网）

从任意同网段机器：

```bash
# 1. 容器健康
NAS "echo 'c790414J' | sudo -S /usr/local/bin/docker ps --filter name=agentmemory-mcp-bridge"
# 期望：Up X minutes (healthy)

# 2. MCP initialize 握手
curl -i -X POST -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}' \
  http://192.168.123.104:8014/mcp
# 期望：HTTP 200，body 含 serverInfo agentmemory v1.27.1

# 3. tools/list（带 session-id）
SID=$(curl -sS -D - -o /dev/null -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}' \
  http://192.168.123.104:8014/mcp | awk -F': ' '/mcp-session-id/{gsub(/\r/,"");print $2}')

curl -sS -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  http://192.168.123.104:8014/mcp

curl -sS -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  http://192.168.123.104:8014/mcp | jq '.result.tools | length'
# 期望：53
```

---

## 6. Hermes 接入

在 `~/.hermes/config.yaml` 末尾追加（与 MemPalace 同结构）：

```yaml
mcp_servers:
  agentmemory:
    transport: http
    url: http://192.168.123.104:8014/mcp
```

> 键名 `mcp_servers` 是 Hermes 框架硬编码的（不是 `mcp.servers`），跟 MemPalace 接入规则一致。

```bash
hermes mcp list                 # 应显示 agentmemory + 53 工具
hermes mcp test agentmemory     # 连通测试
```

接入需**新开会话**才生效。

---

## 7. 运维

### 查日志
```bash
NAS "echo 'c790414J' | sudo -S /usr/local/bin/docker logs agentmemory-mcp-bridge --tail 100 -f"
```

### 重启
```bash
NAS "cd /volume5/docker5/agentmemory-mcp-bridge && \
  echo 'c790414J' | sudo -S /usr/local/bin/docker compose restart"
```

### 升级 @agentmemory/mcp 版本

镜像里 npm 包是 build 时钉死的。要升级：

```bash
# 本地：改 Dockerfile 里的 @agentmemory/mcp@<新版本> → rebuild → 传 NAS → load → compose up -d
docker compose build --no-cache
docker save agentmemory-mcp-bridge:0.1-nas | gzip > /tmp/mcp-bridge.tar.gz
# 传 NAS、load（见第 3 节），然后：
NAS "cd /volume5/docker5/agentmemory-mcp-bridge && \
  echo 'c790414J' | sudo -S /usr/local/bin/docker compose up -d"
```

mcp-proxy 本体要升级同理（改 Dockerfile FROM tag）。

### 调试子进程

子进程 `agentmemory-mcp` 的 stdout 会被 mcp-proxy 捕获，但 stderr 会冒到容器日志。要看完整子进程日志，临时加 `--log-level=DEBUG`：

```yaml
command:
  - --pass-environment
  - --host=0.0.0.0
  - --port=8014
  - --debug           # ← 加这行
  - --
  - agentmemory-mcp
```

---

## 8. 故障排查

| 症状 | 原因 / 处理 |
|---|---|
| `POST /mcp` 返回 500，子进程报 `ECONNREFUSED 192.168.123.104:3111` | agentmemory 本体挂了；查 `docker ps` 看 `agentmemory-agentmemory-1` 是否 healthy |
| `tools/list` 只返回 7 个工具 | `AGENTMEMORY_TOOLS=all` 没设；检查 compose 的 environment |
| `mcp-proxy` 启动报 `KeyError: 'AGENTMEMORY_SECRET'` | `--pass-environment` 漏了；compose command 第一个参数必须是它 |
| 容器 unhealthy 但能响应 curl | healthcheck 用 `nc -z`，只测端口；服务真挂了要靠 `/mcp` 探活 |
| Hermes 报 `406 Not Acceptable` | URL 错（可能配了 `/sse` 但客户端要 streamable）；改 `/mcp` |
| 子进程报 401 unauthorized | `AGENTMEMORY_SECRET` 跟 NAS 上 agentmemory 的 `data/.hmac` 不一致；取最新值 |
| Hermes 同时显示 agentmemory + 重复工具 | 之前用 stdio 模式接过；从 `~/.hermes/config.yaml` 删掉旧的 stdio 配置 |

---

## 9. 安全注意

- **8014 没有 Bearer 鉴权**——mcp-proxy 不内置鉴权，任何能连到 NAS 8014 端口的客户端都能调 53 个工具
- 跟 agentmemory 本体（3111 有 Bearer）不同，bridge 默认信任内网
- 要在公网/半信任网络暴露：在前面加 nginx + Bearer 校验，或者只允许特定 IP（Synology 防火墙规则）
- `AGENTMEMORY_SECRET` 写在 compose 文件里——文件本身在 NAS root 下，权限 `----------`（root only），相对安全；但要避免 compose 文件进任何 git repo

---

## 10. 已知限制

- **mcp-proxy 的 server 模式是 stateful streamable http**：每个 session 占内存（agentmemory-mcp 子进程也跟着 session 数量增长）。Hermes 单客户端没问题；多客户端并发要看 mcp-proxy 的 session timeout 配置
- **子进程 cold start ~80ms**：mcp-proxy 每个新 session 会 spawn 一个 `agentmemory-mcp` 子进程；首个请求会感觉慢
- **不支持 SSE 与 Streamable HTTP 同时被一个客户端使用**：Hermes 选 streamable http 就用 `/mcp`，要用 SSE 就改 transport 字段并指向 `/sse`
- **AGENTMEMORY_TOOLS 必须在 spawn 时确定**：运行中改环境变量不生效，要重启容器
