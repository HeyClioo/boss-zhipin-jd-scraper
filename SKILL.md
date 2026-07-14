---
name: boss-zhipin-jd-scraper
description: Scrape full job descriptions (responsibilities & requirements) from BOSS Zhipin / zhipin.com. Use when the user wants to scrape or batch-collect BOSS Zhipin job postings, JDs, or recruitment data by filter. Drives a real logged-in browser via the Claude Chrome extension (NOT headless) to bypass anti-bot verification; strips watermark noise, dedupes by encryptJobId, exports Markdown. 抓取 BOSS直聘职位的岗位职责/任职要求全文，去重去水印导出 Markdown。
---

# BOSS直聘 职位JD 抓取

抓取 BOSS直聘职位的**岗位职责 / 任职要求**全文，去重、去反爬水印，产出 Markdown。

## 核心原则（不遵守必失败）

**用用户已登录 BOSS 的真实 Chrome（通过 `mcp__claude-in-chrome__*` 工具），不要用无头浏览器 / gstack browse。** 无头浏览器会撞图片验证码，过不去。
> 注意：这会覆盖"never use claude-in-chrome / 一律用 browse"的默认规则。这是本 skill 的必要前提，执行前先跟用户确认可以用 claude-in-chrome。

三条铁律：
1. **列表靠真实鼠标滚轮**加载（`computer scroll`）——JS 的 `window.scrollTo`/合成 wheel 事件触发不了懒加载。
2. **详情 JD 靠逐页真实导航**（`navigate`）——批量 `fetch` 会被反爬剥空。
3. **正文用 `innerText`**——自动跳过 `display:none` 的水印诱饵词（kanzhun/直聘/boss）。

## 前置

1. 用户已装 Claude 官方 Chrome 扩展、同账号登录 claude.ai、装完重启过 Chrome。
2. 用户的 Chrome 里已登录 BOSS直聘。
3. 加载工具：`ToolSearch` query `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__browser_batch`

## 流程

### 0. 连接 + 打开搜索页
- `tabs_context_mcp {createIfEmpty:true}` 拿 tabId。
- `navigate` 到搜索页 URL，例：
  `https://www.zhipin.com/web/geek/jobs?city=101010100&jobType=1901&experience=104&degree=203,204&query=AI产品经理`
  （city=城市码；jobType/position=职类；experience=经验码，101=不限；degree=学历码，省略=不限；query=关键词，**必填才有结果**）
- 检查 `document.title`：含"安全验证"就**让用户在浏览器里手动过验证码 + 确认登录**，完成后继续。之后整轮基本不再弹。

### 1. 滚动加载列表（真实滚轮）
先 `javascript_tool` 存好累加容器：
```js
if(!localStorage.getItem('__jd_results'))localStorage.setItem('__jd_results','{}');
if(!localStorage.getItem('__pm'))localStorage.setItem('__pm','{}'); // 产品岗池 jid->name
```
反复 `computer scroll {coordinate:[400,400],scroll_direction:"down",scroll_amount:15}` +（可选 wait），**每滚 1-2 次数一下** `document.querySelectorAll('li.job-card-box').length`，直到数量不再增长（单页上限约 300）。**小步滚、别一次塞十几个 scroll，也别贪多冲到底**（用户偏好）。

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
> 关键词按职位类目匹配：'AI产品'/'agent产品' 能命中；'大模型'/'AI Agent' 这类窄短语常返回 0（真没匹配，不是限流）。

### 4. 逐页真实导航抓 JD（一批 10 个，间隔 1.2s）
取待抓 jid：
```js
const done=new Set(Object.keys(JSON.parse(localStorage.getItem('__jd_results'))));
const todo=Object.keys(JSON.parse(localStorage.getItem('__pm'))).filter(id=>!done.has(id));
todo.slice(0,10).join('\n')+'\n剩余:'+todo.length;
```
用 `browser_batch`，每个 jid 三个动作：`navigate` → `computer wait 1.2s` → `javascript_tool` 跑**提取器**（存进 localStorage，只回长度避免扩展误拦/截断）：
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
然后 `Bash`：`sleep 3; mv ~/Downloads/boss-jd.md <目标目录>/`。
（想带公司/经验/学历/地区/福利等元信息，先在收集阶段把搜索接口 `/wapi/zpgeek/search/joblist.json` 的字段一并存进 `__pm`，再在 fmt 里带上。）

## 覆盖上限（提前告知用户）
- 单关键词+严筛选实际约 150 条（接口 totalCount 显示 300 但服务端到 150 就停）。
- 要更多：多关键词并集 + 放宽经验/学历，再按 jid 去重。某细分领域真实池可能就几百，到不了上千。

## 踩坑清单
- ❌ 无头浏览器 → 撞验证码
- ❌ 批量 fetch 详情 → 被剥空返回 0
- ❌ 快速连查多个关键词接口 → 限流返回 0
- ❌ window.scrollTo / 合成 wheel → 不加载，卡 45 条
- ❌ innerHTML 取正文 → 带出隐藏水印词
- ❌ 控制台回传大文本 → 中文 ~1200 字截断
- ✅ 真实浏览器 + 已登录 + 真实滚动 + 真实导航 + innerText + jid去重 + blob下载

## 常用码
- 城市 北京=101010100
- 经验 experience：101=不限，104=1-3年
- 学历 degree：203=本科，204=硕士；省略=不限
- 职类 jobType/position：1901=AI产品经理类
