---
name: xhs-cdp-scraping
description: 用 CDP 接已登录的真实 Chrome 抓小红书帖子+评论，避开 X-S/X-T 签名反爬。当用户要"抓小红书 / 爬 RED note / xhs comment 数据 / 小红书 web 接口结构"等场景时召回。覆盖：浏览器选型、登录态复用、不破签名借页面 JS 自签、API vs DOM 双层抓取、风控信号识别、行为节奏、滑块 fail-safe、踩过的坑。
---

# Xiaohongshu (小红书) CDP Scraping · Skill

> 决策地图——给要做小红书帖子/评论抓取的 agent 用。**不是 boilerplate，是教 agent 该怎么思考这件事。**
> 出处：基于 2026-05-09 圈内调研（MediaCrawler 49k★ 等）。

## 适用场景

> 这个 skill 教你做的是 **「用用户已登录的日常 Chrome 抓自己平时看得到的小红书内容」**——把"自动化用户自己的浏览行为"这件事做对。

| 场景 | 用本 skill | 改方案 |
|---|---|---|
| 个人调研 N 篇帖子 + 评论 | ✅ | — |
| 高并发批量（>100/h） | ❌ | 出本 skill 范围 |
| 不打开浏览器纯 HTTP 调用 | ❌ | 用 [cv-cat/Spider_XHS](https://github.com/cv-cat/Spider_XHS)（自己实现 X-S 签名，但每月会被反爬升级搞坏） |

---

## 核心心法：**不破签名，借页面 JS 自签**

小红书反爬的核心是 `X-S` / `X-T` / `X-S-Common` 三个签名 header（计算依据：URL path + body + `a1` cookie + 时间戳，算法在 `mns.js` / `webmsxyw` 里高频 rotate）。

**圈内多年共识**：
- ❌ **逆向 mns.js 自己实现签名**——能做但每月坏，维护噩梦（见 [cv-cat/Spider_XHS](https://github.com/cv-cat/Spider_XHS) 的 113 个 open issues）
- ✅ **让页面 JS 自己签**——CDP attach 到一个真正打开了小红书页面的 Chrome，所有发出的请求都是页面 JS 自己签好的，agent 只读不改

两条具体路径：

| 路径 | 做法 | 适用 |
|---|---|---|
| **A. CDP 拦响应** | `Network.enable` + 监听 `Network.responseReceived` + `Network.getResponseBody` 抓后端 JSON | **推荐**，结构化数据，免 DOM 解析 |
| **B. `page.evaluate` 触发** | 在页面 context 里调 `fetch('/api/sns/web/...')`——浏览器拦截器自动加签 | 主动想拉某条数据时用 |

---

## 决策地图（10 个维度）

### 1. 浏览器选型

| 选项 | 反爬等级 | 维护成本 |
|---|---|---|
| **CDP attach 用户日常 Chrome**（本 skill 选） | 最低（指纹 = 真人）| 0 |
| Playwright + stealth + browserforge | 中 | 中（stealth 包要跟版本） |
| Headless puppeteer | 高（必触发 captcha）| 高 |

**Trigger 词**：`chrome --remote-debugging-port=9222 --user-data-dir=<existing_profile>` 启动 → CDP `ws://localhost:9222/...` 接入。或者用现成的 `browser-cdp` skill。

### 2. 登录态复用

- **绝不程序化登录**：QR 扫码 / 短信验证码不应该 agent 触发
- **复用用户日常 Chrome 的 user profile**：`a1` / `web_session` / `webId` cookies 已经在那里
- **首次启动**：用户在自己浏览器里登录一次小红书 → CDP attach → 之后 cookies 自动续

### 3. 抓取层选择：API vs DOM

```
首选 API 拦截 (CDP Network domain)
    ↓ 风控触发或字段缺失
回退 DOM scrape (CDP Runtime.evaluate / Page snapshot)
```

**永远做 DOM fallback**——风控时 API 会返回 `{success:true, has_more:true, items:[]}`（成功状态码 + 空数组）伪装正常，**只有 DOM 还能拿到内容**。

### 4. ⚠ 风控信号识别（必须做）

| 信号 | 含义 | 应对 |
|---|---|---|
| `items: []` 但 `has_more: true` | **soft block**（最阴险，返回 200 不报错）| 切 DOM，停手 30+ min |
| HTTP 412 / 461 / 418 | 频率限制 | 指数退避，5-30 min |
| 跳转 `/punish` / `/verify` 页面 | 滑块/验证码 | **暂停 + 通知用户手动过**，禁止程序化解 |
| 接口返回 `code: 10001` / `success: false` | 签名/cookie 失效 | 重新刷页面让 JS 重签 |

### 5. 行为节奏

- **请求间隔**：2-5 秒随机 jitter（不是固定值）
- **单 session 上限**：≤ 100 帖 / 小时（再多必触发风控）
- **滚动加载评论**：每滚动一次等 1-3 秒让懒加载完成
- **不要并发**：单串行，模拟人手刷

### 6. 主账号绝对禁用

- **小号专用**：xhs 一旦怀疑账号在爬，会先 shadow throttle（`items: []`）→ 7 天展示限流 → 永久封号
- **真实案例**：[xisuo67/XHS-Spider issue #97](https://github.com/xisuo67/XHS-Spider/issues/97)
- agent 应该开口就提醒用户："请用专门的小号"

### 7. 滑块 fail-safe

- 检测到 `/punish` 或 `/verify` URL → **立即暂停**，输出"请手动通过验证码"
- **不要程序化解滑块** —— 解了也没用，触发滑块本身已经被风控盯上，硬冲只会被升级到更高强度的拦截

### 8. 评论分页机制

- 顶级评论：`/api/sns/web/v2/comment/page`（cursor-based 分页）
- 子评论：`/api/sns/web/v2/comment/sub/page`（按 root comment_id 拉）
- DOM fallback：滚动评论区到底 → 等 "查看更多回复" 全部点开 → DOM 提取

### 9. 数据持久化建议

- **去标识化优先**：评论数据含 `user_id` / `nickname` / `avatar`，研究用只存 hash(user_id) + 内容文本 + 时间戳
- **本地化存储**：原始 dump 不进版本控制 / 不发公网仓
- **结构清晰命名**：单独 `raw/` 目录，方便后续清理

### 10. 单 session 边界

- 一次 session = 一次 Chrome 启动 + 一段连续抓取
- session 之间留间隔（小时级）
- 不要 24/7 daemon——长跑必上风控雷达

---

## 主要 Endpoints 速查（来自 [ReaJason/xhs](https://github.com/ReaJason/xhs)）

```
POST /api/sns/web/v1/feed                 帖子详情（by note_id）→ note_card
GET  /api/sns/web/v1/homefeed             首页推荐流
POST /api/sns/web/v1/search/notes         关键词搜索
GET  /api/sns/web/v2/comment/page         顶级评论（cursor 分页）
GET  /api/sns/web/v2/comment/sub/page     子评论（按 root id）
GET  /api/sns/web/v1/user/otherinfo       用户主页 profile
GET  /api/sns/web/v1/user_posted          用户发布的帖子列表
POST /api/sns/web/v1/login/qrcode/create  扫码登录二维码（仅首次手动）
GET  /api/sns/web/v1/login/qrcode/status  扫码状态轮询
```

所有接口都需要 `x-s` / `x-t` / `x-s-common` headers + `a1` / `web_session` cookies——用本 skill 的 CDP 路径 **全部由浏览器自动加好**，agent 不用碰。

---

## 反模式（别学）

- ❌ 用 headless puppeteer / chromedp 默认配置
- ❌ 自己实现 X-S 签名算法（每月坏）
- ❌ 程序化解滑块 / 接打码平台
- ❌ 多账号并发矩阵
- ❌ 主账号上跑
- ❌ 不做 DOM fallback（被 soft block 后数据全空）
- ❌ 24/7 长跑 daemon

---

## 反面教材（活的踩坑）

- [xisuo67/XHS-Spider #97](https://github.com/xisuo67/XHS-Spider/issues/97) — 用户主账号被封，每次都要 QR 重登
- [imputnet/cobalt #1457](https://github.com/imputnet/cobalt/issues/1457) — 共享 IP 基础设施被全网封，没人修得动
- [jackwener/OpenCLI #10](https://github.com/jackwener/OpenCLI/issues/10) — search API 静默返回空，团队最后改 DOM scrape

---

## 圈内可参考方案（致谢）

| 项目 | Stars | 适合干嘛 |
|---|---|---|
| [NanmiCoder/MediaCrawler](https://github.com/NanmiCoder/MediaCrawler) | 49k | **思路同款**（CDP attach + 不破签名），事实标准，可直接抄实现 |
| [ReaJason/xhs](https://github.com/ReaJason/xhs) | — | 最干净的 endpoint reference，~40 条 path 列全 |
| [yangsijie666/xiaohongshu-crawler](https://github.com/yangsijie666/xiaohongshu-crawler) | 7 | Playwright + stealth 路径，跟本 skill 是互补关系（用户没现成日常 Chrome 时用） |
| [Cloxl/xhshow](https://github.com/Cloxl/xhshow) | 中 | 想理解签名算法时读，**不建议照抄**（维护负担） |
| [DeliciousBuding/xiaohongshu-skill](https://github.com/DeliciousBuding/xiaohongshu-skill) | 15 | 现有 Claude Code skill，Playwright 路径，跟本 skill 不冲突 |

---

## 出处与版本锚点

- 沉淀时间：2026-05-09
- 圈内调研覆盖：MediaCrawler / ReaJason/xhs / Spider_XHS / xhshow / cobalt issues
- 沉淀人：千夏（基于研究报告整理）
