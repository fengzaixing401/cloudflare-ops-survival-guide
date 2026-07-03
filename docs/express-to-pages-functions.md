# Pages Functions：全栈应用 (Express/API) 迁移指南

> 你有一个 Express.js 后端跑在 VPS 上，现在想迁移到 Cloudflare Pages Functions（零服务器运维）。这里是坑点和正确姿势。

## 什么是 Pages Functions？

Pages Functions 是 Cloudflare Pages 内置的无服务器计算能力，基于 Workers 运行时。它让你可以在静态站点旁边放一些 API 路由，实现全栈部署。

## 架构对比

```text
之前（VPS + Express）:
  用户 → CF CDN → VPS:3000（Express 全栈）

之后（Pages Functions）:
  用户 → CF Pages（静态资源）
       → CF Pages Functions（/api/* 路由）→ D1/KV/外部API
```

## 迁移步骤

### 1. 项目结构改造

```text
my-project/
├── functions/           ← Pages Functions 目录
│   └── api/
│       ├── users.ts     ← GET/POST /api/users
│       └── auth/
│           └── login.ts ← POST /api/auth/login
├── src/                 ← 前端源码
├── dist/                ← 构建产物（publish 目录）
└── package.json
```

### 2. 路由映射

| Express 路由 | Pages Functions 文件 | 说明 |
|-------------|---------------------|------|
| `app.get('/api/users', ...)` | `functions/api/users.ts` → `onRequestGet` | 文件路径即路由 |
| `app.post('/api/users', ...)` | `functions/api/users.ts` → `onRequestPost` | 同文件多方法 |
| `app.get('/api/users/:id', ...)` | `functions/api/users/[id].ts` | 动态路由用 `[param]` |

### 3. Handler 写法

```typescript
// functions/api/users.ts
interface Env {
  DB: D1Database;
}

// GET /api/users
export const onRequestGet: PagesFunction<Env> = async (context) => {
  const { results } = await context.env.DB.prepare(
    "SELECT * FROM users LIMIT 50"
  ).all();
  return Response.json(results);
};

// POST /api/users
export const onRequestPost: PagesFunction<Env> = async (context) => {
  const body = await context.request.json();
  // 验证、插入...
  return Response.json({ ok: true }, { status: 201 });
};
```

### 4. 中间件迁移

Express 的 `app.use()` 中间件 → Pages Functions 的 `_middleware.ts`：

```typescript
// functions/_middleware.ts
export const onRequest: PagesFunction = async (context) => {
  // CORS
  if (context.request.method === "OPTIONS") {
    return new Response(null, {
      headers: {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE",
        "Access-Control-Allow-Headers": "Content-Type, Authorization",
      },
    });
  }

  // 继续执行下一个 handler
  const response = await context.next();

  // 添加 CORS 头
  response.headers.set("Access-Control-Allow-Origin", "*");
  return response;
};
```

## 常见坑点

### 坑 1：`node:*` 模块不可用

Workers 运行时不是 Node.js！以下不能用：
- `fs`, `path`, `crypto`（部分可用）, `child_process`
- 任何依赖这些的 npm 包（如 `bcrypt`）

替代方案：
- `bcrypt` → `bcryptjs`（纯 JS 实现）
- `fs` → 使用 KV/R2/D1
- `crypto` → Web Crypto API

### 坑 2：环境变量

Express: `process.env.DB_URL`
Pages Functions: 通过 `context.env.BINDING_NAME` 访问

在 `wrangler.toml` 或 Dashboard 中配置 bindings。

### 坑 3：请求体大小限制

Workers 免费计划：请求体 ≤ 100MB（实际建议 <10MB）。
大文件上传走 R2 presigned URL。

### 坑 4：执行时间限制

Workers 免费：10ms CPU time（注意是 CPU 时间，不是挂钟时间）
Workers 付费：30s CPU time

长时间任务（如发邮件、批量处理）考虑用 Queues 或 Durable Objects。

## Git 部署配置

```toml
# wrangler.toml
name = "my-project"
pages_build_output_dir = "./dist"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"
database_name = "my-db"
database_id = "xxx"
```

## 本地开发

```bash
# 带 bindings 的本地开发
npx wrangler pages dev dist --d1 DB=my-db --port 8788
```

---

*踩坑等级：⭐⭐⭐⭐ | 出现频率：中 | 影响范围：整个后端架构迁移*
