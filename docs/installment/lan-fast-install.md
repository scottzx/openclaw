# OpenClaw局域网快速安装指南

## 适用场景

当您需要在ARM设备（如LubanCat、Orange Pi、树莓派等）上安装OpenClaw，但：

- 网络速度较慢
- npm安装时间过长
- 想要跳过编译步骤
- 多台设备需要部署

## 前置条件

- Mac和目标设备在同一局域网
- 已在Mac上克隆OpenClaw仓库
- 目标设备已安装Node.js 22+

## 安装步骤

### 第一步：在Mac上准备安装包

```bash
# 进入OpenClaw目录
cd /path/to/openclaw

# 确保依赖已安装
npm install

# 构建项目
npm run build

# 创建安装包（只包含必要的文件）
tar -czf openclaw-arm64.tar.gz \
  node_modules/ \
  dist/ \
  openclaw.mjs \
  assets/ \
  docs/ \
  extensions/ \
  skills/

# 检查包大小
ls -lh openclaw-arm64.tar.gz
```

### 第二步：传输到目标设备

```bash
# 查看Mac的IP地址
ifconfig | grep "inet "

# 通过scp传输（替换为目标设备的IP和用户名）
scp openclaw-arm64.tar.gz username@target-ip:~/
```

### 第三步：在目标设备上安装

```bash
# SSH连接到目标设备
ssh username@target-ip

# 解压到用户目录
cd ~
tar -xzf openclaw-arm64.tar.gz -C openclaw-install

# 创建软链接到npm全局目录
sudo ln -sf ~/openclaw-install/openclaw.mjs /usr/local/bin/openclaw
sudo chmod +x /usr/local/bin/openclaw

# 验证安装
openclaw --version
```

## 常见设备信息

| 设备类型  | 默认用户名      | 默认密码        |
| --------- | --------------- | --------------- |
| LubanCat  | cat             | temppwd         |
| Orange Pi | root / orangepi | orangepi / 1234 |
| NanoPi    | pi / root       | pi / 1234       |
| 树莓派    | pi              | raspberry       |

## 优化配置（2GB RAM设备）

创建优化配置文件：

```bash
# 创建配置目录
mkdir -p ~/.openclaw

# 创建优化配置
cat > ~/.openclaw/openclaw.json << 'EOF'
{
  "gateway": {
    "mode": "local",
    "bind": "loopback"
  },
  "agents": {
    "defaults": {
      "maxConcurrent": 2,
      "subagents": {
        "maxConcurrent": 3
      },
      "contextPruning": {
        "mode": "cache-ttl",
        "ttl": "30m",
        "keepLastAssistants": 2
      },
      "heartbeat": {
        "every": "45m"
      },
      "models": {
        "anthropic/claude-sonnet-4-6": {
          "params": {
            "cacheRetention": "short"
          }
        }
      }
    }
  },
  "session": {
    "maxDiskBytes": "512mb",
    "maxEntries": 200
  },
  "tools": {
    "media": {
      "concurrency": 1
    }
  },
  "logging": {
    "level": "warn",
    "redactSensitive": "tools"
  }
}
EOF
```

## 环境变量优化

```bash
# 添加到 ~/.bashrc
cat >> ~/.bashrc << 'EOF'

# OpenClaw 2GB 优化
export NODE_OPTIONS="--max-old-space-size=1024"
export OPENCLAW_NO_RESPAWN=1
EOF

# 应用环境变量
source ~/.bashrc
```

## 故障排查

### 安装包过大

如果安装包超过500MB，可以只传输核心文件：

```bash
# 创建精简安装包
tar -czf openclaw-arm64-minimal.tar.gz \
  node_modules/@agentclientprotocol \
  node_modules/@anthropic \
  node_modules/@clack \
  node_modules/@mariozechner \
  dist/ \
  openclaw.mjs
```

### 权限问题

如果遇到权限错误：

```bash
# 使用sudo安装
sudo tar -xzf openclaw-arm64.tar.gz -C /usr/local/lib/openclaw
sudo ln -sf /usr/local/lib/openclaw/openclaw.mjs /usr/local/bin/openclaw
```

### 验证安装

```bash
# 检查OpenClaw版本
openclaw --version

# 查看帮助信息
openclaw --help

# 检查系统状态
openclaw doctor
```

## 性能对比

| 安装方式    | 时间      | 说明                   |
| ----------- | --------- | ---------------------- |
| 直接npm安装 | 10-30分钟 | 依赖网络速度，需要编译 |
| 局域网传输  | 2-5分钟   | 快速，跳过编译         |
| 本地安装包  | 1-2分钟   | 最快，适合批量部署     |

## 相关文档

- [树莓派安装指南](/platforms/raspberry-pi.md)
- [配置优化](/gateway/configuration.md)
- [故障排查](/troubleshooting.md)
