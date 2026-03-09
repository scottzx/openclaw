# 局域网文件访问配置指南

## 适用场景

当您需要在局域网内从 Windows 或 Mac 电脑访问开发板上的 OpenClaw 文件时，可以通过 Samba 实现文件共享，无需每次使用 SSH。

**优势：**

- ✅ Windows/Mac 原生支持，无需安装额外软件
- ✅ 图形界面操作，像使用本地文件夹一样简单
- ✅ 实时编辑配置文件、查看会话日志
- ✅ 适合小白用户使用

## 快速开始

### 第一步：在开发板上配置 Samba

SSH 连接到开发板，运行以下命令：

```bash
curl -sSL https://openclaw.ai/scripts/setup-openclaw-samba.sh | bash
```

或者手动下载运行：

```bash
# 下载配置脚本
wget https://openclaw.ai/scripts/setup-openclaw-samba.sh

# 运行配置
chmod +x setup-openclaw-samba.sh
./setup-openclaw-samba.sh
```

配置完成后，脚本会显示您的访问地址。

### 第二步：从 Windows/Mac 访问

根据您的操作系统，选择相应的访问方式（见下方详细说明）。

---

## Windows 用户使用指南

### 方法 1：直接访问（推荐）

1. 打开"文件资源管理器"（快捷键 `Win + E`）
2. 在地址栏输入：`\\开发板IP\openclaw`
3. 按回车，即可看到 OpenClaw 文件夹
4. 双击打开文件，可以直接编辑

**示例：**

```
\\192.168.1.100\openclaw
```

### 方法 2：映射网络驱动器（更方便）

将远程文件夹映射为本地驱动器（如 Z: 盘）：

1. 打开"文件资源管理器"
2. 点击顶部"映射网络驱动器"按钮
   - 或者在"此电脑"上右键 → "映射网络驱动器"
3. 在弹出的对话框中：
   - **驱动器**：选择一个盘符（如 `Z:`）
   - **文件夹**：输入 `\\开发板IP\openclaw`
   - ✅ 勾选"登录时重新连接"
4. 点击"完成"

以后您可以在"此电脑"中直接看到这个驱动器，像使用本地硬盘一样。

### 方法 3：创建快捷方式

1. 在桌面上右键 → 新建 → 快捷方式
2. 输入位置：`\\开发板IP\openclaw`
3. 命名为"OpenClaw 开发板"

---

## Mac 用户使用指南

### 方法 1：通过 Finder 连接

1. 打开 Finder
2. 按快捷键 `⌘ + K`，或点击菜单"前往" → "连接服务器..."
3. 在弹出的对话框中输入：`smb://开发板IP/openclaw`
4. 点击"连接"

**示例：**

```
smb://192.168.1.100/openclaw
```

### 方法 2：添加到侧边栏（推荐）

连接成功后：

1. 找到挂载的 OpenClaw 文件夹
2. 将其拖到 Finder 侧边栏的"位置"区域
3. 以后可以直接点击侧边栏快速访问

### 方法 3：自动挂载（开机自动连接）

1. 打开"系统设置" → "通用" → "服务器"
2. 点击"+"添加服务器
3. 输入：`smb://开发板IP/openclaw`
4. 勾选"登录时连接"

---

## 常见使用场景

### 场景 1：编辑配置文件

直接在共享文件夹中编辑 `openclaw.json`：

```
Windows: \\192.168.1.100\openclaw\openclaw.json
Mac: smb://192.168.1.100/openclaw/openclaw.json
```

修改后保存，OpenClaw 会自动重新加载配置（如果启用了热重载）。

### 场景 2：查看会话日志

浏览历史会话：

```
Windows: \\192.168.1.100\openclaw\sessions\
Mac: smb://192.168.1.100/openclaw/sessions/
```

会话文件是 `.jsonl` 格式，可以用文本编辑器查看。

### 场景 3：访问工作空间文件

查看 Agent 生成的文件：

```
Windows: \\192.168.1.100\openclaw\workspace\
Mac: smb://192.168.1.100/openclaw/workspace/
```

### 场景 4：备份配置

直接将整个 `openclaw` 文件夹复制到本地电脑作为备份：

**Windows：**

1. 打开 `\\开发板IP\openclaw`
2. 全选文件（`Ctrl + A`）
3. 复制到本地目录（`Ctrl + C`，然后 `Ctrl + V`）

**Mac：**

1. 打开 Finder 中的 OpenClaw 共享
2. 全选文件（`⌘ + A`）
3. 拖拽到本地文件夹

---

## 配置脚本说明

### 脚本功能

`setup-openclaw-samba.sh` 脚本会自动完成以下操作：

1. ✅ 检测操作系统（仅支持 Debian/Ubuntu 系的 Linux）
2. ✅ 安装 Samba 服务
3. ✅ 配置共享目录（~/.openclaw）
4. ✅ 设置正确的文件权限
5. ✅ 配置防火墙规则（如果使用 ufw）
6. ✅ 启动并启用 Samba 服务
7. ✅ 显示访问地址和说明

### 脚本内容

```bash
#!/bin/bash
# setup-openclaw-samba.sh - OpenClaw 局域网文件共享配置脚本

set -e

echo "🔧 正在配置 OpenClaw 文件共享..."

# 检测系统
if [ ! -f /etc/os-release ]; then
    echo "❌ 不支持的系统"
    exit 1
fi

# 检查是否为 root
if [ "$EUID" -ne 0 ]; then
    echo "请使用 sudo 运行此脚本"
    exit 1
fi

# 安装 Samba
echo "📦 安装 Samba..."
apt-get update
apt-get install -y samba

# 获取当前用户
SUDO_USER=${SUDO_USER:-pi}
OPENCLAW_DIR="/home/$SUDO_USER/.openclaw"

# 检查 OpenClaw 目录是否存在
if [ ! -d "$OPENCLAW_DIR" ]; then
    echo "⚠️  警告: OpenClaw 目录不存在: $OPENCLAW_DIR"
    echo "请先安装并运行 OpenClaw"
    exit 1
fi

# 配置 Samba 共享
echo "⚙️  配置共享目录..."
cat > /etc/samba/smb.conf << EOF
[global]
   workgroup = WORKGROUP
   server string = OpenClaw Board
   security = user
   map to guest = bad user
   obey interface restrictions = yes
   bind interfaces only = yes

[openclaw]
   path = $OPENCLAW_DIR
   browseable = yes
   read only = no
   create mask = 0644
   directory mask = 0755
   guest ok = yes
   force user = $SUDO_USER
EOF

# 设置权限
chmod 755 "$OPENCLAW_DIR"
chown -R $SUDO_USER:$SUDO_USER "$OPENCLAW_DIR"

# 配置防火墙（如果存在）
if command -v ufw &> /dev/null; then
    echo "🔥 配置防火墙..."
    ufw allow samba
fi

# 重启 Samba 服务
echo "🚀 启动 Samba 服务..."
systemctl restart smbd
systemctl enable smbd

# 获取 IP 地址
IP_ADDR=$(hostname -I | awk '{print $1}')

echo ""
echo "✅ 配置完成！"
echo ""
echo "📁 访问方式："
echo ""
echo "   Windows 用户："
echo "   1. 打开文件资源管理器"
echo "   2. 在地址栏输入: \\\\$IP_ADDR\\openclaw"
echo "   3. 或映射为网络驱动器"
echo ""
echo "   Mac 用户："
echo "   1. 打开 Finder"
echo "   2. 按 ⌘K 或点击"前往" → "连接服务器""
echo "   3. 输入: smb://$IP_ADDR/openclaw"
echo ""
echo "💡 提示: 可以映射为网络驱动器，方便日常使用"
echo ""
echo "🔧 管理命令："
echo "   启动服务: sudo systemctl start smbd"
echo "   停止服务: sudo systemctl stop smbd"
echo "   重启服务: sudo systemctl restart smbd"
echo "   查看状态: sudo systemctl status smbd"
```

---

## 高级配置

### 设置访问密码

如果需要设置密码保护：

```bash
# 在开发板上运行
sudo smbpasswd -a pi  # 将 pi 替换为您的用户名
```

然后修改 `/etc/samba/smb.conf`，将 `guest ok = yes` 改为：

```ini
[openclaw]
   ...
   guest ok = no
   valid users = pi
```

重启服务：

```bash
sudo systemctl restart smbd
```

### 添加多个共享目录

编辑 `/etc/samba/smb.conf`，添加新的共享配置：

```ini
[openclaw-workspace]
   path = /home/pi/.openclaw/workspace
   browseable = yes
   read only = no
   guest ok = yes

[openclaw-sessions]
   path = /home/pi/.openclaw/sessions
   browseable = yes
   read only = yes  # 只读
   guest ok = yes
```

### 仅允许特定 IP 访问

在 `/etc/samba/smb.conf` 中添加：

```ini
[global]
   ...
   hosts allow = 192.168.1. 192.168.2. 127.0.0.1
   hosts deny = 0.0.0.0/0
```

---

## 故障排查

### 问题 1：无法访问共享

**检查开发板 IP：**

```bash
# 在开发板上运行
hostname -I
```

**检查 Samba 服务状态：**

```bash
sudo systemctl status smbd
```

**重启 Samba 服务：**

```bash
sudo systemctl restart smbd
```

### 问题 2：Windows 提示"无法访问"

**检查网络连接：**

```bash
# 在 Windows 上运行
ping 开发板IP
```

**检查防火墙：**

```bash
# 在开发板上运行
sudo ufw status
```

如果防火墙阻止，添加规则：

```bash
sudo ufw allow samba
```

### 问题 3：Mac 无法连接

尝试使用 `cifs://` 而不是 `smb://`：

```
cifs://192.168.1.100/openclaw
```

### 问题 4：文件权限问题

确保 OpenClaw 目录权限正确：

```bash
chmod 755 ~/.openclaw
chmod -R 644 ~/.openclaw/*
chmod -R 755 ~/.openclaw/*/
```

### 问题 5：修改配置后不生效

修改 Samba 配置后需要重启服务：

```bash
sudo systemctl restart smbd nmbd
```

---

## 安全建议

### 局域网使用

1. ✅ 确保开发板在安全的局域网内
2. ✅ 不要将开发板直接暴露在公网
3. ✅ 定期备份重要配置文件

### 生产环境

如果需要公网访问，建议：

1. ❌ 不要使用 Samba（不安全）
2. ✅ 使用 VPN 连接到局域网
3. ✅ 使用 SSHFS 或 SFTP
4. ✅ 配置防火墙限制访问来源

---

## 相关文档

- [局域网快速安装](/installment/lan-fast-install.md)
- [完整配置指南](/installment/complete-configuration-guide.md)
- [故障排查](/troubleshooting.md)
- [网关配置](/gateway/configuration.md)

---

## 技术支持

如果遇到问题：

1. 查看[故障排查](#故障排查)部分
2. 运行诊断命令：`openclaw doctor`
3. 在 GitHub 上提 Issue：[https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
