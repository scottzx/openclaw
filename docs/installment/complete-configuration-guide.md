---
summary: "Complete guide: tool policy + multi-agent configuration"
read_when:
  - Setting up OpenClaw for the first time
  - Configuring different agents with different tool permissions
  - Troubleshooting tool access issues
title: "OpenClaw 完整配置指南：工具策略 + 多智能体"
---

# OpenClaw 完整配置指南：工具策略 + 多智能体

本文档整合了工具策略权限系统和多智能体配置的完整知识，帮助你快速理解和配置 OpenClaw。

## 核心概念

### 1. 工具策略系统

OpenClaw 使用**多层策略管道**来控制工具访问：

```
tools.profile (全局配置文件)
    ↓
tools.byProvider.profile (提供商级配置)
    ↓
tools.allow (全局允许列表)
    ↓
tools.byProvider.allow (提供商级允许列表)
    ↓
agents.<agentId>.tools.allow (Agent 级允许列表) ← 重点！
    ↓
agents.<agentId>.tools.byProvider.allow (Agent 提供商级)
    ↓
group tools.allow (组级策略)
```

**关键原则**：

- ✅ **deny 优先**：拒绝列表总是优先于允许列表
- ✅ **渐进限制**：每层只能移除工具，不能添加
- ✅ **继承覆盖**：子配置覆盖父配置
- ✅ **通配符支持**：`*` 匹配所有，`web_*` 匹配模式

### 2. 多智能体系统

OpenClaw 支持多个智能体，每个都可以有独立的：

- 工具权限 (`tools`)
- 模型配置 (`model`)
- 工作空间 (`workspace`)
- 沙箱设置 (`sandbox`)

## 工具配置文件 (Tool Profiles)

| Profile     | 工具集            | 包含的工具                                                                                                        |
| ----------- | ----------------- | ----------------------------------------------------------------------------------------------------------------- |
| `minimal`   | 最小工具集        | `session_status`                                                                                                  |
| `messaging` | 消息工具集        | `message`, `sessions_list`, `sessions_history`, `sessions_send`                                                   |
| `coding`    | **编码工具集** ✅ | `exec`, `process`, `read`, `write`, `edit`, `apply_patch`, `web_search`, `web_fetch`, memory tools, session tools |
| `full`      | 完整工具集        | 所有工具（无限制）                                                                                                |

## 配置文件位置

主配置文件：`~/.openclaw/openclaw.json`

## 完整配置示例

### 示例 1：简单多智能体（入门推荐）

```json5
{
  meta: {
    lastTouchedVersion: "2026.3.3",
    lastTouchedAt: "2026-03-06T11:32:15.227Z",
  },
  agents: {
    defaults: {
      model: {
        primary: "custom-open-bigmodel-cn/GLM-5",
      },
      workspace: "/Users/scott/.openclaw/workspace",
      compaction: { mode: "safeguard" },
      maxConcurrent: 4,
      subagents: { maxConcurrent: 8 },
    },
    list: [
      {
        id: "messenger",
        name: "消息助手",
        default: true,
        tools: {
          profile: "messaging",
        },
      },
      {
        id: "coder",
        name: "编程助手",
        tools: {
          profile: "coding",
          exec: {
            host: "sandbox",
            security: "allowlist",
            ask: "on-miss",
          },
        },
      },
      {
        id: "admin",
        name: "管理员",
        tools: {
          profile: "full",
          exec: {
            host: "gateway",
            security: "allowlist",
            ask: "on-miss",
          },
        },
      },
    ],
  },
  tools: {
    profile: "coding",
  },
  channels: {
    "dingtalk-connector": {
      clientId: "your-client-id",
      clientSecret: "your-client-secret",
      gatewayToken: "your-gateway-token",
    },
  },
  gateway: {
    port: 18789,
    mode: "local",
    bind: "loopback",
    auth: {
      mode: "token",
      token: "your-auth-token",
    },
  },
}
```

### 示例 2：按功能分离的多智能体

```json5
{
  agents: {
    defaults: {
      model: { primary: "custom-open-bigmodel-cn/GLM-5" },
    },
    list: [
      {
        id: "chat",
        name: "聊天助手",
        default: true,
        tools: {
          profile: "messaging",
          deny: ["exec", "process"],
        },
      },
      {
        id: "dev",
        name: "开发助手",
        tools: {
          profile: "coding",
          exec: {
            host: "sandbox",
            security: "allowlist",
            ask: "on-miss",
          },
          alsoAllow: ["browser", "canvas"],
        },
      },
      {
        id: "ops",
        name: "运维助手",
        tools: {
          profile: "coding",
          exec: {
            host: "gateway",
            security: "allowlist",
            ask: "always",
          },
          allow: ["cron", "gateway"],
        },
      },
    ],
  },
}
```

### 示例 3：按权限级别分离

```json5
{
  agents: {
    list: [
      {
        id: "user",
        name: "普通用户",
        default: true,
        tools: {
          profile: "messaging",
          deny: ["exec", "read", "write", "edit", "process", "cron", "gateway"],
        },
      },
      {
        id: "trusted",
        name: "可信用户",
        tools: {
          profile: "coding",
          exec: {
            host: "sandbox",
            security: "allowlist",
            ask: "on-miss",
          },
        },
      },
      {
        id: "admin",
        name: "管理员",
        tools: {
          profile: "full",
          elevated: {
            enabled: true,
            allowFrom: {
              "dingtalk-connector": ["user:ADMIN_USER_ID"],
            },
          },
          exec: {
            host: "gateway",
            security: "full",
            ask: "off",
          },
        },
      },
    ],
  },
}
```

### 示例 4：提供商特定策略

```json5
{
  agents: {
    defaults: {
      tools: {
        profile: "coding",
      },
    },
    list: [
      {
        id: "multi-channel-bot",
        name: "多渠道机器人",
        tools: {
          profile: "coding",
          byProvider: {
            "dingtalk-connector": {
              allow: ["exec", "read", "write", "edit", "message", "web_search"],
            },
            discord: {
              profile: "messaging",
              deny: ["exec", "process"],
            },
            feishu: {
              profile: "full",
            },
          },
        },
      },
    ],
  },
}
```

### 示例 5：Per-Sender 策略（细粒度权限控制）

```json5
{
  channels: {
    slack: {
      groups: {
        engineering: {
          agentId: "coder",
          toolsBySender: {
            "id:ADMIN_USER_ID": {
              allow: ["*"],
            },
            "username:senior-dev": {
              allow: ["exec", "read", "write", "edit", "web_search"],
              deny: ["gateway", "cron"],
            },
            "id:JUNIOR_DEV_ID": {
              profile: "messaging",
              alsoAllow: ["web_search", "memory_search"],
            },
            "*": {
              profile: "messaging",
              deny: ["exec", "process", "read", "write", "edit"],
            },
          },
        },
      },
    },
  },
}
```

## 配置说明

### 工具策略配置结构

```typescript
type AgentToolsConfig = {
  // 基础工具配置文件
  profile?: "minimal" | "coding" | "messaging" | "full";

  // 显式工具列表
  allow?: string[]; // 允许的工具（空列表 = 允许所有）
  deny?: string[]; // 拒绝的工具（优先于 allow）
  alsoAllow?: string[]; // 额外允许的工具（追加到 profile/allow）

  // 提供商特定策略
  byProvider?: Record<string, AgentToolsConfig>;

  // Sub-agent 策略
  subagents?: {
    tools?: {
      allow?: string[];
      deny?: string[];
    };
  };

  // Exec 工具配置
  exec?: {
    host?: "sandbox" | "gateway" | "node";
    security?: "deny" | "allowlist" | "full";
    ask?: "off" | "on-miss" | "always";
    node?: string;
    timeoutSec?: number;
  };

  // Elevated 模式
  elevated?: {
    enabled?: boolean;
    allowFrom?: {
      [provider: string]: string[]; // 允许的发送者列表
    };
  };

  // 文件系统工具
  fs?: {
    workspaceOnly?: boolean;
  };
};
```

### 智能体配置结构

```typescript
type AgentEntry = {
  id: string; // 智能体 ID（必需）
  name?: string; // 显示名称
  default?: boolean; // 是否为默认智能体

  // 模型配置
  model?:
    | string
    | {
        primary?: string;
        fallbacks?: string[];
      };

  // 工具配置
  tools?: AgentToolsConfig;

  // 工作空间
  workspace?: string;

  // 沙箱配置
  sandbox?: {
    mode?: "off" | "non-main" | "all";
    workspaceAccess?: "none" | "ro" | "rw";
  };

  // Sub-agent 配置
  subagents?: {
    maxDepth?: number;
    maxConcurrent?: number;
  };
};
```

## 常见配置场景

### 场景 1：只有默认智能体，想启用 exec 工具

**问题**：配置了 `tools.profile: "messaging"`，没有 exec 工具

**解决方案 1**：修改全局 profile

```bash
openclaw config set tools.profile coding
```

**解决方案 2**：在配置文件中修改

```json5
{
  tools: {
    profile: "coding", // 改为 coding 或 full
  },
}
```

**解决方案 3**：使用 alsoAllow 添加特定工具

```json5
{
  tools: {
    profile: "messaging",
    alsoAllow: ["exec", "process", "read", "write", "edit"],
  },
}
```

### 场景 2：创建两个智能体，一个能执行命令，一个只能聊天

```json5
{
  agents: {
    defaults: {
      model: { primary: "custom-open-bigmodel-cn/GLM-5" },
    },
    list: [
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
          exec: {
            host: "sandbox",
          },
        },
      },
    ],
  },
}
```

### 场景 3：不同渠道使用不同智能体

```json5
{
  agents: {
    list: [
      {
        id: "feishu-bot",
        name: "飞书机器人",
        tools: { profile: "messaging" },
      },
      {
        id: "dingtalk-dev",
        name: "钉钉开发助手",
        tools: { profile: "coding" },
      },
    ],
  },
  channels: {
    feishu: {
      agentId: "feishu-bot",
    },
    "dingtalk-connector": {
      agentId: "dingtalk-dev",
    },
  },
}
```

### 场景 4：同一渠道，不同用户有不同权限

```json5
{
  channels: {
    slack: {
      groups: {
        general: {
          toolsBySender: {
            "id:ADMIN_ID": {
              allow: ["*"],
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

## 配置继承机制

理解配置如何继承和覆盖非常重要：

```
agents.defaults (全局默认)
    ↓ 继承
agents.list[i] (具体智能体)
    ↓ 覆盖
生效配置
```

**示例**：

```json5
{
  agents: {
    defaults: {
      model: { primary: "custom-open-bigmodel-cn/GLM-5" },
      tools: {
        profile: "messaging",
        deny: ["gateway", "cron"],
      },
    },
    list: [
      {
        id: "messenger",
        // 完全继承 defaults
      },
      {
        id: "coder",
        tools: {
          profile: "coding", // 覆盖 profile，但保留 deny
        },
      },
      {
        id: "admin",
        tools: {
          allow: ["*"], // 完全覆盖，忽略 defaults 的 profile 和 deny
          deny: [], // 清空 deny 列表
        },
      },
    ],
  },
}
```

## 如何选择和使用智能体

### 方法 1：通过命令切换（推荐）

在钉钉或其他渠道中发送：

```
/agent              # 查看当前智能体
/agent coder        # 切换到编程助手
/agent messenger    # 切换到消息助手
/agent list         # 列出所有可用智能体
```

### 方法 2：通过配置绑定

在配置文件中为渠道绑定特定智能体：

```json5
{
  channels: {
    "dingtalk-connector": {
      agentId: "coder", // 钉钉始终使用编程助手
    },
  },
}
```

### 方法 3：通过 session 前缀

在 session key 中指定智能体（高级用法）：

```
agent:coder:dingtalk:group:GROUP_ID
```

## 权限系统架构

### Sub-agent 限制

Sub-agent（被主智能体创建的子智能体）有额外限制：

**始终拒绝**（不管配置如何）：

- `gateway`, `agents_list` - 系统管理
- `whatsapp_login` - 交互式设置
- `session_status`, `cron` - 状态/调度
- `memory_search`, `memory_get` - 内存访问
- `sessions_send` - 直接消息发送

**叶子节点额外拒绝**（depth >= maxSpawnDepth）：

- `sessions_list`, `sessions_history`, `sessions_spawn`

### Owner-Only 工具

这些工具只对所有者可用（在 `commands.ownerAllowFrom` 中配置）：

- `whatsapp_login`
- `cron`
- `gateway`

### Elevated 模式

用于控制特权操作：

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        "dingtalk-connector": ["user:ADMIN_USER_ID"],
        discord: ["id:SPECIFIC_USER_ID"],
      },
    },
  },
}
```

## 故障排查

### 问题 1：工具不可用

**检查步骤**：

1. 检查 profile：

   ```bash
   openclaw config get tools.profile
   openclaw config get agents.defaults.tools.profile
   ```

2. 检查 deny 列表：

   ```bash
   openclaw config get tools.deny
   openclaw config get agents.defaults.tools.deny
   ```

3. 检查当前智能体：

   ```
   /agent  # 在聊天中查看
   ```

4. 检查智能体特定配置：
   ```bash
   openclaw config get agents.list
   ```

### 问题 2：exec 工具需要批准

这是正常行为，当：

- `host: "gateway"` 或 `host: "node"`
- `security: "allowlist"`
- `ask: "on-miss"` 或 `ask: "always"`

**解决方案**：

```json5
{
  tools: {
    exec: {
      host: "sandbox", // 沙箱模式不需要批准
      // 或
      ask: "off", // 跳过批准提示
    },
  },
}
```

⚠️ **警告**：在 `host: gateway` 时禁用批准允许无限制执行命令。

### 问题 3：智能体切换不生效

1. 检查智能体 ID 是否正确：

   ```bash
   openclaw agents list
   ```

2. 确认配置文件格式正确（JSON 语法）

3. 重启 gateway：
   ```bash
   openclaw gateway restart
   ```

## 配置最佳实践

### ✅ 推荐做法

1. **使用默认配置**：在 `agents.defaults` 中设置通用配置
2. **明确智能体用途**：每个智能体有明确的功能定位
3. **最小权限原则**：默认拒绝，显式允许
4. **使用 profile**：优先使用预定义 profile，而非手动列举工具
5. **分层配置**：全局 → 智能体 → 提供商 → 用户

### ❌ 避免做法

1. 不要给所有智能体 `full` profile
2. 不要在 `host: gateway` 时设置 `ask: off`
3. 不要忽略 `deny` 列表的作用
4. 不要让多个智能体的功能重叠太多
5. 不要在配置文件中硬编码敏感信息（使用环境变量）

## 配置管理命令

```bash
# 查看配置
openclaw config get                          # 所有配置
openclaw config get tools                    # 工具配置
openclaw config get agents.defaults.tools    # 默认智能体工具配置
openclaw config get agents.list              # 智能体列表

# 设置配置
openclaw config set tools.profile coding
openclaw config set agents.defaults.tools.profile coding

# 智能体管理
openclaw agents list                         # 列出智能体
openclaw agents get <agent-id>              # 获取智能体详情

# 重启服务
openclaw gateway restart
```

## 应用配置更改

修改 `~/.openclaw/openclaw.json` 后：

```bash
# 重启 gateway
openclaw gateway restart

# 验证配置
openclaw config get
```

## 相关文档

- [Exec Tool](/tools/exec) - exec 工具详细文档
- [Exec Approvals](/tools/exec-approvals) - 批准系统
- [Elevated Mode](/tools/elevated) - 特权操作
- [Subagents](/tools/subagents) - 子智能体配置
- [Agent Configuration](/cli/agent) - 智能体配置参考
- [Tool Policy](/tools/index) - 工具策略总览

## 快速参考

### 工具配置文件对比

| 场景         | 推荐配置                                     |
| ------------ | -------------------------------------------- |
| 纯聊天机器人 | `profile: "messaging"`                       |
| 需要执行命令 | `profile: "coding"`                          |
| 开发环境     | `profile: "coding"` + `exec.host: "sandbox"` |
| 生产环境     | `profile: "messaging"` + `deny: ["exec"]`    |
| 管理员       | `profile: "full"` + `elevated.enabled: true` |

### 多智能体推荐结构

```
agents.defaults
  ├─ list[0]: messenger (默认，消息工具)
  ├─ list[1]: coder (编码工具)
  └─ list[2]: admin (完全访问)
```

### 配置层级记忆口诀

```
全局 defaults → 智能体 list → 提供商 byProvider → 用户 bySender
         ↓             ↓                ↓                ↓
      基础配置      功能分离         渠道适配         权限细化
```
