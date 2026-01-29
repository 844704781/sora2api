# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Sora2API 是一个 OpenAI 兼容的 API 服务，为 OpenAI Sora 提供统一接口，支持文生图、图生图、文生视频、图生视频、视频角色、Remix、分镜等功能。

## 常用命令

### 本地开发
```bash
# 安装依赖
pip install -r requirements.txt

# 安装 Playwright 浏览器（用于 PoW 验证）
playwright install chromium

# 启动服务
python main.py
```

### Docker 部署
```bash
# 标准模式
docker-compose up -d

# WARP 代理模式
docker-compose -f docker-compose.warp.yml up -d

# 查看日志
docker-compose logs -f
```

## 架构概述

```
src/
├── main.py              # FastAPI 应用入口
├── core/                # 核心模块
│   ├── config.py        # 配置管理（读取 setting.toml）
│   ├── database.py      # SQLite 异步数据库操作
│   ├── models.py        # Pydantic 数据模型
│   ├── auth.py          # API Key 和管理员认证
│   └── logger.py        # 调试日志
├── services/            # 业务逻辑层
│   ├── token_manager.py      # Token 生命周期管理（AT/ST/RT 刷新）
│   ├── sora_client.py        # Sora API 客户端（PoW、Sentinel Token）
│   ├── generation_handler.py # 生成任务处理（模型配置、重试、去水印）
│   ├── load_balancer.py      # Token 负载均衡（随机/轮询）
│   ├── concurrency_manager.py # 并发限制管理
│   ├── token_lock.py         # Token 分布式锁
│   ├── file_cache.py         # 文件缓存服务
│   └── proxy_manager.py      # 代理配置管理
├── api/                 # API 路由
│   ├── routes.py        # 用户 API（/v1/models, /v1/chat/completions）
│   └── admin.py         # 管理后台 API
└── utils/               # 工具函数
```

## 关键模块说明

### Token 类型
- **AT (Access Token)**: OpenAI 标准 token，以 `eyJhbGciOiJ` 开头
- **ST (Session Token)**: 以 `sess-` 开头的会话 token
- **RT (Refresh Token)**: 用于刷新其他 token

### 生成流程
1. 用户请求 → `routes.py` 验证 API Key
2. `generation_handler.py` 解析模型配置、选择 Token
3. `load_balancer.py` 根据策略选择可用 Token
4. `sora_client.py` 调用 Sora 后端 API
5. 轮询任务状态直到完成
6. 返回流式/非流式响应

### 数据库表
- `tokens` - Token 存储和配额
- `token_stats` - Token 使用统计
- `tasks` - 任务记录
- `request_logs` - 请求日志
- `*_config` - 各类配置表

## 配置文件

主配置：`config/setting.toml`

关键配置项：
- `global.api_key` - 用户 API 密钥
- `sora.base_url` - Sora 后端地址
- `admin.error_ban_threshold` - 连续错误禁用阈值
- `call_logic.call_mode` - Token 选择模式（default/polling）

## API 端点

### 用户 API
- `GET /v1/models` - 列出可用模型
- `POST /v1/chat/completions` - 创建生成任务

### 管理 API（需登录）
- `/admin/tokens/*` - Token 管理
- `/admin/config/*` - 配置管理
- `/admin/tasks/*` - 任务管理
- `/admin/logs/*` - 日志查询

## 支持的模型

- **图片**: `gpt-image`, `gpt-image-landscape`, `gpt-image-portrait`
- **视频标准版**: `sora2-{landscape/portrait}-{10s/15s/25s}`
- **视频 Pro 版**: `sora2pro-{landscape/portrait}-{10s/15s/25s}`
- **视频 Pro HD**: `sora2pro-hd-{landscape/portrait}-{10s/15s}`
- **提示词优化**: `prompt-enhance-{short/medium/long}-{10s/15s/20s}`
