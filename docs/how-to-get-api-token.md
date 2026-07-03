# 如何正确获取 Cloudflare API Token 与 Account ID

在自动化部署（如 Wrangler、GitHub Actions）或通过 API 管理 Cloudflare 资源时，身份认证是最容易踩坑的第一步。

## 1. 致命误区：Global API Key vs API Token

Cloudflare 提供两种凭证，**千万不要用错**：

- ❌ **Global API Key (全局 API 密钥)**：拥有你账号的**所有**权限。一旦泄露，攻击者可以删除你的域名、转移资产。**强烈建议永远不要在脚本或 CI/CD 中使用它**（除非极少数极老旧的 API 强制要求）。
- ✅ **API Token (API 令牌)**：可以精确控制权限（比如“只能修改 A 域名的 DNS”或“只能部署 Pages”）。**这是自动化部署的唯一正确选择**。

## 2. 获取 API Token 的标准流程

1. 登录 Cloudflare 控制台。
2. 点击右上角头像 -> **My Profile (我的个人资料)**。
3. 在左侧菜单选择 **API Tokens (API 令牌)**。
4. 点击 **Create Token (创建令牌)**。
5. 你可以选择模板（如 *Edit zone DNS*），或者点击底部的 **Create Custom Token (创建自定义令牌)**。

### 常用权限配置清单 (Permissions)

如果你要进行以下操作，请在自定义 Token 中赋予对应的权限：

- **部署 Pages 项目**：
  - `Account` | `Cloudflare Pages` | `Edit`
- **部署 Workers 脚本**：
  - `Account` | `Workers Scripts` | `Edit`
  - `Account` | `Workers Tail` | `Read`
- **修改 DNS 记录**：
  - `Zone` | `DNS` | `Edit`
- **修改 SSL/TLS 设置**：
  - `Zone` | `SSL and Certificates` | `Edit`

**Account Resources (账户资源)** 和 **Zone Resources (区域资源)**：
强烈建议在这里选择 `Include` -> `Specific Account` 或 `Specific Zone`，将 Token 的作用域限制在特定的域名或账户下，防止误操作其他项目。

6. 点击 **Continue to summary** -> **Create Token**。
7. ⚠️ **保存 Token**：Token 的明文**只会显示这一次**！请立即复制并保存到密码管理器或 GitHub Secrets 中。如果丢失，只能重新生成（Roll）或新建。

## 3. 如何获取 Account ID (账户 ID) 和 Zone ID (区域 ID)

很多 Wrangler 命令（如 `pages deploy`）在非交互式环境下，除了需要 Token，还**强制需要 Account ID**。

**获取方法**：
1. 登录 Cloudflare 控制台。
2. 在主页点击你的任意一个**域名 (Zone)** 进入概览页面 (Overview)。
3. 向下滚动，看页面的**右侧边栏 (Right Sidebar)**。
4. 在 **API** 区域，你会看到：
   - **Zone ID**：用于操作该域名特定资源的 API（如 DNS、Origin Rules）。
   - **Account ID**：用于操作账户级资源的 API（如 Pages、Workers、D1 数据库）。

## 4. 环境变量注入规范

在 CI/CD（如 GitHub Actions）或本地终端中，请严格使用以下环境变量名，Wrangler 会自动读取它们：

```bash
export CLOUDFLARE_API_TOKEN="你的真实Token"
export CLOUDFLARE_ACCOUNT_ID="你的真实AccountID"

# 然后直接执行命令，不需要在命令里再写一遍
npx wrangler pages deploy dist/ --project-name my-project
```

**避坑**：永远不要在代码库里硬编码 Token，也不要在终端里输入 `CLOUDFLARE_API_TOKEN="***"` 这种打码的假字符串去测试，这会触发 `9106` 错误死循环（详见 [Wrangler 9106 陷阱](wrangler-9106-token-trap.md)）。
