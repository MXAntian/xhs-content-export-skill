# xhs-cdp-scraping · Claude Skill

A Claude Code [skill](https://docs.claude.com/claude-code/skills) for **scraping Xiaohongshu (小红书 / RED) posts and comments by attaching CDP to the user's already-logged-in real Chrome** — bypassing X-S/X-T anti-bot signatures without ever reverse-engineering them.

## What it covers

A 10-dimension decision map for the agent to think through:

1. **Browser choice**: real Chrome via CDP > Playwright + stealth > headless puppeteer
2. **Login state reuse**: never automate login, reuse the user's daily Chrome profile
3. **Capture layer**: API XHR interception (primary) + DOM scrape (fallback)
4. **⚠ Risk-control signal detection**: `items: []` soft block, `412/461/418` rate limits, `/punish` redirects
5. **Behavior cadence**: 2–5s jitter, ≤100 notes/hour, single-thread
6. **Burner account only**: never main account
7. **Captcha fail-safe**: pause and let the user solve manually, never automate
8. **Comment pagination**: cursor-based + sub-comment endpoints
9. **Data persistence**: pseudonymize, no raw `user_id` in version control
10. **Session boundary**: hour-level intervals, no 24/7 daemons

Plus: API endpoint cheat sheet (9 paths from [ReaJason/xhs](https://github.com/ReaJason/xhs)), 7-item anti-pattern list, and 3 cautionary tales (real GitHub issues).

## Core idea

**Don't reverse-engineer X-S signatures — let the page's own JavaScript sign them for you.**

Two paths:
- **CDP intercept** the real page's outgoing requests (`Network.responseReceived` + `Network.getResponseBody`)
- **Trigger fetches via `page.evaluate`** so the site's own interceptor signs them naturally

Both bypass the maintenance treadmill of reimplementing the rotating mns.js signing algorithm (community projects that go that route get broken every month — see [cv-cat/Spider_XHS's 113 open issues](https://github.com/cv-cat/Spider_XHS/issues)).

## Install (Claude Code skill)

Drop `SKILL.md` into your skills directory:

```bash
mkdir -p ~/.claude/skills/xhs-cdp-scraping
cp SKILL.md ~/.claude/skills/xhs-cdp-scraping/SKILL.md
```

Claude Code auto-loads it; the skill triggers when the conversation hits things like "scrape Xiaohongshu", "小红书 爬虫", "xhs comment data", "RED note API" etc.

For project-scoped install, drop it under your project's `.claude/skills/xhs-cdp-scraping/` instead.

## When NOT to use this skill

- ❌ High-throughput batch scraping (>100/hour) — out of scope
- ❌ Pure HTTP-only scraping without a browser — use [cv-cat/Spider_XHS](https://github.com/cv-cat/Spider_XHS) instead, but expect monthly breakage
- ❌ Production / commercial pipelines — talk to Xiaohongshu's official API team

For ✅ personal research with a burner account on a small number of public posts, this skill is the cleanest path.

## Acknowledgements

The decision map is distilled from a 2026-05-09 community survey covering:

- [NanmiCoder/MediaCrawler](https://github.com/NanmiCoder/MediaCrawler) (49k★) — the canonical CDP-attach-real-Chrome implementation
- [ReaJason/xhs](https://github.com/ReaJason/xhs) — clean endpoint reference
- [cv-cat/Spider_XHS](https://github.com/cv-cat/Spider_XHS) — pure-HTTP alternative (cautionary)
- [yangsijie666/xiaohongshu-crawler](https://github.com/yangsijie666/xiaohongshu-crawler) — Playwright + stealth alternative
- [DeliciousBuding/xiaohongshu-skill](https://github.com/DeliciousBuding/xiaohongshu-skill) — peer Claude Code skill (Playwright path; complementary to this one)

All credit for the actual implementation work goes upstream.

## License

[MIT](./LICENSE)

---

> Skill format follows the [Anthropic SKILL.md spec](https://docs.claude.com/claude-code/skills) — YAML frontmatter (name + description) + Markdown body.
