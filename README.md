# powpow-workbuddy-connector

PowPop（泡泡）的 WorkBuddy 连接器包。提交 WorkBuddy 审核用，**不是部署产物**——MCP 服务端本身运行在 Supabase Edge Function（`durenzidu/powpow` 仓库的 `supabase/functions/mcp-server/`），本仓库只包含 WorkBuddy 平台要求的接入描述文件。

- MCP Server：`https://global.powpow.online/api/mcp`（streamableHttp + OAuth 2.1 + PKCE）
- 线上状态：`powpow-mcp` v0.14.5，9 个只读工具

## 目录结构

```
├── connector-meta.json   # 连接器元信息（名称/描述/示例/source/版本）
├── mcp.json              # MCP Server 连接配置
├── icon.svg              # 市场图标（64×64）
├── skills/powpow/
│   └── SKILL.md          # AI 使用指导（9 工具用法/错误恢复/坐标系约定）
└── README.md             # 本文件
```

## 维护纪律

1. **送审前必读** `docs`：https://open.workbuddy.cn/docs/connector（「提交前检查」12 条过一遍）
2. **SKILL.md 同步时机**：MCP server 发生**工具契约变更**（新增/删除/改参数/改返回语义）时，更新 SKILL.md + `connector-meta.json` 版本号 +1，重新打包送审；纯 server bug fix / 性能优化不需要动这里
3. **不要在本仓库写入任何凭证**：mcp.json 只用公开 URL，OAuth 由 WorkBuddy 的 OAuth 管理器自动完成
4. **source（powpow）一旦送审通过不可更改**——换 source 等于新连接器

## 送审流程（2026-10-18 已核实，无官方送审 CLI）

1. **前置条件**：在 WorkBuddy 开放平台（open.workbuddy.cn）完成开发者认证（个人或企业），否则无法提交
2. **打包 zip**：解压后必须是**单一顶层目录** `powpow/`，内含 `connector-meta.json` + `mcp.json` + `icon.svg` + `skills/`（本仓库内容原样打包，排除 `.git`）
3. **提交审核**：在开放平台提交 zip 包给 WorkBuddy 团队审核；zip 解析失败可发邮件 openworkbuddy@tencent.com 或加开放平台首页社群
4. **过审后**进入连接器市场；后续更新重新提交，约 10–15 分钟同步生效
5. 送审时需向平台确认 `source`（powpow）未被他人注册——source 过审后不可更改

## 已验证的服务端兼容性（2026-10-18）

- DCR 接受 WorkBuddy 两种回调：`workbuddy://workbuddy/mcp/connector%3A<powpow>/oauth/callback` 与 `http://127.0.0.1:{port}/oauth/callback`；DCR 注册响应正确回显 redirect_uris
- 协议双形态兼容：传统 JSON-RPC（initialize 握手，2025-06-18）与 modern envelope（2026-07-28 + SEP-2663 头镜像）均 200
- OAuth 全链路：authorize 302 → remembered consent 自动批准 → token 交换 → whoami 返回账号信息
- PKCE：授权请求带 `code_challenge_method=S256`，token 端点校验 `code_verifier`
- 线上服务端版本 powpow-mcp v0.14.5，9 个只读工具，单次请求 <30 秒
