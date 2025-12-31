# 子服务接入 Fence 网关指南

## 快速开始

Fence 是统一认证网关，子服务接入后无需自行处理 JWT 验证，只需从请求 Header 中获取用户信息即可。

## 接入步骤

### 1. 确认网关地址

```
http://fence:5479/verify
```

### 2. 获取用户身份

认证通过后，用户 ID 会通过 `CH-USER` Header 传递到你的服务：

```go
// Go
userID := r.Header.Get("CH-USER")
```

```python
# Python (Flask)
user_id = request.headers.get('CH-USER')
```

```javascript
// Node.js (Express)
const userId = req.headers['ch-user'];
```

```java
// Java (Spring)
String userId = request.getHeader("CH-USER");
```

### 3. 处理未认证情况

如果请求未通过认证，网关会直接返回 `401 Unauthorized`，请求不会到达你的服务。

因此，当请求到达你的服务时，可以认为：
- `CH-USER` 存在 → 已登录用户
- `CH-USER` 不存在 → 白名单路径（公开接口）

## 客户端 Token 传递

告知前端/客户端以下两种方式传递 Token：

### 方式一：Cookie（推荐，Web 端）
```
Cookie: CowboyHat=<jwt_token>
```

### 方式二：Header（推荐，移动端/API）
```
Authorization: Bearer <jwt_token>
```

## 白名单配置

如果你的服务有公开接口（无需登录），需要将路径加入白名单。

### 申请格式
```
METHOD:PATH
```

### 示例
| 接口 | 白名单值 |
|------|----------|
| `GET /api/health` | `GET:/api/health` |
| `POST /api/login` | `POST:/api/login` |
| `GET /api/public/*` | 需逐个添加具体路径 |

### 添加方式
联系运维或自行操作 Redis：
```bash
LPUSH fence:whitelist "GET:/api/your-public-path"
```

## 常见问题

### Q: 如何判断用户是否登录？
检查 `CH-USER` Header 是否存在且非空。

### Q: 白名单路径会有 CH-USER 吗？
不会。白名单路径直接放行，不解析 Token。

### Q: Token 从哪里获取？
由认证服务（如登录接口）签发，签名密钥为 `cowboy_hat`，算法 HS256。

### Q: 如何生成 Token？
```go
claims := jwt.MapClaims{
    "sub": "user_id_here",
    "exp": time.Now().Add(24 * time.Hour).Unix(),
}
token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
tokenString, _ := token.SignedString([]byte("cowboy_hat"))
```

### Q: 请求超时或网关不可用怎么办？
网关返回 `500 Internal Server Error`，建议客户端重试或提示用户稍后再试。

## 架构示意

```
┌──────────┐    ┌─────────┐    ┌──────────┐    ┌──────────┐
│  Client  │───▶│  Nginx  │───▶│  Fence   │    │  Redis   │
│          │    │         │    │ (验证)    │◀──▶│ (白名单)  │
└──────────┘    └────┬────┘    └────┬─────┘    └──────────┘
                     │              │
                     │  ✓ 200 OK    │
                     │  + CH-USER   │
                     ▼              │
               ┌──────────┐        │
               │ 你的服务  │◀───────┘
               │          │
               └──────────┘
```

## 检查清单

- [ ] 确认服务已配置在 Nginx/Traefik 的 auth_request 后
- [ ] 代码中正确读取 `CH-USER` Header
- [ ] 公开接口已加入白名单
- [ ] 前端/客户端已配置 Token 传递方式
