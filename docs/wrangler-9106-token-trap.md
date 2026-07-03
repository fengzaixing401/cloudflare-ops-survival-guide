# Wrangler 部署 9106/6111 假 Token 死循环陷阱

## 症状
在 CI/CD、自动化脚本或通过 AI Agent 执行 `wrangler pages deploy` 或 `wrangler secret put` 时，终端反复报错：
- `Authentication failed (status: 400) [code: 9106]`
- `Invalid request headers [code: 6003]`
- `Invalid format for Authorization header [code: 6111]`

## 致命陷阱：你被“打码”骗了

当你看到 `9106` 或 `6111` 错误时，**第一反应绝对不应该是“我的 Token 过期了”或“我的权限不够”**。

**99% 的情况是：你在命令行里直接输入了被打码的假 Token。**

很多官方文档、博客教程或 AI 生成的代码示例中，为了安全，会这样写：
```bash
export CLOUDFLARE_API_TOKEN="***"
# 或者
export CLOUDFLARE_API_TOKEN="cfut_T...f5"
```
如果你（或者你的自动化脚本/Agent）**原样复制**了这行代码并在终端执行，Wrangler 就会把字面量 `***` 或 `cfut_T...f5` 作为真实的 Token 发送给 Cloudflare API。

Cloudflare API 收到 `***`，当然会返回格式错误（6111）或认证失败（9106）。

### 防死循环强制规则

如果你在自动化脚本或 AI Agent 中遇到了这个错误，**绝对禁止原样重试**。

1. **立即检查命令字符串**：打印出你实际执行的命令，看看 `CLOUDFLARE_API_TOKEN` 的值是不是 `***`、`[REDACTED]` 或包含省略号。
2. **填入真实值**：必须使用真实的、未打码的完整字符串。
3. **不要无脑轮询**：如果在自动化任务中遇到 401/403/9106，立即停止执行。重试一百次 `***` 也不会变成真 Token，只会触发 Cloudflare 的防刷机制。

## 另一个陷阱：缺少 Account ID

即使你提供了真实的 `CLOUDFLARE_API_TOKEN`，在非交互式环境（如 CI/CD）下，某些 Wrangler 命令（特别是 `pages deploy` 和 `secret put`）仍然会报错。

**原因**：Wrangler 无法推断你要操作哪个账号下的资源。

**修复**：必须同时提供 `CLOUDFLARE_ACCOUNT_ID`。

```bash
# 错误示范（在 CI 环境中会失败）
CLOUDFLARE_API_TOKEN="真实Token" npx wrangler pages deploy dist/

# 正确示范
CLOUDFLARE_API_TOKEN="真实Token" CLOUDFLARE_ACCOUNT_ID="真实AccountID" npx wrangler pages deploy dist/ --project-name my-project
```

## 总结
看到 9106 / 6111 -> 检查 Token 是否是字面量 `***` -> 填入真值 -> 确保同时传入 Account ID。
