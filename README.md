# BOSS Zhipin JD Scraper — A Skill for Scraping Job Descriptions from zhipin.com

**English** | [中文](README.zh-CN.md)

<p>
  <img alt="skill" src="https://img.shields.io/badge/type-agent%20skill-6f42c1">
  <img alt="agents" src="https://img.shields.io/badge/agents-Claude%20Code%20%C2%B7%20Codex%20%C2%B7%20WorkBuddy-2ea44f">
  <img alt="browser" src="https://img.shields.io/badge/requires-real%20logged--in%20browser-orange">
  <img alt="license" src="https://img.shields.io/badge/license-MIT-blue">
</p>

> A **Claude Code / Codex / WorkBuddy skill** that scrapes full **job descriptions** — responsibilities & requirements — from **BOSS Zhipin (zhipin.com)**, a Chinese online recruiting platform.
> It drives your **real, logged-in browser** (not a headless one), so it walks straight past the anti-bot verification that blocks every headless scraper. It strips hidden anti-copy watermarks, de-duplicates by job ID, and exports clean Markdown.

## ✨ What it does

- Collects the **岗位职责 / 任职要求** (responsibilities & requirements) full text from BOSS Zhipin job pages
- **Beats the anti-bot wall** by using your own logged-in browser instead of a headless one
- **Strips watermark noise** — BOSS injects hidden `display:none` decoy words into the text; `innerText` skips them cleanly
- **De-duplicates by `encryptJobId`** across multiple keywords and filters — one job, one record, zero false merges
- **Configurable role filtering** — as an example for *AI Product Manager*, it keeps **product / project manager** roles and drops **engineer / developer / algorithm** roles. It's a one-line regex: change it to keep whatever roles you want, or turn filtering off entirely
- **Runs in rounds** — collect ~30, scroll, repeat; scrape in batches with checkpoints, so long runs stay resumable and never lose progress
- **Exports Markdown** via an in-browser download, sidestepping console text-truncation on long CJK output

## 🧠 Why BOSS Zhipin is hard (and how this beats it)

BOSS Zhipin runs three defenses. This skill answers each one:

| Defense | The trap | The fix |
|---|---|---|
| **Entry CAPTCHA** | A headless browser gets redirected to an image CAPTCHA — jobs never render | Use a **real, logged-in browser**; solve the CAPTCHA **once** by hand, then 300+ visits stay clean |
| **Watermark injection** | Hidden `display:none` decoy words (`kanzhun` / `直聘` / `boss`) pollute copied text | Read `.job-sec-text` with **`innerText`**, which ignores hidden nodes |
| **API anti-scraping** | Bulk `fetch` returns stripped pages; rapid keyword queries get rate-limited | List via **real scroll**, details via **real navigation** — neither is throttled |

Two more non-obvious traps the skill handles: infinite scroll **only** responds to a real mouse wheel (synthetic `scrollTo` / `WheelEvent` stall at ~45 cards), and long Chinese output truncates in the tool channel (so it exports through a **blob download** instead).

## 🚀 Install (`npx skills add`)

```bash
npx skills add HeyClioo/boss-zhipin-jd-scraper
```

Installs across the **skills** ecosystem — **Claude Code, Codex, Gemini CLI, GitHub Copilot, WorkBuddy** and other compatible agents.

> **Runtime requirement:** this skill drives a *real, logged-in* browser, so it needs an agent with browser automation over **your own session** — e.g. Claude Code's [Chrome extension](https://claude.ai/chrome), or an equivalent MCP / browser tool in Codex / WorkBuddy. A **headless** browser hits the CAPTCHA wall and will not work.

## 🎯 Usage

1. Give your agent a real-browser tool (e.g. Claude Code's Chrome extension) and sign in to BOSS Zhipin in that browser.
2. Tell the agent: **"scrape BOSS Zhipin job descriptions for &lt;role&gt; in &lt;city&gt;"** — or hand it a `zhipin.com` search URL.
3. When the CAPTCHA appears the first time, solve it yourself, then let the skill run.

The skill collects jobs by scrolling the search page, filters to the roles you want, scrapes each detail page, and hands you a Markdown file.

## ⚙️ Filters — works for any role & city

Everything is driven by the search-page URL, so the skill handles **any city, any role, any filter combination**. The codes below are only the example this README uses (*AI Product Manager · Beijing*). To get codes for **your** search, apply the filters on BOSS Zhipin in the browser and read them straight off the resulting URL.

`?city=&jobType=&experience=&degree=&query=`

| Param | Example value (AI PM · Beijing) |
|---|---|
| `city` · Beijing | `101010100` |
| `experience` · any | `101` |
| `experience` · 1–3 yrs | `104` |
| `degree` · Bachelor | `203` |
| `degree` · Master | `204` |
| `degree` · any | *omit* |
| `jobType` · AI Product Manager | `1901` |
| `query` · keyword | `<term>` *(required for results)* |

## 📊 Output

A single Markdown file, one section per job:

```markdown
## 1. AI Product Manager
- Company: ByteDance (Internet · 10,000+)
- Salary: 30–60K/mo · 15-month pay
- Experience / Degree: 1–3 yrs / Bachelor
- Location: Beijing · Haidian
- Link: https://www.zhipin.com/job_detail/xxxx.html

Responsibilities: ...
Requirements: ...
```

## 🔑 Keywords

BOSS Zhipin scraper · zhipin.com scraper · job description scraper · JD scraper · recruitment data · China job market · web scraping · browser automation · anti-bot bypass · Claude skill · Claude Code skill · agent skill · job posting crawler · 招聘数据抓取 · BOSS直聘爬虫

## ⚠️ Disclaimer

For **personal research and job-seeking use only**. Respect BOSS Zhipin's Terms of Service and `robots.txt`, keep request rates low, and do not redistribute scraped personal data. You are responsible for how you use this skill.

## License

[MIT](LICENSE)
