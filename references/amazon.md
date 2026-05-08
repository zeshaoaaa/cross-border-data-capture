# Amazon 数据获取流程

本流程用于 Amazon 公开商品页、搜索结果、评论页、A+ 图文详情和竞品证据采集。它继承原 `amazon-data-capture` 的核心规则：自适应浏览器流程，不依赖固定脚本。

## 适用任务

- 搜索 Amazon 商品。
- 打开 Amazon 商品链接或 ASIN。
- 获取商品标题、价格、评分、评论数、变体、配送显示、商品要点、详情字段。
- 获取主图、图集、A+ 图文详情图片。
- 获取评论页状态和评论元数据。
- 为跨境选品、竞品分析、利润测算提供公开页面证据。

## 成功标准

1. 第一步打开 headful Chrome 或用户已登录的真实浏览器环境。
2. 确认 Amazon 页面是否登录；如果未登录且目标内容不可见，停止采集并标记 `blocked_by_login`。
3. 确认打开的是真实 Amazon 公开商品页或搜索结果页。
4. 商品详情保存到 Markdown。
5. 结构化数据保存到 CSV 或 JSON。
6. 图片下载到本地 `images/` 目录，并在 Markdown 中使用相对路径。
7. 遇到验证码、地区限制、登录墙、评论不可见时记录状态，不伪造。

## 安全边界

只处理 Amazon 公开页面。

不要访问：

- Seller Central。
- Brand Registry。
- 购物车。
- 订单。
- 账号资料。
- 收货地址。
- 支付页面。

## 推荐工具顺序

1. `web-access`：通过用户 Chrome / CDP 使用真实登录态。
2. Playwright headful browser：当 web-access 不可用但浏览器可访问页面时使用。
3. 普通 HTTP：只作为辅助，不作为评论和动态内容的强依赖。

## 采集步骤

### 1. 明确目标

确认：

- marketplace：例如 `amazon.com`、`amazon.de`、`amazon.co.uk`。
- query 或 ASIN 或商品链接。
- 是否需要评论。
- 是否需要图片下载。
- 输出目录。

用户未指定 marketplace 时，不要擅自默认到某个国家站；先追问，除非上下文已经明确。

### 2. 打开搜索页或商品页

- query：打开 `https://www.amazon.<tld>/s?k=<query>`。
- ASIN：打开 `/dp/<ASIN>`。
- 商品链接：直接打开，记录跳转后的最终 URL。

如果搜索结果页只显示广告、登录墙或验证码，记录阻塞状态。

### 3. 判断页面是否可采

检查：

- URL 是否是 Amazon 域名。
- 是否出现 `/ap/signin`。
- 是否出现 CAPTCHA / robot check。
- 商品标题是否存在。
- 商品图片区域是否存在。
- 价格、评分、评论数是否可见。

关键内容不可见时，不要猜。

### 4. 提取商品基础字段

推荐字段：

```text
asin,title,brand,price,currency,rating,review_count,availability,shipping_text,variation_summary,feature_bullets,detail_fields,source_url,captured_at,page_status
```

自适应定位顺序：

1. 语义明确的 DOM：标题、价格、评分、评论数、要点、详情表格。
2. 可见文本邻近关系：例如 Brand、About this item、Customer reviews。
3. URL 和页面结构：提取 ASIN。
4. JSON-LD 或页面内数据：只作为辅助，必须与可见页面交叉核对。

### 5. 提取图片

图片类型：

- 主图。
- 图集。
- A+ / From the manufacturer / Product Description 图片。

流程：

1. 滚动页面触发懒加载。
2. 提取高分辨率图片 URL。
3. 去重。
4. 过滤透明占位图、loading 图、极小图标。
5. 下载到 `images/`。
6. Markdown 中使用 HTML figure：

```html
<figure>
  <img src="./images/01-gallery.jpg" alt="商品图集-第1张" style="max-width: 720px; width: 100%; height: auto;" />
  <figcaption>商品图集-第1张<br />原图 URL：https://...</figcaption>
</figure>
```

不要使用无意义图片名或 alt，例如 `![1](images/23.jpg)`。

### 6. 采集评论

如果用户要求评论页：

1. 打开 `/product-reviews/<ASIN>/?reviewerType=all_reviews&sortBy=recent&pageNumber=1`。
2. 按用户要求采集页数；未指定时至少采 1 页。
3. 每页写一行 `review_page_status`。
4. 评论正文只保存短摘录，不批量复制完整正文。

CSV 表头：

```text
record_type,asin,product_title,page_number,review_index,reviewer_name,review_rating,review_title,review_date,verified_purchase,review_excerpt,page_status,requested_url,source_url,captured_at
```

### 7. Markdown 结构

```markdown
# Amazon 商品数据采集

## 采集说明
- 来源：...
- 采集时间：...
- 页面状态：...

## 商品信息
- ASIN：...
- 标题：...
- 价格：...
- 评分：...
- 评论数量：...

## 商品主图和图集
<figure>...</figure>

## 商品要点
- ...

## 商品详情字段
- ...

## A+ 图文详情
<figure>...</figure>

## 评论页状态
- ...

## 阻塞与缺失字段
- ...
```

## 结束检查

- 已记录登录状态或可见性状态。
- Markdown 存在。
- CSV/JSON 存在。
- 图片文件真实存在。
- 图片 alt 有意义。
- 评论页状态覆盖用户要求的页码，或写明未覆盖原因。
