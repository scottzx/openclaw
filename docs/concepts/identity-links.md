---
summary: "跨渠道身份链接和会话共享配置指南"
read_when:
  - 配置多渠道机器人
  - 需要跨渠道共享对话上下文
title: "身份链接与跨渠道会话"
---

# 身份链接与跨渠道会话

## 概述

OpenClaw 支持多个消息渠道（飞书、钉钉、Telegram、Discord 等），每个渠道默认创建独立的会话和记忆存储。通过**身份链接 (identityLinks)** 功能，可以让同一用户在不同渠道的对话共享同一个会话和上下文。

## 默认行为

### 会话隔离

默认情况下，不同渠道的会话是**完全隔离**的：

| 渠道     | 会话密钥格式                          | 示例                                   |
| -------- | ------------------------------------- | -------------------------------------- |
| 飞书私聊 | `agent:main:feishu:direct:<用户ID>`   | `agent:main:feishu:direct:ou_89b6...`  |
| 钉钉私聊 | `agent:main:dingtalk:direct:<用户ID>` | `agent:main:dingtalk:direct:$.xxx...`  |
| Telegram | `agent:main:telegram:direct:<用户ID>` | `agent:main:telegram:direct:123456789` |

这意味着：

- ❌ 你在飞书说的话，钉钉机器人**不知道**
- ❌ 你在钉钉说的话，飞书机器人**不知道**
- ❌ 每个渠道有独立的记忆和上下文

### 配置示例

```json5
// ~/.openclaw/openclaw.json
{
  session: {
    dmScope: "per-channel-peer", // 每个渠道+用户独立会话
  },
}
```

## 身份链接

### 工作原理

身份链接通过 `session.identityLinks` 配置实现，将不同渠道的同一用户**识别为同一个人**。

```json5
{
  session: {
    dmScope: "per-channel-peer",
    identityLinks: {
      scott: [
        "feishu:ou_89b64dd85e73a84649cb1725d9224a1c",
        "dingtalk:$.xxx...",
        "telegram:123456789",
      ],
    },
  },
}
```

### 效果对比

| 配置                 | 飞书会话                   | 钉钉会话                 | 结果    |
| -------------------- | -------------------------- | ------------------------ | ------- |
| **无 identityLinks** | `feishu:direct:ou_89b6...` | `dingtalk:direct:xxx...` | ❌ 隔离 |
| **有 identityLinks** | `feishu:direct:scott`      | `dingtalk:direct:scott`  | ✅ 共享 |

### 核心逻辑

OpenClaw 通过以下步骤识别身份链接：

1. **收集候选者 ID**：
   - 用户 ID 本身：`ou_89b64dd8...`
   - 渠道前缀 + 用户 ID：`feishu:ou_89b64dd8...`

2. **匹配 identityLinks**：
   - 遍历配置的身份链接映射
   - 如果候选者 ID 在映射表中，返回规范化的身份名称

3. **生成会话密钥**：
   - 使用规范化的身份名称替代原始用户 ID
   - 不同渠道的同一用户获得相同的会话密钥

## 配置指南

### 步骤 1: 获取用户 ID

#### 飞书用户 ID

在飞书中发送任意消息给机器人，查看配对码：

```bash
# 飞书机器人会返回：
Pairing code: MLQHTPU6
Ask the bot owner to approve with:
openclaw pairing approve feishu MLQHTPU6
```

配对后，查看已配对用户：

```bash
openclaw pairing list feishu
```

#### 钉钉用户 ID

类似飞书，在钉钉中发送消息获取配对码。

#### 其他渠道

每个渠道的配对流程类似，使用相应的命令：

```bash
openclaw pairing list <channel>
```

### 步骤 2: 配置身份链接

编辑配置文件：

```bash
openclaw config edit
```

或直接编辑 `~/.openclaw/openclaw.json`：

```json5
{
  session: {
    dmScope: "per-channel-peer",
    identityLinks: {
      你的名字或ID: [
        "feishu:ou_89b64dd85e73a84649cb1725d9224a1c",
        "dingtalk:你的钉钉ID",
        "telegram:你的telegramID",
      ],
    },
  },
}
```

### 步骤 3: 重启 Gateway

```bash
# 重启 gateway 使配置生效
pkill -f "openclaw.*gateway"
nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &
```

### 步骤 4: 验证配置

在不同渠道中与机器人对话，检查：

- ✅ 在飞书说的话，钉钉机器人能记住
- ✅ 在钉钉说的话，飞书机器人能记住
- ✅ 对话上下文在不同渠道间保持一致

## 使用场景

### 场景 1: 个人多渠道使用

**需求**：你同时使用飞书和钉钉，希望两个机器人都认识你，共享对话历史。

**配置**：

```json5
{
  session: {
    dmScope: "per-channel-peer",
    identityLinks: {
      scott: ["feishu:ou_89b64dd85e73a84649cb1725d9224a1c", "dingtalk:$.xxx..."],
    },
  },
}
```

**效果**：

- 在飞书中讨论的项目计划，在钉钉中继续讨论时机器人能记住
- 在钉钉中设置的任务提醒，在飞书中机器人能识别

### 场景 2: 团队协作

**需求**：团队成员在不同渠道工作，需要保持对话上下文。

**配置**：

```json5
{
  session: {
    dmScope: "per-channel-peer",
    identityLinks: {
      alice: ["feishu:ou_alice_feishu_id", "discord:alice_discord_id"],
      bob: ["feishu:ou_bob_feishu_id", "telegram:bob_telegram_id"],
    },
  },
}
```

**效果**：

- Alice 在飞书和 Discord 的对话是连续的
- Bob 在飞书和 Telegram 的对话是连续的
- Alice 和 Bob 的会话仍然保持隔离

### 场景 3: 多账户管理

**需求**：同一用户在多个平台账户中使用机器人。

**配置**：

```json5
{
  session: {
    dmScope: "per-account-channel-peer",
    identityLinks: {
      scott: [
        "feishu:ou_89b64dd85e73a84649cb1725d9224a1c",
        "feishu:ou_another_account_id",
        "dingtalk:$.xxx...",
      ],
    },
  },
}
```

**效果**：

- 同一用户在不同飞书账户的对话共享
- 跨平台对话保持一致

## 安全考虑

### 多用户场景

如果你的机器人接收**多个用户**的消息，建议：

1. **启用安全 DM 模式**：

   ```json5
   {
     session: {
       dmScope: "per-channel-peer",
     },
   }
   ```

2. **谨慎使用身份链接**：
   - 只链接你确认是同一用户的 ID
   - 定期审查 `identityLinks` 配置
   - 避免错误的身份关联导致隐私泄露

### 隐私保护

**警告**：错误配置身份链接可能导致不同用户的对话混合。

**示例**：

```json5
// ❌ 错误配置：将不同用户链接在一起
{
  session: {
    identityLinks: {
      shared: [
        "feishu:ou_alice_id", // Alice 的飞书 ID
        "dingtalk:$.bob_id...", // Bob 的钉钉 ID
      ],
    },
  },
}
```

这会导致：

- Alice 的私密对话可能被 Bob 看到
- Bob 的对话历史会混合 Alice 的内容

**正确做法**：

```json5
// ✅ 正确配置：每个用户独立
{
  session: {
    identityLinks: {
      alice: ["feishu:ou_alice_id", "discord:alice_discord_id"],
      bob: ["dingtalk:$.bob_id...", "telegram:bob_telegram_id"],
    },
  },
}
```

## 会话作用域 (dmScope)

### 选项说明

| dmScope                    | 会话格式                                   | 适用场景                     | 隔离级别 |
| -------------------------- | ------------------------------------------ | ---------------------------- | -------- |
| `main`                     | `agent:main:main`                          | 单用户或需要全局上下文       | 无隔离   |
| `per-peer`                 | `agent:main:direct:<用户ID>`               | 需要区分用户但不需要区分渠道 | 中等隔离 |
| `per-channel-peer`         | `agent:main:<渠道>:direct:<用户ID>`        | 多用户多渠道（推荐）         | 高隔离   |
| `per-account-channel-peer` | `agent:main:<渠道>:<账户>:direct:<用户ID>` | 多账户多渠道                 | 最高隔离 |

### 选择建议

**单用户场景**：

```json5
{ dmScope: "main" }
```

- 所有渠道共享一个会话
- 最简单，上下文完全共享

**多用户单渠道**：

```json5
{ dmScope: "per-peer" }
```

- 按用户隔离，但不区分渠道
- 适用于只有一种消息渠道的场景

**多用户多渠道（推荐）**：

```json5
{ dmScope: "per-channel-peer" }
```

- 按渠道+用户隔离
- 配合 identityLinks 实现跨渠道共享
- 平衡了隔离和灵活性

**多账户场景**：

```json5
{ dmScope: "per-account-channel-peer" }
```

- 按账户+渠道+用户隔离
- 适用于复杂的多账户部署

## 记忆系统

### 记忆隔离

OpenClaw 的记忆系统（QMD）也支持访问控制：

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [
          {
            action: "allow",
            match: { chatType: "direct" },
          },
        ],
      },
    },
  },
}
```

### 跨渠道记忆

当使用身份链接时，记忆也会自动关联到规范化的身份：

- 未配置 identityLinks：`feishu:ou_89b6...` 和 `dingtalk:xxx...` 的记忆是分开的
- 配置 identityLinks：`scott` 的记忆在所有渠道中共享

## 故障排查

### 问题：跨渠道不共享上下文

**检查清单**：

1. **验证 identityLinks 配置**：

   ```bash
   openclaw config get session.identityLinks
   ```

2. **检查 dmScope 设置**：

   ```bash
   openclaw config get session.dmScope
   ```

   应该是 `per-peer`、`per-channel-peer` 或 `per-account-channel-peer`

3. **确认用户 ID 正确**：

   ```bash
   openclaw pairing list <channel>
   ```

4. **重启 Gateway**：
   ```bash
   pkill -f "openclaw.*gateway"
   nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &
   ```

### 问题：错误的身份关联

**症状**：不同用户的对话混在一起。

**解决方案**：

1. 检查 `identityLinks` 配置，确保没有错误关联
2. 移除错误的身份链接
3. 重启 Gateway

### 问题：配置后无效果

**可能原因**：

1. **dmScope 设置为 `main`**：
   - `main` 模式下所有用户共享同一个会话
   - identityLinks 不生效
   - 解决：将 dmScope 改为 `per-channel-peer`

2. **用户 ID 格式错误**：
   - 确保使用 `渠道:用户ID` 格式
   - 例如：`feishu:ou_89b64dd8...`

3. **Gateway 未重启**：
   - 配置修改后需要重启 Gateway

## 相关文档

- [Session Management](./session.md) - 会话管理详解
- [Configuration Reference](/configuration) - 完整配置参考
- [Security Model](/security/threat-model) - 安全模型说明

## 代码参考

- `src/routing/session-key.ts` - 会话密钥生成和身份链接解析
- `src/config/types.base.ts` - 配置类型定义
- `src/config/sessions/session-key.ts` - 会话密钥处理逻辑
