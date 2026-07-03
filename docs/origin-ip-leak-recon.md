# Cloudflare 真实源 IP 泄露的常见途径与防御

当你把网站接入 Cloudflare（开启橙色云）后，你以为源站 IP 就彻底隐藏了。但实际上，攻击者有无数种方法可以绕过 CF 直接找到你的真实 IP，从而发起直接的 DDoS 攻击。

## 常见泄露途径与排查方法

### 1. 证书透明度日志 (Certificate Transparency Logs)
这是最致命、最常见的泄露途径。
如果你在源站服务器上申请了 Let's Encrypt 证书，证书的 SAN (Subject Alternative Name) 列表中包含了你的域名。攻击者可以通过查询 CT 日志，找到所有包含你域名的证书，并扫描这些证书关联的 IP。

**排查命令**：
```bash
# 使用 CertSpotter API 查询
curl -s "https://api.certspotter.com/v1/issuances?domain=example.com&include_subdomains=true&expand=dns_names"
```

### 2. 未代理的子域名 (Unproxied Subdomains)
很多管理员只把主域名 `www` 和 `@` 开启了 CF 代理，却遗漏了其他子域名。

**高危子域名**：
- `direct.example.com` / `origin.example.com`
- `mail.example.com` (MX 记录通常不能走 CF 代理)
- `ftp.example.com` / `ssh.example.com`
- `dev.example.com` / `test.example.com`

**特别注意双重子域名模式**：
形如 `username.username.example.com`。这种自动化面板生成的子域名经常被遗漏。

### 3. SPF 记录泄露 (高价值)
SPF (Sender Policy Framework) 记录用于防止邮件伪造，它必须声明哪些 IP 有权代表该域名发送邮件。

**排查命令**：
```bash
dig +short example.com TXT
# 如果输出类似 "v=spf1 ip4:192.140.166.89 -all"，那么 192.140.166.89 极有可能就是源站 IP。
```
*注意：SPF 记录通常在域名注册时配置，不会随 Cloudflare 代理而隐藏。*

### 4. 历史 DNS 记录
在接入 Cloudflare 之前，你的域名可能已经解析到了源站 IP。这些历史记录会被 SecurityTrails、Hackertarget 等安全大数据平台永久记录。

### 5. GitHub 源码泄露
如果你的项目开源或不慎泄露了源码，攻击者可以在代码中寻找硬编码的 IP、API 端点或数据库连接字符串。

**排查重点**：
- `.env.example` 或 `.env` 文件
- `config.js` / `database.js`
- 包含 `process.env.SECRET || "fallback"` 的认证逻辑

## 终极防御方案：Authenticated Origin Pulls

即使攻击者找到了你的真实 IP，只要你的源站配置得当，他们也无法直接访问。

**最佳实践**：配置 Cloudflare Authenticated Origin Pulls (认证回源)。
1. 在 Cloudflare 控制台开启该功能。
2. 在源站 Nginx 中配置，**强制要求**所有入站请求必须携带 Cloudflare 的客户端证书。
3. 任何不携带 CF 证书的直接 IP 访问（如攻击者的扫描器）都会被 Nginx 直接拒绝（403 或 444）。

这样，即使源 IP 泄露，源站也只接受来自 Cloudflare 边缘节点的流量。
