# Pages 部署 SPA (React/Vite) 出现白屏的根因与修复

> 你把 React/Vue/Vite 项目部署到 CF Pages，首页加载是一片白屏，标题正常但内容为空。控制台报资源 404。

## 症状

- 部署成功，CF Pages Dashboard 显示 "Success"
- 访问 `https://project.pages.dev/`：标题正常，页面白屏
- 浏览器控制台：`GET /assets/index-xxxxx.js 404 (Not Found)`
- 刷新非首页路由（如 `/about`）返回 404

## 根因分析

### 原因 1：Build Output 目录配错

Pages 的 `Build output directory` 设置决定了哪个目录的文件被上传。

```text
❌ 错误: Build output = "."（把整个仓库都传了）
❌ 错误: Build output = "build"（但实际产物在 "dist"）
✅ 正确: Build output = "dist"（Vite 默认）
✅ 正确: Build output = "build"（CRA 默认）
```

### 原因 2：纯静态仓库配了 Build Command

如果你的仓库里已经有构建好的静态文件（比如直接推 `dist/` 目录），但 Pages 配了 `Build command: npm run build`：

- CF 会尝试运行 `npm run build`
- 如果没有 `package.json` 或构建失败，输出目录为空
- 上一次成功部署的文件仍在线上（所以 `curl` 返回 200），但新内容没上去

### 原因 3：SPA 路由未配置

SPA 框架使用客户端路由（如 React Router），直接访问 `/about` 时：
- 服务器没有 `/about.html` 这个文件
- 需要一个 fallback 规则把所有请求指向 `index.html`

## 修复方案

### 修复 Build Output

```text
CF Dashboard → Pages → 你的项目 → Settings → Build & deployments
  Build command: npm run build（或留空如果是纯静态）
  Build output directory: dist（根据框架调整）
```

### 修复 SPA 路由

在项目根目录添加 `public/_redirects`（或 `dist/_redirects`）：

```text
/*  /index.html  200
```

或者用 `_headers` + `_redirects` 的组合：

```text
# _redirects
/*    /index.html   200
```

### 修复 Git 部署配置不匹配

如果是纯静态仓库（没有 build 步骤）：

```text
Build command: （留空）
Build output directory: .（或具体的静态文件目录）
Root directory: （留空，除非静态文件在子目录）
```

## 验证步骤

```bash
# 1. 触发新部署（改一下然后 push，或手动重新部署）
git commit --allow-empty -m "trigger redeploy" && git push

# 2. 等部署完成，检查新部署状态
# CF Dashboard → Pages → Deployments → 最新的是否 Success

# 3. 验证资源加载
curl -sI "https://project.pages.dev/assets/index.js" | head -5
# 预期: HTTP/2 200

# 4. 验证 SPA 路由 fallback
curl -s "https://project.pages.dev/about" | grep -o '<title>.*</title>'
# 预期: 返回 index.html 的 title
```

## Wrangler 部署 vs Git 部署的差异

⚠️ **重要陷阱**：`wrangler pages deploy` 是直接上传本地文件，绕过 Git 构建流程！

```bash
# 这个能成功，不代表 Git 部署也能成功
wrangler pages deploy ./dist --project-name my-project
```

如果用户反馈"手动部署正常但 push 后还是白屏"，问题出在 **Git 构建配置**，不是文件本身。

## 常见框架的正确配置

| 框架 | Build Command | Output Directory |
|------|--------------|-----------------|
| Vite (React/Vue) | `npm run build` | `dist` |
| Create React App | `npm run build` | `build` |
| Next.js (static) | `npx next build && npx next export` | `out` |
| Nuxt 3 (static) | `npx nuxt generate` | `.output/public` |
| Astro | `npm run build` | `dist` |
| Hugo | `hugo` | `public` |

---

*踩坑等级：⭐⭐⭐ | 出现频率：高 | 影响范围：整站白屏*
