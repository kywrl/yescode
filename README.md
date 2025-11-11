# YesCode CLI (yc)

简洁的 YesCode API 调用工具，纯 Bash 实现，支持余额查询等功能。

## 功能特点

- **纯 Bash 实现** - 轻量级，仅依赖 curl 和 jq
- **一键安装** - 自包含脚本，可自行安装/卸载
- **自动配置** - 首次运行自动提示输入 API Key
- **彩色输出** - 美观的终端界面和进度条
- **安全可靠** - API Key 自动安全存储

## 快速开始

### 1. 安装

```bash
./yc install
```

安装过程会自动检测依赖（curl 和 jq），如有缺失会提示安装。

### 2. 使用

```bash
# 查询余额（首次运行会自动提示输入 API Key）
yc balance

# 查看帮助
yc --help

# 卸载
yc uninstall
```

## 命令说明

- `yc balance` - 查询账户余额和订阅信息
- `yc providers` - 查询可用的 AI 提供商
- `yc provider-alternatives <id>` - 查询指定提供商提供的所有可用服务方案
- `yc provider-selection <id>` - 查询指定提供商当前生效的服务方案
- `yc set-provider-selection <id> <alt>` - 设置指定提供商的生效服务方案
- `yc install` - 安装 yc 到系统
- `yc uninstall` - 卸载 yc
- `yc --help` - 显示帮助信息

## 输出示例

### 余额查询

```
💰 余额信息
订阅余额：¥34.80
总余额：¥74.40

📊 本周消费
周限额：¥100.00
已消费：¥5.59 (5.6%)
剩余额度：¥94.41 (94.4%)
消费进度：[█░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] 5.6%
```

### 提供商查询

```
🔌 可用提供商

Basic/Swift API Endpoint [默认]
  ID: provider-123  |  类型: claude  |  费率: ×0.8  |  来源: 订阅

OpenAI [默认]
  ID: provider-456  |  类型: openai  |  费率: ×0.1  |  来源: 订阅

GoogleAPI [默认]
  ID: provider-789  |  类型: google  |  费率: ×0.1  |  来源: 订阅

PAYGO APIs
  ID: provider-abc  |  类型: claude  |  费率: ×0.5  |  来源: 按需
```

### 查询提供商的所有可用服务方案

查看提供商提供的所有可选服务方案（包括原方案和替代方案）：

```
🔄 提供商替代方案

原提供商：OpenAI (openai, ID: 3)

可用替代方案 (4)：

1. OpenAIBackup
   ID: 9  |  类型: openai  |  费率: ×0.2

2. CodexTest
   ID: 11  |  类型: openai  |  费率: ×0.1

3. YesCodeTest(Codex)
   ID: 14  |  类型: openai  |  费率: ×0.1

4. OpenAIOffical
   ID: 16  |  类型: openai  |  费率: ×1
```

### 查询提供商当前生效的服务方案

查看提供商当前实际使用的是哪个服务方案：

```
✓ 当前选择的提供商

提供商 ID：3

当前使用：OpenAI
  ID: 3  |  类型: openai  |  费率: ×0.1
  描述: OpenAI provider (migrated from Admin)
```

### 设置提供商的生效服务方案

将提供商的生效服务方案切换到指定的替代方案：

```
正在设置生效的服务方案...
✓ 服务方案设置成功!

提供商 ID：5
已切换到：YesCodeTest
  ID: 12  |  类型: claude  |  费率: ×0.1
```

**典型使用流程：**
```bash
# 1. 查看提供商 5 有哪些可用服务方案
yc provider-alternatives 5

# 2. 查看提供商 5 当前生效的是哪个方案
yc provider-selection 5

# 3. 切换提供商 5 到服务方案 12
yc set-provider-selection 5 12

# 4. 确认切换成功
yc provider-selection 5
```

## 扩展开发

添加新的 API 功能只需两步：

**1. 添加 API 调用函数**（参考 `api_get_balance()`）

```bash
api_your_function() {
    local url="${API_BASE_URL}/api/v1/your/endpoint"
    local response=$(curl -s -w "\n%{http_code}" \
        -H "X-API-Key: ${API_KEY}" "$url")

    local http_code=$(echo "$response" | tail -n1)
    local body=$(echo "$response" | sed '$d')

    [[ "$http_code" == "200" ]] && echo "$body" || return 1
}
```

**2. 添加命令处理函数**（在 `main()` 的 case 中注册）

```bash
cmd_your_command() {
    local data=$(api_your_function)
    [[ $? -eq 0 ]] && echo "$data" | jq '.'
}
```

## 安全说明

- API Key 自动安全存储，仅本人可访问
- 定期轮换 API Key 以保障安全
- API Base URL 固定为 `https://co.yes.vg`

## 故障排除

### 依赖问题

`./yc install` 会自动检测并提示安装 curl 和 jq。如果自动安装失败，请手动安装：

```bash
# Ubuntu/Debian
sudo apt install curl jq

# RHEL/CentOS
sudo yum install curl jq

# Fedora
sudo dnf install curl jq

# macOS
brew install curl jq

# Arch Linux
sudo pacman -S curl jq

# openSUSE
sudo zypper install curl jq
```

### 认证问题

**401 错误** - 程序会自动提示重新输入 API Key

**重置配置** - 删除配置文件后重新运行：
```bash
rm ~/.yescode/config.json
yc balance  # 将提示重新输入
```

### 网络问题

检查网络连接，确认可访问 `https://co.yes.vg`
