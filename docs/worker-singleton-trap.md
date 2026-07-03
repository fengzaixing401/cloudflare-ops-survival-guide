# Worker 单例状态陷阱 (Works Once Then Fails)

> 你的 Worker 第一次请求正常，后续请求开始出错或返回过期数据。这不是 Bug，是你把 Worker 当成了有状态服务。

## 症状

- Worker 第一次部署/调用后工作正常
- 后续请求偶尔返回过期数据、空响应、或直接报错
- 重新部署后暂时恢复，然后又坏了
- 本地 `wrangler dev` 测试一切正常

## 根因：Worker 的执行模型

Cloudflare Workers 是 **无状态**的 Isolate，但有一个致命的误解区域：

```javascript
// ❌ 错误！模块级变量在多次请求间共享（同一个 Isolate 实例内）
let counter = 0;
let cachedData = null;

export default {
  async fetch(request) {
    counter++;  // 在同一个 Isolate 内累加
    if (!cachedData) {
      cachedData = await fetchExpensiveData();
    }
    return Response.json({ counter, data: cachedData });
  }
}
```

**问题**：
1. `counter` 会在同一个 Isolate 的多次请求间累加，但 Isolate 何时被回收不可预测
2. `cachedData` 可能过期但永远不刷新（在 Isolate 生命周期内）
3. 不同边缘节点有不同的 Isolate 实例，状态不同步

## 常见陷阱模式

### 陷阱 1：模块级缓存

```javascript
// ❌ 这不是可靠的缓存
let dbConnection = null;

export default {
  async fetch(request, env) {
    if (!dbConnection) {
      dbConnection = await connectDB(env.DB_URL);
    }
    // dbConnection 可能已经断开了，但代码不知道
    return dbConnection.query("SELECT ...");
  }
}
```

**修复**：每次请求重新获取连接，或使用带 TTL 的缓存逻辑：

```javascript
export default {
  async fetch(request, env) {
    // 每次请求独立获取，让 D1/Hyperdrive 管连接池
    const result = await env.DB.prepare("SELECT ...").all();
    return Response.json(result);
  }
}
```

### 陷阱 2：请求间共享的 Auth Token

```javascript
// ❌ Token 过期后不刷新
let authToken = null;
let tokenExpiry = 0;

export default {
  async fetch(request, env) {
    if (!authToken || Date.now() > tokenExpiry) {
      const resp = await fetch(env.AUTH_URL, { method: 'POST', ... });
      const data = await resp.json();
      authToken = data.token;
      tokenExpiry = Date.now() + data.expires_in * 1000;
    }
    // 如果 Isolate 的时钟偏移，或者 token 被服务端撤销...
  }
}
```

**修复**：加错误处理和 fallback：

```javascript
async function getToken(env) {
  // 每次检查 token 有效性，失败则重新获取
  if (authToken && Date.now() < tokenExpiry - 60000) {
    return authToken;
  }
  const resp = await fetch(env.AUTH_URL, { method: 'POST', ... });
  if (!resp.ok) throw new Error(`Auth failed: ${resp.status}`);
  const data = await resp.json();
  authToken = data.token;
  tokenExpiry = Date.now() + data.expires_in * 1000;
  return authToken;
}
```

### 陷阱 3：依赖全局 `Date.now()` 的定时逻辑

Workers Isolate 的存活时间不可预测（可能几秒，可能几小时）。不要用模块级的 `setInterval` 或时间戳做"每 N 分钟刷新"的逻辑。

## 正确的状态管理方案

| 需求 | 方案 |
|------|------|
| 短期缓存（<1min） | Cache API (`caches.default`) |
| KV 存储（最终一致） | Workers KV |
| 强一致状态 | Durable Objects |
| 数据库 | D1 或 Hyperdrive |
| 跨请求计数 | Analytics Engine 或 Durable Objects |

## 调试技巧

```javascript
// 加日志看 Isolate 生命周期
let isolateId = crypto.randomUUID();
let requestCount = 0;

export default {
  async fetch(request) {
    requestCount++;
    console.log(`Isolate ${isolateId} | Request #${requestCount}`);
    // 如果你看到 requestCount 一直累加，说明是同一个 Isolate
    // 如果突然重置为 1，说明 Isolate 被回收重建了
  }
}
```

```bash
# 实时查看 Worker 日志
wrangler tail --format pretty
```

## 关键教训

1. **Worker 不是服务器**：没有持久进程，没有可靠的内存状态
2. **模块级变量是 Isolate 级别的**：可以用于同一 Isolate 内的"热缓存"，但必须有 fallback
3. **本地测试会骗你**：`wrangler dev` 是单个长期运行的进程，和生产环境的 Isolate 池完全不同
4. **多节点不共享状态**：东京和纽约的边缘节点各有各的 Isolate，全局状态用 KV/DO/D1

---

*踩坑等级：⭐⭐⭐⭐⭐ | 出现频率：高 | 影响范围：Worker 行为不可预测*
