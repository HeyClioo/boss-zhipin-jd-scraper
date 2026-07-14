# BOSS直聘 JD 抓取技能 — 从 zhipin.com 批量抓取岗位描述

[English](README.md) | **中文**

<p>
  <img alt="skill" src="https://img.shields.io/badge/类型-agent%20skill-6f42c1">
  <img alt="platform" src="https://img.shields.io/badge/Claude%20Code-Chrome%20扩展-2ea44f">
  <img alt="license" src="https://img.shields.io/badge/license-MIT-blue">
</p>

> 一个 **Claude Code / Codex / WorkBuddy 技能**，从 **BOSS直聘（zhipin.com，一个中国在线招聘平台）** 批量抓取职位的**岗位职责 / 任职要求**全文。
> 它驱动你**已登录的真实浏览器**（不是无头浏览器），因此能绕过拦住所有无头爬虫的安全验证；同时去掉隐藏的反爬水印词，按职位 ID 去重，导出干净的 Markdown。

## ✨ 能做什么

- 抓取 BOSS直聘职位页的**岗位职责 / 任职要求**全文
- **绕过安全验证**：用你自己已登录的浏览器，而不是无头浏览器
- **去水印**：BOSS 往正文塞了 `display:none` 的隐藏诱饵词，用 `innerText` 自动跳过
- **按 `encryptJobId` 去重**：跨多个关键词和筛选，同一职位只留一份，零误判
- **标题过滤**：只保留产品经理岗，剔除工程师/开发/算法等技术岗（可自由配置）
- **导出 Markdown**：用浏览器内下载，绕开长中文文本在工具通道里的截断

## 🧠 BOSS直聘为什么难爬（以及怎么破）

BOSS直聘有三道防线，本技能逐一破解：

| 防线 | 陷阱 | 破解 |
|---|---|---|
| **入口验证码** | 无头浏览器被跳转到图片验证码，职位根本不渲染 | 用**真实已登录浏览器**，首次**人工过一次**验证，之后 300+ 次访问零验证 |
| **水印注入** | 隐藏的 `display:none` 诱饵词（`kanzhun` / `直聘` / `boss`）污染复制文本 | 用 **`innerText`** 读 `.job-sec-text`，自动忽略隐藏节点 |
| **接口反爬** | 批量 `fetch` 返回被剥空的页面；快速换关键词查接口被限流 | 列表走**真实滚动**、详情走**真实导航**，两者都不被限流 |

还有两个不显眼的坑技能也处理了：无限滚动**只认真实鼠标滚轮**（合成 `scrollTo`/`WheelEvent` 会卡在约 45 条）；长中文在工具通道里会截断（所以改用 **blob 下载**导出）。

## 🚀 安装（`npx skills add`）

```bash
npx skills add HeyClioo/boss-zhipin-jd-scraper
```

可安装到 **skills** 生态的各类 agent —— **Claude Code、Codex、Gemini CLI、GitHub Copilot、WorkBuddy** 等。

> **运行要求：** 本技能驱动**真实、已登录**的浏览器，所以需要 agent 具备对**你自己会话**的浏览器自动化能力 —— 例如 Claude Code 的 [Chrome 扩展](https://claude.ai/chrome)，或 Codex / WorkBuddy 里等价的 MCP / 浏览器工具。**无头**浏览器会撞验证码，跑不通。

## 🎯 用法

1. 装好 Claude Chrome 扩展，并在该 Chrome 里登录 BOSS直聘。
2. 对 agent 说：**"抓取 BOSS直聘上 &lt;城市&gt; 的 &lt;岗位&gt; 职位描述"** —— 或直接给它一个 `zhipin.com` 搜索页 URL。
3. 首次弹验证码时你手动过一下，之后让技能自己跑。

技能会滚动搜索页收集职位、按你要的岗位过滤、逐个抓详情页，最后给你一个 Markdown 文件。

## ⚙️ 筛选码

BOSS直聘把筛选写在搜索 URL 里：`?city=&jobType=&experience=&degree=&query=`

| 筛选 | 值 |
|---|---|
| 城市 · 北京 | `city=101010100` |
| 经验 · 不限 | `experience=101` |
| 经验 · 1-3年 | `experience=104` |
| 学历 · 本科 | `degree=203` |
| 学历 · 硕士 | `degree=204` |
| 学历 · 不限 | *省略* |
| 职类 · AI产品经理 | `jobType=1901` |
| 关键词 | `query=<词>` *（必填才有结果）* |

## 📊 产出

一个 Markdown 文件，每个职位一节：

```markdown
## 1. AI产品经理
- 公司：字节跳动（互联网 · 10000人以上）
- 薪资：30-60K·15薪
- 经验/学历：1-3年 / 本科
- 地点：北京·海淀区
- 链接：https://www.zhipin.com/job_detail/xxxx.html

岗位职责：……
任职要求：……
```

## 🔑 关键词

BOSS直聘爬虫 · zhipin 抓取 · 职位描述抓取 · JD 抓取 · 招聘数据 · 岗位职责任职要求 · 网页爬虫 · 浏览器自动化 · 反爬绕过 · Claude 技能 · Claude Code skill · agent skill · 职位采集

## ⚠️ 免责声明

仅供**个人调研与求职**使用。请遵守 BOSS直聘的服务条款与 `robots.txt`，控制请求频率，不要二次分发抓取到的个人数据。使用后果由你自己负责。

## 许可

[MIT](LICENSE)
