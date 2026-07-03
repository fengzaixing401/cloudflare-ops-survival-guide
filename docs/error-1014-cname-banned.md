# Error 1014: CNAME Cross-User Banned 跨账户绑定拦截

> 自定义域名指向 Pages 项目后，访问返回 `Error 1014`。DNS 看起来没问题，但 Cloudflare 就是不认。

## 症状

```text
HTTP/2 403
cf-error-code: 1014
```

浏览器显示 Cloudflare 错误页面：**Error 1014 - CNAME Cross-User Banned**

## 根因

Cloudflare Pages 自定义域名的 CNAME 目标必须精确匹配**拥有该自定义域名的 Pages 项目**。

常见翻车场景：

1. **项目改名/重建**：之前用 `old-project.pages.dev`，重建后变成 `new-project.pages.dev`，但 DNS CNAME 还指着旧的
2. **多项目混淆**：有 `my-site` 和 `my-site-git`（Git 集成自动创建的），CNAME 指错了
3. **跨账户**：域名在 A 账户，Pages 项目在 B 账户

## 排查 SOP

### Step 1：确认公网报错

```bash
curl -4 -skI https://custom.example.com/
# 找 cf-error-code: 1014
```

### Step 2：查看 DNS CNAME 目标（通过 API，不是 dig）

```bash
# dig 看不到真实 CNAME（被代理了），用 API 查
curl -s "$CF_API/zones/$ZONE_ID/dns_records?name=custom.example.com" \
  -H "Authorization: Bearer $CF_TOKEN" | jq '.result[] | {type,content,proxied}'
```

### Step 3：列出 Pages 项目，找到正确目标

```bash
CLOUDFLARE_API_TOKEN="$CF_TOKEN" CLOUDFLARE_ACCOUNT_ID="$ACCOUNT_ID" \
  npx wrangler pages project list
```

找到你的项目实际的 `*.pages.dev` 子域名。

### Step 4：检查 Pages 项目的自定义域名状态

```bash
curl -s "$CF_API/accounts/$ACCOUNT_ID/pages/projects/<correct-project>/domains" \
  -H "Authorization: Bearer $CF_TOKEN" | jq '.result[]'
```

如果 `status` 不是 `active`，或 `verification_data` 有错误，就是这里的问题。

### Step 5：修复

```bash
# 更新 CNAME 指向正确的 Pages 项目
curl -X PATCH "$CF_API/zones/$ZONE_ID/dns_records/$RECORD_ID" \
  -H "Authorization: Bearer $CF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":"correct-project.pages.dev"}'
```

如果自定义域名没有在正确项目中注册，先添加：
```bash
curl -X POST "$CF_API/accounts/$ACCOUNT_ID/pages/projects/correct-project/domains" \
  -H "Authorization: Bearer $CF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"custom.example.com"}'
```

## 易混淆点

| 情况 | 结果 |
|------|------|
| CNAME → 正确项目的 `*.pages.dev` | ✅ 正常 |
| CNAME → 错误项目/旧项目的 `*.pages.dev` | ❌ 1014 |
| CNAME → `pages.dev`（不带项目名） | ❌ 1014 |
| A 记录指向 Pages IP | ❌ 不支持，用 CNAME |

## 关键教训

- Cloudflare Pages 不靠 IP 路由，靠 **Host header + CNAME target** 匹配项目
- `wrangler pages deploy` 的直传部署和 Git 集成部署可能创建**不同名字**的项目
- 改项目名后，`*.pages.dev` 子域名也会变，DNS 必须同步更新

---

*踩坑等级：⭐⭐⭐⭐ | 出现频率：高 | 影响范围：自定义域名完全不可用*
