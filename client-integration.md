# 前端 & CLI 客户端接入指南

面向前端项目和 CLI 工具，说明如何通过 Fence 网关联调后端子服务。

## Token 传递方式

Fence 按以下优先级提取 Token：

| 优先级 | 方式 | 适用场景 |
|--------|------|----------|
| 1 | Cookie `CowboyHat` | Web 前端 |
| 2 | Header `Authorization: Bearer <token>` | CLI / 移动端 / 非浏览器客户端 |

## 前端项目接入

### 登录后存储 Token

登录接口返回 JWT 后，存入名为 `CowboyHat` 的 Cookie：

```javascript
// 登录成功后设置 Cookie
document.cookie = `CowboyHat=${token}; path=/; SameSite=Lax`;

// 如果网关使用 HTTPS
document.cookie = `CowboyHat=${token}; path=/; SameSite=Lax; Secure`;
```

如果登录接口由后端通过 `Set-Cookie` 设置，前端无需手动处理。

### 请求发送

使用 Cookie 方式时，浏览器会自动携带，确保 `credentials` 配置正确：

```javascript
// fetch
const res = await fetch('/api/some-endpoint', {
  credentials: 'include',  // 必须，否则不带 Cookie
});

// axios
axios.defaults.withCredentials = true;
const res = await axios.get('/api/some-endpoint');
```

如果选择 Header 方式：

```javascript
const res = await fetch('/api/some-endpoint', {
  headers: {
    'Authorization': `Bearer ${token}`,
  },
});
```

### 处理 401 响应

网关认证失败时返回 `401`，前端应统一拦截并跳转登录：

```javascript
// axios 拦截器示例
axios.interceptors.response.use(
  (res) => res,
  (err) => {
    if (err.response?.status === 401) {
      window.location.href = '/login';
    }
    return Promise.reject(err);
  }
);

// fetch 封装示例
async function request(url, options = {}) {
  const res = await fetch(url, { credentials: 'include', ...options });
  if (res.status === 401) {
    window.location.href = '/login';
    return;
  }
  return res;
}
```

### 本地开发代理配置

本地开发时前端通常运行在 `localhost:3000` 等端口，需要代理到网关所在环境：

**Vite:**

```javascript
// vite.config.js
export default {
  server: {
    proxy: {
      '/api': {
        target: 'https://your-gateway-domain.com',
        changeOrigin: true,
        cookieDomainRewrite: 'localhost',
      },
    },
  },
};
```

**Webpack (create-react-app):**

```javascript
// src/setupProxy.js
const { createProxyMiddleware } = require('http-proxy-middleware');

module.exports = function (app) {
  app.use('/api', createProxyMiddleware({
    target: 'https://your-gateway-domain.com',
    changeOrigin: true,
    cookieDomainRewrite: 'localhost',
  }));
};
```

**Next.js:**

```javascript
// next.config.js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'https://your-gateway-domain.com/api/:path*',
      },
    ];
  },
};
```

## CLI 工具接入

### curl

```bash
# 使用 Bearer Token
curl -H "Authorization: Bearer <token>" https://your-gateway-domain.com/api/endpoint

# 使用 Cookie
curl -b "CowboyHat=<token>" https://your-gateway-domain.com/api/endpoint
```

### httpie

```bash
# 使用 Bearer Token
http https://your-gateway-domain.com/api/endpoint "Authorization: Bearer <token>"

# 使用 Cookie
http https://your-gateway-domain.com/api/endpoint "Cookie: CowboyHat=<token>"
```

### 在脚本中复用 Token

```bash
# 将 Token 存入变量，避免重复粘贴
export FENCE_TOKEN="eyJhbGciOiJI..."

curl -H "Authorization: Bearer $FENCE_TOKEN" https://your-gateway-domain.com/api/users
curl -H "Authorization: Bearer $FENCE_TOKEN" https://your-gateway-domain.com/api/orders
```

## 生成测试 Token

联调时需要一个合法的 JWT。以下是各语言的生成方式：

**Go:**

```go
claims := jwt.MapClaims{
    "sub": "test-user-123",
    "exp": time.Now().Add(24 * time.Hour).Unix(),
}
token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
tokenString, _ := token.SignedString([]byte("cowboy_hat"))
fmt.Println(tokenString)
```

**Node.js:**

```bash
npm install jsonwebtoken
```

```javascript
const jwt = require('jsonwebtoken');

const token = jwt.sign(
  { sub: 'test-user-123' },
  'cowboy_hat',
  { algorithm: 'HS256', expiresIn: '24h' }
);
console.log(token);
```

**Python:**

```bash
pip install pyjwt
```

```python
import jwt, datetime

token = jwt.encode(
    {"sub": "test-user-123", "exp": datetime.datetime.utcnow() + datetime.timedelta(hours=24)},
    "cowboy_hat",
    algorithm="HS256",
)
print(token)
```

**命令行一键生成（需要 Node.js）：**

```bash
node -e "console.log(require('jsonwebtoken').sign({sub:'test-user-123'},'cowboy_hat',{expiresIn:'24h'}))"
```

### Token 规范

| 字段 | 说明 |
|------|------|
| 签名算法 | HS256 |
| 签名密钥 | `cowboy_hat` |
| 用户标识 | `sub` 字段 |
| 过期时间 | `exp` 字段（Unix 时间戳） |

## 白名单路径（免认证）

白名单路径无需携带 Token，网关直接放行。

### 格式

```
METHOD:PATH_PATTERN
```

### 通配符支持

| 模式 | 说明 | 示例 |
|------|------|------|
| 精确匹配 | 完全一致 | `GET:/api/health` |
| `*` | 匹配单个路径段 | `GET:/api/*/info` 匹配 `/api/user/info`、`/api/order/info` |
| `/**` | 匹配任意后续路径（仅限末尾） | `GET:/public/**` 匹配 `/public/css/a.css`、`/public/js/b.js` |

### 查看当前白名单

```bash
redis-cli LRANGE fence:whitelist 0 -1
```

### 添加白名单

```bash
redis-cli LPUSH fence:whitelist "GET:/api/public/health"
redis-cli LPUSH fence:whitelist "POST:/api/auth/login"
redis-cli LPUSH fence:whitelist "GET:/static/**"
```

## 联调排查

### 请求被 401 拦截

1. **确认 Token 是否正确传递**
   ```bash
   # 用 curl 直接测试
   curl -v -H "Authorization: Bearer $FENCE_TOKEN" https://your-gateway-domain.com/api/endpoint
   ```

2. **确认 Token 是否过期**
   ```bash
   # 解码 Token 查看 payload（不验证签名）
   echo "<token>" | cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool
   ```

3. **确认 Cookie 是否发送**
   - 浏览器 DevTools → Network → 选中请求 → 查看 Request Headers 中是否有 `Cookie: CowboyHat=...`
   - 检查 `credentials: 'include'` 是否配置

4. **确认路径是否在白名单中**
   ```bash
   redis-cli LRANGE fence:whitelist 0 -1
   ```

### 白名单不生效

- 检查 method 是否匹配（区分大小写，必须大写）：`GET:/path` 不等于 `get:/path`
- 检查路径格式：必须以 `/` 开头
- `/**` 通配符不匹配基础路径本身：`GET:/public/**` 不匹配 `/public`，匹配 `/public/xxx`

### 本地联调 CORS 问题

如果直接从 `localhost` 请求远程网关遇到 CORS 错误，使用前端代理（见上方 Vite/Webpack/Next.js 配置）将请求转发到网关，避免跨域。

## 架构示意

```
前端 / CLI
    │
    │  携带 Token（Cookie 或 Bearer Header）
    ▼
┌─────────────┐
│  Nginx      │ ── auth_request ──▶ Fence(:5479) ◀──▶ Redis
│  (反向代理)  │                        │
└──────┬──────┘                        │
       │                               │
       │  200 OK + CH-USER header      │
       ▼                               │
┌─────────────┐                        │
│  后端子服务   │  ◀── 401 则请求不到达 ──┘
└─────────────┘
```
