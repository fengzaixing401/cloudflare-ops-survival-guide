# 源站 8043 端口与 Origin Rules 冲突陷阱

> CF Origin Rules 可以修改回源端口，但如果配错了，你会得到一个诡异的 522 或 521——源站明明活着，CF 就是连不上。

## 症状

- 域名在 CF 开了代理，网站返回 522 / 521
- 直连源站 IP + 正确端口完全正常
- Nginx 日志里没有任何请求记录（说明请求根本没到 Nginx）

## 根因

Cloudflare Origin Rules 可以 override 回源端口：

```text
CF 边缘 → 回源到 ORIGIN_IP:PORT
```

如果你设了 Origin Rule 把端口改成 8043，但：
1. 源站 Nginx 只监听 443（没监听 8043）
2. 防火墙只开了 443 没开 8043
3. 或者 Origin Rule 的匹配条件写错了，把不该改的域名也改了

## 诊断流程

### Step 1：确认 Origin Rules 配置

```bash
# CF Dashboard → 选择域名 → Rules → Origin Rules
# 或通过 API：
curl -s "$CF_API/zones/$ZONE_ID/rulesets" \
  -H "Authorization: Bearer $CF_TOKEN" | jq '.result[] | select(.phase == "http_request_origin")'
```

重点看：
- `action_parameters.origin.port` — 端口 override
- `expression` — 匹配条件是什么

### Step 2：验证源站端口

```bash
# 检查源站实际监听什么端口
ssh root@$ORIGIN_IP "ss -tlnp | grep -E 'nginx|node'"

# 从外部测试 TCP 连通
nc -zv $ORIGIN_IP 443 -w 3
nc -zv $ORIGIN_IP 8043 -w 3
```

### Step 3：直连源站测试

```bash
# 用 CF 实际回源的端口测试
curl -4 -skS --resolve example.com:8043:$ORIGIN_IP \
  -w 'HTTP=%{http_code}\n' https://example.com:8043/

# 对比默认 443
curl -4 -skS --resolve example.com:443:$ORIGIN_IP \
  -w 'HTTP=%{http_code}\n' https://example.com/
```

## 常见场景

### 场景 1：Nginx 443 迁移后遗症

之前源站监听 8043（因为前面有其他代理），后来改成 443 了，但 CF Origin Rule 还指着 8043：

```text
Origin Rule: destination port = 8043   ← 过时了！
Nginx: listen 443 ssl;                 ← 现在监听 443
结果: CF → ORIGIN:8043 → 无人应答 → 522
```

**修复**：删除或更新 Origin Rule 的端口 override。

### 场景 2：多域名共用 Origin Rule

一条 Origin Rule 的 expression 用了 `http.host contains "example"`，结果把 `sub1.example.com` 和 `sub2.example.com` 都匹配了，但只有 sub1 需要改端口：

**修复**：缩小 expression 的匹配范围，用精确的 `http.host eq "sub1.example.com"`。

### 场景 3：Origin Rule 的 append 行为

**注意**：CF Origin Rules 的 API 是用 PUT 替换整个 ruleset，不是 append！如果你用 API 添加规则时覆盖了已有规则，之前的规则就没了。

```bash
# 正确做法：先 GET 现有规则，修改后整体 PUT 回去
RULESET_ID=$(curl -s "$CF_API/zones/$ZONE_ID/rulesets" \
  -H "Authorization: Bearer $CF_TOKEN" | jq -r '.result[] | select(.phase=="http_request_origin") | .id')

# GET 现有规则
curl -s "$CF_API/zones/$ZONE_ID/rulesets/$RULESET_ID" \
  -H "Authorization: Bearer $CF_TOKEN" | jq '.result.rules'

# 修改后 PUT 回去（包含所有规则）
```

## 预防清单

- [ ] 改 Nginx 监听端口时，同步检查 CF Origin Rules
- [ ] Origin Rule expression 用精确匹配，不用模糊匹配
- [ ] 改规则后立刻 `curl -4 -skI https://domain/` 验证
- [ ] 用 API 操作 Origin Rules 时，先 GET 再改再 PUT，不要盲 PUT

---

*踩坑等级：⭐⭐⭐⭐⭐ | 出现频率：中 | 影响范围：整站 522 不可用*
