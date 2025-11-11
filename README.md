# YesCode CLI (yc)

YesCode API 调用工具，纯 Bash 实现。

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

# 交互式切换提供商（推荐）
yc switch [id]

# 查看帮助
yc --help

# 卸载
yc uninstall
```

## 命令说明

- `yc balance` - 查询账户余额和订阅信息
- `yc switch [id]` - **交互式切换提供商（推荐）**
- `yc providers` - 查询可用提供商分组信息
- `yc provider-alternatives <id>` - 查询指定分组的所有可用提供商
- `yc provider-selection <id>` - 查询指定分组的当前提供商
- `yc set-provider-selection <id> <alt>` - 设置指定分组的提供商
- `yc install` - 安装 yc 到系统
- `yc uninstall` - 卸载 yc
- `yc --help` - 显示帮助信息

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
