# Error 525 (SSL handshake failed) 终极排查指南

## 症状
- 浏览器访问 Cloudflare 代理的域名返回 525 错误。
- `curl -v https://domain` 显示 `CF-RAY` 头，且 HTTP 状态为 525。
- 腾讯 EdgeOne 等其他 CDN 可能表现为 521 或 525。

## 核心原理解析

**Error 525 的唯一定义**：Cloudflare 边缘节点与你的源站服务器之间的 SSL 握手失败。

Cloudflare 与源站之间的 SSL 连接模式决定了握手的严格程度：

| 模式 | CF 到源站加密 | 验证源站证书 |
|------|--------------|--------------|
| Flexible | HTTP（不加密） | 不适用 |
| Full | HTTPS | **不验证**有效性（允许自签名、过期、链不完整）|
| Full (strict) | HTTPS | **严格验证**（必须由受信任 CA 签发、链完整、未过期）|

**最常见死因**：源站证书链不完整（缺少中间证书），且 Cloudflare 处于 `Full (strict)` 模式。

## 诊断步骤（绝对详尽）

### 1. 确认是否真的走了 Cloudflare 代理
```bash
dig +short <domain>
```
若返回 `172.67.x.x` 或 `104.21.x.x`，说明已代理（橙色云）。

### 2. 测试源站直连（绕过 CF）
获取你的源站真实 IP，在本地终端执行：
```bash
openssl s_client -connect <源站IP>:443 -servername <domain> -verify_return_error 2>&1 | grep -E "Verify return code|subject|issuer"
```
- 若输出 `Verify return code: 0 (ok)` → 证书链完整，问题可能在 CF 侧配置。
- 若输出 `verify error:num=20:unable to get local issuer certificate` → **证书链不完整**。这就是 525 的元凶。

### 3. 查看 Nginx 当前使用的证书文件
```bash
grep -E "ssl_certificate|ssl_trusted_certificate" /etc/nginx/sites-enabled/<domain>
```
如果只有 `ssl_certificate` 而缺少 `ssl_trusted_certificate`（或证书文件本身没有打包中间证书），在 Strict 模式下必报 525。

## 解决方案

### 方案 A：降级为 `Full` 模式（最快，无需修改源站）
**适用场景**：源站证书链不完整，且你不关心源站证书的绝对有效性（如内部系统、测试环境、仅需 CF 的 CDN 功能）。

1. 登录 Cloudflare 控制台 → `SSL/TLS` → `Overview`
2. 将当前模式从 `Full (strict)` 改为 **`Full`**
3. 等待 5 秒，执行 `curl -I https://<domain>`，应返回 200。

*原理：`Full` 模式下 CF 不验证源站证书，只要源站能返回任意证书即可建立加密连接。*

### 方案 B：补全源站证书链（保持 `Full (strict)`，更安全）
**适用场景**：需要严格安全合规。

1. 获取中间证书（例如，如果你用的是 Cloudflare Origin 证书，需要下载 `origin-ca-bundle.crt`）。
2. 在 Nginx 配置中添加：
   ```nginx
   ssl_trusted_certificate /path/to/chain.crt;
   ```
3. 重载 Nginx：`sudo nginx -t && sudo systemctl reload nginx`
4. 再次使用 `openssl s_client` 验证，必须输出 `0 (ok)`。

### 方案 C：改用 Let's Encrypt 证书（推荐长期方案）
```bash
sudo certbot --nginx -d <domain> --non-interactive --agree-tos --email admin@example.com
```
Certbot 会自动配置完整的受信任证书链，无需手动折腾中间证书。

## 常见误区 & 陷阱

| 错误认知 | 事实 |
|---------|------|
| “证书链不完整没事，只要能加密就行” | 在 `Full (strict)` 模式下会直接阻断连接 → 525 |
| “修改 CF 模式会很复杂，生效很慢” | 只需一次下拉选择，通常 5 秒内生效 |
| “自签名证书永远不能用” | 在 `Full` 模式下完全可用 |
| “525 一定是 Cloudflare Full Strict 证书链问题” | 也可能是源站 Nginx 没有配置对应的 `443 ssl server_name`，或者实际后端只是 HTTP 服务。先测源站 `curl -skI -H 'Host: <domain>' https://127.0.0.1/`。 |
| “公网 521/525 就说明源站服务挂了” | 源站后端端口可能正常（如 8000 返回 200），但 CDN 到源站的协议或端口错了。必须同时验证后端端口、Nginx vhost、CDN 回源协议三层。 |
