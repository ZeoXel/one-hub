# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

One Hub 是基于 [one-api](https://github.com/songquanpeng/one-api) 开发的 AI API 网关服务，支持多个 AI 供应商的统一接入和管理。

## 核心开发命令

### 构建与运行
```bash
# 使用 Task (推荐方式)
task build          # 构建完整项目（包含前端和后端）
task web            # 仅构建前端
task run            # 构建并运行服务
task clean          # 清理构建文件

# 使用 Makefile
make one-api        # 构建完整项目
make web            # 构建前端
make clean          # 清理构建文件

# 直接运行
go run main.go      # 开发模式运行
```

### 代码质量与测试
```bash
# 格式化和代码检查
task fmt            # 运行 go mod tidy、gofmt、goimports 和 golangci-lint
task lint           # 运行 gofmt 和 golangci-lint
task gofmt          # 格式化 Go 代码
task golint         # 运行 golangci-lint

# Go 模块管理
task gomod          # 运行 go mod tidy
```

### 前端开发
```bash
cd web
yarn install       # 安装依赖
yarn dev           # 开发模式运行前端
yarn build         # 构建前端用于生产
yarn lint          # 检查代码格式
yarn lint:fix      # 自动修复代码格式问题
```

### Docker 构建
```bash
task docker         # 构建 Docker 镜像
docker-compose up   # 使用 docker-compose 运行
```

## 项目架构

### 后端架构（Go + Gin）
- **路由层** (`router/`): API 路由、中继路由、仪表板路由和 MCP 路由
- **控制器层** (`controller/`): 处理 HTTP 请求，包括渠道管理、用户管理、计费等
- **中间件层** (`middleware/`): 认证、限流、日志、指标收集等
- **服务提供商** (`providers/`): 各种 AI 服务提供商的适配器（OpenAI、Claude、Gemini 等）
- **中继层** (`relay/`): 请求转发和响应处理
- **模型层** (`model/`): 数据库操作和数据结构定义
- **通用组件** (`common/`): 配置、缓存、通知、限流等共享功能

### 前端架构（React + Vite）
- **框架**: React 18 + Material-UI
- **构建工具**: Vite
- **状态管理**: Redux
- **路由**: React Router v6
- **国际化**: i18next
- **图表**: ApexCharts

### 数据库支持
程序支持三种数据库，通过全局变量判断：
- `common.UsingPostgreSQL` - 使用 PostgreSQL
- `common.UsingSQLite` - 使用 SQLite  
- 都为 false 时使用 MySQL

使用 GORM 作为 ORM，确保代码兼容多种数据库。

### 缓存系统
- **Redis**: 通过 `common.RedisEnabled` 判断是否启用
- **内存缓存**: 可选的内存缓存以减少数据库查询

## 关键配置

### 环境配置
- 主要配置文件: `config.yaml`（参考 `config.example.yaml`）
- 环境变量优先级高于配置文件
- 支持通过 CLI 参数传递配置

### 重要配置项
```yaml
port: 3000                    # 服务端口
sql_dsn: ""                   # 数据库连接字符串
redis_conn_string: ""         # Redis 连接字符串
memory_cache_enabled: false   # 内存缓存开关
node_type: "master"           # 节点类型（master/slave）
mcp:
  enable: false              # MCP 服务开关
```

## 开发规范

### Go 代码规范
- 使用 Gin 框架进行 HTTP 服务开发
- 使用 GORM 进行数据库操作，确保多数据库兼容性
- Redis 操作前检查 `common.RedisEnabled`
- 遵循标准的 Go 项目结构和命名规范

### JavaScript 代码规范  
- 使用 Vite + React 技术栈
- 遵循 Material-UI 设计规范
- 使用 ESLint 和 Prettier 保持代码风格一致

### 数据库操作注意事项
- 所有数据库操作必须兼容 MySQL、PostgreSQL 和 SQLite
- 使用 GORM 提供的数据库无关性功能
- 避免使用特定数据库的 SQL 语法

## 主要功能模块

### 供应商管理 (`providers/`)
支持 20+ AI 服务供应商，包括 OpenAI、Claude、Gemini、百度文心、通义千问等。每个供应商都有独立的适配器实现。

### 渠道管理 (`controller/channel.go`)
- 渠道配置和测试
- 负载均衡和故障转移
- 渠道状态监控

### 用户和令牌管理
- 用户认证和授权
- API 令牌管理
- 使用量统计和计费

### 计费系统 (`payment/`)
支持多种支付方式：支付宝、微信支付、Stripe 等

### 通知系统 (`common/notify/`)
支持多种通知方式：邮件、企业微信、钉钉、飞书、Telegram 等

## 测试

目前项目包含一些测试文件：
```bash
# 运行特定测试
go test ./common/stmp/...     # 测试邮件功能
go test ./providers/ali/...   # 测试阿里云供应商
```

## 部署

### 配置文件部署
1. 复制 `config.example.yaml` 为 `config.yaml`
2. 根据环境修改配置
3. 使用 `task build` 构建
4. 运行生成的二进制文件

### Docker 部署
```bash
# 构建镜像
task docker

# 使用 docker-compose
docker-compose up -d
```

### 环境变量
支持通过环境变量覆盖配置文件，变量名格式：将配置项的点号替换为下划线并转为大写。
例如：`mcp.enable` → `MCP_ENABLE`

## MCP (Model Context Protocol) 支持

项目支持 MCP 协议，相关代码在 `mcp/` 目录：
- `mcp/server.go`: MCP 服务器实现
- `mcp/tools/`: MCP 工具实现
- 通过配置 `mcp.enable: true` 启用