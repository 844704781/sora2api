# Sora 官方 API 接口文档

本文档基于对 Sora2API 项目的深入分析，整理了 Sora 官方 API 的完整接口规范。

> **Base URL**: `https://sora.chatgpt.com/backend`
>
> **API 版本**: 基于 2024-2025 年 Sora API 分析

---

## 目录

- [第一章：认证与 Token 管理](#第一章认证与-token-管理)
- [第二章：Sentinel Token 与 PoW 验证](#第二章sentinel-token-与-pow-验证)
- [第三章：用户信息与订阅](#第三章用户信息与订阅)
- [第四章：文件上传](#第四章文件上传)
- [第五章：生成任务创建](#第五章生成任务创建)
- [第六章：任务状态查询](#第六章任务状态查询)
- [第七章：无水印视频获取](#第七章无水印视频获取)
- [第八章：角色/Cameo 管理](#第八章角色cameo-管理)
- [第九章：参数枚举值速查表](#第九章参数枚举值速查表)
- [第十章：调用流程图](#第十章调用流程图)
- [附录：代理配置说明](#附录代理配置说明)
- [附录：User-Agent 池](#附录user-agent-池)
- [附录：错误码参考](#附录错误码参考)

---

## 第一章：认证与 Token 管理

**调用顺序：1**

Sora API 使用 OpenAI 的认证体系，支持三种 Token 类型：
- **AT (Access Token)**: 以 `eyJhbGciOiJ` 开头的 JWT Token，用于 API 请求认证
- **ST (Session Token)**: 以 `sess-` 开头的会话 Token，可转换为 AT
- **RT (Refresh Token)**: 用于刷新获取新的 AT

### 1.1 Session Token (ST) 转换为 Access Token (AT)

**详情**: 使用 Session Token 获取 Access Token，适用于已登录用户的会话凭证转换。

**URL**: `GET https://sora.chatgpt.com/api/auth/session`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Cookie | string | 是 | 包含 Session Token |
| Accept | string | 是 | 固定值 `application/json` |
| Origin | string | 是 | 固定值 `https://sora.chatgpt.com` |
| Referer | string | 是 | 固定值 `https://sora.chatgpt.com/` |

**Cookie 格式**:
```
__Secure-next-auth.session-token={session_token}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| accessToken | string | Access Token (JWT) |
| user | object | 用户信息对象 |
| user.email | string | 用户邮箱 |
| expires | string | 过期时间 (ISO 8601) |

**示例**:

```bash
curl -X GET "https://sora.chatgpt.com/api/auth/session" \
  -H "Cookie: __Secure-next-auth.session-token=sess-xxxxxxxxxxxxx" \
  -H "Accept: application/json" \
  -H "Origin: https://sora.chatgpt.com" \
  -H "Referer: https://sora.chatgpt.com/"
```

**响应示例**:
```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "email": "user@example.com",
    "name": "User Name"
  },
  "expires": "2025-02-01T00:00:00.000Z"
}
```

**错误码**:

| 状态码 | 错误 | 说明 |
|--------|------|------|
| 401 | Unauthorized | Session Token 无效或已过期 |

---

### 1.2 Refresh Token (RT) 转换为 Access Token (AT)

**详情**: 使用 Refresh Token 刷新获取新的 Access Token，RT 可能会同时更新。

**URL**: `POST https://auth.openai.com/oauth/token`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Accept | string | 是 | 固定值 `application/json` |
| Content-Type | string | 是 | 固定值 `application/json` |

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| client_id | string | 是 | 客户端 ID，默认 `app_LlGpXReQgckcGGUo2JrYvtJK` |
| grant_type | string | 是 | 固定值 `refresh_token` |
| redirect_uri | string | 是 | 回调 URI |
| refresh_token | string | 是 | Refresh Token |

**示例**:

```bash
curl -X POST "https://auth.openai.com/oauth/token" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "app_LlGpXReQgckcGGUo2JrYvtJK",
    "grant_type": "refresh_token",
    "redirect_uri": "com.openai.chat://auth0.openai.com/ios/com.openai.chat/callback",
    "refresh_token": "your_refresh_token_here"
  }'
```

**响应示例**:
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "new_refresh_token_here",
  "expires_in": 86400,
  "token_type": "Bearer"
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| access_token | string | 新的 Access Token |
| refresh_token | string | 新的 Refresh Token（可能更新） |
| expires_in | integer | 有效期（秒） |
| token_type | string | Token 类型，固定值 `Bearer` |

**错误码**:

| 状态码 | 错误 | 说明 |
|--------|------|------|
| 400 | invalid_grant | Refresh Token 无效或已过期 |
| 401 | Unauthorized | 客户端 ID 无效 |

---

### 1.3 Client ID 说明

**详情**: Client ID 是 OAuth 2.0 协议中的客户端标识符，用于在 RT 刷新 AT 时标识请求来源。不同客户端（iOS App、Web 等）使用不同的 Client ID。

#### 为什么需要 Client ID

- RT (Refresh Token) 与特定的 Client ID 绑定
- 刷新时必须使用获取 RT 时相同的 Client ID
- 使用错误的 Client ID 会导致 `invalid_grant` 错误

#### 已知 Client ID 列表

| Client ID | 客户端类型 | redirect_uri |
|-----------|------------|--------------|
| `app_LlGpXReQgckcGGUo2JrYvtJK` | iOS 客户端 (默认) | `com.openai.chat://auth0.openai.com/ios/com.openai.chat/callback` |
| `pdlLIX2Y72MIl2rhLhTE9VV9bN905kBh` | Web 客户端 | `https://chatgpt.com/api/auth/callback/login-web` |

#### 使用场景

1. **Token 录入时选填**：如果 RT 来自非 iOS 客户端，需要指定对应的 Client ID
2. **自动刷新**：系统使用存储的 Client ID 进行 Token 自动刷新
3. **纯 RT 批量导入**：必须指定与 RT 匹配的 Client ID

#### 注意事项

- 如果不确定 RT 来源，可以先尝试默认的 iOS Client ID
- 如果刷新失败返回 `invalid_grant`，尝试使用其他 Client ID
- Client ID 与 redirect_uri 需要配对使用
- 存储 Token 时建议同时保存 Client ID，以便后续自动刷新

---

## 第二章：Sentinel Token 与 PoW 验证

**调用顺序：2（每次生成任务前需要获取）**

Sora 使用 Sentinel Token 机制进行请求验证，包含 Proof of Work (PoW) 计算以防止滥用。

### 2.1 获取 Sentinel Token

**详情**: 获取 PoW 挑战并计算 Sentinel Token，用于生成任务请求的认证。

**URL**: `POST https://chatgpt.com/backend-api/sentinel/req`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Accept | string | 是 | 固定值 `*/*` |
| Content-Type | string | 是 | 固定值 `text/plain;charset=UTF-8` |
| Origin | string | 是 | 固定值 `https://chatgpt.com` |
| Referer | string | 是 | 固定值 `https://chatgpt.com/backend-api/sentinel/frame.html` |
| User-Agent | string | 是 | 浏览器 UA + PoW 初始信息 |
| sec-ch-ua | string | 是 | Chrome UA 信息 |
| sec-ch-ua-mobile | string | 是 | 是否移动端 |
| sec-ch-ua-platform | string | 是 | 平台信息 |

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| p | string | 是 | 初始 PoW Token，以 `gAAAAAC` 开头 |
| id | string | 是 | 请求 ID (UUID v4) |
| flow | string | 是 | 流程标识，固定值 `sora_init` |

**示例**:

```bash
curl -X POST "https://chatgpt.com/backend-api/sentinel/req" \
  -H "Accept: */*" \
  -H "Content-Type: text/plain;charset=UTF-8" \
  -H "Origin: https://chatgpt.com" \
  -H "Referer: https://chatgpt.com/backend-api/sentinel/frame.html" \
  -H 'sec-ch-ua: "Not(A:Brand";v="8", "Chromium";v="131", "Google Chrome";v="131"' \
  -H "sec-ch-ua-mobile: ?1" \
  -H 'sec-ch-ua-platform: "Android"' \
  -d '{"p":"gAAAAAC...","id":"550e8400-e29b-41d4-a716-446655440000","flow":"sora_init"}'
```

**响应示例**:
```json
{
  "token": "sentinel_token_value",
  "turnstile": {
    "dx": "turnstile_dx_value"
  },
  "proofofwork": {
    "required": true,
    "seed": "pow_seed_value",
    "difficulty": "0fffff"
  }
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| token | string | Sentinel Token 核心值 |
| turnstile | object | Turnstile 验证信息 |
| turnstile.dx | string | Turnstile dx 值 |
| proofofwork | object | PoW 挑战信息 |
| proofofwork.required | boolean | 是否需要 PoW 计算 |
| proofofwork.seed | string | PoW 种子值 |
| proofofwork.difficulty | string | 难度值（十六进制） |

---

### 2.2 PoW 计算算法说明

**详情**: 当 `proofofwork.required` 为 `true` 时，需要进行 SHA3-512 哈希碰撞计算。

**算法流程**:

1. 构造配置数组 `config_list`（包含屏幕尺寸、时间、UA 等信息）
2. 迭代计算，直到找到满足难度的哈希值：
   - 将配置数组 JSON 编码后 Base64
   - 拼接 `seed + base64_config`
   - 计算 SHA3-512 哈希
   - 检查哈希前 N 字节是否 <= 难度值

**配置数组结构**:

| 索引 | 类型 | 说明 |
|------|------|------|
| [0] | integer | 屏幕宽度 |
| [1] | string | 本地时间字符串 |
| [2] | integer | jsHeapSizeLimit |
| [3] | integer | 迭代次数（动态） |
| [4] | string | User-Agent |
| [5] | string | Sora CDN 脚本 URL |
| [6] | null | 保留字段 |
| [7] | string | 主语言 |
| [8] | string | 语言列表 |
| [9] | integer | 随机初始值 |
| [10] | string | Navigator 键 |
| [11] | string | Document 键 |
| [12] | string | Window 键 |
| [13] | float | Performance 时间 |
| [14] | string | UUID |
| [15] | string | 空字符串 |
| [16] | integer | CPU 核心数 |
| [17] | float | 时间原点 |

**可选屏幕尺寸**: `1266`, `1536`, `1920`, `2560`, `3000`, `3072`, `3120`, `3840`

**可选 CPU 核心数**: `4`, `8`, `12`, `16`, `24`, `32`

**可选语言**:
| 主语言 | 语言列表 |
|--------|----------|
| zh-CN | zh-CN,zh |
| en-US | en-US,en |
| ja-JP | ja-JP,ja,en |
| ko-KR | ko-KR,ko,en |

**最终 Sentinel Token 格式**:

```json
{
  "p": "gAAAAAB{pow_solution}~S",
  "t": "{turnstile_dx}",
  "c": "{sentinel_token}",
  "id": "{request_id}",
  "flow": "sora_2_create_task__auto"
}
```

---

## 第三章：用户信息与订阅

**调用顺序：3（Token 验证后）**

### 3.1 获取用户信息

**详情**: 获取当前登录用户的基本信息，包括邮箱、用户名等。

**URL**: `GET {base_url}/me`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Accept | string | 是 | 固定值 `application/json` |
| Origin | string | 否 | `https://sora.chatgpt.com` |
| Referer | string | 否 | `https://sora.chatgpt.com/` |

**示例**:

```bash
curl -X GET "https://sora.chatgpt.com/backend/me" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Accept: application/json"
```

**响应示例**:
```json
{
  "email": "user@example.com",
  "name": "User Name",
  "username": "username123",
  "picture": "https://example.com/avatar.jpg",
  "created_at": "2024-01-01T00:00:00Z"
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| email | string | 用户邮箱 |
| name | string | 用户显示名称 |
| username | string \| null | 用户名（可能为 null，需要设置） |
| picture | string | 头像 URL |
| created_at | string | 账户创建时间 |

**错误码**:

| 状态码 | 错误码 | 说明 |
|--------|--------|------|
| 401 | token_invalidated | Token 已失效 |
| 401 | Unauthorized | 未授权 |

---

### 3.2 获取订阅信息

**详情**: 获取用户的 ChatGPT 订阅信息，包括套餐类型和到期时间。

**URL**: `GET https://sora.chatgpt.com/backend/billing/subscriptions`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |

**示例**:

```bash
curl -X GET "https://sora.chatgpt.com/backend/billing/subscriptions" \
  -H "Authorization: Bearer eyJhbGciOiJ..."
```

**响应示例**:
```json
{
  "data": [
    {
      "plan": {
        "id": "chatgpt_plus",
        "title": "ChatGPT Plus"
      },
      "end_ts": "2025-02-15T00:00:00Z"
    }
  ]
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| data | array | 订阅列表 |
| data[].plan.id | string | 套餐 ID |
| data[].plan.title | string | 套餐名称 |
| data[].end_ts | string | 订阅到期时间 (ISO 8601) |

**套餐类型枚举**:

| plan.id | plan.title | 说明 |
|---------|------------|------|
| chatgpt_plus | ChatGPT Plus | 个人 Plus |
| chatgpt_team | ChatGPT Business | 团队版 |
| chatgpt_pro | ChatGPT Pro | 专业版 |

**错误码**:

| 状态码 | 错误码 | 说明 |
|--------|--------|------|
| 401 | token_expired | Token 已过期 |

---

### 3.3 获取 Sora2 邀请码

**详情**: 获取用户的 Sora2 邀请码信息，包括已使用和总数。

**URL**: `GET https://sora.chatgpt.com/backend/project_y/invite/mine`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Accept | string | 是 | 固定值 `application/json` |

**示例**:

```bash
curl -X GET "https://sora.chatgpt.com/backend/project_y/invite/mine" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Accept: application/json"
```

**响应示例**:
```json
{
  "invite_code": "SORA-XXXX-XXXX",
  "redeemed_count": 2,
  "total_count": 5
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| invite_code | string | 邀请码 |
| redeemed_count | integer | 已使用次数 |
| total_count | integer | 总可用次数 |

**错误码**:

| 状态码 | 错误码 | 说明 |
|--------|--------|------|
| 401 | Unauthorized | 账户不支持 Sora2 |
| 403 | unsupported_country_code | 所在地区不支持 Sora |

---

### 3.4 获取 Sora2 剩余配额

**详情**: 获取 Sora2 视频生成的剩余次数和限制状态。

**URL**: `GET https://sora.chatgpt.com/backend/nf/check`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Accept | string | 是 | 固定值 `application/json` |
| User-Agent | string | 否 | 推荐使用移动端 UA |

**示例**:

```bash
curl -X GET "https://sora.chatgpt.com/backend/nf/check" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Accept: application/json" \
  -H "User-Agent: Sora/1.2026.007 (Android 15; 24122RKC7C; build 2600700)"
```

**响应示例**:
```json
{
  "rate_limit_and_credit_balance": {
    "estimated_num_videos_remaining": 27,
    "rate_limit_reached": false,
    "access_resets_in_seconds": 46833
  }
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| rate_limit_and_credit_balance | object | 配额信息 |
| estimated_num_videos_remaining | integer | 估计剩余视频次数 |
| rate_limit_reached | boolean | 是否达到速率限制 |
| access_resets_in_seconds | integer | 配额重置倒计时（秒） |

---

### 3.5 激活 Sora2

**详情**: 对于首次使用 Sora2 的账户，需要先调用此接口激活。

**URL**: `GET https://sora.chatgpt.com/backend/m/bootstrap`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |

**示例**:

```bash
curl -X GET "https://sora.chatgpt.com/backend/m/bootstrap" \
  -H "Authorization: Bearer eyJhbGciOiJ..."
```

**响应示例**:
```json
{
  "success": true
}
```

---

### 3.6 激活 Sora2 邀请码

**详情**: 使用邀请码激活 Sora2 访问权限。

**URL**: `POST https://sora.chatgpt.com/backend/project_y/invite/accept`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | 固定值 `application/json` |
| Cookie | string | 否 | 包含设备 ID: `oai-did={uuid}` |

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| invite_code | string | 是 | Sora2 邀请码 |

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/project_y/invite/accept" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -H "Cookie: oai-did=550e8400-e29b-41d4-a716-446655440000" \
  -d '{"invite_code": "SORA-XXXX-XXXX"}'
```

**响应示例**:
```json
{
  "success": true,
  "already_accepted": false
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| success | boolean | 是否激活成功 |
| already_accepted | boolean | 是否已经激活过 |

---

### 3.7 用户名检查与设置

#### 3.7.1 检查用户名可用性

**详情**: 检查指定用户名是否可用。

**URL**: `POST https://sora.chatgpt.com/backend/project_y/profile/username/check`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | 固定值 `application/json` |

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| username | string | 是 | 要检查的用户名 |

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/project_y/profile/username/check" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -d '{"username": "desired_username"}'
```

**响应示例**:
```json
{
  "available": true
}
```

#### 3.7.2 设置用户名

**详情**: 设置账户用户名（首次使用时必需）。

**URL**: `POST https://sora.chatgpt.com/backend/project_y/profile/username/set`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | 固定值 `application/json` |

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| username | string | 是 | 要设置的用户名 |

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/project_y/profile/username/set" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -d '{"username": "my_username"}'
```

**响应示例**:
```json
{
  "username": "my_username",
  "display_name": "User Name"
}
```

---

## 第四章：文件上传

**调用顺序：4（生成任务前，如需要）**

### 4.1 上传图片文件

**详情**: 上传图片用于图生图或图生视频任务，返回 media_id。

**URL**: `POST {base_url}/uploads`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | multipart/form-data |

**请求体 (multipart/form-data)**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| file | file | 是 | 图片文件（支持 PNG/JPG/WEBP） |
| file_name | string | 是 | 文件名 |

**支持的图片格式**:
- `image/png`
- `image/jpeg`
- `image/webp`

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/uploads" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -F "file=@image.png;type=image/png" \
  -F "file_name=image.png"
```

**响应示例**:
```json
{
  "id": "upload_xxxxxxxxxxxx"
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| id | string | 上传文件的 media_id，用于生成任务 |

---

### 4.2 上传角色视频

**详情**: 上传视频用于创建角色（Cameo），返回 cameo_id。

**URL**: `POST {base_url}/characters/upload`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | multipart/form-data |

**请求体 (multipart/form-data)**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| file | file | 是 | 视频文件（MP4） |
| timestamps | string | 是 | 时间戳范围，格式 `start,end`（如 `0,3`） |

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/characters/upload" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -F "file=@video.mp4;type=video/mp4" \
  -F "timestamps=0,3"
```

**响应示例**:
```json
{
  "id": "cameo_xxxxxxxxxxxx"
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| id | string | cameo_id，用于后续角色创建流程 |

---

### 4.3 上传角色头像

**详情**: 上传角色头像图片，返回 asset_pointer。

**URL**: `POST {base_url}/project_y/file/upload`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | multipart/form-data |

**请求体 (multipart/form-data)**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| file | file | 是 | 图片文件（推荐 WEBP） |
| use_case | string | 是 | 固定值 `profile` |

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/project_y/file/upload" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -F "file=@profile.webp;type=image/webp" \
  -F "use_case=profile"
```

**响应示例**:
```json
{
  "asset_pointer": "asset_ptr_xxxxxxxxxxxx"
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| asset_pointer | string | 资源指针，用于完成角色创建 |

---

## 第五章：生成任务创建

**调用顺序：5**

### 5.1 图片生成

**详情**: 创建图片生成任务，支持文生图和图生图。

**URL**: `POST {base_url}/video_gen`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | 固定值 `application/json` |
| OpenAI-Sentinel-Token | string | 是 | Sentinel Token（JSON 格式） |
| OAI-Device-Id | string | 否 | 设备 ID (UUID) |
| OAI-Language | string | 否 | 语言设置，如 `en-US` |

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| type | string | 是 | 固定值 `image_gen` |
| operation | string | 是 | 操作类型：`simple_compose`（文生图）或 `remix`（图生图） |
| prompt | string | 是 | 生成提示词 |
| width | integer | 是 | 图片宽度 |
| height | integer | 是 | 图片高度 |
| n_variants | integer | 是 | 变体数量，固定值 `1` |
| n_frames | integer | 是 | 帧数，固定值 `1`（图片） |
| inpaint_items | array | 否 | 输入图片列表（图生图时使用） |

**inpaint_items 数组元素**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| type | string | 是 | 固定值 `image` |
| frame_index | integer | 是 | 帧索引，固定值 `0` |
| upload_media_id | string | 是 | 上传图片的 media_id |

**尺寸配置**:

| 模型 | width | height | 说明 |
|------|-------|--------|------|
| gpt-image | 360 | 360 | 正方形 |
| gpt-image-landscape | 540 | 360 | 横屏 |
| gpt-image-portrait | 360 | 540 | 竖屏 |

**示例（文生图）**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/video_gen" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -H "OpenAI-Sentinel-Token: {\"p\":\"gAAAAAB...\",\"t\":\"...\",\"c\":\"...\",\"id\":\"...\",\"flow\":\"sora_2_create_task__auto\"}" \
  -d '{
    "type": "image_gen",
    "operation": "simple_compose",
    "prompt": "A beautiful sunset over mountains",
    "width": 540,
    "height": 360,
    "n_variants": 1,
    "n_frames": 1,
    "inpaint_items": []
  }'
```

**示例（图生图）**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/video_gen" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -H "OpenAI-Sentinel-Token: {...}" \
  -d '{
    "type": "image_gen",
    "operation": "remix",
    "prompt": "Transform to oil painting style",
    "width": 540,
    "height": 360,
    "n_variants": 1,
    "n_frames": 1,
    "inpaint_items": [
      {
        "type": "image",
        "frame_index": 0,
        "upload_media_id": "upload_xxxxxxxxxxxx"
      }
    ]
  }'
```

**响应示例**:
```json
{
  "id": "task_xxxxxxxxxxxx"
}
```

---

### 5.2 视频生成（标准）

**详情**: 创建视频生成任务，支持文生视频和图生视频。

**URL**: `POST {base_url}/nf/create`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | 固定值 `application/json` |
| OpenAI-Sentinel-Token | string | 是 | Sentinel Token（JSON 格式） |
| User-Agent | string | 是 | 浏览器 UA |
| OAI-Device-Id | string | 否 | 设备 ID (UUID) |
| OAI-Language | string | 否 | 语言设置 |

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| kind | string | 是 | 固定值 `video` |
| prompt | string | 是 | 生成提示词 |
| orientation | string | 是 | 视频方向：`landscape` 或 `portrait` |
| size | string | 是 | 视频尺寸：`small`（标准）或 `large`（高清） |
| n_frames | integer | 是 | 帧数：`300`（10s）/`450`（15s）/`750`（25s） |
| model | string | 是 | 模型：`sy_8`（标准）或 `sy_ore`（Pro） |
| inpaint_items | array | 否 | 输入图片列表（图生视频时使用） |
| style_id | string | 否 | 风格 ID |

**inpaint_items 数组元素（视频）**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| kind | string | 是 | 固定值 `upload` |
| upload_id | string | 是 | 上传图片的 media_id |

**模型配置**:

| 模型名称 | model | size | 说明 |
|----------|-------|------|------|
| 标准版 | sy_8 | small | 标准视频 |
| Pro 版 | sy_ore | small | Pro 质量 |
| Pro HD | sy_ore | large | Pro 高清 |

**帧数对应时长**:

| n_frames | 时长 | 说明 |
|----------|------|------|
| 300 | 10s | 短视频 |
| 450 | 15s | 中等视频 |
| 750 | 25s | 长视频（仅标准版和 Pro 版支持） |

**示例（文生视频）**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/nf/create" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -H "OpenAI-Sentinel-Token: {...}" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -d '{
    "kind": "video",
    "prompt": "A cat running through a field of flowers",
    "orientation": "landscape",
    "size": "small",
    "n_frames": 450,
    "model": "sy_8",
    "inpaint_items": [],
    "style_id": null
  }'
```

**示例（图生视频）**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/nf/create" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -H "OpenAI-Sentinel-Token: {...}" \
  -d '{
    "kind": "video",
    "prompt": "The cat starts to run",
    "orientation": "landscape",
    "size": "small",
    "n_frames": 300,
    "model": "sy_8",
    "inpaint_items": [
      {
        "kind": "upload",
        "upload_id": "upload_xxxxxxxxxxxx"
      }
    ]
  }'
```

**响应示例**:
```json
{
  "id": "gen_xxxxxxxxxxxx"
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| id | string | 生成任务 ID，用于查询状态 |

---

### 5.3 分镜视频生成

**详情**: 使用分镜模式生成视频，可以精确控制每个镜头的时长和内容。

**URL**: `POST {base_url}/nf/create/storyboard`

**请求头**: 同 5.2 视频生成

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| kind | string | 是 | 固定值 `video` |
| prompt | string | 是 | 格式化的分镜提示词 |
| title | string | 否 | 标题，默认 `Draft your video` |
| orientation | string | 是 | 视频方向 |
| size | string | 是 | 视频尺寸 |
| n_frames | integer | 是 | 总帧数 |
| storyboard_id | string | 否 | 分镜 ID（可选） |
| inpaint_items | array | 否 | 输入图片列表 |
| remix_target_id | string | 否 | Remix 目标 ID |
| model | string | 是 | 模型类型 |
| metadata | object | 否 | 元数据 |
| style_id | string | 否 | 风格 ID |
| cameo_ids | array | 否 | 角色 ID 列表 |
| cameo_replacements | object | 否 | 角色替换映射 |
| audio_caption | string | 否 | 音频描述 |
| audio_transcript | string | 否 | 音频转录 |
| video_caption | string | 否 | 视频描述 |

**分镜提示词格式**:

```
current timeline:
Shot 1:
duration: 5.0sec
Scene: 描述第一个镜头的内容

Shot 2:
duration: 5.0sec
Scene: 描述第二个镜头的内容

instructions:
整体视频的总述（可选）
```

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/nf/create/storyboard" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -H "OpenAI-Sentinel-Token: {...}" \
  -d '{
    "kind": "video",
    "prompt": "current timeline:\nShot 1:\nduration: 5.0sec\nScene: A cat jumping from a roof\n\nShot 2:\nduration: 5.0sec\nScene: The cat landing gracefully\n\ninstructions:\nCinematic style",
    "title": "Draft your video",
    "orientation": "landscape",
    "size": "small",
    "n_frames": 300,
    "model": "sy_8",
    "storyboard_id": null,
    "inpaint_items": [],
    "remix_target_id": null,
    "metadata": null,
    "style_id": null,
    "cameo_ids": null,
    "cameo_replacements": null,
    "audio_caption": null,
    "audio_transcript": null,
    "video_caption": null
  }'
```

---

### 5.4 Remix 视频

**详情**: 基于已发布的视频进行二次创作。

**URL**: `POST {base_url}/nf/create`

**请求体**（在基础视频生成请求上增加）:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| remix_target_id | string | 是 | 源视频的 Post ID（如 `s_690d100857248191b679e6de12db840e`） |
| cameo_ids | array | 否 | 角色 ID 列表 |
| cameo_replacements | object | 否 | 角色替换映射 |

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/nf/create" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -H "OpenAI-Sentinel-Token: {...}" \
  -d '{
    "kind": "video",
    "prompt": "Change the background to a beach",
    "inpaint_items": [],
    "remix_target_id": "s_690d100857248191b679e6de12db840e",
    "cameo_ids": [],
    "cameo_replacements": {},
    "model": "sy_8",
    "orientation": "portrait",
    "n_frames": 450,
    "style_id": null
  }'
```

---

### 5.5 提示词优化

**详情**: 使用 AI 优化和扩展提示词，生成更详细的视频描述。

**URL**: `POST {base_url}/editor/enhance_prompt`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | 固定值 `application/json` |

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| prompt | string | 是 | 原始提示词 |
| expansion_level | string | 是 | 扩展级别：`short`/`medium`/`long` |
| duration_s | integer | 是 | 目标时长（秒）：`10`/`15`/`20` |

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/editor/enhance_prompt" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "A cat playing",
    "expansion_level": "medium",
    "duration_s": 15
  }'
```

**响应示例**:
```json
{
  "enhanced_prompt": "A fluffy orange tabby cat playfully pounces on a colorful toy mouse in a sunlit living room. The cat's movements are swift and graceful, with its tail swishing excitedly. Soft afternoon light streams through sheer curtains, casting warm shadows across the hardwood floor. The cat pauses momentarily, ears perked forward, before leaping again with joyful energy."
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| enhanced_prompt | string | 优化后的提示词 |

---

## 第六章：任务状态查询

**调用顺序：6（创建任务后轮询）**

### 6.1 获取待处理任务

**详情**: 获取当前正在处理的视频生成任务列表及进度信息。

**URL**: `GET {base_url}/nf/pending/v2`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |

**示例**:

```bash
curl -X GET "https://sora.chatgpt.com/backend/nf/pending/v2" \
  -H "Authorization: Bearer eyJhbGciOiJ..."
```

**响应示例**:
```json
[
  {
    "id": "gen_xxxxxxxxxxxx",
    "status": "processing",
    "progress": 0.45,
    "created_at": "2025-01-15T10:30:00Z",
    "prompt": "A cat running through flowers",
    "orientation": "landscape",
    "n_frames": 450
  }
]
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| id | string | 任务 ID |
| status | string | 状态：`pending`/`processing`/`completed`/`failed` |
| progress | float | 进度 (0.0 - 1.0) |
| created_at | string | 创建时间 |
| prompt | string | 提示词 |
| orientation | string | 视频方向 |
| n_frames | integer | 帧数 |

**状态枚举**:

| status | 说明 |
|--------|------|
| pending | 等待处理 |
| processing | 正在生成 |
| completed | 已完成 |
| failed | 失败 |

---

### 6.2 获取已完成任务（草稿）

**详情**: 获取已完成的视频草稿列表。

**URL**: `GET {base_url}/project_y/profile/drafts?limit={limit}`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |

**查询参数**:

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| limit | integer | 否 | 15 | 返回数量限制 |

**示例**:

```bash
curl -X GET "https://sora.chatgpt.com/backend/project_y/profile/drafts?limit=15" \
  -H "Authorization: Bearer eyJhbGciOiJ..."
```

**响应示例**:
```json
{
  "drafts": [
    {
      "id": "gen_xxxxxxxxxxxx",
      "status": "completed",
      "created_at": "2025-01-15T10:30:00Z",
      "prompt": "A cat running through flowers",
      "video_url": "https://example.com/video.mp4",
      "thumbnail_url": "https://example.com/thumb.jpg",
      "duration_seconds": 15,
      "orientation": "landscape"
    }
  ]
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| drafts | array | 草稿列表 |
| drafts[].id | string | 任务 ID (generation_id) |
| drafts[].status | string | 状态 |
| drafts[].video_url | string | 视频 URL（带水印） |
| drafts[].thumbnail_url | string | 缩略图 URL |
| drafts[].duration_seconds | integer | 视频时长（秒） |

---

### 6.3 获取最近图片任务

**详情**: 获取最近的图片生成任务列表。

**URL**: `GET {base_url}/v2/recent_tasks?limit={limit}`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |

**查询参数**:

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| limit | integer | 否 | 20 | 返回数量限制 |

**示例**:

```bash
curl -X GET "https://sora.chatgpt.com/backend/v2/recent_tasks?limit=20" \
  -H "Authorization: Bearer eyJhbGciOiJ..."
```

**响应示例**:
```json
{
  "tasks": [
    {
      "id": "task_xxxxxxxxxxxx",
      "type": "image_gen",
      "status": "completed",
      "created_at": "2025-01-15T10:30:00Z",
      "prompt": "A beautiful sunset",
      "image_url": "https://example.com/image.png"
    }
  ]
}
```

---

## 第七章：无水印视频获取

**调用顺序：7（视频生成完成后，可选）**

Sora 生成的视频默认带有水印，需要通过发布流程获取无水印版本。

### 7.1 发布视频

**详情**: 将草稿视频发布为帖子，发布后可获取无水印视频 URL。

**URL**: `POST {base_url}/project_y/post`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | 固定值 `application/json` |
| OpenAI-Sentinel-Token | string | 是 | Sentinel Token |

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| attachments_to_create | array | 是 | 附件列表 |
| post_text | string | 是 | 帖子文本（可为空） |

**attachments_to_create 数组元素**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| generation_id | string | 是 | 视频生成 ID |
| kind | string | 是 | 固定值 `sora` |

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/project_y/post" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -H "OpenAI-Sentinel-Token: {...}" \
  -d '{
    "attachments_to_create": [
      {
        "generation_id": "gen_01k9btrqrnen792yvt703dp0tq",
        "kind": "sora"
      }
    ],
    "post_text": ""
  }'
```

**响应示例**:
```json
{
  "post": {
    "id": "s_690ce161c2488191a3476e9969911522",
    "video_url": "https://example.com/watermark_free_video.mp4",
    "created_at": "2025-01-15T10:35:00Z"
  }
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| post.id | string | 帖子 ID，用于删除或分享 |
| post.video_url | string | 无水印视频 URL |

---

### 7.2 删除已发布视频

**详情**: 删除已发布的帖子（获取无水印 URL 后可删除以保持隐私）。

**URL**: `DELETE {base_url}/project_y/post/{post_id}`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |

**路径参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| post_id | string | 是 | 帖子 ID |

**示例**:

```bash
curl -X DELETE "https://sora.chatgpt.com/backend/project_y/post/s_690ce161c2488191a3476e9969911522" \
  -H "Authorization: Bearer eyJhbGciOiJ..."
```

**响应**: 成功返回 HTTP 204 No Content

---

### 7.3 第三方解析服务

**详情**: 通过第三方服务获取无水印视频下载链接。

**URL**: `POST {parse_url}/get-sora-link`

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| url | string | 是 | Sora 分享链接 |
| token | string | 是 | 解析服务 Token |

**示例**:

```bash
curl -X POST "http://parse-server.example.com/get-sora-link" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://sora.chatgpt.com/p/s_690c0f574c3881918c3bc5b682a7e9fd",
    "token": "your_parse_token"
  }'
```

**响应示例**:
```json
{
  "download_link": "https://example.com/direct_download.mp4"
}
```

---

## 第八章：角色/Cameo 管理

**调用顺序：独立流程**

角色（Cameo）功能允许用户创建自定义角色用于视频生成。

### 8.1 获取角色处理状态

**详情**: 查询角色视频的处理进度。

**URL**: `GET {base_url}/project_y/cameos/in_progress/{cameo_id}`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |

**示例**:

```bash
curl -X GET "https://sora.chatgpt.com/backend/project_y/cameos/in_progress/cameo_xxxxxxxxxxxx" \
  -H "Authorization: Bearer eyJhbGciOiJ..."
```

**响应示例**:
```json
{
  "status": "completed",
  "display_name_hint": "Suggested Name",
  "username_hint": "suggested_username",
  "profile_asset_url": "https://example.com/profile.webp",
  "instruction_set_hint": "Character description..."
}
```

**响应参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| status | string | 处理状态：`processing`/`completed`/`failed` |
| display_name_hint | string | 建议的显示名称 |
| username_hint | string | 建议的用户名 |
| profile_asset_url | string | 生成的头像 URL |
| instruction_set_hint | string | 角色描述提示 |

---

### 8.2 完成角色创建

**详情**: 完成角色创建流程，确认角色信息。

**URL**: `POST {base_url}/characters/finalize`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | 固定值 `application/json` |

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| cameo_id | string | 是 | Cameo ID |
| username | string | 是 | 角色用户名 |
| display_name | string | 是 | 角色显示名称 |
| profile_asset_pointer | string | 是 | 头像资源指针 |
| instruction_set | null | 是 | 固定值 `null` |
| safety_instruction_set | null | 是 | 固定值 `null` |

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/characters/finalize" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -d '{
    "cameo_id": "cameo_xxxxxxxxxxxx",
    "username": "my_character",
    "display_name": "My Character",
    "profile_asset_pointer": "asset_ptr_xxxxxxxxxxxx",
    "instruction_set": null,
    "safety_instruction_set": null
  }'
```

**响应示例**:
```json
{
  "character": {
    "character_id": "char_xxxxxxxxxxxx"
  }
}
```

---

### 8.3 设置角色公开

**详情**: 将角色设置为公开，允许其他用户使用。

**URL**: `POST {base_url}/project_y/cameos/by_id/{cameo_id}/update_v2`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |
| Content-Type | string | 是 | 固定值 `application/json` |

**请求体**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| visibility | string | 是 | 可见性：`public` 或 `private` |

**示例**:

```bash
curl -X POST "https://sora.chatgpt.com/backend/project_y/cameos/by_id/cameo_xxxxxxxxxxxx/update_v2" \
  -H "Authorization: Bearer eyJhbGciOiJ..." \
  -H "Content-Type: application/json" \
  -d '{"visibility": "public"}'
```

---

### 8.4 删除角色

**详情**: 删除已创建的角色。

**URL**: `DELETE {base_url}/project_y/characters/{character_id}`

**请求头**:

| Header | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Authorization | string | 是 | Bearer {access_token} |

**示例**:

```bash
curl -X DELETE "https://sora.chatgpt.com/backend/project_y/characters/char_xxxxxxxxxxxx" \
  -H "Authorization: Bearer eyJhbGciOiJ..."
```

**响应**: 成功返回 HTTP 200 或 204

---

## 第九章：参数枚举值速查表

### 视频方向 (orientation)

| 值 | 说明 |
|----|------|
| landscape | 横屏 (16:9) |
| portrait | 竖屏 (9:16) |

### 模型 (model)

| 值 | 说明 |
|----|------|
| sy_8 | 标准版模型 |
| sy_ore | Pro 版模型 |

### 视频尺寸 (size)

| 值 | 说明 |
|----|------|
| small | 标准分辨率 |
| large | 高清分辨率（仅 Pro 版支持） |

### 提示词扩展级别 (expansion_level)

| 值 | 说明 |
|----|------|
| short | 简短扩展 |
| medium | 中等扩展 |
| long | 详细扩展 |

### 视频风格 (style_id)

| 值 | 说明 |
|----|------|
| festive | 节日风格 |
| kakalaka | Kakalaka 风格 |
| news | 新闻风格 |
| selfie | 自拍风格 |
| handheld | 手持摄影风格 |
| golden | 金色调风格 |
| anime | 动漫风格 |
| retro | 复古风格 |
| nostalgic | 怀旧风格 |
| comic | 漫画风格 |

### 视频帧数与时长对应 (n_frames)

| n_frames | 时长 | 支持的模型 |
|----------|------|------------|
| 300 | 10s | 全部 |
| 450 | 15s | 全部 |
| 750 | 25s | sy_8, sy_ore (small) |

### 完整模型名称映射

| 模型名称 | type | model | size | orientation | n_frames |
|----------|------|-------|------|-------------|----------|
| gpt-image | image | - | - | - | - |
| gpt-image-landscape | image | - | - | landscape | - |
| gpt-image-portrait | image | - | - | portrait | - |
| sora2-landscape-10s | video | sy_8 | small | landscape | 300 |
| sora2-portrait-10s | video | sy_8 | small | portrait | 300 |
| sora2-landscape-15s | video | sy_8 | small | landscape | 450 |
| sora2-portrait-15s | video | sy_8 | small | portrait | 450 |
| sora2-landscape-25s | video | sy_8 | small | landscape | 750 |
| sora2-portrait-25s | video | sy_8 | small | portrait | 750 |
| sora2pro-landscape-10s | video | sy_ore | small | landscape | 300 |
| sora2pro-portrait-10s | video | sy_ore | small | portrait | 300 |
| sora2pro-landscape-15s | video | sy_ore | small | landscape | 450 |
| sora2pro-portrait-15s | video | sy_ore | small | portrait | 450 |
| sora2pro-landscape-25s | video | sy_ore | small | landscape | 750 |
| sora2pro-portrait-25s | video | sy_ore | small | portrait | 750 |
| sora2pro-hd-landscape-10s | video | sy_ore | large | landscape | 300 |
| sora2pro-hd-portrait-10s | video | sy_ore | large | portrait | 300 |
| sora2pro-hd-landscape-15s | video | sy_ore | large | landscape | 450 |
| sora2pro-hd-portrait-15s | video | sy_ore | large | portrait | 450 |
| prompt-enhance-short-10s | prompt | - | - | - | - |
| prompt-enhance-medium-15s | prompt | - | - | - | - |
| prompt-enhance-long-20s | prompt | - | - | - | - |

---

## 第十章：调用流程图

### 完整 API 调用流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           完整 API 调用流程                                   │
└─────────────────────────────────────────────────────────────────────────────┘

1. 认证阶段
   ┌─────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │ ST / RT │ ──► │ /api/auth/session       │ ──► │  Access Token   │
   └─────────┘     │ /oauth/token            │     └─────────────────┘
                   └─────────────────────────┘

2. 验证阶段（每次生成任务前）
   ┌─────────────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │  Access Token   │ ──► │ /backend-api/sentinel/  │ ──► │ Sentinel Token  │
   │  + PoW 计算     │     │ req (含 PoW 挑战)       │     │                 │
   └─────────────────┘     └─────────────────────────┘     └─────────────────┘

3. 用户信息获取（可选/初始化时）
   ┌─────────────────┐     ┌─────────────────────────┐
   │  Access Token   │ ──► │ /me                     │ ──► 用户基本信息
   └─────────────────┘     │ /billing/subscriptions  │ ──► 订阅信息
                           │ /nf/check               │ ──► 剩余配额
                           └─────────────────────────┘

4. 资源上传（按需）
   ┌─────────────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │  图片/视频文件  │ ──► │ /uploads                │ ──► │   upload_id     │
   └─────────────────┘     │ /characters/upload      │     │   cameo_id      │
                           └─────────────────────────┘     └─────────────────┘

5. 任务创建
   ┌─────────────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │  Prompt +       │ ──► │ /video_gen (图片)       │ ──► │    task_id      │
   │  Sentinel Token │     │ /nf/create (视频)       │     │ generation_id   │
   └─────────────────┘     │ /nf/create/storyboard   │     └─────────────────┘
                           └─────────────────────────┘

6. 状态轮询
   ┌─────────────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │    task_id      │ ──► │ /nf/pending/v2          │ ──► │    进度信息     │
   └─────────────────┘     │ /project_y/profile/     │     │    完成结果     │
              ▲            │ drafts                  │     └────────┬────────┘
              │            └─────────────────────────┘              │
              │                                                     │
              └─────────────── 未完成则继续轮询 ◄───────────────────┘

7. 后处理（可选 - 获取无水印视频）
   ┌─────────────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │ generation_id   │ ──► │ /project_y/post         │ ──► │    post_id      │
   └─────────────────┘     └─────────────────────────┘     │ 无水印视频 URL   │
                                                           └────────┬────────┘
                                                                    ▼
   ┌─────────────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │    post_id      │ ──► │ DELETE /project_y/post  │ ──► │   删除发布      │
   └─────────────────┘     └─────────────────────────┘     └─────────────────┘
```

### 角色创建流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           角色创建流程                                        │
└─────────────────────────────────────────────────────────────────────────────┘

1. 上传角色视频
   ┌─────────────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │   视频文件      │ ──► │ /characters/upload      │ ──► │   cameo_id      │
   └─────────────────┘     └─────────────────────────┘     └─────────────────┘

2. 轮询处理状态
   ┌─────────────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │   cameo_id      │ ──► │ /project_y/cameos/      │ ──► │ 状态 + 头像 URL │
   └─────────────────┘     │ in_progress/{cameo_id}  │     │ + 建议信息      │
              ▲            └─────────────────────────┘     └────────┬────────┘
              │                                                     │
              └─────────────── status != completed ◄────────────────┘

3. 下载并重新上传头像
   ┌─────────────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │ profile_asset_  │ ──► │ 下载图片                │ ──► │ 图片数据        │
   │ url             │     └─────────────────────────┘     └────────┬────────┘
   └─────────────────┘                                              │
                                                                    ▼
   ┌─────────────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │   图片数据      │ ──► │ /project_y/file/upload  │ ──► │ asset_pointer   │
   └─────────────────┘     └─────────────────────────┘     └─────────────────┘

4. 完成角色创建
   ┌─────────────────┐     ┌─────────────────────────┐     ┌─────────────────┐
   │ cameo_id +      │ ──► │ /characters/finalize    │ ──► │ character_id    │
   │ asset_pointer   │     └─────────────────────────┘     └─────────────────┘
   └─────────────────┘

5. 设置公开（可选）
   ┌─────────────────┐     ┌─────────────────────────┐
   │   cameo_id      │ ──► │ /project_y/cameos/      │ ──► 公开成功
   └─────────────────┘     │ by_id/{id}/update_v2    │
                           └─────────────────────────┘
```

---

## 附录：代理配置说明

本项目支持三种层级的代理配置，用于不同的使用场景。

### 代理类型概览

| 代理类型 | 配置位置 | 作用范围 | 使用场景 |
|----------|----------|----------|----------|
| Token 级别代理 | 录入账号时填写 | 仅该 Token | Token 来自特定地区，需要单独代理 |
| 全局代理 | 系统配置 → 代理配置 | 所有未配置单独代理的 Token | 统一使用一个代理访问 Sora API |
| POW 代理 | 系统配置 → POW代理配置 | Sentinel Token 获取 | 浏览器 PoW 验证需要单独代理 |

### 1. Token 级别代理

**配置位置**：录入账号时的 `proxy_url` 字段

**作用**：为单个 Token 指定专属代理，该代理用于该 Token 的所有 API 请求（获取用户信息、订阅信息、创建任务、轮询状态等）。

**优先级**：最高。如果 Token 配置了代理，则忽略全局代理。

**使用场景**：
- Token 来自不同地区，需要使用不同的代理
- 部分 Token 需要代理，部分不需要
- 每个 Token 对应不同的代理账户（避免 IP 关联）

**示例**：
```
http://127.0.0.1:7890
socks5://user:pass@proxy.example.com:1080
```

### 2. 全局代理

**配置位置**：系统配置 → 代理配置

**作用**：为所有未配置单独代理的 Token 提供统一的代理。

**优先级**：次于 Token 级别代理。仅当 Token 没有配置单独代理时生效。

**使用场景**：
- 所有 Token 都需要使用相同的代理
- 服务器本身无法直接访问 Sora API
- 作为默认代理，部分 Token 可通过配置单独代理覆盖

**配置项**：
| 字段 | 类型 | 说明 |
|------|------|------|
| proxy_enabled | bool | 是否启用全局代理 |
| proxy_url | string | 代理地址 |

### 3. POW 代理

**配置位置**：系统配置 → POW代理配置

**作用**：专门用于 Playwright 浏览器获取 Sentinel Token。

**独立性**：完全独立于上面两种代理，即使全局代理和 Token 代理都配置了，POW 代理也可以单独开启/关闭。

**使用场景**：
- 服务器 IP 被 Cloudflare 识别，需要通过代理获取 Sentinel Token
- 浏览器自动化需要特定的代理（如需要住宅 IP 才能通过 PoW 验证）
- API 请求和 PoW 验证需要使用不同的代理

**为什么要分开配置**：
1. **网络环境不同**：API 请求通常用数据中心代理即可，但 Cloudflare 的 PoW 验证可能需要住宅 IP
2. **代理稳定性**：PoW 验证通过 Playwright 浏览器执行，需要更稳定的代理，避免浏览器超时
3. **成本考虑**：住宅代理比数据中心代理贵，可以只在 PoW 环节使用

**配置项**：
| 字段 | 类型 | 说明 |
|------|------|------|
| pow_proxy_enabled | bool | 是否启用 POW 代理 |
| pow_proxy_url | string | POW 代理地址 |

### 代理使用流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           代理选择流程                                        │
└─────────────────────────────────────────────────────────────────────────────┘

                            API 请求代理选择
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │  Token 有单独配置代理吗？    │
                    └─────────────────────────────┘
                         │              │
                        有              无
                         │              │
                         ▼              ▼
           ┌───────────────────┐  ┌─────────────────────────┐
           │ 使用 Token 代理    │  │  全局代理开启了吗？      │
           └───────────────────┘  └─────────────────────────┘
                                       │              │
                                      是              否
                                       │              │
                                       ▼              ▼
                         ┌───────────────────┐  ┌───────────────────┐
                         │ 使用全局代理       │  │ 不使用代理         │
                         └───────────────────┘  └───────────────────┘

────────────────────────────────────────────────────────────────────────────────

                            POW 验证代理选择（独立）
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │  POW 代理开启了吗？          │
                    └─────────────────────────────┘
                         │              │
                        是              否
                         │              │
                         ▼              ▼
           ┌───────────────────┐  ┌───────────────────┐
           │ 浏览器使用 POW 代理 │  │ 浏览器不使用代理    │
           └───────────────────┘  └───────────────────┘
```

### 代理格式说明

支持 HTTP 和 SOCKS5 代理：

```
# HTTP 代理
http://127.0.0.1:7890
http://username:password@proxy.example.com:8080

# SOCKS5 代理
socks5://127.0.0.1:1080
socks5://username:password@proxy.example.com:1080
```

### 常见问题

**Q: 为什么配置了全局代理但某些 Token 还是连接失败？**
A: 检查该 Token 是否配置了单独的代理（Token 代理优先级高于全局代理）。

**Q: 什么情况下需要配置 POW 代理？**
A: 当遇到 `Cloudflare challenge` 或 `Sentinel Token 获取失败` 错误时，可能需要配置住宅 IP 代理作为 POW 代理。

**Q: 可以只配置 POW 代理不配置其他代理吗？**
A: 可以。三种代理完全独立，可以任意组合使用。

---

## 附录：User-Agent 池

### 桌面端 User-Agent

```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36
Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36
Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36
Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36
Mozilla/5.0 (Windows NT 11.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36
```

### 移动端 User-Agent (Sora App)

```
Sora/1.2026.007 (Android 15; 24122RKC7C; build 2600700)
Sora/1.2026.007 (Android 14; SM-G998B; build 2600700)
Sora/1.2026.007 (Android 15; Pixel 8 Pro; build 2600700)
Sora/1.2026.007 (Android 14; Pixel 7; build 2600700)
Sora/1.2026.007 (Android 15; 2211133C; build 2600700)
Sora/1.2026.007 (Android 14; SM-S918B; build 2600700)
Sora/1.2026.007 (Android 15; OnePlus 12; build 2600700)
```

---

## 附录：错误码参考

### 通用错误码

| HTTP 状态码 | 错误码 | 说明 |
|-------------|--------|------|
| 400 | bad_request | 请求参数错误 |
| 401 | token_invalidated | Token 已失效 |
| 401 | token_expired | Token 已过期 |
| 401 | Unauthorized | 未授权 |
| 403 | unsupported_country_code | 所在地区不支持 |
| 429 | rate_limit_exceeded | 请求频率超限 |
| 500 | internal_error | 服务器内部错误 |
| 503 | heavy_load | 服务器负载过高 |

### 生成任务错误

| 错误码 | 说明 |
|--------|------|
| content_policy_violation | 内容违规 |
| safety_filter_triggered | 安全过滤器触发 |
| generation_failed | 生成失败 |
| timeout | 任务超时 |

---

**文档版本**: 1.0
**最后更新**: 2025-01
**基于项目**: Sora2API
