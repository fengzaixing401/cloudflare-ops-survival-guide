# 403 白页与 ICP 备案拦截的混合内容 (Mixed Content) 原理

> 国内用户访问你的 CF 站点偶尔遇到 403 白页，但国外用户正常。这不是 CF 的锅，是备案和混合内容的锅。

## 症状

- 国内部分地区用户访问返回 403 空白页面
- 不是 CF 的标准错误页面（没有 Ray ID），是运营商拦截页
- 时有时无，不同运营商表现不同
- 挂 VPN 后正常

## 根因分析

### 场景 1：ICP 备案缺失

中国大陆的 HTTP/HTTPS 流量经过运营商时，如果域名没有 ICP 备案：
- 部分运营商直接拦截返回 403
- 部分运营商插入跳转到备案提示页
- 部分运营商不管（所以表现不一致）

**判断方法：** 如果你的域名没有备案，且使用了中国大陆的 CDN 节点或直连大陆服务器，基本就是这个原因。

### 场景 2：Mixed Content 混合内容

**更隐蔽的坑：** 你的主页是 HTTPS，但页面里引用了 HTTP 资源（图片、脚本、样式），浏览器可能：
- 静默阻止加载（Chrome/Firefox）
- 显示不安全警告
- 部分运营商的 DPI 检测到 HTTP 明文流量，触发拦截

### 场景 3：CF 大陆节点 + 无备案

Cloudflare 免费计划不提供中国大陆节点，流量走海外。但如果你用了 CF 的中国网络合作（Enterprise），或者域名被错误地路由到了大陆节点，无备案就会被拦。

## 排查 SOP

### Step 1：确认是否运营商拦截

```bash
# 从大陆服务器测试
curl -4 -skI https://example.com/
# 看返回的 Server 头：
# - 如果是 "cloudflare" → CF 返回的，继续排查
# - 如果是空的或者奇怪的 → 运营商劫持

# 对比海外
curl -4 -skI --resolve example.com:443:104.x.x.x https://example.com/
```

### Step 2：检查 ICP 状态

```bash
# ICP 备案查询
# https://beian.miit.gov.cn/
# 或者用第三方 API 查
```

### Step 3：扫描 Mixed Content

```bash
# 用 curl 拉取页面，grep HTTP 资源
curl -sL https://example.com/ | grep -oE 'http://[^"'"'"' >]+'
# 如果有任何 http:// 的资源引用，就是 mixed content
```

## 解决方案

| 问题 | 解决 |
|------|------|
| 无 ICP 备案 | 确保 CF 走海外节点（免费计划默认就是），不要用大陆 CDN |
| Mixed Content | 全站 HTTPS，所有资源改 `//` 或 `https://`，配置 CSP upgrade-insecure-requests |
| 运营商劫持 | 启用 HSTS，使用 `<meta http-equiv="Content-Security-Policy">` |

### CSP Header 配置

在 CF Pages 的 `_headers` 文件或 Workers 中添加：

```text
/*
  Content-Security-Policy: upgrade-insecure-requests
  Strict-Transport-Security: max-age=31536000; includeSubDomains
```

## 注意

- 这个问题**无法通过 CF 侧配置完全解决**，因为拦截发生在用户到 CF 边缘之间的运营商链路上
- 最可靠的方案是：要么备案，要么确保所有大陆用户走海外节点（接受延迟代价）
- 不要因为"部分用户正常"就忽略这个问题——不同运营商策略不同

---

*踩坑等级：⭐⭐⭐ | 出现频率：中（仅影响国内用户）| 影响范围：国内部分地区完全无法访问*
