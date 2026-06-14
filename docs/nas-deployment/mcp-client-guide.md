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

## 2. 连接前你要拿到什么

| 项 | 值 | 说明 |
|---|---|---|
| 服务 URL | `http://192.168.123.104:3111` | REST + MCP HTTP 端点 |
| WebSocket | `ws://192.168.123.104:3112` | 实时观察流（可选） |
| Viewer | `http://192.168.123.104:3113` | 管理员 UI（默认 loopback，需 SSH 隧道） |
| **Bearer Token** | `<向部署者索取>` | 鉴权必需，部署文档保留在 `data/.hmac` |

**没 token 一切免谈**——服务对 `/agentmemory/*` 全部要 `Authorization: Bearer <token>`。

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
        "AGENTMEMORY_SECRET": "<你的 token>",
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
    "AGENTMEMORY_SECRET": "<你的 token>"
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
    "AGENTMEMORY_SECRET": "<你的 token>"
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
        "AGENTMEMORY_SECRET": "<你的 token>"
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
        "AGENTMEMORY_SECRET": "<你的 token>"
      }
    }
  }
}
```

---

## 6. 不走 MCP？直接 REST API

如果你的 agent / 脚本不走 MCP 协议，直接 HTTP 调用：

### 写一条 memory
```bash
curl -X POST http://192.168.123.104:3111/agentmemory/remember \
  -H "Authorization: Bearer $TOKEN" \
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
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "代码风格偏好",
    "project": "my-app"
  }'
```

### 列出所有 memory
```bash
curl http://192.168.123.104:3111/agentmemory/memories \
  -H "Authorization: Bearer $TOKEN"
```

### 探活（无鉴权）
```bash
curl http://192.168.123.104:3111/agentmemory/livez
# → {"service":"agentmemory","status":"ok","viewerPort":3113,...}
```

完整 endpoint 列表：见部署后的 `/agentmemory/health`（带 token），返回里 `functionMetrics` 段列了 263 个可用 function。

---

## 7. MCP 工具速览

接入 MCP 后，你的 agent 会拿到这些常用工具（共 53 个，节选）：

| 工具 | 用途 |
|---|---|
| `memory_smart_search` | 语义 + BM25 + 图遍历融合搜索（最常用） |
| `memory_save` | 主动保存一条 memory |
| `memory_sessions` | 列出 / 查询 session |
| `memory_governance_delete` | 按 ID 删除 memory / session |
| `memory_compress` | 触发对一批 observation 的压缩 |
| `memory_summarize` | 对一个 session 生成摘要 |
| `memory_consolidate` | 触发 4 层精炼 pipeline |
| `memory_graph_query` | 查询概念图 |

Skills（agent 可调用的"何时用工具"指令）：
- `/remember` `/recall` `/forget` `/recap` `/handoff` `/session-history` `/commit-context` `/commit-history`

---

## 8. 多项目 / 多 agent 共享策略

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

## 9. 验证你的连接

接好后跑一遍：

```bash
# 1. npx 能拉起 MCP shim
npx -y @agentmemory/mcp --help

# 2. 客户端连上后，让 agent 调一次 memory_smart_search
#    （Claude Code: 直接说 "search my memory for test setup"）

# 3. 直接验证 REST：
curl http://192.168.123.104:3111/agentmemory/livez   # 不带 token 也能 200
curl -H "Authorization: Bearer $TOKEN" \
     http://192.168.123.104:3111/agentmemory/memories  # 期望 {"memories":[...], "total":N}
```

---

## 10. 故障排查

| 症状 | 原因 / 处理 |
|---|---|
| MCP 客户端启动报 `ECONNREFUSED 192.168.123.104:3111` | 网络不通 / NAS 关机 / 容器挂了；部署者查 `docker ps` |
| 401 unauthorized | token 错；找部署者拿最新的（`data/.hmac`） |
| `npx @agentmemory/mcp` 卡住下载 | npm 源问题；可改 `npm config set registry https://registry.npmmirror.com` |
| 写入很慢（>1s） | 正常——embedding-3 调 bigmodel 算向量；如果 >5s 找部署者查 bigmodel 限流 |
| 搜索结果跨语言匹配不上 | 检查是否真的连了 NAS（连 localhost 是另一个实例）；查 `project` 字段是否一致 |
| 工具栏只看到 7 个（不是 53） | 默认 `AGENTMEMORY_TOOLS=core`；客户端 env 加 `AGENTMEMORY_TOOLS=all` |
| `MCP server cannot reach localhost:3111` | 客户端 sandbox 屏蔽了局域网；加 `AGENTMEMORY_FORCE_PROXY=1` |

---

## 11. 行为约束 / 礼仪

- **不要写大量垃圾 observation**：每条都会触发 embedding API（一次 ~0.5s + 算力）。把"用户键入了 a"这种事件性的东西写到日志，不要丢给 agentmemory。
- **不要把机密信息当 memory 存**：API key、密码、token 这种东西不该进 memory 库——会被搜索返回。
- **删除前确认**：`memory_governance_delete` 不可逆，没有软删。
- **跨项目共享要谨慎**：默认 shared 模式下，A 项目的 memory 会被 B 项目搜出来。如果 A 是私密项目，给所有 memory 强制带 `project` 字段。

---

## 12. 我要给别的 agent / 项目接入，怎么开口要 token？

给部署者发：

> 我要把 agentmemory 接入 `<agent 名字 / 项目名>`。
> 服务地址 `http://192.168.123.104:3111` 我有了，
> 请给我一个 AGENTMEMORY_SECRET。
> 用途：`<写什么类型的 memory / 频率多高>`。

部署者会决定：
- 给你**同一个 secret**（共享模式，跟其他用户互相可见）
- 还是**单独起一个实例 / 端口**（隔离模式）

---

## 附录：curl 一行流（复制即用）

```bash
# 设环境变量
export AM_URL=http://192.168.123.104:3111
export AM_TOKEN=<你的 token>

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
