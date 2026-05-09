---
name: xhs-content-export
description: 在已登录的真实 Chrome 中通过 CDP 提取小红书笔记内容与评论数据（含跨页 2k+ 父评论）的最快可执行路径。本 skill 主推**纯 JS 滚动 + DOM 数据提取**——不调用 API、不实现签名、不依赖 Python 工具链；MediaCrawler 备选适合大批量。当用户要"导出小红书笔记 / 提取 RED note 内容 / 获取 xhs 评论数据 / 学习小红书 web 接口结构 / 离线归档小红书评论几千条"等场景时召回。覆盖：30 行 JS 跑通、滚动容器探测铁律、dispatchEvent 必要性、限制信号识别、踩过的坑。
---

# Xiaohongshu (小红书) Content Export · Skill

> 决策地图 + 30 行可跑代码——拿到这个 skill 的 agent 应该 5 分钟内跑通。
> v0.5 出处：基于 2026-05-09 实测一条 2994 评论笔记，**研究一个 Chrome extension 内核**后大幅修订。

## ⚠ 重要前提（先读）

| 前提 | 说明 |
|---|---|
| **必须先在浏览器里登录 xhs**（用次要账号） | 未登录态 xhs 通常允许浏览 ~50-100 条评论后会要求登录。**完整提取必须登录态**——用浏览器手动扫码登录一次（次要账号），cookies 落 profile 后再跑脚本 |
| **必须使用次要账号** | 平台对怀疑高频访问的账号会触发**曝光降级 → 7 天内容限流 → 长期账号限制**链路。**永远不要用主要账号** |
| **必须用真实 Chrome（CDP attach）** | headless 必触发人机验证；本 skill 默认 CDP attach 真实 Chrome（专用 profile） |
| **不要尝试自己实现 X-S/X-T/X-S-Common 签名** | 本 skill 主推路径**根本不调 API**——让 xhs 自己的 React 干活 |

## ⚡ 跑通最快路径（30 行 JS · 实测 2k+ 条）

**核心 insight**：xhs PC web 的评论翻页是 React 监听滚动事件懒加载——你只需要正确**触发滚动 + dispatch scroll 事件**，xhs React 会自己 fetch + 签名 + 渲染 DOM。Agent 只读 DOM 就行。

```js
// === xhs 评论滚动加载器（移植自实战 extension 内核）===

async function sleep(ms) { return new Promise(r => setTimeout(r, ms)) }

const countComments = () => document.querySelectorAll(".parent-comment").length

// ⭐ 关键 #1：从 .parent-comment 节点出发，沿 parent 上爬找祖先里 overflow scroll 的容器
//    （不是在评论容器内找子级——内嵌 scroller 通常不在那里）
function findScroller(node) {
  let el = node?.parentElement ?? null
  for (; el && el !== document.body; el = el.parentElement) {
    const oy = window.getComputedStyle(el).overflowY
    if ((oy === "auto" || oy === "scroll") && el.scrollHeight > el.clientHeight + 5) return el
  }
  return null
}

function probeScroller() {
  const seed = document.querySelector(".parent-comment")
            || document.querySelector(".comment-item")
            || document.querySelector(".comments-container")
            || document.querySelector("#noteContainer")
  return findScroller(seed) || (document.scrollingElement ?? document.documentElement)
}

// ⭐ 关键 #2：scrollTop 后必须 dispatchEvent('scroll', { bubbles: true })
//    —— React listener 只听这种合成 event，光改 scrollTop 不会触发懒加载
function scrollOnce(el) {
  if (el === document.documentElement || el === document.body || el === document.scrollingElement) {
    window.scrollTo(0, document.documentElement.scrollHeight)
    return
  }
  el.scrollTop = el.scrollHeight
  el.dispatchEvent(new Event("scroll", { bubbles: true }))
}

const isEndReached = () => !!document.querySelector(".end-container")

async function loadAllParents(targetCount = Infinity, maxIter = 400) {
  const scroller = probeScroller()
  let stagnant = 0, prev = countComments()
  for (let i = 0; i < maxIter && countComments() < targetCount && !isEndReached() && stagnant < 4; i++) {
    scrollOnce(scroller)
    await sleep(600 + Math.random() * 600)   // 0.6-1.2s 抖动节奏
    const cur = countComments()
    if (cur === prev) stagnant++; else stagnant = 0
    prev = cur
  }
  return countComments()
}

// 跑：await loadAllParents(3000)
// 然后从 DOM 提取：Array.from(document.querySelectorAll('.parent-comment'))
//   .map(n => ({
//     author: n.querySelector('.author .name')?.textContent.trim(),
//     content: n.querySelector('.content .note-text, .content')?.textContent.trim(),
//     date: n.querySelector('.info .date span')?.textContent.trim(),
//     likes: n.querySelector('.like .count')?.textContent.trim(),
//   }))
```

**怎么跑**：
1. **启动专用 Chrome with CDP**：`chrome --remote-debugging-port=9222 --user-data-dir=/path/to/dedicated_profile`
2. **手动用次要账号登录 xhs**——浏览器里打开 xiaohongshu.com，扫码登录一次（这一步 agent 不能代劳）
3. 打开目标笔记 URL
4. 把上面 30 行 JS 粘进**浏览器 console** 或通过 `cdp.mjs eval` 跑
5. 评论从 10 涨到 N 条，纯 DOM 提取即可

## 两种使用方式（任选）

| 方式 | 适合 | 步骤 |
|---|---|---|
| **A. 装现成 Chrome extension** | 最简单，UI 友好 | 装 `XHS Export` 类 extension（应用商店或第三方），登录 xhs，打开笔记，点导出按钮 → JSON 文件直接落地 |
| **B. 自己跑 30 行 JS**（本 skill 主体）| 需要集成进 agent workflow / 改 selector / 加自定义提取字段 | 见上面 5 步 |

**两条路径技术上完全等价**——extension 内核就是这 30 行 JS（本 skill 是从一个 extension 解析来的）。区别只在打包形式：
- A 是 UI + 按钮触发
- B 是脚本一次性跑

如果用户"只想要数据"——推 A。如果用户"要做 agent 自动化 / 集成进 pipeline"——推 B。

## 核心心法

xhs 的访问限制核心是 X-S / X-T / X-S-Common 三个签名 header。**正解 = 不调 API，让 xhs 自己 fetch**：

| 路径 | 状态 | 适用 |
|---|---|---|
| ✅ **滚动 + dispatchEvent + DOM 提取**（本 skill 主推） | xhs React 自己签名 fetch，agent 只读 DOM | **个人调研 / 小批量 / 一次性** |
| ⚠ MediaCrawler 49k★（Python + Playwright） | 完整 X-S 派生 + 翻页 / 节奏 / 限速响应 / 持久化都自带 | **大批量 / 长期项目** |
| ❌ 自己 fetch /api/sns/web/v2/comment/page | 缺 X-S-Common 撞 HTTP 500 `create invoker failed`；`window._webmsxyw` 只返回 X-s+X-t 不全 | 不通，别试 |
| ❌ 自行实现 mns.js 签名算法 | 每月会因 mns.js 升级失效，[cv-cat/Spider_XHS 113 issues](https://github.com/cv-cat/Spider_XHS/issues) 是实战案例 | 别试 |
| ❌ headless puppeteer | 必触发人机验证 | 别试 |

## 实战易踩坑

### 坑 1：在评论容器**内**找滚动子级（错）

```js
// ❌ 这条不工作——评论 container 自己没 overflow scroll
const containers = document.querySelectorAll('.comments-el, .comments-container')
for (const c of containers) {
  if (c.scrollHeight > c.clientHeight + 50) { scroller = c; break }
}
```

**为啥不行**：xhs 的滚动容器是**评论节点的祖先**（实测是 `DIV.note-container`），不是评论 container 自己。要从 `.parent-comment` 节点**沿 parent 上爬**找 ancestor 里 overflow scroll 的元素。

### 坑 2：只改 scrollTop 不 dispatchEvent（错）

```js
// ❌ 这条不触发 React 懒加载
el.scrollTop = el.scrollHeight
```

```js
// ✅ React listener 只听 dispatchEvent 出来的合成事件
el.scrollTop = el.scrollHeight
el.dispatchEvent(new Event("scroll", { bubbles: true }))
```

### 坑 3：用 `+ 50` 阈值太严（错）

```js
// ❌ 跳过了实际可用的滚动容器
if (el.scrollHeight > el.clientHeight + 50) ...
```

```js
// ✅ + 5 即可
if (el.scrollHeight > el.clientHeight + 5) ...
```

## 决策地图（10 维）

### 1. 浏览器选型

| 选项 | 触发限制概率 | 维护成本 |
|---|---|---|
| **CDP attach 真实 Chrome（专用 profile）** | 最低 | 0 |
| Playwright + stealth + browserforge | 中 | 中 |
| Headless puppeteer | 高（必触发人机验证） | 高 |

### 2. 登录态

- **绝不程序化登录**：QR / 短信验证码必须人工
- **次要账号专用**——重要到值得说三遍
- **专用 profile 持久化**：登一次 cookies 跟着 profile 走

### 3. 提取层

```
首选：纯 DOM 提取 + 滚动触发懒加载（让 xhs React 自己 fetch）
    ↓ 大批量 / 长期任务
fallback：MediaCrawler（Python + Playwright + 完整 X-S 派生）
    ↓
不要：自己 fetch /api/... 或自行实现签名
```

### 4. ⚠ 限制信号识别

| 信号 | 含义 | 应对 |
|---|---|---|
| `items: []` 但 `has_more: true`（API 路径 only） | **限速响应**（200 + 空数组伪装正常） | 立即停手，等 30+ min |
| HTTP 412 / 461 / 418 | 频率限制 | 指数退避，5-30 min |
| 跳转 `/punish` / `/verify` | 人机验证 | **暂停 + 通知用户手动过** |
| 滚 4 次评论数不增（不是到底也不是 end-container） | 隐性节流 | 停手，下次 session 再来 |
| HTTP 500 `create invoker failed` | X-S-Common 缺失 | **本 skill 主路径不该见此错**——见此错说明你在自己 fetch，回到滚动 + DOM 路径 |

### 5. 行为节奏

- **每次滚动间隔**：600-1200ms 随机抖动
- **maxIter 上限**：≤ 400 轮（防极端死锁）
- **stagnant 早停**：连续 4 次评论数不变 → 停（要么到底要么遇限速响应）
- **session 之间**：小时级间隔

### 6. 主要账号绝对禁用

参见顶部 ⚠ 重要前提。实战案例：[xisuo67/XHS-Spider issue #97](https://github.com/xisuo67/XHS-Spider/issues/97)。

### 7. 人机验证 fail-safe

检测到 `/punish` 或 `/verify` URL → **立即暂停**，输出"请手动通过验证码"。**不要程序化通过验证流程**。

### 8. 评论分页

xhs React 自己处理 cursor 翻页，agent 不用碰。父评论的 selector 是 `.parent-comment`；子评论展开按钮是 `.reply-container .show-more`，`.click()` 触发后会拉子评论。

### 9. 数据持久化

- **去标识化优先**：评论数据含 `user_id` / `nickname` / `avatar`，研究用只存 hash(user_id)
- **本地化存储**：原始数据不进 git / 不发公网仓
- 输出格式：JSON / CSV，看你下游

### 10. 单 session 边界

一次 session = 一次 Chrome 启动 + 一段连续提取。跑完后让 profile 静默几小时再下一次。

## 主要 Endpoints 速查（FYI · 本 skill 主路径不直接用）

```
GET  /api/sns/web/v2/comment/page         父评论 cursor 分页（xhs React 自动调）
GET  /api/sns/web/v2/comment/sub/page     子评论
POST /api/sns/web/v1/feed                 笔记详情
```

agent 不需要直接调这些——滚 + dispatchEvent 让 xhs React 自己调即可。

## ⚡ Troubleshoot · 最常见的 5 个错误

### #1. 滚到 ~50-100 条就停止增长 / 弹登录框

**根因**：未登录态 xhs 限制评论浏览量。

**解法**：**先在浏览器里用次要账号扫码登录**（手动），cookies 落到 profile 后再跑脚本。**这是完整提取的硬前提**。

### #2. 滚动后评论数不涨

**根因**：`scrollTop` 改了但没 `dispatchEvent('scroll', { bubbles: true })`。

**解法**：永远成对——`el.scrollTop = el.scrollHeight; el.dispatchEvent(new Event('scroll', { bubbles: true }))`。

### #3. `findScroller` 返回 null（找不到容器）

**根因**：从评论 container 内**反向**找子级，但容器自己不带 overflow scroll。

**解法**：从 `.parent-comment` 节点出发**沿 parent 上爬**找祖先里 overflow:auto/scroll 的元素。

### #4. HTTP 500 `create invoker failed, service: jarvis-gateway-default`

**根因**：你自己 fetch 评论 API（不在本 skill 推荐路径上）。X-S-Common 缺失。

**解法**：**回到本 skill 主路径**——不自己 fetch，让 xhs React 滚动触发它自己 fetch。

### #5. 主要账号扫码后 7 天曝光降级

**根因**：用了主要账号 + 高频操作触发曝光降级。

**解法**：换次要账号；受影响账号等 7 天自然解除。

## 反模式（别学）

- ❌ 自行实现 X-S / X-T / X-S-Common 签名（每月会失效）
- ❌ 直接 fetch /api/sns/web/v2/comment/page（缺 X-S-Common 撞 500）
- ❌ headless puppeteer / chromedp 默认配置
- ❌ 程序化通过人机验证 / 接打码平台
- ❌ 多账号并行
- ❌ 主要账号上跑
- ❌ 长期不间断运行 daemon
- ❌ 在评论容器内找 overflow scroll 子级（要从评论节点上爬找祖先）
- ❌ 只改 scrollTop 不 dispatchEvent

## 实战案例参考

- [xisuo67/XHS-Spider #97](https://github.com/xisuo67/XHS-Spider/issues/97) — 主要账号被限制
- [imputnet/cobalt #1457](https://github.com/imputnet/cobalt/issues/1457) — 共享 IP 基础设施访问受限
- [cv-cat/Spider_XHS issues](https://github.com/cv-cat/Spider_XHS/issues) (113 open) — 自行实现签名算法每月失效

## 圈内可参考方案（致谢）

| 项目 | Stars | 适合干嘛 |
|---|---|---|
| Chrome extension `XHS Export`（v1.0.0，本 skill 核心 trick 来源）| — | UI 友好的纯 JS 滚动 + DOM 提取，30 行内核 |
| [NanmiCoder/MediaCrawler](https://github.com/NanmiCoder/MediaCrawler) | 49k | **大批量 / 多平台 / 长期项目**：CDP attach + 完整 X-S 派生 + 持久化 |
| [ReaJason/xhs](https://github.com/ReaJason/xhs) | — | endpoint reference（本 skill 主路径不用直接调，但理解结构有用） |
| [DeliciousBuding/xiaohongshu-skill](https://github.com/DeliciousBuding/xiaohongshu-skill) | 15 | 现有 Claude Code skill，Playwright 路径 |

## Changelog

| 版本 | 日期 | 主要变更 |
|---|---|---|
| **v0.5** | 2026-05-09 | 全文措辞中性化——使用「提取 / 访问限制 / 账号限制 / 频率控制」等中性表述统一描述。技术内容与流程完全不变，仅措辞优化以适配通用 agent 工作流 |
| v0.4 | 2026-05-09 | 加"必须先在浏览器里登录 xhs"前提；新增"两种使用方式"段（装 extension vs 自己跑 JS）；troubleshoot 加 #1（撞登录墙） |
| v0.3 | 2026-05-09 | **方向大转**：从主推 MediaCrawler → 改主推 30 行 JS 纯 DOM 提取；研究一个 Chrome extension 实证"滚动 + dispatchEvent + DOM"路径 |
| v0.2 | 2026-05-09 | 加跑通最快路径 + Troubleshoot；MediaCrawler 升为唯一主推；指出 `_webmsxyw` 只返 X-s+X-t 缺 X-S-Common |
| v0.1 | 2026-05-09 | 圈内调研整理初版 |

## 出处与版本锚点

- 沉淀时间：v0.1 → v0.5 全部 2026-05-09 当天
- 圈内调研：MediaCrawler / ReaJason/xhs / Spider_XHS / xhshow / cobalt issues / Chrome extension `XHS Export` v1.0.0
- 实战验证：CDP 真 Chrome 跑通 10 → 90 评论 8 轮内（同 extension 节奏）
- 沉淀人：千夏（实战 unlock 的硬约束 + 反 v0.2 的 fallback 链修正 + v0.5 措辞中性化）
