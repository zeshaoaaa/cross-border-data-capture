---
name: cross-border-data-capture
description: Use this whenever the user asks to collect cross-border e-commerce data from Amazon or 1688, including Amazon product details, public reviews, A+ images, 1688 supplier candidates, supplier shop/company profile, prices, MOQ, SKU variants, packaging, logistics, cross-border attributes, and evidence files for product selection or supplier feasibility analysis. This skill routes to site-specific workflows in references/amazon.md and references/1688.md.
---

# Cross-border E-commerce Data Capture

这是跨境电商数据获取的统一入口 Skill。它把 Amazon 商品证据采集和 1688 供应商证据采集合并到一个工作流里，用于选品、竞品、供应商可行性、利润测算和合规前置判断。

核心原则：先观察真实页面，再提取证据；不要依赖固定脚本；不要伪造页面不可见的数据。

## 什么时候使用

当用户要求以下任务时使用本 Skill：

- 获取 Amazon 商品、评论、A+ 图文、公开价格、评分、评论数、变体或竞品证据。
- 获取 1688 搜索结果、供应商候选、商品详情、价格、起批量、SKU、库存、包装尺寸、重量、跨境属性、店铺服务分或公司档案。
- 做跨境电商选品、供应商筛选、采购价核验、货源证据留档、利润测算前的数据准备。
- 同时比较 Amazon 售卖端和 1688 供给端数据。

## 先选数据源

根据任务目标加载对应 reference：

| 数据源 | 读取 |
| --- | --- |
| Amazon 商品、评论、图片、A+、公开竞品证据 | `references/amazon.md` |
| 1688 供应商、货源、公司档案、价格、SKU、包装、跨境属性 | `references/1688.md` |
| Amazon + 1688 都需要 | 两个 reference 都读取，先采 Amazon 需求/竞品，再采 1688 供应商 |

不要把所有流程一次性塞进上下文。先读当前任务需要的 reference。

## 通用成功标准

一次采集任务至少要做到：

1. 使用真实浏览器观察页面结构；优先使用 `web-access`，不可用时使用当前环境可用的 headful browser / Playwright。
2. 记录来源 URL、实际跳转后的 URL、采集时间、登录/验证码/风控状态。
3. 把每条关键结论连接到可核验证据：页面文本、商品链接、供应商链接、图片 URL、本地文件或结构化 JSON。
4. 遇到登录墙、验证码、Cloudflare、淘宝/1688 风控页、地区限制时停止对应页面采集，标记阻塞，不绕过。
5. 输出文件要能复查：Markdown 写人能读的报告，CSV/JSON 写机器能读的数据。
6. 图片如需保存，下载到本地 `images/` 目录；视频如需保存，下载到本地 `videos/` 目录；Markdown 中都使用相对路径。

## 安全边界

只采集公开页面和用户已授权登录后可见的交易前页面。

不要做这些事：

- 不访问订单、购物车、账号资料、支付、收货地址、聊天私信等私人页面。
- 不绕过验证码、登录墙或风控。
- 不批量复制完整评论正文；评论以元数据和短摘录为主。
- 不伪造销量、价格、库存、供应商资质、认证、MOQ 或物流信息。
- 不把搜索摘要当成已验证证据；必须打开页面读取正文后才算 `verified_evidence`。

## 推荐输出目录

默认按任务建立一个目录：

```text
跨境电商数据获取/<task_id>/
├── evidence.json
├── capture-summary.md
├── amazon-products.csv              # 如果采 Amazon
├── amazon-reviews.csv               # 如果采 Amazon 评论
├── 1688-suppliers.csv               # 如果采 1688
├── 1688-offers.csv                  # 如果采 1688 商品
├── images/
└── videos/
```

## 证据状态

统一使用这些状态：

- `verified_evidence`：真实浏览器打开并读取过正文。
- `pending_evidence`：搜索结果、未打开链接、未完整加载页面、仅摘要可见。
- `missing_evidence`：页面没有该字段或当前任务没有提供。
- `blocked_by_login`：登录页阻塞。
- `blocked_by_captcha`：验证码阻塞。
- `blocked_by_browser_challenge`：浏览器授权、风控或人机挑战阻塞。
- `not_visible`：页面打开成功，但字段不可见。

## 合并采集建议

当用户要判断一个 SKU 能不能做时，优先这样组织：

1. Amazon：确认公开竞品价格、评分、评论数、卖点、差评痛点、图片/A+表达。
2. 1688：确认候选供应商、采购价、MOQ、SKU、包装尺寸、重量、店铺服务分、公司档案、跨境属性。
3. 汇总：把 Amazon 售卖端和 1688 供给端合并成证据表，不直接下最终商业结论，除非用户要求。

## 结束前检查

- 已读取对应 reference。
- 已记录实际访问方式和阻塞状态。
- 已输出 Markdown 汇总。
- 已输出 CSV 或 JSON 结构化数据。
- 所有本地图片引用都存在。
- 没有把登录页、验证码页、搜索摘要写成 verified evidence。
