# BOSS Zhipin JD Scraper — A Skill for Scraping Job Descriptions from zhipin.com

**English** | [中文](README.zh-CN.md)

<p>
  <img alt="skill" src="https://img.shields.io/badge/type-agent%20skill-6f42c1">
  <img alt="platform" src="https://img.shields.io/badge/Claude%20Code-Chrome%20extension-2ea44f">
  <img alt="license" src="https://img.shields.io/badge/license-MIT-blue">
</p>

> A Claude Code **skill** that scrapes full **job descriptions** — responsibilities & requirements — from **BOSS Zhipin (zhipin.com)**, China's largest recruiting platform.
> It drives your **real, logged-in Chrome** (not a headless browser), so it walks straight past the anti-bot verification that blocks every headless scraper. It strips hidden anti-copy watermarks, de-duplicates by job ID, and exports clean Markdown.

## ✨ What it does

- Collects the **岗位职责 / 任职要求** (responsibilities & requirements) full text from BOSS Zhipin job pages
- **Beats the anti-bot wall** by using your own logged-in browser instead of a headless one
- **Strips watermark noise** — BOSS injects hidden `display:none` decoy words into the text; `innerText` skips them cleanly
- **De-duplicates by `encryptJobId`** across multiple keywords and filters — one job, one record, zero false merges
- **Title filtering** — keep product/PM roles, drop engineer/dev/algorithm roles (fully configurable)
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

Built for **Claude Code** with the [Claude Chrome extension](https://claude.ai/chrome). It drives a real browser, so it needs an interactive, logged-in Chrome session.

## 🎯 Usage

1. Install the Claude Chrome extension and sign in to BOSS Zhipin in that Chrome.
2. Tell the agent: **"scrape BOSS Zhipin job descriptions for &lt;role&gt; in &lt;city&gt;"** — or hand it a `zhipin.com` search URL.
3. When the CAPTCHA appears the first time, solve it yourself, then let the skill run.

The skill collects jobs by scrolling the search page, filters to the roles you want, scrapes each detail page, and hands you a Markdown file.

## ⚙️ Filter codes

BOSS Zhipin encodes filters in the search URL: `?city=&jobType=&experience=&degree=&query=`

| Filter | Value |
|---|---|
| City · Beijing | `city=101010100` |
| Experience · any | `experience=101` |
| Experience · 1–3 yrs | `experience=104` |
| Degree · Bachelor | `degree=203` |
| Degree · Master | `degree=204` |
| Degree · any | *omit* |
| Job type · AI Product Manager | `jobType=1901` |
| Keyword | `query=<term>` *(required for results)* |

## 📊 Output

A single Markdown file, one section per job:

```markdown
## 1. AI Product Manager
- Company: ByteDance (Internet · 10,000+)
- Salary: 30-60K·15 months
- Experience / Degree: 1-3 yrs / Bachelor
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
