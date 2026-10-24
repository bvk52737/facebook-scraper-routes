# facebook scraper python：为什么 requests 直接返回 HTTP 400，哪些路线还能拿到公开数据，以及 Scraper API 的真实成本

搜索 facebook scraper python 的人，多半已经踩过同一个坑：写好一段 requests 加 BeautifulSoup 的代码，对着 Facebook 的公开页面发请求，结果拿回来的是一个 1.5 KB 的 HTTP 400 错误页。这不是你代码写错了，是这条路本身在 2026 年已经基本走不通。现在还能工作的路线剩下三条：未登录状态下解析页面里残留的 `og:` 标签和内嵌 JSON、走官方 Graph API（限制很多），或者把抓取外包给 ScraperAPI 这类抓取 API 服务。这篇文章把三条路线的能力边界、开源库的现状、以及 ScraperAPI 全部 7 个套餐的价格和 credit 消耗规则一次讲清楚。

> **快速结论**：如果你已经有 Python 爬虫代码、只是被 Facebook 的封锁卡住，ScraperAPI 的 Hobby 套餐（$49/月，年付 $44.10，100,000 credits）是最低门槛的付费档；付费前可以先白嫖——免费方案送 1,000 credits，另有 7 天试用含 5,000 credits，不需要信用卡。👉 [查看 ScraperAPI 全部套餐与免费试用](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## 为什么 requests 加 BeautifulSoup 在 Facebook 上直接失败

因为 Facebook 的边缘服务器按「整个请求的形态」做拦截决策，而最常见的写法恰好命中了被拦的组合。2026 年 6 月一篇公开发表的实测把这件事拆得很细：对 `facebook.com/Meta` 发请求，带桌面 Chrome User-Agent 的那次返回 HTTP 400，错误页只有 1,542 字节；完全不带 User-Agent 的同一次请求反而返回 HTTP 200，拿到 472 KB 的完整公开页面。换一个 `Mozilla/5.0 (compatible; bot)` 的 UA，同样返回 200 和完整数据。

也就是说，伪装得最像真人的那个请求头，反而是唯一被拒绝的组合。你在网上抄来的「设置 Chrome UA」的教程，在 Facebook 这里会起反作用。

未登录状态下，公开 Page 的数据确实存在：`og:title`、点赞数、封面图直链，以及一个内嵌的 `__bbox` JSON 数据块。同一个测试从 Meta 主页的标记里解析出了 106,675,535 这个点赞数。但这条路的坑在于：`__bbox` 的键名会被混淆和轮换，深层解析路径经常失效；HTML 结构一变，解析代码就得重写；而且循环请求很快会触发 IP 限流。它适合一次性验证，不适合当生产方案。

还有一个容易忽略的信号：Facebook 的 robots.txt 对默认 User-Agent 直接 `Disallow: /`，并且点名禁止 Scrapy，文件开头写着「未经 Facebook 明确书面许可，禁止通过自动化手段收集 Facebook 数据」。

## 开源库 facebook-scraper：还能装，但基本停在原地

PyPI 上的 facebook-scraper（kevinzg 维护）最后一个版本发布于 2022 年 8 月 31 日，之后没有再更新。它至今仍是搜 facebook scraper python 时排名最靠前的结果，`pip install facebook-scraper` 也依然能装上，所以值得花一段说清楚它能干什么、不能干什么。

它能做的：`get_posts()` 抓公开主页和小组的帖子，返回评论数、点赞数、分享数、正文、时间戳等字段；传 `group` 参数抓小组；`options={"comments": True}` 提取评论内容，`options={"reactors": True}` 提取点反应的用户；带 `credentials` 或 `cookies` 登录后能解锁更多字段（cookie 必须同时包含 `c_user` 和 `xs` 两个值，否则抛 `InvalidCookies` 异常）；还有 CLI 用法，一行命令把帖子存成 CSV。

它文档里自己承认的限制更值得关注：

- 没有任何保证每个字段都能提取到，很多字段会是 `None`
- 小组抓取可能只返回一页，私密小组直接不可用
- 抓太多，Facebook 可能临时封你的 IP

更麻烦的是登录这件事本身。传 cookie 登录抓取，意味着以登录账号身份做自动化访问，这直接撞上 Meta 的自动化数据收集条款，账号和 IP 的风险都由你自己承担。一个两年多没跟上 Facebook 前端改动的库，加上登录的法律风险，这就是为什么社区里已经有人在转而封装托管服务——原有方案跟不上反爬更新的速度。

## 2026 年还能用的三条路线

| 路线 | 能拿到什么 | 主要限制 | 成本形态 |
| --- | --- | --- | --- |
| 未登录 HTML 解析（`og:` 标签、`__bbox`） | 公开 Page 的名称、点赞数、封面图、部分帖子数据 | UA 组合全凭运气，`__bbox` 键名经常轮换，IP 限流快 | 免费，但维护成本高 |
| 官方 Graph API | 干净的 JSON，仅限你拥有或管理的资产 | 每小时 `200 × 用户数` 次调用限额；需 App 审核；小组内容已无 API 途径（Groups API 于 2024 年 4 月弃用） | 免费但覆盖面极窄 |
| 抓取 API（ScraperAPI 等） | 传 URL 进去，拿回处理过的响应，代理轮换、重试、反爬绕过都由服务端处理 | 按请求计费；具体到 Facebook 的单次消耗要在仪表板里查 | 按用量付费，可预估 |

第三条路线的本质是把代理轮换、请求头博弈、验证码处理这些脏活外包出去。你的 Python 代码只负责拼参数和解析返回值，这正是搜索 facebook scraper python 的多数开发者真正卡住的地方。

## ScraperAPI 在 Python 里怎么用

接入方式是一个 HTTP 端点：把你的 API key 和目标 URL 作为参数发过去，服务端返回 HTML 或解析好的 JSON。官方文档给出的建议是客户端超时设 70 秒——简单页面几秒就回，硬目标最多可能等 60 秒。参数的写法大致是这样：

python
import requests

params = {
    "api_key": "YOUR_KEY",
    "url": "https://www.facebook.com/Meta",  # 目标页面
    "country_code": "us",   # 地区定位，不加收 credits
}
response = requests.get(ENDPOINT, params=params, timeout=70)


端点地址和完整示例在注册后的文档里，参数顺序注意把 ScraperAPI 自己的参数放在 `url` 之前。

真正决定你花多少钱的，是 credit 消耗规则。官网的口径是：标准页面 1 credit，Amazon 5，Google 和 Bing 25，LinkedIn 30；被 Cloudflare、DataDome、PerimeterX 这类反爬系统保护、但被成功绕过的站点，每次请求加 10 credits。打开 JS 渲染会进一步加倍：`premium=true` 是 10 credits，premium 加渲染 25，ultra_premium 加渲染 75。

Facebook 本身的单次消耗官方没有公布固定数字，但这恰恰是可以自己查的：仪表板里的 Domain Cost Estimator 能算出任意 URL 的精确成本，你还可以给每次抓取设置 `max_cost` 上限，防止单个请求吃掉一大块额度。买套餐之前先跑一遍这个估算器，比看任何评测都准。👉 [用免费 credits 测试 Facebook 页面的实际消耗](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## ScraperAPI 全部套餐与价格

以下是官网 Pricing Page 当前展示的全部 7 个套餐，月付与年付（年付统一 10% off）价格都来自官方页面：

| 套餐 | 月付 | 年付 | Credits | 并发连接 | 购买 |
| --- | --- | --- | --- | --- | --- |
| Hobby | $49 | $44.10 | 100,000 | 20 | [ 开通 Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [ 开通 Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [ 开通 Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling（官方标注最受欢迎） | $475 | $427.50 | 5,000,000 | 200 | [ 开通 Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [ 开通 Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [ 开通 Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 自定义报价 | — | 22,000,000+ | 500+ | [ 联系获取报价](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

免费方案之外，注册即送 1,000 credits（最多 5 个并发连接），测试额度不够还可以联系支持团队申请追加。

功能清单各套餐完全一致：JS 渲染、premium 住宅与移动 IP、验证码与反爬检测、自动重试、JSON 自动解析、不限带宽、99.9% 正常运行时间保证。真正把套餐拉开差距的是四个维度：credits 数量、并发连接数、地区定位范围、以及分析仪表板的数据保留时长（低档套餐只保留 30 天，Business 起不限时长）。

## 你的 credits 实际能买多少页

这是多数排行榜文章不会算给你看的一笔账：ScraperAPI 的页均成本主要由 credit 乘数决定，而不是套餐价格。第三方评测算过账：按 1 credit 的标准页面口径，Hobby 套餐折合每千页 $0.49，Startup 降到 $0.15，Business 约 $0.10；但如果目标页需要渲染加反爬绕过，同样的 credits 只能买到十分之一的页数。这就是「100 万 credits」听起来很多、实际可能不够用的原因。

几个只写在条款和文档里的规则，买之前应该知道：

1. **Credits 不累积**。每月清零，按季度跑一次大批量抓取的工作负载，会为闲置的月份付费。
2. **超额处理分两档**。并发超限返回 429，不额外收费，请求直接被拒而不是排队；credits 用尽后默认暂停抓取。
3. **续命手段按套餐分层**。Auto-upgrade（credits 用尽自动升到下一档）需要主动开启，覆盖 Hobby 到 Professional；按量付费（pay-as-you-go）只在 Scaling 及以上可用，按固定单价继续跑，受你设定的月度上限约束。
4. **年付省 10%**，比如 Hobby 从 $49 降到 $44.10，一年下来省 $58.80。

一个值得注意的价位断点：从 Startup（$149）到 Business（$299），价格翻了整整一倍，触发条件却可能只是额度差 10%。评测里给的建议很实用——如果你的用量贴近套餐上限，给请求加 `max_cost`，并把渲染限制在真正需要 JS 的页面上，往往能把用量压回低一档。

## 哪些人不需要 ScraperAPI

把限制说在前面，比事后退款省事。以下几种情况，这笔钱可以不花：

- **目标地区在美国和欧盟之外，且预算在 $299 以下**。Hobby 和 Startup 的地区定位只有 US 和 EU，国家级定位从 Business 起步。
- **想要免代码的现成爬虫**。ScraperAPI 需要你自己写请求和解析。像 Apify 那类平台提供现成的 Facebook Pages Scraper actor，按结果付费、无需写代码——如果你完全不想碰 Python，那个方向更对口。
- **抓取目标几乎全是无反爬的普通网站**。有第三方评测直言，对博客、新闻站、小型电商这类目标，Business 套餐「genuinely good value」；但同一个评测也认为它在强反爬站点上表现吃力。目标构成决定它值不值，别只看套餐价格。

## 动手之前值得知道的法律边界

先把最有分量的判决放在这里：2024 年 1 月 23 日，美国地区法官 Edward Chen 在 Meta 诉 Bright Data 案中裁定，「the Facebook and Instagram Terms do not bar logged-off scraping of public data」——Meta 的服务条款约束的是登录用户，管不住未登录状态下的公开数据抓取。Meta 在次月撤回了这起诉讼。再往前，2022 年 hiQ Labs 诉 LinkedIn 案已确立抓取无需登录的公开数据不违反 CFAA。

但判决的边界同样清晰：

- Meta 服务条款第 3.2.2 条仍禁止未经许可的自动化采集，登录态抓取的违约风险真实存在
- 个人数据是另一条独立的红线。Facebook 上几乎一切内容——姓名、照片、点赞关系——都属于个人数据，存储欧盟居民信息即触发 GDPR，加州居民适用 CCPA/CPRA
- 前车之鉴摆在那里：2022 年 11 月，爱尔兰数据保护委员会因 5.33 亿用户被爬资料外流，对 Meta 开出 2.65 亿欧元罚单

实操上的可辩护姿势：只收集公开的、页面级的业务数据（粉丝数、页面分类、公开帖文的聚合互动数），不要碰点赞用户名单这类个人级数据，不要用机器人账号登录。

## 常见问题

**facebook-scraper 这个 pip 包现在还能用吗？**
能安装，但 2022 年 8 月之后没有再发版。它自己的文档承认字段可能提取不到、小组可能只返回一页、抓多了会被封 IP。用之前建议先用小批量验证目标页面还能不能返回数据。

**用 ScraperAPI 抓 Facebook 一次请求花多少 credits？**
官方没有给 Facebook 设固定费率；标准页面 1 credit，反爬绕过成功加 10，具体到你的目标 URL 可以在仪表板的 Domain Cost Estimator 里精确查询，并可用 `max_cost` 限制单次消耗。

**Facebook 小组的内容还能抓吗？**
Graph API 这条路已经没有了——Meta 在 2024 年 4 月弃用了 Groups API，小组内容不存在任何官方获取途径。开源库对小组的支持也有限（可能只返回一页，私密小组不可用）。剩下的可行路径只有未登录抓取或抓取 API，且只限公开小组。

**有没有免费试用的方式？**
有。免费方案注册即得 1,000 credits、5 个并发连接；另有 7 天试用含 5,000 credits，全程不需要信用卡。先跑通一个页面再决定要不要付费，顺序别反了。

## 按你的情况做最后选择

- 只是想验证某个公开主页能不能抓到数据：先用免费方案的 1,000 credits，配合 `og:` 标签解析，不花一分钱
- 已有 Python 爬虫代码、需要稳定跑公开页面数据：Hobby（$49/月）或 Startup（$149/月），年付再省 10%，下单前用 Cost Estimator 确认你的目标域名的单次消耗
- 需要美国和欧盟之外的地区定位，或月用量在 300 万 credits 以上：直接从 Business（$299）往上选，低于这个档位拿不到国家级定位
- 完全不想写代码：ScraperAPI 不适合你，现成 actor 平台按结果付费更省事（具体价格去对应平台核实，这里不代劳）

下单前最后确认两件事：你的目标 URL 在 Cost Estimator 里的单次 credit 消耗，以及你的月用量是否能稳在一个套餐额度之内——credits 清零不滚存，这两点比套餐价格更影响实际账单。

👉 [领取 5,000 credits 的 7 天试用，先测再买](https://www.scraperapi.com/pricing/?fp_ref=coupons)
