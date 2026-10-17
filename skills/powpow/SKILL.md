---
name: powpow
display_name: PowPop 泡泡
display_name_en: PowPop
description: 指导 AI 正确使用 PowPop MCP 连接器的 9 个工具：附近发现（数字人/用户/帖子）、话题搜索、数字人与帖子详情、景点商圈、活动与会员方案、身份检查。
description_zh: PowPop（泡泡）连接器的使用指导：发现附近的数字人、用户与帖子，搜索话题，查看数字人/帖子详情、景点商圈、打卡活动与会员方案。
description_en: How to use the PowPop connector's 9 tools: nearby discovery (digital humans/users/posts), topic search, digital human & post details, attractions, campaigns, membership plans and identity check.
category: life
version: 1.0.0
author: durenzidu
---

# PowPop（泡泡）MCP 连接器使用指导

PowPop 是一款基于地理位置的社交应用：用户在地图上发现附近的公开数字人、真实用户与帖子，可以和数字人对话、参与打卡活动。本连接器把这些能力封装为 9 个只读工具。

## 认证与前置条件

- **公开内容无需登录**：附近的数字人、帖子、景点、活动、会员方案等公开数据，匿名即可查询。
- **账号相关能力需要授权**：首次调用若返回 401，说明用户尚未完成 OAuth 授权。引导用户在连接器设置中找到 PowPop 并完成授权（授权页由 PowPop 提供服务），完成后重试即可。授权态由平台自动维护（token 自动续期），无需反复询问。
- **坐标系注意**：涉及坐标的工具统一使用 **GCJ-02（高德/火星坐标系）**。用户直接提供 GPS（WGS-84）经纬度时会有约 300–600 米偏差；优先让用户提供地名，由 `geocode_place` 或工具的 `place` 参数转换。

## 工具总览

| 工具 | 用途 | 关键参数 | 典型场景 |
| --- | --- | --- | --- |
| `find_nearby` | 发现附近的数字人、真实用户与帖子 | `lat`+`lng` 或 `place`（二选一）；`radius_meters` 默认 1000；`types` 可只返回某类 | 「我附近有什么人/帖子」 |
| `geocode_place` | 地名转经纬度，并报告匹配置信度 | `place` | 需要坐标前的第一步 |
| `get_digital_human` | 数字人完整资料（人设/简介/位置/情绪） | `id`（来自 find_nearby） | 找到数字人后的「下一跳」 |
| `get_post_detail` | 帖子全文 + 评论列表 | `post_id`（来自 find_nearby/search_posts） | 展开某条帖子的讨论 |
| `nearby_attractions` | 附近景点与商圈（按距离） | `lat`+`lng` 或 `place`；`radius_meters` 默认 3000 | 「附近有什么好玩的」 |
| `search_posts` | 按关键词检索公开帖子 | `keyword`（匹配正文，不区分大小写） | 「关于故宫的帖子」 |
| `list_campaigns` | 全部公开活动（内嵌打卡点列表） | 无 | 「现在有什么活动」 |
| `list_membership_plans` | 会员/套餐价格 | 无 | 「会员多少钱」 |
| `whoami` | 检查当前授权身份 | 无 | 「我是谁」「登录了吗」 |

> **参数 schema 以 `tools/list` 的实时返回为准**。上面只列常用参数；可选参数、默认值若有出入，不要与用户争辩，按实时 schema 调用。

## 各工具用法要点

### find_nearby —— 附近发现核心工具
- 位置三选一：`place`（地名，推荐，自动编码）> `lat`+`lng`（GCJ-02）。三者都不传时服务端无法定位，会返回错误。
- `types` 可按需过滤：`digital_humans`（数字人）/ `users`（真实用户）/ `posts`（帖子），默认全部返回。
- 返回含每类的摘要与距离；数字人卡片可继续用 `get_digital_human` 展开。
- 示例：用户说「国贸附近有什么人」→ `{ "place": "国贸", "types": ["digital_humans"] }`。

### geocode_place —— 地名编码 + 置信度
- 返回 `lat`/`lng`、`matched_name`（服务端实际匹配到的 POI 名）与 `confidence`（0~1）。
- **置信度低于 0.8 时，必须把「你输入的地名」与「实际匹配到的地名」都转告用户确认**，不要直接把模糊匹配当成答案继续查询。

### get_digital_human —— 数字人详情
- `id` 必须来自 `find_nearby` 等工具返回的数字人 id，不要凭空构造。
- 返回人设（persona）、简介、位置与当前情绪，适合先介绍再引导用户对话。

### get_post_detail —— 帖子详情
- `post_id` 来自 `find_nearby` 或 `search_posts` 的返回。
- 仅公开帖子（is_public）可查；非公开帖子会返回可读错误，如实转告用户即可。

### nearby_attractions —— 景点与商圈
- 与 `find_nearby` 共享位置参数约定（`place` 或 `lat`+`lng`，GCJ-02），默认半径 3000 米。
- 返回景点（attractions）与商圈（business_districts）两组，各带距离，可合并进「附近推荐」类回答。

### search_posts —— 关键词搜帖
- `keyword` 对帖子正文做不区分大小写的模糊匹配。
- 命中结果含摘要、位置与头图链接；引用具体帖子时优先带上其位置信息，回答更完整。
- 示例：「搜索关于秋天的帖子」→ `{ "keyword": "秋天" }`。

### list_campaigns —— 活动列表
- 无参数。返回全部公开活动（如「北京 100 家博物馆」打卡活动），每个活动内嵌打卡点（acts）列表。
- 用户问「有什么活动/打卡」时优先调用本工具。

### list_membership_plans —— 会员方案
- 无参数。返回按渠道分组的价格：支付宝时长包与 PayPal/华为订阅。
- 用户问价格、会员、套餐时调用；如实转述价格，不要替用户做购买建议。

### whoami —— 身份检查
- 无参数。已授权时返回 PowPop 账号信息（昵称、徽章数、会员状态）；未授权时提示连接方式。
- 在「以用户身份」执行任何操作前，或用户询问身份/登录状态时，先调用本工具。

## 错误处理与恢复

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 401 / 未授权 | OAuth 未完成或已失效 | 引导用户在连接器设置中重新授权 PowPop，然后重试 |
| `error` 字段的可读错误（如「缺少参数」「无法解析」） | 参数缺失或输入无效 | 按错误信息修正参数后重试；不要原样把英文错误堆给用户，翻译成 actionable 的中文提示 |
| 空结果（0 条） | 该区域/关键词确实没有公开内容 | 如实告知「没有找到」，可建议用户换关键词、扩大半径或换个地点 |
| 超时（>30 秒） | 服务端无响应 | 不要连续重试；告知用户稍后再试 |

## 高风险操作确认

本连接器全部 9 个工具均为**只读**，不存在写操作或支付动作，无需额外确认。涉及购买会员的意向，只提供 `list_membership_plans` 的价格信息，引导用户前往 PowPop 应用内完成，不要在对话中代用户决策。

---

_最后同步：2026-10-18，对应 MCP server 版本 powpow-mcp 0.14.5（9 工具）。server 工具契约发生变更（新增/删除/改参）时，需同步更新本文件并提升连接器版本重新送审；纯 server 内部修复不需要更新。_
