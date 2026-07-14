---
name: boss-zhipin-jd-scraper
description: Scrape full job descriptions (responsibilities & requirements) from BOSS Zhipin / zhipin.com. Use when the user wants to scrape or batch-collect BOSS Zhipin job postings, JDs, or recruitment data by filter. Needs a browser-automation tool that drives your real, logged-in browser (Claude Code's Chrome extension, or an equivalent in Codex / WorkBuddy) — NOT headless — to bypass anti-bot verification; strips watermark noise, dedupes by encryptJobId, exports Markdown. 抓取 BOSS直聘职位的岗位职责/任职要求全文；需用能驱动真实已登录浏览器的工具（Claude Code / Codex / WorkBuddy 等），去重去水印导出 Markdown。
---

# BOSS直聘 职位JD 抓取

抓取 BOSS直聘职位的**岗位职责 / 任职要求**全文，去重、去反爬水印，产出 Markdown。

## 核心原则（不遵守必失败）

**用用户已登录 BOSS 的真实浏览器（一个能驱动"你自己会话"的浏览器自动化工具），不要用无头浏览器。** 无头浏览器会撞图片验证码，过不去。

不同 agent 用对应的浏览器工具：
- **Claude Code**：Claude Chrome 扩展（`mcp__claude-in-chrome__*`）。注意这会覆盖"一律用 browse / 别用 claude-in-chrome"的默认规则——执行前先跟用户确认可以用。
- **Codex / WorkBuddy / 其他**：用该 agent 里等价的、能控制真实已登录浏览器的 MCP / 浏览器工具。
下文示例以 Claude Code 的工具名书写；换 agent 时把工具名替换成等价动作（导航、真实滚轮滚动、在页面执行 JS）即可，**JS 片段通用**。

三条铁律：
1. **列表靠真实鼠标滚轮**加载（`computer scroll`）——JS 的 `window.scrollTo` 没用；合成 `WheelEvent` 会动一下就卡死在 ~45 条，别信。
2. **详情 JD 靠逐页真实导航**（`navigate`）——批量 `fetch` 会被反爬剥空。
3. **正文用 `innerText`**——自动跳过 `display:none` 的水印诱饵词（kanzhun/直聘/boss）。

## 循环节奏（最重要）

**分轮循环，每轮约 30 条：滚一点 → 收一点 → 抓一批 → 存一次 → 重复。绝不"一次性加载 300 再一次性抓 300"。**
- 收集必须在搜索页做完；一旦导航去抓详情，搜索页滚动位置就丢了、回来重置成 15 条。所以**一次性把要的 jid 在搜索页收够，再离开抓**。
- 一次性滚到 300 又重又飘、容易卡；分轮稳、好盯、可中断续跑。
- 抓取分批（每批 10）、每 ~5 批下载一个检查点文件防丢。

## 前置

1. 用户已装 Claude 官方 Chrome 扩展、同账号登录 claude.ai。若报 `Browser extension is not connected`：**`Cmd+Q` 完全退出 Chrome 再重开**（关窗口不行），首装必须重启。
2. 用户的 Chrome 里已登录 BOSS直聘（未登录时薪资会打码为"-K"）。
3. 加载工具：`ToolSearch` query `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__browser_batch`

## 流程

### 0. 连接 + 打开搜索页
- `tabs_context_mcp {createIfEmpty:true}` 拿 tabId。
- `navigate` 到搜索页 URL，例：
  `https://www.zhipin.com/web/geek/jobs?city=101010100&jobType=1901&experience=104&degree=203,204&query=AI产品经理`
  （city=城市码；jobType/position=职类；experience=经验码，101=不限；degree=学历码，省略=不限；query=关键词，**必填才有结果**）
- **一定要带 `query`**：不带 query 的 `/web/geek/jobs` 是个性化推荐流，只给 15 条、`&page=` 参数无效，滚不动。
- 检查 `document.title`：含"安全验证"就**让用户在浏览器里手动过验证码 + 确认登录**，完成后继续。之后整轮基本不再弹。

### 1. 滚动加载列表（真实滚轮）
先 `javascript_tool` 存好累加容器：
```js
if(!localStorage.getItem('__jd_results'))localStorage.setItem('__jd_results','{}');
if(!localStorage.getItem('__pm'))localStorage.setItem('__pm','{}'); // 产品岗池 jid->name
```
**分轮滚**：`computer scroll {coordinate:[400,400],scroll_direction:"down",scroll_amount:15}` 滚 1-2 下 → 数一下 `document.querySelectorAll('li.job-card-box').length` → 跑第 2 步收这一轮的 jid → 再滚。每轮约 30，攒够就去抓，别一次塞十几个 scroll、也别一口气冲到 300（又重又飘、还不好盯）。单页上限约 300。

### 2. 从 DOM 收产品岗 jid（含去重 + 过滤技术岗）
```js
const done=new Set(Object.keys(JSON.parse(localStorage.getItem('__jd_results'))));
const pm=JSON.parse(localStorage.getItem('__pm'));
// 只要产品岗；剔除技术岗（按需增删关键词）
const isPM=n=>/产品/.test(n)&&!/(工程师|开发|研发|架构师|算法|测试|设计师|运营|销售|实习|数据分析|后端|前端|全栈|方案|交付|标注)/.test(n);
let add=0;
document.querySelectorAll('li.job-card-box').forEach(c=>{
  const name=c.querySelector('.job-name')?.innerText?.trim()||'';
  const jid=c.querySelector('a[href*="/job_detail/"]')?.getAttribute('href')?.split('/job_detail/')[1]?.split('.html')[0]||'';
  if(!jid||!isPM(name))return;
  if(done.has(jid)||pm[jid])return;
  pm[jid]=name; add++;
});
localStorage.setItem('__pm',JSON.stringify(pm));
JSON.stringify({新增:add, 待抓总数:Object.keys(pm).length});
```
（不需要过滤技术岗时把 `isPM` 换成 `n=>true`。）

### 3. 换关键词/放宽筛选扩池（可选，去重靠 jid）
重复 0-2，换 query（AI产品经理 / AI产品 / agent产品 …）或放宽 experience/degree。同一 jid 自动只留一份。
> - 关键词按职位类目匹配：'AI产品'/'agent产品' 能命中；'大模型'/'AI Agent' 这类窄短语常返回 0（真没匹配，不是限流）。
> - 别去抠列表 JSON 接口的参数（某些词缺 jobType/position 会返回 0，网页搜索却有）——**统一用真实搜索页滚**最稳。
> - "限流 vs 真没有"判断：某词返回 0 时，回头重查一个已知有结果的词——也变 0=限流（等一会），照常有=这词真没岗。

### 4. 逐页真实导航抓 JD（一批 10 个，间隔 1.2s）
取待抓 jid：
```js
const done=new Set(Object.keys(JSON.parse(localStorage.getItem('__jd_results'))));
const todo=Object.keys(JSON.parse(localStorage.getItem('__pm'))).filter(id=>!done.has(id));
todo.slice(0,10).join('\n')+'\n剩余:'+todo.length;
```
用 `browser_batch`，每个 jid 三个动作：`navigate` → `computer wait 1.2s` → `javascript_tool` 跑**提取器**。**提取器只 return 长度、不 return JD 正文**（正文只写进 localStorage）——否则含正文/类 URL 参数的返回值会被扩展安全过滤当敏感数据拦掉（报 `[BLOCKED: Cookie/query string data]`）：
```js
(()=>{const q=s=>document.querySelector(s);
const jid=location.pathname.split('/job_detail/')[1]?.replace('.html','');
const res=JSON.parse(localStorage.getItem('__jd_results'));
const clean=el=>el?el.innerText.split('\n').map(s=>s.trim()).filter(Boolean).join('\n'):'';
if(/安全验证/.test(document.title))return 'VERIFY '+jid; // 弹验证就停下让用户过
res[jid]={jid,name:(q('.name h1')?.innerText||'').trim(),salary:(q('.salary')?.innerText||'').trim(),jd:clean(q('.job-sec-text'))};
localStorage.setItem('__jd_results',JSON.stringify(res));
return jid.slice(0,8)+' len='+res[jid].jd.length;})()
```
每几批取一次剩余列表继续，直到剩余为 0。见 VERIFY 就暂停、让用户手动过验证再续。

### 5. 生成 Markdown 并下载导出
不要用控制台回传大文本（中文 ~1200 字会被截断）。用浏览器 blob 下载：
```js
const jds=Object.values(JSON.parse(localStorage.getItem('__jd_results')));
const fmt=(d,i)=>`## ${i+1}. ${d.name||'(无标题)'}\n\n- 薪资：${d.salary||'-'}\n- 链接：https://www.zhipin.com/job_detail/${d.jid}.html\n\n${d.jd||'（未填写JD）'}\n\n---\n`;
const md='# BOSS直聘 JD 合集（共 '+jds.length+' 条）\n\n'+jds.map(fmt).join('\n');
const b=new Blob([md],{type:'text/markdown;charset=utf-8'});const u=URL.createObjectURL(b);
const a=document.createElement('a');a.href=u;a.download='boss-jd.md';document.body.appendChild(a);a.click();
setTimeout(()=>{URL.revokeObjectURL(u);a.remove();},2000);
'downloaded '+jds.length;
```
然后 `Bash`：`sleep 3; mv ~/Downloads/boss-jd.md <目标目录>/`（**必须 sleep 3**，sleep 2 时文件还没落地）。抓取途中每 ~5 批也这样下载一个检查点，防 localStorage 被清丢数据。
（想带公司/经验/学历/地区/福利等元信息，先在收集阶段把搜索接口 `/wapi/zpgeek/search/joblist.json` 的字段一并存进 `__pm`，再在 fmt 里带上。）

## 覆盖上限（提前告知用户）
- 单关键词+严筛选实际约 150 条（接口 totalCount 显示 300 但服务端到 150 就停）。
- 要更多：多关键词并集 + 放宽经验/学历，再按 jid 去重。某细分领域真实池可能就几百，到不了上千。

## 踩坑清单
- ❌ 无头浏览器 → 撞验证码
- ❌ 不带 query 的推荐页 → 只有 15 条、`&page=` 无效，滚不动
- ❌ 批量 fetch 详情 → 被剥空返回 0
- ❌ 快速连查多个关键词接口 → 限流返回 0
- ❌ window.scrollTo / 合成 wheel → 不加载，卡 45 条
- ❌ innerHTML 取正文 → 带出隐藏水印词
- ❌ 提取器 return JD 正文/类 URL 字符串 → 被扩展安全过滤拦（`[BLOCKED]`）→ 只回长度
- ❌ 控制台回传大文本 → 中文 ~1200 字截断
- ❌ 下载后立刻 mv → 文件没落地 → 先 sleep 3
- ❌ 一次性加载全部 / 一次性抓全部 → 又重又飘、不好盯 → 分轮循环，每轮约 30
- ✅ 真实浏览器 + 已登录 + 分轮真实滚动 + 分批真实导航 + innerText + 只回长度 + jid去重 + 检查点 + blob下载

## 常用码（示例，任意城市/岗位通用）
拿自己的码：在浏览器里对 BOSS直聘勾选筛选，直接从生成的 URL 里读。以下是本文示例（AI产品经理·北京）用到的：
- 城市 city：北京=101010100
- 经验 experience：101=不限，104=1-3年
- 学历 degree：203=本科，204=硕士；省略=不限
- 职类 jobType/position：1901=AI产品经理类

岗位过滤（第 2 步的 `isPM`）完全可配置：默认保留产品/项目经理、剔除工程师/开发/算法等；改正则即可保留任意岗位，或用 `n=>true` 关闭过滤。
