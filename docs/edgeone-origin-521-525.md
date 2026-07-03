# EdgeOne / 非 CF CDN 的 521/525 回源变体

> 当你把域名从 Cloudflare 切到腾讯 EdgeOne（或其他 CDN），源站明明活着，却收到 521/525 报错。这里记录常见的变体坑点和排查 SOP。

## 背景：为什么 EdgeOne 的 525 和 CF 的 525 不一样？

Cloudflare 的 525 (`SSL handshake failed`) 通常意味着源站 TLS 配置问题。但 EdgeOne 的回源链路与 CF 不同：

- EdgeOne 有自己的回源证书和 SNI 行为
- EdgeOne 支持 WebSocket 但需要显式启用
- 切换期间，DNS 的 CNAME flattening 可能导致诡异的路由循环

## 常见变体

### 变体 1：CNAME Flattening 导致 EdgeOne 无法验证域名

**症状：** EdgeOne Dashboard 一直提示"请添加 CNAME 记录"，但你明明在 Cloudflare 加了。

**根因：** Cloudflare 对 apex 域名自动执行 CNAME flattening，公共解析器查不到原始 CNAME，EdgeOne 验证脚本检测失败。

**排查：**
```bash
# 检查公网能否看到 CNAME
dig +short @1.1.1.1 example.com CNAME    # 可能为空（被 flatten 了）
dig +short @8.8.8.8 example.com A         # 返回 EdgeOne 边缘 IP

# 确认 CF 侧确实配了
curl -s "$CF_API/zones/$ZONE_ID/dns_records?name=example.com" \
  -H "Authorization: Bearer $CF_TOKEN" | jq '.result[] | {type,content}'
```

**解决：**
1. 如果 EdgeOne 支持文件验证，走 `/.well-known/teo-verification/` 路径
2. 如果 Dashboard 硬卡，临时将 apex CNAME 改为 `DNS-only`（关代理），验证通过后再开

### 变体 2：子域名切换时源站端口不匹配

**症状：** 子域名 CNAME 从 CF 切到 EdgeOne 后返回 521/522。

**根因：** EdgeOne 默认回源端口可能是 443，但你的源站 Nginx 监听的是 8043 或其他非标端口。

**排查：**
```bash
# 直连源站测试
curl -4 -skS --resolve sub.example.com:443:<ORIGIN_IP> \
  -w 'HTTP=%{http_code}\n' https://sub.example.com/
# 如果超时，换端口试：
curl -4 -skS --resolve sub.example.com:8043:<ORIGIN_IP> \
  -w 'HTTP=%{http_code}\n' https://sub.example.com:8043/
```

**解决：** 在 EdgeOne 回源配置里指定正确的回源端口和协议。

### 变体 3：WebSocket Upgrade 被拒

**症状：** 网站页面能加载，但 WebSocket 连接报 521。

**根因：** EdgeOne 需要在域名配置中显式开启 WebSocket 支持。

**排查：**
```bash
# 测试 WebSocket upgrade
curl -sI -H "Upgrade: websocket" -H "Connection: Upgrade" \
  https://sub.example.com/ws
# 预期: 101 Switching Protocols
# 实际: 521 或 connection refused
```

**解决：** EdgeOne 控制台 → 域名管理 → 找到对应域名 → 开启 WebSocket。

## 通用排查 SOP

```bash
# 1. 确认 DNS 解析路径
dig +short @1.1.1.1 $DOMAIN A
dig +short @8.8.8.8 $DOMAIN A

# 2. 直连源站验证 TLS
curl -4 -skS --resolve $DOMAIN:443:$ORIGIN_IP \
  -w '\nHTTP=%{http_code} SSL=%{ssl_verify_result}\n' \
  https://$DOMAIN/

# 3. 检查源站 Nginx 是否在监听
ssh root@$ORIGIN_IP "ss -tlnp | grep -E ':443|:8043'"

# 4. 查看源站 Nginx 错误日志
ssh root@$ORIGIN_IP "tail -20 /var/log/nginx/error.log"
```

## 关键教训

| 坑点 | 说明 |
|------|------|
| CNAME Flattening | Apex 域名的 CNAME 会被 CF 拍平，第三方 CDN 可能验证失败 |
| 回源端口 | 不要假设 CDN 回源一定走 443，检查源站实际监听端口 |
| WebSocket | EdgeOne 默认不开 WS，需要手动启用 |
| 切换顺序 | 先配好 EdgeOne 回源，再切 DNS，不要反过来 |

---

*踩坑等级：⭐⭐⭐ | 出现频率：中高 | 影响范围：整站不可用*
