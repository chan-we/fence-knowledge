# Fence 架构文档

## 概述

Fence 是一个轻量级的 JWT 认证网关服务，设计用于与反向代理（如 Nginx、Traefik）配合使用，提供 Forward Auth 功能。

## 核心功能

- JWT Token 验证
- 基于 Redis 的路径白名单
- 用户身份透传

## 系统架构

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│   Nginx     │────▶│  Backend    │
│             │     │  (Proxy)    │     │  Service    │
└─────────────┘     └──────┬──────┘     └─────────────┘
                          │
                          │ Forward Auth
                          ▼
                   ┌─────────────┐     ┌─────────────┐
                   │   Fence     │────▶│   Redis     │
                   │  (:5479)    │     │             │
                   └─────────────┘     └─────────────┘
```

## 认证流程

```
Request ──▶ 检查白名单 ──▶ [匹配] ──▶ 200 OK (放行)
                │
                ▼ [不匹配]
           获取 Token
           (Cookie/Header)
                │
                ▼
           验证 JWT
                │
        ┌───────┴───────┐
        ▼               ▼
   [验证成功]       [验证失败]
        │               │
        ▼               ▼
   设置 ch-user      401 Unauthorized
   返回 200 OK
```

## API 端点

### GET /verify

反向代理调用的认证端点。

**请求 Headers:**
| Header | 说明 |
|--------|------|
| `X-Forwarded-Uri` | 原始请求路径 |
| `X-Forwarded-Method` | 原始请求方法 |
| `Cookie: CowboyHat=<token>` | JWT Token (方式一) |
| `Authorization: Bearer <token>` | JWT Token (方式二) |

**响应:**
| 状态码 | 说明 | Headers |
|--------|------|---------|
| 200 | 认证成功 | `ch-user: <user_id>` |
| 401 | 认证失败 | - |
| 405 | 方法不允许 | - |
| 500 | 服务内部错误 | - |

## Token 规范

### JWT 配置
- **签名算法**: HS256
- **签名密钥**: `cowboy_hat`
- **用户标识字段**: `sub`

### Token 获取优先级
1. Cookie `CowboyHat`
2. Header `Authorization: Bearer <token>`

### Claims 结构
```json
{
  "sub": "user_id",
  ...
}
```

## 白名单机制

### Redis 配置
- **Key**: `fence:whitelist`
- **类型**: List
- **格式**: `METHOD:PATH`

### 示例
```bash
# 添加白名单
LPUSH fence:whitelist "GET:/api/public"
LPUSH fence:whitelist "POST:/api/login"

# 查看白名单
LRANGE fence:whitelist 0 -1
```

## 环境变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `REDIS_HOST` | Redis 主机 | `127.0.0.1` |
| `REDIS_PORT` | Redis 端口 | `6379` |
| `REDIS_PASS` | Redis 密码 | `password` |

## 子服务接入指南

### 1. Nginx 配置示例

```nginx
server {
    location /api/ {
        auth_request /auth;
        auth_request_set $user $upstream_http_ch_user;

        proxy_set_header CH-USER $user;
        proxy_pass http://backend;
    }

    location = /auth {
        internal;
        proxy_pass http://fence:5479/verify;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Forwarded-Uri $request_uri;
        proxy_set_header X-Forwarded-Method $request_method;
    }
}
```

### 2. Traefik 配置示例

```yaml
http:
  middlewares:
    fence-auth:
      forwardAuth:
        address: "http://fence:5479/verify"
        authResponseHeaders:
          - "ch-user"
```

### 3. 子服务获取用户信息

认证成功后，用户 ID 通过 `ch-user` header 传递给后端服务：

```go
func handler(w http.ResponseWriter, r *http.Request) {
    userID := r.Header.Get("CH-USER")
    // userID 即为 JWT 中的 sub 字段
}
```

## 依赖

| 包 | 版本 | 用途 |
|---|------|------|
| `github.com/go-redis/redis/v8` | v8 | Redis 客户端 |
| `github.com/golang-jwt/jwt/v5` | v5 | JWT 解析 |
| `github.com/joho/godotenv` | - | 环境变量加载 |
