# agentmemory NAS 部署指南

> 部署目标：Synology NAS `192.168.123.104:223`（用户 `john`），让局域网内任意 MCP 客户端共享同一份 AI 记忆库。
> 适用版本：agentmemory v0.9.27 / iii-engine v0.11.2 / bigmodel GLM-4-Flash + embedding-3。

---

## 1. 部署架构

```
┌─────────────────────────────────────────────────────────────┐
│  Synology NAS  192.168.123.104                              │
│                                                              │
│  /volume5/docker5/agentmemory/                              │
│    ├── docker-compose.yml      ← 服务定义                   │
│    └── data/                   ← 持久化卷（bind mount）      │
│        ├── .hmac               ← AGENTMEMORY_SECRET         │
│        └── (KV state files inside container /data)          │
│                                                              │
│  Docker container: agentmemory-agentmemory-1                │
│    image: agentmemory:0.9.27-nas   (458MB, linux/amd64)     │
│    ports: 3111 / 3112 / 3113 → 0.0.0.0                      │
│    restart: unless-stopped                                  │
└─────────────────────────────────────────────────────────────┘
            ↑                            ↑            ↑
       REST + Bearer              WebSocket     Viewer (loopback only)
```

| 端口 | 协议 | 用途 | 对外 |
|---|---|---|---|
| 3111 | HTTP REST | API + MCP HTTP 端点 + `/agentmemory/livez` `/agentmemory/health` | ✅ 0.0.0.0 |
| 3112 | WebSocket | 实时观察流（hooks 推送） | ✅ 0.0.0.0 |
| 3113 | HTTP | Viewer 控制台（管理员 UI） | ❌ 容器内 127.0.0.1，需 SSH 隧道 |
| 49134 | WebSocket | iii-engine bridge（内部） | 不暴露 |

---

## 2. 前置依赖

**本地构建机**：
- Node.js ≥ 20（实测 24.7）
- Docker（实测 29.5.3）
- npm（实测 11.6）

**NAS 端**：
- Docker（实测 24.0.2）
- SSH 已开（端口 223）
- `john` 用户具备 `sudo docker` 权限（密码 `sudo -S`）

**LLM 服务**：
- BigModel（智谱）API key，开通 GLM-4-Flash + embedding-3
- 端点：`https://open.bigmodel.cn/api/coding/paas/v4`

---

## 3. 完整部署流程

### 阶段 1：本地测试（5-10 min）

```bash
cd /opt/claude/agentmemory

npm install --no-fund --no-audit
npm test                          # 期望：128 文件 / 1415 用例通过
npm run build                     # 产出 dist/cli.mjs 等

# 烟雾测试：起本地 server → demo → 关服务
nohup env AGENTMEMORY_DEBUG=1 node dist/cli.mjs > /tmp/am.log 2>&1 &
disown
sleep 8
curl -sf http://127.0.0.1:3111/agentmemory/livez && echo " READY"
node dist/cli.mjs demo            # 期望：3 sessions, smart-search 命中

pkill -9 -f "/root/.agentmemory/bin/iii"
pkill -9 -f "node dist/cli.mjs"
```

任一步失败 → 停。**不要部署有问题的版本到 NAS。**

### 阶段 2：构建 docker 镜像

复用项目自带 `deploy/coolify/Dockerfile`：

```bash
docker build \
  --build-arg AGENTMEMORY_VERSION=0.9.27 \
  --build-arg III_VERSION=0.11.2 \
  --build-arg III_SDK_VERSION=0.11.2 \
  -t agentmemory:0.9.27-nas \
  -f deploy/coolify/Dockerfile \
  deploy/coolify/

docker images agentmemory:0.9.27-nas    # 期望 ~458MB
```

镜像内容：`node:22-slim` + iii 0.11.2 binary + npm 装的 `@agentmemory/agentmemory@0.9.27` + entrypoint（首启自动生成 HMAC secret 并写入 `/data/.hmac`）。

### 阶段 3：传输到 NAS

NAS 禁用了 SFTP 子系统，scp 必须加 `-O`（legacy 协议）：

```bash
docker save agentmemory:0.9.27-nas | gzip > /tmp/am.tar.gz    # ~130MB

sshpass -p 'c790414J' scp -O \
  -o StrictHostKeyChecking=no \
  -o PreferredAuthentications=password \
  -o PubkeyAuthentication=no \
  -P 223 /tmp/am.tar.gz \
  john@192.168.123.104:/tmp/am.tar.gz

NAS() { sshpass -p 'c790414J' ssh \
  -o StrictHostKeyChecking=no \
  -o PreferredAuthentications=password \
  -o PubkeyAuthentication=no \
  -p 223 john@192.168.123.104 "$@"; }

NAS "echo 'c790414J' | sudo -S /usr/local/bin/docker load -i /tmp/am.tar.gz"
NAS "rm /tmp/am.tar.gz"
rm /tmp/am.tar.gz
```

### 阶段 4：compose 部署

```bash
NAS "mkdir -p /volume5/docker5/agentmemory/data"

NAS "cat > /volume5/docker5/agentmemory/docker-compose.yml << 'EOF'
services:
  agentmemory:
    image: agentmemory:0.9.27-nas
    restart: unless-stopped
    environment:
      OPENAI_API_KEY: <BIGMODEL_KEY>
      OPENAI_BASE_URL: https://open.bigmodel.cn/api/coding/paas/v4
      OPENAI_MODEL: glm-4-flash
      OPENAI_EMBEDDING_MODEL: embedding-3
      OPENAI_EMBEDDING_DIMENSIONS: \"2048\"
      EMBEDDING_PROVIDER: openai
    ports:
      - \"0.0.0.0:3111:3111\"
      - \"0.0.0.0:3112:3112\"
      - \"0.0.0.0:3113:3113\"
    volumes:
      - ./data:/data
    healthcheck:
      test: [\"CMD-SHELL\", \"curl -fsS http://127.0.0.1:3111/agentmemory/livez || exit 1\"]
      interval: 30s
      timeout: 5s
      start_period: 30s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: \"10m\"
        max-file: \"3\"
EOF"

NAS "cd /volume5/docker5/agentmemory && \
  echo 'c790414J' | sudo -S /usr/local/bin/docker compose up -d"
```

### 阶段 5：验证

```bash
# 1. 容器健康
NAS "echo 'c790414J' | sudo -S /usr/local/bin/docker ps --filter name=agentmemory"
# 期望：Up X minutes (healthy)

# 2. 无鉴权探活
NAS "curl -sf http://127.0.0.1:3111/agentmemory/livez"
# 期望：{"service":"agentmemory","status":"ok",...}

# 3. 启动日志无 noop 警告 = LLM provider 已识别
NAS "echo 'c790414J' | sudo -S /usr/local/bin/docker logs agentmemory-agentmemory-1 2>&1 | grep noop"
# 期望：空输出

# 4. 取 secret
SECRET=$(NAS "echo 'c790414J' | sudo -S cat /volume5/docker5/agentmemory/data/.hmac")
echo "AGENTMEMORY_SECRET=$SECRET"

# 5. 写入测试 memory（会触发 embedding-3 调用，延迟 ~700ms）
NAS "curl -sS -X POST -H 'Authorization: Bearer $SECRET' \
  -H 'Content-Type: application/json' \
  -d '{\"content\":\"deploy verify\",\"project\":\"smoke\"}' \
  http://127.0.0.1:3111/agentmemory/remember"
# 期望：HTTP 201，返回 memory 对象

# 6. 跨语言语义搜索（验证向量召回）
NAS "curl -sS -X POST -H 'Authorization: Bearer $SECRET' \
  -H 'Content-Type: application/json' \
  -d '{\"query\":\"部署验证\",\"project\":\"smoke\"}' \
  http://127.0.0.1:3111/agentmemory/smart-search"
# 期望：results[] 非空，命中刚才的英文 memory
```

---

## 4. AGENTMEMORY_SECRET 管理

- 首次启动时由 entrypoint 用 `openssl rand -hex 32` 生成，存到 `/data/.hmac`（chmod 600）
- 容器重启 / 升级都保留同一个 secret（持久化卷）
- **轮换**：`NAS "sudo rm /volume5/docker5/agentmemory/data/.hmac"` 然后 `docker compose restart`，新 secret 会打印到启动日志一次
- 当前值：`b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836`

---

## 5. LLM 模型选型

**为什么 GLM-4-Flash 而不是 GLM-5.x**：

agentmemory 调 LLM 的场景是**机械文本压缩 / 摘要 / 归并**——高频、低延迟、不需要推理。GLM-5.x 是思考模型，单次压缩会烧 300+ reasoning tokens、延迟 6s+，明显过重。GLM-4-Flash 实测：1.3s / 19 tokens / 0 reasoning。

**Embedding 维度坑**：
- bigmodel `embedding-3` 默认输出 2048 维
- 必须显式设 `OPENAI_EMBEDDING_DIMENSIONS=2048`，否则 agentmemory 误判维度丢弃索引（`AGENTMEMORY_DROP_STALE_INDEX` 触发）

**可选环境变量**（按需启用）：

| 变量 | 默认 | 启用效果 |
|---|---|---|
| `AGENTMEMORY_AUTO_COMPRESS=true` | off | 每批 observation 调 GLM-4-Flash 压缩（高频，省 token 慎开） |
| `CONSOLIDATION_ENABLED=true` | off | 4 层精炼 pipeline（memory → semantic → procedural） |
| `GRAPH_EXTRACTION_ENABLED=true` | off | 抽概念图边，启用图遍历召回路径 |
| `AGENTMEMORY_REFLECT=true` | off | 周期性从 memory 合成 lesson |
| `TEAM_ID` / `USER_ID` / `AGENT_ID` | 未设 | 多 agent 隔离；见 README "Multi-agent memory" 段 |

---

## 6. 运维操作

### 查看日志
```bash
NAS "echo 'c790414J' | sudo -S /usr/local/bin/docker logs agentmemory-agentmemory-1 --tail 100 -f"
```

### 重启
```bash
NAS "cd /volume5/docker5/agentmemory && \
  echo 'c790414J' | sudo -S /usr/local/bin/docker compose restart"
```

### 升级 agentmemory 版本
```bash
# 本地：改 Dockerfile ARG（如 0.9.28）→ rebuild → re-tag
docker build --build-arg AGENTMEMORY_VERSION=0.9.28 ... -t agentmemory:0.9.28-nas ...
docker save agentmemory:0.9.28-nas | gzip > /tmp/am.tar.gz
# 传 NAS → load → 改 compose 的 image: 字段 → docker compose up -d
```
**数据卷保留**，所有 memory/session/secret 跨升级不丢。

### Viewer 访问（SSH 隧道）
3113 故意绑容器内 127.0.0.1，远端访问走隧道：
```bash
ssh -L 3113:127.0.0.1:3113 -p 223 john@192.168.123.104
# 然后浏览器开 http://localhost:3113
```
要让 viewer 直接对外：在 compose 加
```yaml
environment:
  AGENTMEMORY_VIEWER_HOST: 0.0.0.0
  VIEWER_ALLOWED_HOSTS: nas.local:3113   # 你访问用的 Host header
```
（`AGENTMEMORY_SECRET` 已设，鉴权自动启用）

### 备份
```bash
NAS "echo 'c790414J' | sudo -S tar -czf - -C /volume5/docker5/agentmemory data/" > am-backup-$(date +%F).tar.gz
```

---

## 7. 故障排查

| 症状 | 排查 |
|---|---|
| 容器 `unhealthy` | `docker logs` 看 iii-engine 是否起来；通常是 49134 端口冲突或 `/data` 权限 |
| `/agentmemory/health` 401 | Bearer token 错；从 `data/.hmac` 取最新 |
| `/agentmemory/health` 404 | iii-engine 起来了但 worker 没注册上，等 30s 或查 worker 日志 |
| 写 memory 延迟 < 50ms | embedding 没生效；检查 `OPENAI_API_KEY` 是否进容器、`EMBEDDING_PROVIDER` 是否设 |
| smart-search 全是英文匹配不上中文 | 向量索引被维度不匹配丢弃，看启动日志 `dim guard fires`；加 `AGENTMEMORY_DROP_STALE_INDEX=true` 重启 |
| 启动日志有 `No LLM provider key found` | OPENAI_API_KEY 没注入；`docker exec ... env` 确认 |
| bigmodel 401/403 | key 失效或限额用尽；控制台看用量 |
| bigmodel 429 | 限流，glm-4-flash 默认 RPM 较松，但批量 consolidate 时可能触发 |

---

## 8. 已知限制

- **没启用 LLM 压缩**：默认 noop synthetic 压缩。如要 LLM 摘要，加 `AGENTMEMORY_AUTO_COMPRESS=true`。
- **没启 TEAM/AGENT 隔离**：所有项目共享记忆。要隔离加 `AGENT_ID` + `AGENTMEMORY_AGENT_SCOPE=isolated`。
- **没启用持久化快照**：`SNAPSHOT_ENABLED` 默认 off。容器重启不丢，但 volume 文件损坏会丢——建议定期 tar 备份。
- **镜像没签名校验**：本地 build → scp → load，没走 registry。可信度来自本机构建。
- **glm-4-flash 限流**：智谱免费档有 RPM/RPM 限制，agentmemory 高频压缩时会触发，建议默认 `AGENTMEMORY_AUTO_COMPRESS=false`。
