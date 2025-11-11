# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

YesCode CLI (`yc`) 是一个纯 Bash 实现的命令行工具，用于调用 YesCode API。项目特点：
- **单文件架构**：所有功能集中在 `yc` 脚本中
- **零配置依赖**：仅依赖系统工具 `curl` 和 `jq`
- **自包含**：可自行安装/卸载，配置管理完全独立

## 核心架构

### 配置管理
- API Key 存储在 `~/.yescode/config.json`，权限 600
- 配置文件格式：`{"api_key": "xxx"}`
- API Base URL 固定为 `https://co.yes.vg`
- 首次运行或认证失败时自动提示配置

### 功能分层

脚本按功能模块组织：

1. **颜色系统** (行 8-20)：ANSI 转义码定义
2. **配置管理** (行 23-84)：`load_config()`, `prompt_api_key()`
3. **UI 工具** (行 87-134)：动态颜色、进度条
4. **依赖检测** (行 137-205)：跨平台包管理器检测和自动安装
5. **API 层** (行 208-281)：`api_get_balance()`, `api_get_providers()`
6. **显示层** (行 284-389)：格式化输出函数
7. **命令处理** (行 392-643)：`cmd_balance()`, `cmd_providers()`, 等
8. **主程序** (行 646-702)：参数路由

### API 调用模式

所有 API 函数遵循统一模式：

```bash
api_function_name() {
    local url="${API_BASE_URL}/api/v1/endpoint"

    # curl 带状态码
    local response=$(curl -s -w "\n%{http_code}" \
        -H "X-API-Key: ${API_KEY}" "$url")

    # 分离 HTTP 状态码和响应体
    local http_code=$(echo "$response" | tail -n1)
    local body=$(echo "$response" | sed '$d')

    # 401 返回特殊标记用于重新认证
    [[ "$http_code" == "401" ]] && echo "AUTH_ERROR_401" && return 2

    # 其他错误返回 1
    [[ "$http_code" != "200" ]] && return 1

    echo "$body"
}
```

**重要**：退出码 2 表示认证失败，命令处理函数会自动触发重新配置流程。

## 开发指南

### 添加新 API 功能

遵循两步法：

**1. 实现 API 调用函数**

```bash
api_your_feature() {
    local url="${API_BASE_URL}/api/v1/your/endpoint"
    local response=$(curl -s -w "\n%{http_code}" \
        -H "X-API-Key: ${API_KEY}" "$url")

    local http_code=$(echo "$response" | tail -n1)
    local body=$(echo "$response" | sed '$d')

    [[ "$http_code" == "401" ]] && echo "AUTH_ERROR_401" && return 2
    [[ "$http_code" != "200" ]] && return 1

    echo "$body"
}
```

**2. 实现命令处理函数**

```bash
cmd_your_feature() {
    echo "正在查询..."

    local data
    local exit_code

    set +e
    data=$(api_your_feature)
    exit_code=$?
    set -e

    # 处理 401 认证错误
    if [[ $exit_code -eq 2 && "$data" == "AUTH_ERROR_401" ]]; then
        echo -e "${RED}✗ 认证失败${RESET}"
        prompt_api_key false
        load_config

        set +e
        data=$(api_your_feature)
        exit_code=$?
        set -e
    fi

    if [[ $exit_code -eq 0 ]]; then
        display_your_feature "$data"
    else
        echo -e "${RED}✗ 操作失败${RESET}"
        exit 1
    fi
}
```

**3. 注册到 main()**

在 `main()` 的 case 语句中添加：

```bash
your-feature)
    load_config
    cmd_your_feature
    ;;
```

并更新 `show_help()` 函数。

### 颜色使用规范

- 余额：动态颜色 `get_balance_color()` (红<20, 黄<50, 绿≥50)
- 消费：动态颜色 `get_spending_color()` (红≥95%, 黄≥80%, 绿<80%)
- 标题：`${CYAN}${BOLD}`
- 成功/错误：`${GREEN}` / `${RED}`
- 强调数据：`${WHITE}${BOLD}`

### 错误处理原则

1. **set -e 全局启用**：默认遇错退出
2. **API 调用需临时禁用**：使用 `set +e` ... `set -e` 包裹以捕获退出码
3. **认证错误特殊处理**：返回码 2 + "AUTH_ERROR_401" 标记
4. **用户友好错误信息**：所有错误输出到 stderr 并带彩色标记

### 交互式命令实现

`switch` 命令是一个交互式的提供商切换工具，展示了如何构建用户友好的交互流程：

**特点：**
- **两步交互**：先选择分组，再选择该分组的提供商
- **智能标记**：自动显示 [当前] 和 （官方）标记
- **支持直达**：可选参数 `provider_id` 跳过第一步
- **并行加载**：同时获取可用提供商列表和当前选择

**实现要点：**

```bash
# 1. 交互式选择函数返回格式化字符串
interactive_select_provider() {
    # 返回格式: "id|name"
    echo "${provider_id}|${provider_name}"
}

# 2. 并行 API 调用提升性能
api_get_provider_alternatives "$provider_id" > "$temp_file1" &
api_get_provider_selection "$provider_id" > "$temp_file2" &
wait  # 等待两个 API 调用完成

# 3. 标记逻辑
[[ "$alt_id" == "$current_alt_id" ]] && echo "[当前]"
[[ "$is_self" == "true" ]] && echo "（官方）"
```

## 测试和调试

### 手动测试流程

```bash
# 1. 测试安装
./yc install

# 2. 测试 API 功能
yc balance
yc providers

# 3. 测试交互式切换
yc switch         # 完整交互流程
yc switch 5       # 直接切换分组 5

# 4. 测试错误处理（使用无效 API Key）
rm ~/.yescode/config.json
yc balance  # 应提示输入 API Key

# 5. 测试卸载
yc uninstall
```

### 调试技巧

- 查看完整 curl 响应：移除 `-s` 标志
- 检查 JSON 解析：`echo "$data" | jq .`
- 验证 HTTP 状态码：在 API 函数中添加 `echo "HTTP: $http_code" >&2`

## 代码风格约定

- **函数命名**：`snake_case`，API 函数前缀 `api_`，命令函数前缀 `cmd_`
- **局部变量**：总是使用 `local` 声明
- **引号规范**：变量引用总是使用双引号 `"$var"`
- **注释分隔**：使用 `# ====` 样式的 banner 注释分隔大模块
- **缩进**：4 空格

## 限制和注意事项

- **平台兼容性**：依赖 `bc` 进行浮点运算（某些最小化系统可能缺失）
- **jq 依赖**：JSON 解析强依赖 jq，无法降级
- **并发安全**：配置文件读写无锁，不支持并发修改
- **错误恢复**：网络错误不会自动重试，由用户手动重试
