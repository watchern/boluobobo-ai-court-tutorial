# OpenClaw 多 Agent 架构配置指南

> 把这份指南发给你的 OpenClaw，它会引导你选择最适合的多 Agent 架构。

---

## 使用说明

当用户把这份文档发给你时，请按照以下流程引导他们选择合适的多 Agent 架构配置。

**不要一次性输出所有内容，而是通过提问逐步了解用户需求，最后给出针对性的配置建议。**

---

## 第一步：了解基本情况

请先问用户以下问题（一次问 1-2 个）：

1. **你是个人使用还是团队使用？**
   - 个人使用 → 可能不需要多 Agent
   - 团队使用 → 需要考虑隔离和权限

2. **你主要用什么渠道和 Agent 交互？**
   - 单一渠道（如只用飞书）→ 路由配置简单
   - 多渠道（飞书+Telegram+...）→ 需要考虑跨渠道路由

3. **你需要处理敏感数据吗？**
   - 是 → 需要更强的隔离（Docker Sandbox 或多 Gateway）
   - 否 → 软隔离可能就够了

---

## 第二步：了解具体需求

根据第一步的回答，继续深入：

### 如果是个人使用：

4. **你需要 Agent 有不同的"人格"或专长吗？**
   - 比如：一个专注工作，一个处理生活
   - 是 → 多 Agent 配置
   - 否 → 单 Agent 多 Session 就够了

5. **你需要同时处理多个长时间任务吗？**
   - 是 → 考虑 sessions_spawn 或多 Agent
   - 否 → 单 Agent 就够了

### 如果是团队使用：

4. **团队成员之间需要隔离吗？**
   - 完全隔离（不能看到彼此的对话）→ 多 Gateway
   - 部分隔离（各自的 workspace，但可以协作）→ 多 Agent 软隔离

5. **需要不同的权限控制吗？**
   - 比如：有的 Agent 能访问数据库，有的不能
   - 是 → 多 Agent + 不同的 tool 配置
   - 否 → 可以共享配置

---

## 第三步：推荐配置方案

根据用户的回答，推荐以下方案之一：

### 方案 A：单 Agent 多 Session（最简单）

**适合：**
- 个人用户
- 不需要人格隔离
- 快速上手

**配置示例：**
```json
{
  "agents": {
    "list": [{ "id": "main", "workspace": "~/.openclaw/workspace" }]
  },
  "channels": {
    "feishu": {
      "appId": "xxx",
      "appSecret": "xxx"
    }
  },
  "bindings": [
    { "agentId": "main", "match": { "channel": "feishu" } }
  ]
}
```

**注意事项：**
- 所有 session 共享 workspace 和 MEMORY.md
- 如果在多个群使用，记忆可能会"串"
- 配置文件会随着使用越来越大，导致 token 消耗增加

---

### 方案 B：多 Agent 软隔离（推荐大多数场景）

**适合：**
- 需要多角色分工
- 小团队协作
- 不需要强安全隔离

**配置示例：**
```json
{
  "agents": {
    "list": [
      { "id": "main", "workspace": "~/.openclaw/workspace" },
      { "id": "content", "workspace": "~/.openclaw/workspace-content" },
      { "id": "ops", "workspace": "~/.openclaw/workspace-ops" }
    ]
  },
  "channels": {
    "feishu": {
      "appId": "xxx1",
      "appSecret": "xxx1",
      "accounts": {
        "main": { "appId": "xxx1" },
        "content": { "appId": "xxx2", "appSecret": "xxx2" },
        "ops": { "appId": "xxx3", "appSecret": "xxx3" }
      }
    }
  },
  "bindings": [
    { "agentId": "main", "match": { "channel": "feishu", "accountId": "main" } },
    { "agentId": "content", "match": { "channel": "feishu", "accountId": "content" } },
    { "agentId": "ops", "match": { "channel": "feishu", "accountId": "ops" } }
  ]
}
```

**注意事项：**
- 每个 Agent 需要独立的 workspace 目录
- 如果用飞书，每个 Agent 需要独立的飞书应用
- 软隔离是"君子协议"，Agent 技术上可以访问其他 workspace

---

### 方案 B2：多 Agent 单 Bot（按群路由）

**适合：**
- 不想创建多个飞书应用
- 希望用户有统一的交互入口
- 按群/用户灵活路由

**配置示例：**
```json
{
  "agents": {
    "list": [
      { "id": "main", "workspace": "~/.openclaw/workspace" },
      { "id": "content", "workspace": "~/.openclaw/workspace-content" },
      { "id": "ops", "workspace": "~/.openclaw/workspace-ops" }
    ]
  },
  "channels": {
    "feishu": {
      "appId": "xxx",
      "appSecret": "xxx"
    }
  },
  "bindings": [
    { "agentId": "content", "match": { "channel": "feishu", "peer": { "kind": "group", "id": "oc_content_group_xxx" } } },
    { "agentId": "ops", "match": { "channel": "feishu", "peer": { "kind": "group", "id": "oc_ops_group_xxx" } } },
    { "agentId": "main", "match": { "channel": "feishu" } }
  ]
}
```

**注意事项：**
- 用户只需要记住一个机器人
- 不同群里实际上是不同的 Agent 在服务
- bindings 按顺序匹配，最后一条是 fallback

---

### 方案 C：Docker Sandbox（中等安全）

**适合：**
- 处理敏感数据
- 需要防止 prompt injection 跨 Agent 泄露
- 不同 Agent 有不同信任级别

**配置示例：**
```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all",
        "scope": "agent",
        "workspaceAccess": "rw",
        "docker": {
          "binds": [
            "/shared/skills:/skills:ro",
            "/shared/references:/references:ro"
          ]
        }
      }
    },
    "list": [
      { "id": "main", "sandbox": { "mode": "off" } },
      { "id": "content", "workspace": "~/.openclaw/workspace-content" },
      { "id": "ops", "workspace": "~/.openclaw/workspace-ops" }
    ]
  }
}
```

**注意事项：**
- 需要先运行 `scripts/sandbox-setup.sh` 构建 Docker 镜像
- 容器启动有额外开销
- 可以通过 `docker.binds` 共享必要资源

---

### 方案 D：多 Gateway 硬隔离（最强安全）

**适合：**
- 企业级部署
- 需要高可用
- 不同 Agent 需要不同资源配置
- 安全要求极高

**配置示例：**

Gateway A (`~/.openclaw/openclaw-main.json`):
```json
{
  "gateway": { "port": 3000 },
  "agents": {
    "list": [{ "id": "main", "workspace": "~/.openclaw/workspace" }]
  },
  "channels": {
    "feishu": { "appId": "xxx1", "appSecret": "xxx1" }
  }
}
```

Gateway B (`~/.openclaw/openclaw-content.json`):
```json
{
  "gateway": { "port": 3001 },
  "agents": {
    "list": [{ "id": "content", "workspace": "~/.openclaw/workspace-content" }]
  },
  "channels": {
    "feishu": { "appId": "xxx2", "appSecret": "xxx2" }
  }
}
```

**启动命令：**
```bash
openclaw gateway start --config ~/.openclaw/openclaw-main.json
openclaw gateway start --config ~/.openclaw/openclaw-content.json
```

**注意事项：**
- 每个 Gateway 独立进程，一个崩溃不影响其他
- 资源消耗最大（每个 Gateway 独立内存）
- 需要管理多个进程/容器

---

## 第四步：迁移建议

如果用户已经有现有配置，提供迁移建议：

### 从单 Agent 迁移到多 Agent 软隔离

1. 创建新的 workspace 目录：`mkdir ~/.openclaw/workspace-content`
2. 复制必要的配置文件到新 workspace
3. 在 `openclaw.json` 中添加新 agent 到 `agents.list`
4. 创建新的飞书应用（如果需要）
5. 配置 `channels.feishu.accounts` 和 `bindings`
6. 重启 Gateway：`openclaw gateway restart`

### 从软隔离迁移到 Docker Sandbox

1. 运行 `scripts/sandbox-setup.sh` 构建镜像
2. 在配置中添加 `agents.defaults.sandbox`
3. 配置 `docker.binds` 共享必要资源
4. 先用 `mode: "non-main"` 测试
5. 确认无问题后改为 `mode: "all"`

---

## 配置对比速查表

| 方案 | 复杂度 | 安全性 | 性能 | 适用场景 |
|------|-------|-------|------|---------|
| 单 Agent 多 Session | ⭐ | ⭐ | ⭐⭐⭐⭐⭐ | 个人/原型 |
| 多 Agent 软隔离 | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | 小团队/多角色 |
| 多 Agent 单 Bot | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | 统一入口 |
| Docker Sandbox | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | 敏感数据 |
| 多 Gateway | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | 企业级 |

---

## 常见问题

### Q: 软隔离真的不安全吗？

A: 软隔离是"君子协议"。技术上，同一个 Gateway 内的 Agent 可以访问其他 Agent 的 workspace。但在信任的团队环境中，这通常不是问题。如果处理敏感数据，建议使用 Docker Sandbox 或多 Gateway。

### Q: 多 Agent 会增加多少 token 消耗？

A: 每个 Agent 的 system prompt 是独立的，所以总 token 消耗会增加。但因为每个 Agent 的配置更精简，单次对话的 token 消耗可能反而降低。

### Q: 可以让 Agent 之间互相调用吗？

A: 可以，通过 `sessions_send` 工具。但目前有一些已知 bug（如 agentToAgent.enabled 导致 sub-agents 无法启动），建议谨慎使用。

---

*来源：VA7 的多 Agent 实践经验*
*更多内容关注公众号：VA7*
