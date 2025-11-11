# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

**YesCode CLI** 是一个纯 Bash 实现的命令行工具，用于管理 YesCode API 的服务提供商配置和账户余额查询。主要特性：
- 单文件可执行脚本，无需构建过程
- 中文用户界面，带有彩色输出和加载动画
- 自动安装依赖（curl, jq）和系统集成

## 核心架构

### 单体 Bash 脚本结构
整个项目是一个单独的 `yc` 文件，按功能模块组织：

```
配置管理        → API 密钥存储在 ~/.yescode/config.json
API 层       → 5个核心 API 函数（balance, providers, alternatives, selection, set）
UI/显示层     → 彩色输出、加载动画、进度条、交互式菜单
命令处理器  → 每个命令一个 cmd_*() 函数
系统集成  → 包管理器检测和自动安装
```

### 关键交互模式
所有 API 调用遵循统一的模式：
```bash
(api_call > temp_file) &              # 后台执行
show_loading "提示信息" $pid           # 显示加载动画
wait $pid                             # 等待完成
[检查 401 错误] → 重新提示 API Key
```

## 开发命令

### 测试变更
```bash
# 直接运行源文件
./yc balance

# 安装后测试
./yc install
yc balance

# 卸载
yc uninstall
```

### 添加新 API 端点

**步骤 1**: 添加 API 函数
```bash
api_your_function() {
    local url="${API_BASE_URL}/api/v1/your/endpoint"
    local response=$(curl -s -w "\n%{http_code}" \
        -H "X-API-Key: ${API_KEY}" "$url")

    local http_code=$(echo "$response" | tail -n1)
    local body=$(echo "$response" | sed '$d')

    # 关键：401 错误必须返回特殊标记以触发重试
    [[ "$http_code" == "401" ]] && echo "AUTH_ERROR_401" && return 2
    [[ "$http_code" == "200" ]] && echo "$body" || return 1
}
```

**步骤 2**: 添加命令处理函数
```bash
cmd_your_command() {
    # 使用加载动画模式
    local temp_file=$(mktemp)
    (api_your_function > "$temp_file") &
    show_loading "加载中" $!
    wait $!
    local exit_code=$?

    # 处理 401 认证错误
    if [[ $exit_code -eq 2 ]]; then
        local response=$(cat "$temp_file")
        if [[ "$response" == "AUTH_ERROR_401" ]]; then
            # 触发 API Key 重新输入并递归调用
            handle_auth_error
            rm -f "$temp_file"
            cmd_your_command
            return
        fi
    fi

    # 显示结果
    display_your_data "$temp_file"
    rm -f "$temp_file"
}
```

**步骤 3**: 在 `main()` 函数的 case 语句中注册

### 修改显示输出
- 编辑 `display_*()` 函数
- 使用预定义颜色变量：`RED`, `GREEN`, `YELLOW`, `BLUE`, `CYAN`, `GRAY`, `RESET`
- 使用 `jq` 解析 JSON：`jq -r '.path.to.field' file.json`

## 代码规范

### 命名约定
- API 函数：`api_动词_名词()` 例如 `api_get_balance()`
- 命令处理：`cmd_命令名()` 例如 `cmd_balance()`
- 显示函数：`display_名词()` 例如 `display_balance()`
- 工具函数：`动词_名词()` 例如 `check_dependencies()`

### 格式规范
- 4 空格缩进
- 函数间用 `# ============` 分隔符
- 中文注释和用户提示
- 错误检查使用 `[[ ]]` 而非 `[ ]`

### 错误处理模式
```bash
# 脚本级别
set -e  # 遇错即停

# API 级别
[[ "$http_code" == "401" ]] && echo "AUTH_ERROR_401" && return 2
[[ "$http_code" == "200" ]] && echo "$body" || return 1

# 命令级别
if [[ $? -ne 0 ]]; then
    echo "${RED}✗ 操作失败${RESET}"
    return 1
fi
```

## 术语说明（重要！）

项目中的术语在近期提交中已标准化：
- **分组** (Group)：API 端点中的 `id` 参数，表示一组相关的提供商
- **提供商** (Provider)：具体的 AI 服务提供商，在分组内可切换

## API 集成

**Base URL**: `https://co.yes.vg`
**认证**: HTTP Header `X-API-Key: ${API_KEY}`
**超时设置**:
- 连接超时：10 秒
- 最大超时：30 秒

### 核心端点
- `GET /api/v1/balance` - 账户余额
- `GET /api/v1/providers` - 提供商分组列表
- `GET /api/v1/provider-alternatives/{id}` - 分组内的提供商列表
- `GET /api/v1/provider-selection/{id}` - 当前激活的提供商
- `POST /api/v1/provider-selection/{id}` - 切换提供商（Body: `{"alternative":"provider_name"}`）

## 依赖管理

**必需依赖**：
- `curl` - HTTP 请求
- `jq` - JSON 解析

**可选依赖**：
- `bc` - 浮点数计算（用于进度条）
- `tput` - 终端控制（用于光标操作）

安装脚本会自动检测包管理器（apt, yum, dnf, brew, pacman, zypper）并提示安装。

## 配置文件

**位置**: `~/.yescode/config.json`
**权限**: 600（仅所有者可读写）
**格式**:
```json
{
  "api_key": "your-api-key-here"
}
```

首次运行任何命令时，如果配置文件不存在，会自动提示输入 API Key。

## 常见陷阱

1. **临时文件清理**：使用 `mktemp` 创建临时文件后，必须在函数末尾用 `rm -f` 清理
2. **后台进程等待**：使用 `wait $pid` 前要保存 PID：`command & local pid=$!`
3. **401 重试逻辑**：必须返回 `AUTH_ERROR_401` 字符串标记，而非简单的退出码
4. **JSON 解析**：始终使用 `jq -r` 的 `-r` 参数获取原始字符串（无引号）
5. **颜色代码**：输出到管道或非终端时会导致问题，但当前实现未检测 TTY

## 扩展点

- **新 API 端点**：遵循 `api_*()` → `cmd_*()` → `main()` case 三步模式
- **新显示格式**：添加 `display_*()` 函数，使用颜色变量和 Unicode 符号
- **新包管理器**：在 `install_dependencies()` 函数添加检测逻辑
- **新命令**：在 `main()` 的 case 语句和 `cmd_help()` 中同步添加
