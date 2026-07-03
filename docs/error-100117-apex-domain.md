# Error 100117: 根域名 (Apex Domain) 绑定冲突

> 当你尝试将根域名（如 `example.com`）绑定到 Cloudflare Pages 或某些服务时，收到 `Error 100117` 或类似的"域名已被使用"提示。

## 症状

- Cloudflare Pages 添加自定义域名时报错："Domain already in use" 或 Error 100117
- 域名明明在同一个 Cloudflare 账号下，但就是绑不上
- 子域名能绑，根域名不行

## 根因

Cloudflare 对根域名有特殊限制：

1. **Zone apex 只能有一个 CNAME**：如果 apex 已经有 A 记录指向某个 IP，再添加 CNAME 到 Pages 会冲突
2. **Pages 项目名称混淆**：如果之前有个旧 Pages 项目绑定了这个域名（哪怕项目已删除），绑定记录可能残留
3. **CNAME Flattening 副作用**：CF 会自动把 apex CNAME 拍平成 A 记录对外展示，但内部仍然是 CNAME

## 排查步骤

### Step 1：检查当前 apex DNS 记录

```bash
# 查看 CF API 中的真实记录（不是 dig 的结果）
curl -s "$CF_API/zones/$ZONE_ID/dns_records?name=example.com" \
  -H "Authorization: Bearer $CF_TOKEN" | jq '.result[] | {id,type,name,content,proxied}'
```

### Step 2：检查是否有残留 Pages 绑定

```bash
# 列出所有 Pages 项目
CLOUDFLARE_API_TOKEN="$CF_TOKEN" CLOUDFLARE_ACCOUNT_ID="$ACCOUNT_ID" \
  npx wrangler pages project list

# 检查每个项目的自定义域名
curl -s "$CF_API/accounts/$ACCOUNT_ID/pages/projects/<project-name>/domains" \
  -H "Authorization: Bearer $CF_TOKEN" | jq .
```

### Step 3：清理冲突

```bash
# 删除冲突的 A 记录（如果要改用 CNAME 指向 Pages）
curl -X DELETE "$CF_API/zones/$ZONE_ID/dns_records/$RECORD_ID" \
  -H "Authorization: Bearer $CF_TOKEN"

# 添加 apex CNAME 到 Pages
curl -X POST "$CF_API/zones/$ZONE_ID/dns_records" \
  -H "Authorization: Bearer $CF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"CNAME","name":"@","content":"<project>.pages.dev","proxied":true}'
```

## 修复方案

| 场景 | 操作 |
|------|------|
| Apex 有 A 记录，想改绑 Pages | 删 A 记录，加 CNAME → `<project>.pages.dev` |
| 旧项目残留绑定 | 在旧项目设置里删除自定义域名，再在新项目绑定 |
| 跨账户冲突 | 需要在另一个账户解绑，或联系 CF 支持 |

## 注意事项

- 改 apex 记录前，**务必备份现有的 MX、TXT（SPF/DKIM/DMARC）记录**！
- 删 A 记录后，apex 邮件接收不受影响（MX 记录独立），但如果你同时有 `@` 的 A 记录用于收邮件验证，要小心
- CNAME 和 MX 可以共存在 Cloudflare 中（CF 的特殊处理），但标准 DNS 规范不允许

---

*踩坑等级：⭐⭐⭐ | 出现频率：中 | 影响范围：根域名无法绑定服务*
