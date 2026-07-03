# 522 Connection Timed Out 的真实含义与误区

> 522 是 Cloudflare 最容易被误解的错误码之一。它不是"服务器挂了"那么简单。

## 症状

```text
HTTP/2 522
Error: Connection timed out
```

Cloudflare 错误页面显示：源站未在规定时间内响应 TCP 连接请求。

## 常见误区

| 误区 | 真相 |
|------|------|
| "服务器宕机了" | 不一定！可能只是防火墙/安全组没放行 CF IP |
| "Nginx 挂了" | 可能 Nginx 在监听 127.0.0.1 而不是 0.0.0.0 |
| "网络断了" | 可能只是 iptables 规则把 CF 的 IP 段给 DROP 了 |
| "需要重启服务器" | 大概率只需要改一行防火墙规则 |
| "CF 的问题" | 99% 是源站侧的问题 |

## 真实含义

522 表示 **Cloudflare 边缘节点与你的源站 IP 之间的 TCP 三次握手超时**（默认 15 秒）。

这意味着：
1. Cloudflare 确实在尝试连接你的源站
2. 源站 IP 是对的（否则会是其他错误）
3. 但 TCP SYN 包到不了目标端口，或者回不来

## 排查 SOP

### Step 1：确认源站 IP 和端口

```bash
# 从 CF API 获取 DNS 记录
curl -s "$CF_API/zones/$ZONE_ID/dns_records?name=example.com" \
  -H "Authorization: Bearer $CF_TOKEN" | jq '.result[] | {type,content}'
```

### Step 2：直连源站测试 TCP

```bash
# 从外部测试 TCP 连通性（不经过 CF）
nc -zv $ORIGIN_IP 443 -w 5
# 或者
curl -4 -sk --connect-timeout 5 --resolve example.com:443:$ORIGIN_IP https://example.com/
```

### Step 3：检查源站防火墙

```bash
# 查看 iptables
iptables -L INPUT -n | grep -E "443|DROP|REJECT"

# 查看 UFW（如果用的话）
ufw status verbose

# 查看云服务商安全组（阿里云/AWS/腾讯云控制台）
```

### Step 4：确认 Nginx 监听地址

```bash
# 检查 Nginx 是否在监听所有接口
ss -tlnp | grep ':443'
# 预期：0.0.0.0:443 或 :::443
# 问题：127.0.0.1:443（只监听本地！）
```

### Step 5：放行 Cloudflare IP

```bash
# CF 的 IPv4 范围
# https://www.cloudflare.com/ips-v4/
# 确保防火墙放行了这些 IP 段的 443 端口入站

# 示例 iptables 规则
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
  iptables -I INPUT -s $ip -p tcp --dport 443 -j ACCEPT
done
```

## DNS 切换导致的 522 特殊案例

当你把域名从直连切到 CF 代理时，一个常见的坑：

**之前**：DNS 直接解析到源站 IP，用户直连
**之后**：DNS 指向 CF 边缘，CF 回源

如果你的防火墙只允许"来自任意 IP 的 443"，没问题。但如果你用了 IP 白名单（比如只允许特定 IP 访问），切换后 CF 边缘 IP 不在白名单里，就会 522。

## Origin Rules 端口不匹配的 522

另一个隐蔽坑：你设了 Origin Rules 把回源端口改成 8043，但：
- Nginx 只监听 443
- 或者防火墙没放行 8043

```bash
# 确认 Origin Rules 的回源端口
# CF Dashboard → Rules → Origin Rules → 检查 destination port override
# 然后确认源站对应端口开放
ss -tlnp | grep ':8043'
```

## 总结

522 的排查优先级：
1. 🔥 **防火墙/安全组** → 是否放行了 CF IP 段
2. 🔥 **监听地址** → Nginx 是否 bind 到 0.0.0.0
3. ⚡ **端口匹配** → CF 回源端口 vs Nginx 监听端口
4. ⚡ **Origin Rules** → 有没有端口 override 导致不匹配
5. 💤 **服务器负载** → CPU/内存/连接数耗尽（少见）

---

*踩坑等级：⭐⭐⭐⭐ | 出现频率：高 | 影响范围：整站不可用*
