# agentmemory MCP 连接指南

> **目标读者**：想把 agentmemory 接入自己 AI 工作流的开发者 / AI agent。
> **服务地址**：`http://192.168.123.104:3111`（NAS 局域网）
> **状态**：已部署 v0.9.27，LLM 用 GLM-4-Flash，embedding 用 embedding-3。

---

## 1. 这是什么

agentmemory 是一个**跨会话、跨项目、跨 agent** 的 AI 编码记忆服务。你的 AI agent 把每次干活的过程（跑了什么命令、改了什么文件、做了什么决策）丢给它，下次开新会话时它把相关上下文回灌进 prompt，不用你重新解释。

**它能解决的痛点**：
- 每次开新 session 都要重新解释项目架构
- 改一个 bug 后，下次类似问题还要重新查
- 不同 agent / 项目之间的偏好和决策无法共享
- CLAUDE.md / .cursorrules 写满 200 行就装不下

**核心数据流**：

```
Agent 干活 → hook 自动捕获 observation →
  压缩 + embedding 入库 →
下次会话开头 → smart_search 相关 memory → 注入 prompt
```

---

## 2. 连接信息

| 项 | 值 | 说明 |
|---|---|---|
| 服务 URL | `http://192.168.123.104:3111` | REST + MCP HTTP 端点 |
| WebSocket | `ws://192.168.123.104:3112` | 实时观察流（可选） |
| Viewer | `http://192.168.123.104:3113` | 管理员 UI（默认 loopback，需 SSH 隧道） |
| **Bearer Token** | `b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836` | 鉴权必需，部署文档保留在 `data/.hmac` |

服务对 `/agentmemory/*` 全部要 `Authorization: Bearer b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836`。

---

## 3. 快速接入：Claude Code

最简路径（plugin + MCP 一并装）：

```text
打开 Claude Code，运行：
/plugin marketplace add rohitg00/agentmemory
/plugin install agentmemory
```

plugin 会自动装好 12 hooks + 15 skills + MCP 服务器，但默认指向 `localhost:3111`。要让它连 NAS，在 `~/.claude.json` 里把 mcpServers 块改成：

```json
{
  "mcpServers": {
    "agentmemory": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@agentmemory/mcp"],
      "env": {
        "AGENTMEMORY_URL": "http://192.168.123.104:3111",
        "AGENTMEMORY_SECRET": "b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836",
        "AGENTMEMORY_TOOLS": "all"
      }
    }
  }
}
```

或者**纯 MCP 模式**（不装 plugin，只要工具）：

```bash
# 直接合并到 ~/.claude.json 的 mcpServers 字段
"agentmemory": {
  "command": "npx",
  "args": ["-y", "@agentmemory/mcp"],
  "env": {
    "AGENTMEMORY_URL": "http://192.168.123.104:3111",
    "AGENTMEMORY_SECRET": "b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836"
  }
}
```

---

## 4. 快速接入：Cursor / Claude Desktop / Cline / Windsurf / Roo

这些客户端都用 `mcpServers` 配置结构。把下面的块**合并**到对应配置文件：

```json
"agentmemory": {
  "command": "npx",
  "args": ["-y", "@agentmemory/mcp"],
  "env": {
    "AGENTMEMORY_URL": "http://192.168.123.104:3111",
    "AGENTMEMORY_SECRET": "b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836"
  }
}
```

| 客户端 | 配置文件 |
|---|---|
| Cursor | `~/.cursor/mcp.json` 或项目内 `.cursor/mcp.json` |
| Claude Desktop | `~/Library/Application Support/Claude/claude_desktop_config.json` (mac) / `%APPDATA%\Claude\claude_desktop_config.json` (win) |
| Cline | `~/.cline/mcp.json` |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` |
| Roo Code | VSCode settings → `roo-cline.mcpServers` |

改完**重启客户端**。

---

## 5. 快速接入：Gemini CLI / OpenCode / 自定义 MCP host

### Gemini CLI (`~/.gemini/settings.json`)
```json
{
  "mcpServers": {
    "agentmemory": {
      "command": "npx",
      "args": ["-y", "@agentmemory/mcp"],
      "env": {
        "AGENTMEMORY_URL": "http://192.168.123.104:3111",
        "AGENTMEMORY_SECRET": "b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836"
      }
    }
  }
}
```

### OpenCode (`opencode.json`)
```json
{
  "mcp": {
    "agentmemory": {
      "type": "local",
      "command": ["npx", "-y", "@agentmemory/mcp"],
      "enabled": true,
      "environment": {
        "AGENTMEMORY_URL": "http://192.168.123.104:3111",
        "AGENTMEMORY_SECRET": "b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836"
      }
    }
  }
}
```

---

## 6. 快速接入：Hermes Agent

> **实测环境**：Hermes Agent on WSL2，MCP SDK `mcp==1.27.2`，Node v24.7.0

### 一行命令接入

```bash
hermes mcp add agentmemory \
  --command npx \
  --args -y @agentmemory/mcp \
  --env AGENTMEMORY_URL=http://192.168.123.104:3111 \
         AGENTMEMORY_SECRET=b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836 \
         AGENTMEMORY_TOOLS=all
```

执行后会自动连接、发现 7 个工具，提示 `Enable all 7 tools? [Y/n/select]`，输入 `Y` 确认。

### 生效方式

**必须新开会话**。当前会话不会热加载 MCP 工具。重新运行 `hermes` 即可。

### config.yaml 中的结构

`hermes mcp add` 会自动写入 `~/.hermes/config.yaml`，等价于手动添加：

```yaml
mcp_servers:
  agentmemory:
    command: npx
    args:
      - -y
      - "@agentmemory/mcp"
    env:
      AGENTMEMORY_URL: "http://192.168.123.104:3111"
      AGENTMEMORY_SECRET: "b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836"
      AGENTMEMORY_TOOLS: "all"
```

### Hermes Transport 说明

Hermes MCP 客户端支持三种 transport（自动选择）：

| 配置特征 | Transport | 底层 SDK |
|---|---|---|
| `url:` 配置，`transport: http`（默认） | StreamableHTTP | `streamable_http_client` (mcp >= 1.24.0) |
| `url:` 配置，`transport: sse` | 旧 SSE | `sse_client`，sse_read_timeout=300s |
| `command:` 配置（本例） | **stdio** | 子进程 stdin/stdout |

agentmemory 走 **stdio** 模式：`npx @agentmemory/mcp` 作为子进程启动，通过 stdin/stdout 桥接到 NAS REST 服务。

### 管理命令

```bash
hermes mcp list                # 查看所有 MCP server 状态
hermes mcp test agentmemory    # 测试连接（重新发现工具）
hermes mcp remove agentmemory  # 移除
```

### 注意事项

- **首次 npx 下载慢**：`npx -y @agentmemory/mcp` 首次运行需下载包，默认 `hermes mcp add` 连接超时 40s 可能不够。如果首次超时但 config 已保存，第二次执行（包已缓存）即可成功。
- **npm 源**：国内网络可设 `npm config set registry https://registry.npmmirror.com` 加速。
- **Hermes 环境过滤**：Hermes 对 stdio 子进程**不过滤** env（与 Claude Code 不同），但 `AGENTMEMORY_SECRET` 仍需通过 `--env` 显式传入，否则 MCP shim 找不到服务端点。
- **不与其他记忆系统冲突**：Hermes 可同时连接 MemPalace (HTTP) 和 agentmemory (stdio)，工具分别以 `mcp_mempalace_*` 和 `mcp_agentmemory_*`（或 `memory_*`）前缀区分。

### 替代方案：HTTP transport（已部署，与 MemPalace 同风格）

stdio 模式有个不便：每次 Hermes 启动要 spawn `npx -y @agentmemory/mcp`，首次包下载慢、冷启动有延迟。NAS 上已经部署了一层 **mcp-proxy bridge**（端口 8014），把 stdio MCP 翻译成 Streamable HTTP。配置如下：

```yaml
mcp_servers:
  agentmemory:
    transport: http
    url: http://192.168.123.104:8014/mcp
```

接入后 `hermes mcp list` 应显示 agentmemory 暴露 **53 个工具**（stdio 模式默认只有 7 个 core 工具，bridge 端固定 `AGENTMEMORY_TOOLS=all`）。

部署/运维/排错细节见 [mcp-bridge-deployment.md](./mcp-bridge-deployment.md)。

**两种模式选哪个？**

| 场景 | 推荐 |
|---|---|
| Hermes 单机用、嫌 npx 首启慢、要 53 个工具全集 | **HTTP transport（bridge）** |
| 客户端不支持 HTTP、需要本地 stdio、工具集自选 | **stdio 模式**（本节上方） |

---

## 7. 不走 MCP？直接 REST API

如果你的 agent / 脚本不走 MCP 协议，直接 HTTP 调用：

### 写一条 memory
```bash
curl -X POST http://192.168.123.104:3111/agentmemory/remember \
  -H "Authorization: Bearer b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "User prefers tabs over spaces; backend on port 8020-8029",
    "project": "my-app",
    "type": "preference"
  }'
```

### 语义搜索（跨语言、跨会话）
```bash
curl -X POST http://192.168.123.104:3111/agentmemory/smart-search \
  -H "Authorization: Bearer b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "代码风格偏好",
    "project": "my-app"
  }'
```

### 列出所有 memory
```bash
curl http://192.168.123.104:3111/agentmemory/memories \
  -H "Authorization: Bearer b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836"
```

### 探活（无鉴权）
```bash
curl http://192.168.123.104:3111/agentmemory/livez
# → {"service":"agentmemory","status":"ok","viewerPort":3113,...}
```

完整 endpoint 列表：见部署后的 `/agentmemory/health`（带 token），返回里 `functionMetrics` 段列了 263 个可用 function。

---

## 8. MCP 工具速览

接入 MCP 后，你的 agent 会拿到这些工具（Hermes 实测 7 个 core 工具）：

| 工具 | 用途 |
|---|---|
| `memory_smart_search` | 语义 + BM25 + 图遍历融合搜索（最常用） |
| `memory_recall` | 搜索过去 session observation，召回相关上下文 |
| `memory_save` | 主动保存一条 insight / decision / pattern |
| `memory_sessions` | 列出 / 查询 session |
| `memory_export` | 导出所有 memory 数据为 JSON |
| `memory_audit` | 查看 memory 操作审计日志 |
| `memory_governance_delete` | 按 ID 删除 memory（带审计日志，不可逆） |

Claude Code plugin 模式（装了 plugin）还会额外暴露 `memory_compress`、`memory_summarize`、`memory_consolidate`、`memory_graph_query` 等高级工具，以及 `/remember` `/recall` `/forget` `/recap` `/handoff` 等 slash command skills。

> 设 `AGENTMEMORY_TOOLS=all` 可解锁全部 53 个工具；默认 `core` 只给 7 个。

---

## 9. 多项目 / 多 agent 共享策略

**默认（shared 模式）**：所有项目所有 agent 互相可见。简单粗暴，适合单人 / 小团队。

**按项目隔离**：写 memory 时带 `project` 字段，搜索时也带 `project` 字段，自动隔离。无需服务端配置。

**按 agent 角色隔离**（架构师 / 开发 / 审查员互不可见）：在客户端 env 加：
```
AGENT_ID=architect
AGENTMEMORY_AGENT_SCOPE=isolated
TEAM_ID=my-team
USER_ID=alice
```
此时该 agent 只看到自己写的 memory。换成 `shared` 则写入打标但召回不隔离（架构师能看开发写的，但每条都标了作者）。

---

## 10. 验证你的连接

接好后跑一遍：

```bash
# 1. npx 能拉起 MCP shim（首次会下载，可能需要 30-60s）
npx -y @agentmemory/mcp --help

# 2. 客户端连上后，让 agent 调一次 memory_smart_search
#    （Hermes: 直接说 "search my memory for test setup"）

# 3. 直接验证 REST：
curl http://192.168.123.104:3111/agentmemory/livez   # 不带 token 也能 200
curl -H "Authorization: Bearer b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836" \
     http://192.168.123.104:3111/agentmemory/memories  # 期望 {"memories":[...], "total":N}
```

---

## 11. 故障排查

| 症状 | 原因 / 处理 |
|---|---|
| MCP 客户端启动报 `ECONNREFUSED 192.168.123.104:3111` | 网络不通 / NAS 关机 / 容器挂了；部署者查 `docker ps` |
| 401 unauthorized | token 错；找部署者拿最新的（`data/.hmac`） |
| `npx @agentmemory/mcp` 卡住下载 | npm 源问题；可改 `npm config set registry https://registry.npmmirror.com` |
| Hermes `hermes mcp add` 超时 | 首次 npx 下载包 >40s 默认超时；包缓存后第二次执行即可成功。也可手动编辑 config.yaml |
| 写入很慢（>1s） | 正常——embedding-3 调 bigmodel 算向量；如果 >5s 找部署者查 bigmodel 限流 |
| 搜索结果跨语言匹配不上 | 检查是否真的连了 NAS（连 localhost 是另一个实例）；查 `project` 字段是否一致 |
| 工具栏只看到 7 个（不是 53） | 默认 `AGENTMEMORY_TOOLS=core`；客户端 env 加 `AGENTMEMORY_TOOLS=all` |
| `MCP server cannot reach localhost:3111` | 客户端 sandbox 屏蔽了局域网；加 `AGENTMEMORY_FORCE_PROXY=1` |

---

## 12. 行为约束 / 礼仪

- **不要写大量垃圾 observation**：每条都会触发 embedding API（一次 ~0.5s + 算力）。把"用户键入了 a"这种事件性的东西写到日志，不要丢给 agentmemory。
- **不要把机密信息当 memory 存**：API key、密码、token 这种东西不该进 memory 库——会被搜索返回。
- **删除前确认**：`memory_governance_delete` 不可逆，没有软删。
- **跨项目共享要谨慎**：默认 shared 模式下，A 项目的 memory 会被 B 项目搜出来。如果 A 是私密项目，给所有 memory 强制带 `project` 字段。

---

## 附录：curl 一行流（复制即用）

```bash
# 设环境变量
export AM_URL=http://192.168.123.104:3111
export AM_TOKEN=b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836

# 探活
curl -sf $AM_URL/agentmemory/livez && echo OK

# 写一条
curl -sS -X POST $AM_URL/agentmemory/remember \
  -H "Authorization: Bearer $AM_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"content\":\"<你想记的事>\",\"project\":\"<项目名>\"}"

# 搜一条
curl -sS -X POST $AM_URL/agentmemory/smart-search \
  -H "Authorization: Bearer $AM_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"query\":\"<自然语言查询>\",\"project\":\"<项目名>\"}"

# 列出所有
curl -sS -H "Authorization: Bearer $AM_TOKEN" \
     "$AM_URL/agentmemory/memories"
```

---

## 13. 核心注意事项（外部 CLI 集成必读）

### 13.1 鉴权不能忘
所有请求**必须带 Bearer token**，唯一例外是 `/agentmemory/livez`：

```bash
# 正确
curl -H "Authorization: Bearer $AM_TOKEN" http://192.168.123.104:3111/agentmemory/memories

# 探活（无 token）
curl http://192.168.123.104:3111/agentmemory/livez
```

### 13.2 API 烟测数据在 `hermes-primary` project，不是 `default`

Hermes 写入的数据都在 `project=hermes-primary`。外部客户端查询时**必须带这个 project**，否则搜不到：

```bash
# 搜 Hermes 的记录 → 必须带 project
curl -X POST .../smart-search -d '{"query":"YAML", "project":"hermes-primary"}'

# 外部 CLI 用自己的数据 → 用自己的 project 名
curl -X POST .../smart-search -d '{"query":"YAML", "project":"my-cli-project"}'
```

### 13.3 写入走 `/observe` 还是 `/remember`？

| Endpoint | 用途 | 特性 |
|---|---|---|
| `/observe` | 工具事件、对话记录 | 走 4-tier consolidation，写入到检索有 ~5s 延迟 |
| `/remember` | 显式保存 insight/decision/pattern | 即时含 embedding，~600ms |
| `/smart-search` | 语义检索 | 只读 |

- **显式保存**（用户说"记住这个"）→ **`/remember`**
- **流水记录**（自动捕获对话/工具调用）→ **`/observe`**

### 13.4 查询语义问题

agentmemory 使用 bigmodel embedding-3 做向量化。长中文自然语言查询的语义匹配效果较弱：

```
"用户偏好什么文件格式" → 0 条 ❌（向量被稀释，BM25 中文分词不匹配）
"YAML"               → 1 条 ✅
"文件格式偏好 YAML"  → 1 条 ✅
```

**建议**：
- 查询时尽量带关键词：`"YAML 文件格式 偏好"` 优于 `"用户偏好什么文件格式"`
- 或在客户端做 query expansion（提取关键词后再搜索，参考 Hermes provider 的 `_extract_search_keywords` 实现）

### 13.5 不要写垃圾

每条写入都触发 embedding API 调用（~0.5s + 算力 + 限流）。避免：
- 大量短 ack（"好的"、"嗯"、"继续"）
- 低价值工具调用事件（ls、pwd、whoami）
- 频繁写入触发 consolidation pipeline 空转

**建议阈值**：单条内容 < 200 字不写入。

### 13.6 project 隔离

默认 shared 模式下，所有 project 的 memory 互相可见。如果外部 CLI 要隔离数据，**必须**：

```bash
# 写入时带自己的 project 名
curl -X POST .../remember \
  -d '{"content":"...", "project":"your-project-name", "type":"insight"}'

# 查询时也带同一个 project 名
curl -X POST .../smart-search \
  -d '{"query":"...", "project":"your-project-name"}'
```

否则会和其他项目的数据混在一起，或搜不到自己的数据。

### 13.7 工具数量控制

MCP 模式默认暴露 7 个 core 工具。要解锁全部 53 个工具，在客户端 env 加：

```bash
AGENTMEMORY_TOOLS=all
```

但注意：太多工具会让 LLM 选择困难。建议先用 core 7 个，按需开放。也可以用 `AGENTMEMORY_TOOLS=core,memory_graph_query,memory_patterns` 按需白名单。

### 13.8 写入延迟说明

| 操作 | 延迟 | 说明 |
|---|---|---|
| GET `/agentmemory/livez` | ~30ms | 探活，无 embedding |
| POST `/agentmemory/remember` | ~600ms | 含 embedding 调用 |
| POST `/agentmemory/observe` | ~327ms | 入队后异步 consolidation |
| POST `/agentmemory/smart-search` | ~94ms | 仅检索，无写入 |
| 写入到可检索 | ~5s | consolidation pipeline 异步执行 |

如果写入后立刻搜索搜不到，等待几秒后重试。

### 13.9 网络与超时

- NAS 局域网延迟约 1-3ms，embedding 调用增加 ~500ms
- 建议 HTTP 超时设为 **10s**（避免偶发抖动）
- 写入失败应**静默丢弃**，不阻塞主流程

### 13.10 安全性

- Bearer token 有权限读写所有数据，**不要泄露**（不要硬编码在公开仓库、日志、截图里）
- 不要存 API key、密码、token 等机密信息到 memory 库——会被 `smart-search` 返回
- `memory_governance_delete` 不可逆，没有软删除

---

## 14. 多 CLI 多项目记忆隔离

> **场景**：你有多个 CLI（Claude Code、CodeWhale、OpenCode、Hermes），每个 CLI 可能在不同项目目录下工作。你希望每个项目的记忆自动隔离，互不干扰。

### 14.1 核心思路

固定 `AGENTMEMORY_PROJECT=quantix_rust` 不够灵活——一个 CLI 在不同项目目录下需要不同 project。**自动派生**方案：从 git root 名称派生 project 名。

```
你在 /home/user/quantix-rust/ 目录工作
  → git root = "quantix-rust" → AGENTMEMORY_PROJECT=quantix-rust → 记忆隔离

你在 /home/user/myability/ 目录工作
  → git root = "myability"    → AGENTMEMORY_PROJECT=myability    → 记忆隔离
```

### 14.2 安装 wrapper 脚本（一行命令）

```bash
mkdir -p ~/.local/bin && cat > ~/.local/bin/agentmemory-mcp << 'WRAPPER'
#!/bin/bash
# agentmemory MCP wrapper — auto-derive project from git root
# Usage: point all CLI MCP configs to this script instead of npx

# 1. Resolve project: git root name → "quantix-rust" / "myability" / "agentmemory"
PROJECT="default"
GIT_TOP=$(git rev-parse --show-toplevel 2>/dev/null)
if [ -n "$GIT_TOP" ]; then
  PROJECT=$(basename "$GIT_TOP")
fi

# 2. Allow explicit override (highest priority)
if [ -n "$AGENTMEMORY_PROJECT" ]; then
  PROJECT="$AGENTMEMORY_PROJECT"
fi

# 3. Launch MCP shim with the resolved project
export AGENTMEMORY_PROJECT="$PROJECT"
export AGENTMEMORY_URL="${AGENTMEMORY_URL:-http://192.168.123.104:3111}"
export AGENTMEMORY_SECRET="${AGENTMEMORY_SECRET:-b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836}"

exec npx -y @agentmemory/mcp "$@"
WRAPPER
chmod +x ~/.local/bin/agentmemory-mcp
```

### 14.3 各 CLI 配置

所有 CLI 统一把 `command` 从 `npx` 改为指向 wrapper：

#### Claude Code / Cursor / Windsurf / Cline / Roo

```json
{
  "mcpServers": {
    "agentmemory": {
      "command": "~/.local/bin/agentmemory-mcp",
      "args": [],
      "env": {
        "AGENTMEMORY_URL": "http://192.168.123.104:3111",
        "AGENTMEMORY_SECRET": "b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836"
      }
    }
  }
}
```

#### OpenCode（`opencode.json`）

```json
{
  "mcp": {
    "agentmemory": {
      "type": "local",
      "command": ["~/.local/bin/agentmemory-mcp"],
      "enabled": true,
      "environment": {
        "AGENTMEMORY_URL": "http://192.168.123.104:3111",
        "AGENTMEMORY_SECRET": "b95898290680a1692d32ac04ee7757e393076231ce6adc6abcf6943bc1026836"
      }
    }
  }
}
```

#### Hermes（特殊处理）

Hermes 不走 MCP shim（它用 Python memory provider），需要单独配：

**推荐**：Hermes 保持 `AGENTMEMORY_PROJECT=hermes-primary`，负责"对话记忆"。其他 CLI 的 MCP wrapper 自动按项目隔离，负责"项目记忆"。两者互不干扰：

```bash
# ~/.hermes/.env — 保持不动
AGENTMEMORY_PROJECT=hermes-primary
```

如果也希望 Hermes 按项目隔离，启动时手动指定：

```bash
AGENTMEMORY_PROJECT=quantix-rust hermes
```

### 14.4 隔离效果矩阵

| 你在哪个目录工作 | 自动派生的 project | 能搜到 Hermes 的对话？ | 能搜到其他项目的？ |
|---|---|---|---|
| `quantix-rust/` | `quantix-rust` | ❌ 否（project 不同） | ❌ 否（project 不同） |
| `myability/` | `myability` | ❌ 否 | ❌ 否 |
| 显式搜 `project=hermes-primary` | 手动指定 | ✅ 是 | ❌ 否 |
| 不传 project（默认 shared） | 全部 | ✅ 是 | ✅ 是 |

### 14.5 补充说明

- **非 git 仓库目录**：fallback 为 `project=default`，建议项目初始化到 git 仓库中
- **显式 override**：设环境变量 `AGENTMEMORY_PROJECT=quantix-rust` 会覆盖自动派生，优先级最高
- **跨项目查询**：如需跨项目搜索（如架构师审查多个项目），可再加一个独立的 MCP server 配置命名为 `agentmemory-global`，不设 project 限制
- **wrapper 工作原理**：检测当前工作目录的 git root，取目录名作为 project 值。每个 CLI 工具调用 MCP 工具时，工作目录正是当前项目目录，所以自动派生准确

### 14.6 典型工作流

```
早上：cd ~/quantix-rust/  → 开 Claude Code
  → 工具调用自动写入 project=quantix-rust，记忆隔离

下午：cd ~/myability/     → 开 CodeWhale
  → 工具调用自动写入 project=myability，记忆隔离
  → 在 myability 里问 "quantix 的 WebSocket 实现"
  → 主动指定 project=quantix-rust 可查到
  → 不指定则默认只搜 myability，不会混淆

晚上：Hermes 对话
  → 写入 hermes-primary，和其他项目完全隔离
```
