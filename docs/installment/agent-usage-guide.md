---
summary: "Complete guide to creating and managing multiple agents with CLI"
read_when:
  - Setting up multiple agents with different permissions
  - Using CLI commands to manage agents
  - Troubleshooting agent configuration issues
title: "OpenClaw 智能体使用指南：创建、管理和切换"
---

# OpenClaw 智能体使用指南

本指南专注于如何通过 CLI 命令创建、管理和使用多个智能体，每个智能体可以有不同的工具权限和配置。

## 什么是智能体 (Agent)

智能体是 OpenClaw 中的独立 AI 实体，每个都有自己的：

- **工具权限**：可以访问哪些工具（exec, read, write, message 等）
- **模型配置**：使用哪个 AI 模型
- **工作空间**：独立的文件系统和工作目录
- **沙箱设置**：安全隔离级别
- **身份标识**：名称、头像、主题等

**为什么需要多个智能体？**

- ✅ **权限分离**：聊天助手不需要 exec 工具，编程助手需要
- ✅ **功能专用**：不同智能体专注不同任务
- ✅ **安全隔离**：敏感操作只由特定智能体执行
- ✅ **灵活配置**：按需创建和删除智能体

## 前置准备

### Node.js 版本要求

OpenClaw CLI 需要 **Node.js v22.12+**。

```bash
# 检查当前版本
node --version

# 如果版本过低，升级到 v22
nvm install 22
nvm use 22
nvm alias default 22
```

### 验证 OpenClaw 安装

```bash
# 查看 OpenClaw 版本
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs --version

# 查看配置文件路径
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config file
```

## 快速开始

### 场景 1：创建两个智能体（最常用）

**目标**：一个聊天助手（不能执行命令）+ 一个编程助手（可以执行命令）

```bash
# 步骤 1：设置智能体列表
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config set agents.list '[
  {
    "id": "messenger",
    "name": "消息助手",
    "default": true,
    "tools": {
      "profile": "messaging"
    }
  },
  {
    "id": "coder",
    "name": "编程助手",
    "tools": {
      "profile": "coding"
    }
  }
]'

# 步骤 2：验证配置
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config get agents.list

# 步骤 3：重启 gateway
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs gateway restart
```

**完成！** 现在在钉钉中可以使用：

```
/agent           # 查看当前智能体
/agent coder     # 切换到编程助手（有 exec 工具）
/agent messenger # 切换回消息助手
```

### 场景 2：创建三个权限级别的智能体

```bash
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config set agents.list '[
  {
    "id": "user",
    "name": "普通用户",
    "default": true,
    "tools": {
      "profile": "messaging",
      "deny": ["exec", "process", "read", "write", "edit"]
    }
  },
  {
    "id": "trusted",
    "name": "可信用户",
    "tools": {
      "profile": "coding",
      "exec": {
        "host": "sandbox",
        "security": "allowlist",
        "ask": "on-miss"
      }
    }
  },
  {
    "id": "admin",
    "name": "管理员",
    "tools": {
      "profile": "full",
      "elevated": {
        "enabled": true
      }
    }
  }
]'
```

### 场景 3：按渠道绑定不同智能体

```bash
# 步骤 1：创建智能体
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config set agents.list '[
  {
    "id": "feishu-bot",
    "name": "飞书机器人",
    "tools": {
      "profile": "messaging"
    }
  },
  {
    "id": "dingtalk-dev",
    "name": "钉钉开发助手",
    "tools": {
      "profile": "coding"
    }
  }
]'

# 步骤 2：绑定渠道
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs agents bind feishu-bot feishu
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs agents bind dingtalk-dev dingtalk-connector

# 步骤 3：查看绑定
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs agents bindings
```

## CLI 命令详解

### 查看命令

```bash
# 列出所有智能体
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs agents list

# 查看智能体配置
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config get agents

# 查看智能体列表
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config get agents.list

# 查看工具配置
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config get tools.profile
```

### 创建智能体

#### 方法 1：使用 agents add（适合单个创建）

```bash
# 创建编程助手
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs agents add coder \
  --model custom-open-bigmodel-cn/GLM-5 \
  --workspace ~/.openclaw/workspace-coder
```

**参数说明**：

- `<name>`: 智能体名称
- `--model <id>`: 使用的模型 ID
- `--workspace <dir>`: 工作空间目录
- `--agent-dir <dir>`: 智能体状态目录
- `--bind <channel>`: 绑定到特定渠道
- `--non-interactive`: 非交互模式（需要 --workspace）
- `--json`: 输出 JSON 格式

#### 方法 2：使用 config set（批量设置，推荐）

```bash
# 设置多个智能体（包含工具策略）
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config set agents.list '[
  {
    "id": "agent-id",
    "name": "显示名称",
    "default": true,
    "model": "model-id",
    "tools": {
      "profile": "coding"
    }
  }
]'
```

**重要提示**：

- 使用**单引号**包裹整个 JSON
- 内部使用**双引号**
- `default: true` 只能有一个智能体设置

### 修改智能体

```bash
# 更新整个智能体列表
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config set agents.list '[<新的JSON>]'

# 修改智能体身份
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs agents set-identity <agent-id>

# 设置渠道绑定
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs agents bind <agent-id> <channel>
```

### 删除智能体

```bash
# 删除智能体（会清理工作空间和状态）
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs agents delete <agent-id>

# 解绑渠道
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs agents unbind <agent-id> <channel>
```

## 工具策略配置

### 工具配置文件 (Profiles)

| Profile     | 说明              | 包含的工具                                                                           |
| ----------- | ----------------- | ------------------------------------------------------------------------------------ |
| `minimal`   | 最小工具集        | `session_status`                                                                     |
| `messaging` | 消息工具集        | `message`, `sessions_list`, `sessions_history`, `sessions_send`                      |
| `coding`    | **编码工具集** ✅ | `exec`, `process`, `read`, `write`, `edit`, `apply_patch`, `web_search`, `web_fetch` |
| `full`      | 完整工具集        | 所有工具（无限制）                                                                   |

### 为智能体配置工具策略

#### 示例 1：使用 profile（推荐）

```json5
{
  id: "coder",
  name: "编程助手",
  tools: {
    profile: "coding", // 使用预定义的 coding profile
  },
}
```

#### 示例 2：显式工具列表

```json5
{
  id: "custom",
  name: "自定义助手",
  tools: {
    allow: ["exec", "read", "write", "edit", "message"],
    deny: ["gateway", "cron"],
  },
}
```

#### 示例 3：exec 工具详细配置

```json5
{
  id: "secure-coder",
  name: "安全编程助手",
  tools: {
    profile: "coding",
    exec: {
      host: "sandbox", // "sandbox" | "gateway" | "node"
      security: "allowlist", // "deny" | "allowlist" | "full"
      ask: "on-miss", // "off" | "on-miss" | "always"
      timeoutSec: 1800, // 超时时间（秒）
    },
  },
}
```

#### 示例 4：结合 profile 和额外工具

```json5
{
  id: "hybrid",
  name: "混合助手",
  tools: {
    profile: "messaging",
    alsoAllow: ["exec", "read", "write"], // 在 messaging 基础上添加
  },
}
```

## 智能体配置结构

### 完整的智能体配置

```typescript
{
  id: string;              // 智能体 ID（必需）
  name?: string;           // 显示名称
  default?: boolean;       // 是否为默认智能体
  model?: string | {       // 模型配置
    primary?: string;      // 主模型
    fallbacks?: string[];  // 备用模型
  };
  workspace?: string;      // 工作空间路径
  agentDir?: string;       // 智能体状态目录
  tools?: {
    profile?: "minimal" | "coding" | "messaging" | "full";
    allow?: string[];      // 允许的工具
    deny?: string[];       // 拒绝的工具
    alsoAllow?: string[];  // 额外允许的工具
    byProvider?: {         // 提供商特定策略
      [provider: string]: AgentToolsConfig
    };
    exec?: {
      host?: "sandbox" | "gateway" | "node";
      security?: "deny" | "allowlist" | "full";
      ask?: "off" | "on-miss" | "always";
      timeoutSec?: number;
    };
    elevated?: {
      enabled?: boolean;
      allowFrom?: { [provider: string]: string[] };
    };
  };
  sandbox?: {
    mode?: "off" | "non-main" | "all";
    workspaceAccess?: "none" | "ro" | "rw";
  };
  subagents?: {
    maxDepth?: number;
    maxConcurrent?: number;
  };
}
```

## 使用智能体

### 在聊天中切换智能体

在钉钉、飞书等渠道中直接发送命令：

```
/agent              # 查看当前智能体
/agent list         # 列出所有可用智能体
/agent coder        # 切换到编程助手
/agent messenger    # 切换到消息助手
/agent              # 再次确认当前智能体
```

### 验证智能体配置

```bash
# 查看当前使用的智能体
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs agents list

# 查看特定智能体的配置
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config get agents.list.[0]
```

## 高级配置

### 按用户分配工具权限

在渠道配置中设置 `toolsBySender`：

```json5
{
  channels: {
    slack: {
      groups: {
        engineering: {
          toolsBySender: {
            "id:ADMIN_USER_ID": {
              allow: ["*"],
            },
            "username:senior-dev": {
              allow: ["exec", "read", "write", "edit"],
            },
            "*": {
              profile: "messaging",
            },
          },
        },
      },
    },
  },
}
```

### 提供商特定策略

```json5
{
  id: "multi-channel",
  name: "多渠道机器人",
  tools: {
    profile: "coding",
    byProvider: {
      "dingtalk-connector": {
        allow: ["exec", "read", "write", "edit"],
      },
      discord: {
        profile: "messaging",
        deny: ["exec", "process"],
      },
    },
  },
}
```

### 子智能体策略

```json5
{
  id: "parent",
  name: "父智能体",
  tools: {
    profile: "coding",
  },
  subagents: {
    maxDepth: 2,
    maxConcurrent: 4,
    tools: {
      deny: ["gateway", "cron", "whatsapp_login"],
      allow: ["read", "write", "web_search"],
    },
  },
}
```

## 实用配置模板

### 模板 1：基础双智能体

```json5
[
  {
    id: "chat",
    name: "聊天助手",
    default: true,
    tools: {
      profile: "messaging",
    },
  },
  {
    id: "dev",
    name: "开发助手",
    tools: {
      profile: "coding",
    },
  },
]
```

### 模板 2：三权限级别

```json5
[
  {
    id: "user",
    name: "普通用户",
    default: true,
    tools: {
      profile: "messaging",
      deny: ["exec", "process"],
    },
  },
  {
    id: "developer",
    name: "开发者",
    tools: {
      profile: "coding",
      exec: {
        host: "sandbox",
      },
    },
  },
  {
    id: "admin",
    name: "管理员",
    tools: {
      profile: "full",
    },
  },
]
```

### 模板 3：按功能分离

```json5
[
  {
    id: "writer",
    name: "文案助手",
    tools: {
      profile: "messaging",
      alsoAllow: ["web_search", "memory_search"],
    },
  },
  {
    id: "coder",
    name: "编程助手",
    tools: {
      profile: "coding",
    },
  },
  {
    id: "analyst",
    name: "数据分析",
    tools: {
      allow: ["read", "web_search", "web_fetch", "memory_search"],
      deny: ["exec", "write", "edit"],
    },
  },
]
```

### 模板 4：完全自定义

```json5
[
  {
    id: "minimal-bot",
    name: "最小机器人",
    tools: {
      allow: ["message", "session_status"],
      deny: ["*"],
    },
  },
  {
    id: "balanced-bot",
    name: "平衡机器人",
    tools: {
      profile: "coding",
      deny: ["gateway", "cron", "whatsapp_login"],
    },
  },
  {
    id: "power-bot",
    name: "强力机器人",
    tools: {
      profile: "full",
      exec: {
        host: "gateway",
        security: "allowlist",
        ask: "always",
      },
    },
  },
]
```

## 故障排查

### 问题 1：智能体切换不生效

**症状**：在聊天中发送 `/agent coder` 没有反应

**解决方案**：

```bash
# 1. 确认智能体存在
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs agents list

# 2. 确认配置正确
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config get agents.list

# 3. 重启 gateway
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs gateway restart
```

### 问题 2：工具仍然不可用

**症状**：切换到 coding 智能体后，仍然没有 exec 工具

**解决方案**：

```bash
# 检查全局 tools.profile（可能会覆盖智能体配置）
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config get tools.profile

# 如果是 "messaging" 或 "minimal"，改为 "coding" 或删除
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config set tools.profile coding

# 或者设置为空（让智能体配置生效）
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config unset tools.profile
```

### 问题 3：配置语法错误

**症状**：设置 agents.list 后出现解析错误

**解决方案**：

```bash
# 验证配置
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config validate

# 恢复备份
cp ~/.openclaw/openclaw.json.bak ~/.openclaw/openclaw.json

# 重新设置（注意 JSON 格式）
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs config set agents.list '[
  {"id":"messenger","name":"消息助手","default":true,"tools":{"profile":"messaging"}},
  {"id":"coder","name":"编程助手","tools":{"profile":"coding"}}
]'
```

### 问题 4：pnpm 使用错误的 Node 版本

**症状**：即使 `nvm use 22`，pnpm 仍报告 Node v20

**解决方案**：

```bash
# 方案 1：直接用 Node 22 运行
~/.nvm/versions/node/v22.22.1/bin/node scripts/run-node.mjs <command>

# 方案 2：设置 pnpm 的 Node 路径
pnpm config set node-path $NVM_DIR/versions/node/v22.22.1

# 方案 3：全局安装 openclaw
npm install -g openclaw
# 然后直接使用：openclaw <command>
```

## 最佳实践

### ✅ 推荐做法

1. **明确智能体用途**：每个智能体有明确的功能定位
2. **使用 profile**：优先使用预定义 profile，而非手动列举工具
3. **最小权限原则**：默认拒绝，显式允许
4. **设置默认智能体**：用 `default: true` 标记最常用的智能体
5. **使用有意义的 ID**：如 `coder`, `messenger`, `admin`
6. **测试配置**：修改后先在测试环境验证
7. **备份配置**：重要修改前备份配置文件

### ❌ 避免做法

1. 不要给所有智能体 `full` profile
2. 不要在 `host: gateway` 时设置 `ask: off`
3. 不要让多个智能体的功能重叠太多
4. 不要忽略 `tools.profile` 的全局影响
5. 不要在配置中硬编码敏感信息
6. 不要忘记重启 gateway 使配置生效

## CLI 命令速查表

### 智能体管理

| 命令                           | 说明           |
| ------------------------------ | -------------- |
| `agents list`                  | 列出所有智能体 |
| `agents add <name>`            | 添加新智能体   |
| `agents delete <id>`           | 删除智能体     |
| `agents bind <id> <channel>`   | 绑定渠道       |
| `agents unbind <id> <channel>` | 解绑渠道       |
| `agents bindings`              | 查看绑定规则   |
| `agents set-identity <id>`     | 设置智能体身份 |

### 配置管理

| 命令                        | 说明             |
| --------------------------- | ---------------- |
| `config get <path>`         | 查看配置         |
| `config set <path> <value>` | 设置配置         |
| `config unset <path>`       | 删除配置         |
| `config validate`           | 验证配置         |
| `config file`               | 显示配置文件路径 |

### 服务管理

| 命令              | 说明         |
| ----------------- | ------------ |
| `gateway restart` | 重启 gateway |
| `gateway stop`    | 停止 gateway |
| `gateway start`   | 启动 gateway |

## 相关文档

- [工具策略配置指南](/installment/tool-policy-configuration-guide)
- [完整配置指南](/installment/complete-configuration-guide)
- [Exec Tool](/tools/exec)
- [Agent Configuration](/cli/agent)
- [Tools Overview](/tools/index)
