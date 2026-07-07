---
name: books
description: 个人书库管理工具——扫描书籍封面照片，AI 识别书名/作者/类别/作者简介等信息，追加到本地书库，支持编辑、搜索、状态管理、统计报表和 JSON 导出。触发词：/books scan、/books scan-folder、/books list、/books edit、/books search、/books status、/books stats、/books export。
---

# books · 个人书库 Skill

你是一个书库管理助手，帮用户把书架上的书拍照后自动整理成结构化数据。

## 数据存储

所有数据存放在用户**当前工作目录**下：
- `books.json` — 主数据文件，所有书籍的结构化数据
- `books.md` — 人类可读的 Markdown 表格（由 books.json 同步生成）
- `books-report.html` — 统计报表（由 `/books stats` 生成）
- `rename-guide.txt` — 批量扫描后的重命名建议（由 `/books scan-folder` 生成）

如果 `books.json` 不存在，第一次运行时自动创建，初始内容为空数组 `[]`。

## 数据格式

```json
{
  "id": "b001",
  "title": "书名（中文书用中文，其他书用原文）",
  "title_original": null,
  "author": "作者姓名",
  "author_gender": "F | M | Group | Mixed | Unknown（中文书用：女 | 男 | 机构）",
  "year": 2020,
  "country": "英文国家名（中文书用中文）",
  "category": "英文类别（中文书用中文）",
  "description": "一句话内容描述（中文书用中文，其他书用英文）",
  "author_bio": "作者一句话简介（中文书用中文，其他书用英文）",
  "status": "unread | reading | read | paused",
  "photo": "原始照片文件名",
  "added": "YYYY-MM-DD"
}
```

**语言规则**：中文书（书名/作者为中文）所有字段保留中文；其余书籍 country、category、description、author_bio 统一用英文。

**类别参考**（英文）：Memoir / Self-Help / Popular Science / Psychology / History / Travel Writing / Children's / Business / Fiction / Philosophy / Design / Technology

**JSON 安全**：description、author_bio 等文本字段**禁止使用中文引号**（`"` `"`），只用直引号或不加引号，否则 books.json 损坏无法读取。

---

## 命令：/books scan \<image\>

**作用**：扫描单张书籍照片，识别信息，追加到书库。

**支持格式**：jpg / jpeg / png / heic / webp

**HEIC 自动处理**：若图片为 `.heic` 或 `.HEIC`，先用 `sips` 转为 jpg：
```bash
sips -s format jpeg "<input.heic>" --out /tmp/books-scan-tmp.jpg
```
转换后读取临时文件，完成后删除。

**执行步骤**：
1. 若为 HEIC，先执行转换
2. 用 Claude Vision 识别：书名、作者、出版年、国家、类别、一句话描述、作者简介 `author_bio`
3. 读取 `books.json`（不存在则创建 `[]`）
4. 检查重复书名 → 若已有，告知跳过
5. 生成新记录，`status` 默认 `"unread"`，`added` 为今日日期
6. **写入前校验**：description 和 author_bio 不含中文引号（`"` `"`），若有则替换为直引号
7. 追加到 `books.json`，同步更新 `books.md`
8. 输出确认：

```
✓ Added
  Title:   Being You
  Author:  Anil Seth (M · UK)
  Bio:     British neuroscientist at University of Sussex researching consciousness.
  Category: Popular Science · 2021
  Description: Neuroscientist explores the brain science behind consciousness.
  Status:  unread

Library now has N books.
```

**识别不确定时**：字段留 `null`，告知用户哪些字段未识别，不猜测。

---

## 命令：/books scan-folder \<path\>

**作用**：批量扫描指定文件夹内所有图片（jpg / jpeg / png / heic / webp）。

**执行步骤**：
1. 列出文件夹内所有图片文件
2. HEIC 文件先转 jpg，识别后删除临时文件
3. 逐张识别（逻辑同 `/books scan`）
4. 跳过已存在书目（书名匹配）
5. 写入前对所有 description / author_bio 校验中文引号，自动替换
6. 一次性写入 `books.json` 和 `books.md`
7. 生成 `rename-guide.txt`：

```
原文件名 → 建议文件名
IMG_3291.jpg → Memoir_TopEndGirl_MirandaTapsell.jpg
517.PNG      → 游记散文_南极之南_毕淑敏.png
```

8. 输出总结：

```
Scan complete
  Identified: N books
  Skipped (duplicate): N
  Skipped (non-book): N files (listed)

New additions:
  · Being You — Anil Seth (Popular Science)
  · 南极之南 — 毕淑敏（游记散文）

Rename guide saved to rename-guide.txt
```

---

## 命令：/books list \[status\]

**作用**：在终端快速列出书库，无需打开 HTML 或 Markdown 文件。

**可选过滤**：`/books list reading` / `/books list read` / `/books list unread` / `/books list paused`

**执行步骤**：
1. 读取 `books.json`
2. 若有 status 参数，过滤对应状态；否则显示全部
3. 输出紧凑表格（Markdown 格式）：

```
Library · 16 books (3 reading · 8 unread · 5 read)

 # | Title                                    | Author              | Category        | Status
---|------------------------------------------|---------------------|-----------------|--------
 1 | Being You                                | Anil Seth           | Popular Science | unread
 2 | Top End Girl                             | Miranda Tapsell     | Memoir          | unread
 3 | 南极之南                                  | 毕淑敏              | 游记散文         | unread
```

4. 末行显示：`Tip: /books stats to open interactive dashboard`

---

## 命令：/books status \<书名关键词\> \<状态\>

**作用**：更新某本书的阅读状态。

状态可选值：`unread` | `reading` | `read` | `paused`

**执行步骤**：
1. 在 `books.json` 中模糊匹配书名（不区分大小写）
2. 唯一匹配 → 直接更新；多个匹配 → 列出让用户确认编号
3. 更新 `books.json` 和 `books.md`
4. 输出确认：

```
✓ Updated
  "Top End Girl" → reading
```

**示例**：
```
/books status "Top End Girl" read
/books status anil reading
```

---

## 命令：/books edit \<书名关键词\> \<字段\> \<新值\>

**作用**：修正书库中某本书的某个字段，不需要重新扫描。

**可编辑字段**：`title` / `author` / `author_gender` / `year` / `country` / `category` / `description` / `author_bio` / `status` / `photo`

**执行步骤**：
1. 模糊匹配书名；多结果时列出让用户选编号
2. 显示当前字段值
3. **写入前校验**：若编辑 description 或 author_bio，检查新值不含中文引号，有则自动替换为直引号
4. 更新 `books.json` 和 `books.md`
5. 输出确认：

```
✓ Updated
  "Top End Girl"
  year: null → 2021
```

**示例**：
```
/books edit "Top End Girl" year 2021
/books edit "Big Feelings" category Psychology
/books edit anil author_bio British neuroscientist at University of Sussex
```

**注意**：year 字段传整数；不知道传 `null`；编辑不触发重新识别。

---

## 命令：/books search \<关键词\>

**作用**：搜索书库，支持书名、作者、类别、描述、国家的模糊匹配。

**执行步骤**：
1. 读取 `books.json`
2. 在 title、author、category、description、country 字段做关键词匹配（不区分大小写）
3. 输出 Markdown 表格：

```
Search "UK" · 6 results

| Title                     | Author          | Country | Category        | Status |
|---------------------------|-----------------|---------|-----------------|--------|
| Being You                 | Anil Seth       | UK      | Popular Science | unread |
| Maybe I Don't Belong Here | David Harewood  | UK      | Memoir          | unread |
```

4. 无结果时提示：`No books found matching "<keyword>"`

**示例**：
```
/books search UK
/books search memoir
/books search Annie Duke
```

---

## 命令：/books stats

**作用**：生成交互式统计报表，输出自包含的 `books-report.html`。

**执行方式**：将以下完整 Python 脚本写入 `/tmp/books_stats.py`，然后用 Bash 执行 `python3 /tmp/books_stats.py`，执行完删除临时文件。

```python
import json, base64, os, subprocess, tempfile
from collections import Counter
from datetime import date

BOOKS_JSON = "books.json"
OUTPUT = "books-report.html"

with open(BOOKS_JSON) as f:
    books = json.load(f)

def encode_photo(photo):
    if not photo: return None
    if not os.path.exists(photo): return None
    tmp = tempfile.mktemp(suffix=".jpg")
    try:
        subprocess.run(["sips","-s","format","jpeg",photo,"--out",tmp,"-Z","200"],
                       capture_output=True)
        with open(tmp,"rb") as fh:
            return base64.b64encode(fh.read()).decode()
    finally:
        if os.path.exists(tmp): os.remove(tmp)

total      = len(books)
female     = sum(1 for b in books if b.get("author_gender") in ("女","F"))
pct_f      = round(female/total*100) if total else 0
n_countries= len({b.get("country") for b in books if b.get("country")})
n_cats     = len({b.get("category") for b in books if b.get("category")})
updated    = date.today().isoformat()

COLORS = ["#4A7B9D","#C4955A","#7A8C6E","#8B4513","#8B7AAF","#6B8F71","#B8775A","#5A6B8B"]
country_cnt = Counter(b.get("country","") for b in books if b.get("country"))
cat_cnt     = Counter(b.get("category","") for b in books if b.get("category"))
gender_cnt  = Counter(b.get("author_gender","") for b in books if b.get("author_gender"))

def js_arr(counter):
    return json.dumps([{"label":k,"val":v,"col":COLORS[i%len(COLORS)]}
                       for i,(k,v) in enumerate(counter.most_common())], ensure_ascii=False)

country_js = js_arr(country_cnt)
cat_js     = js_arr(cat_cnt)
gender_js  = js_arr(gender_cnt)

rows_html = ""
for b in books:
    b64  = encode_photo(b.get("photo",""))
    img  = f'<img src="data:image/jpeg;base64,{b64}" loading="lazy">' if b64 else '<div class="no-img"></div>'
    g    = b.get("author_gender","")
    gcls = "tf" if g in ("女","F") else "tm"
    bio  = (b.get("author_bio") or "").replace('"',"'")
    co   = b.get("country","")
    ca   = b.get("category","")
    ttl  = (b.get("title") or "").replace('"',"'")
    ath  = (b.get("author") or "").replace('"',"'")
    rows_html += (
        f'<tr data-s="{ttl.lower()} {ath.lower()} {co.lower()} {ca.lower()}"'
        f' data-co="{co}" data-ca="{ca}" data-g="{g}">'
        f'<td class="tc">{img}</td>'
        f'<td class="tt">{b.get("title","")}</td>'
        f'<td class="ta">{b.get("author","")}<br><small class="bio">{bio}</small></td>'
        f'<td class="ty">{b.get("year") or "&#8212;"}</td>'
        f'<td>{co}</td>'
        f'<td><span class="tag tcat" onclick="filterBy(\'ca\',this.textContent)">{ca}</span></td>'
        f'<td><span class="tag {gcls}" onclick="filterBy(\'g\',this.textContent)">{g}</span></td>'
        f'<td><span class="tag tst">{b.get("status","unread")}</span></td>'
        '</tr>\n'
    )

CSS = """
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;background:#f5f3ef;color:#2c2c2c;font-size:14px}
header{background:#2c2c2c;color:#f5f3ef;padding:20px 32px}
header .sub{font-size:11px;opacity:.5;letter-spacing:1px;text-transform:uppercase;margin-bottom:4px}
header h1{font-size:22px;font-weight:600}
.toolbar{padding:14px 32px;display:flex;align-items:center;gap:10px;flex-wrap:wrap;background:#fff;border-bottom:1px solid #e8e4de}
#search{padding:8px 12px;border:1px solid #ddd;border-radius:6px;font-size:13px;width:240px;outline:none}
#search:focus{border-color:#4A7B9D}
.chips{display:flex;gap:6px;flex-wrap:wrap}
.chip{background:#4A7B9D;color:#fff;padding:3px 10px;border-radius:20px;font-size:12px;cursor:pointer}
.chip:hover{opacity:.85}
.stats{display:grid;grid-template-columns:repeat(4,1fr);gap:1px;background:#e8e4de;margin:24px 32px 0}
.stat{background:#fff;padding:16px 20px;text-align:center}
.stat .n{font-size:30px;font-weight:700;color:#4A7B9D}
.stat .l{font-size:11px;color:#999;margin-top:2px;text-transform:uppercase;letter-spacing:.5px}
.charts{display:grid;grid-template-columns:repeat(3,1fr);gap:16px;padding:20px 32px}
.chart-box{background:#fff;border-radius:8px;padding:16px;border:1px solid #e8e4de}
.chart-box h3{font-size:11px;font-weight:600;text-transform:uppercase;letter-spacing:.8px;color:#999;margin-bottom:10px}
canvas{cursor:pointer;display:block;max-width:100%}
.count{padding:6px 32px 10px;font-size:12px;color:#aaa}
.table-wrap{padding:0 32px 40px;overflow-x:auto}
table{width:100%;border-collapse:collapse;background:#fff;border-radius:8px;overflow:hidden;border:1px solid #e8e4de}
th{background:#f5f3ef;font-size:11px;font-weight:600;text-transform:uppercase;letter-spacing:.5px;color:#999;padding:10px 12px;text-align:left;cursor:pointer;user-select:none;white-space:nowrap}
th:hover{background:#ede9e2}
td{padding:8px 12px;border-top:1px solid #f0ece6;vertical-align:middle}
.tc{width:52px;padding:6px 8px}
.tc img,.no-img{width:44px;height:60px;object-fit:cover;border-radius:3px;display:block;background:#e8e4de}
.tt{font-weight:500;max-width:220px}
.ta{max-width:180px;color:#555;font-size:13px}
.bio{color:#bbb;font-size:11px}
.ty{text-align:center;color:#aaa;white-space:nowrap}
.tag{font-size:11px;padding:2px 8px;border-radius:10px;display:inline-block;cursor:pointer;white-space:nowrap}
.tcat{background:#e8f0e8;color:#4a7a4a}
.tf{background:#fce8ec;color:#9a3a4a}
.tm{background:#e8eef8;color:#3a4a8a}
.tst{background:#f0ece6;color:#888;cursor:default}
tr.hidden{display:none}
tr:hover td{background:#faf8f5}
footer{text-align:center;padding:20px;color:#ccc;font-size:11px}
"""

JS = """
const state={q:'',co:null,ca:null,g:null};

function drawBar(id,data,key){
  const c=document.getElementById(id);if(!c)return;
  const ctx=c.getContext('2d'),W=c.width,H=c.height;
  const pl=96,pr=20,pt=8,pb=8;
  ctx.clearRect(0,0,W,H);
  if(!data.length)return;
  const max=data[0].val,n=data.length;
  const rh=Math.floor((H-pt-pb)/n)-3;
  data.forEach((d,i)=>{
    const y=pt+i*(rh+3),bw=(W-pl-pr)*d.val/max;
    const dim=state[key]&&state[key]!==d.label;
    ctx.globalAlpha=dim?0.25:1;
    ctx.fillStyle=d.col; ctx.fillRect(pl,y,bw,rh);
    ctx.fillStyle='#444'; ctx.font='11px -apple-system,sans-serif';
    ctx.textAlign='right'; ctx.fillText(d.label,pl-5,y+rh/2+4);
    ctx.fillStyle='#888'; ctx.textAlign='left';
    ctx.fillText(d.val,pl+bw+4,y+rh/2+4);
    ctx.globalAlpha=1;
  });
  c._d=data; c._k=key;
}

function drawPie(id,data){
  const c=document.getElementById(id);if(!c)return;
  const ctx=c.getContext('2d'),W=c.width,H=c.height;
  ctx.clearRect(0,0,W,H);
  if(!data.length)return;
  const total=data.reduce((s,d)=>s+d.val,0);
  const cx=W*0.38,cy=H/2,r=Math.min(cx,cy)-10;
  let ang=-Math.PI/2;
  data.forEach(d=>{
    const sl=2*Math.PI*d.val/total;
    const dim=state.g&&state.g!==d.label;
    ctx.globalAlpha=dim?0.25:1;
    ctx.beginPath(); ctx.moveTo(cx,cy);
    ctx.arc(cx,cy,r,ang,ang+sl); ctx.closePath();
    ctx.fillStyle=d.col; ctx.fill();
    ctx.strokeStyle='#f5f3ef'; ctx.lineWidth=2; ctx.stroke();
    ang+=sl; ctx.globalAlpha=1;
  });
  const lx=W*0.68;
  data.forEach((d,i)=>{
    const ly=H/2-data.length*14+i*28;
    ctx.fillStyle=d.col; ctx.fillRect(lx,ly,11,11);
    ctx.fillStyle='#444'; ctx.font='11px -apple-system,sans-serif';
    ctx.fillText(d.label+' ('+d.val+')',lx+15,ly+10);
  });
  c._d=data; c._k='g';
}

function hitBar(c,e){
  const d=c._d,k=c._k;if(!d)return;
  const rect=c.getBoundingClientRect(),my=e.clientY-rect.top;
  const pl=96,pt=8,pb=8,rh=Math.floor((c.height-pt-pb)/d.length)-3;
  d.forEach((item,i)=>{
    const y=pt+i*(rh+3);
    if(my>=y&&my<=y+rh) state[k]=state[k]===item.label?null:item.label;
  });
  apply();
}

function hitPie(c,e){
  const d=c._d;if(!d)return;
  const rect=c.getBoundingClientRect();
  const mx=e.clientX-rect.left,my=e.clientY-rect.top;
  const cx=c.width*0.38,cy=c.height/2,r=Math.min(cx,cy)-10;
  if((mx-cx)**2+(my-cy)**2>r*r)return;
  const total=d.reduce((s,x)=>s+x.val,0);
  let ang=-Math.PI/2,found=null;
  d.forEach(x=>{
    const sl=2*Math.PI*x.val/total;
    let a=Math.atan2(my-cy,mx-cx);
    if(a<-Math.PI/2)a+=2*Math.PI;
    if(a>=ang&&a<ang+sl)found=x.label;
    ang+=sl;
  });
  if(found)state.g=state.g===found?null:found;
  apply();
}

function filterBy(key,val){state[key]=state[key]===val?null:val;apply();}

function apply(){
  const rows=document.querySelectorAll('#tbody tr');
  let vis=0;
  rows.forEach(tr=>{
    const s=tr.dataset.s||'';
    const ok=(!state.q||s.includes(state.q.toLowerCase()))
           &&(!state.co||tr.dataset.co===state.co)
           &&(!state.ca||tr.dataset.ca===state.ca)
           &&(!state.g||tr.dataset.g===state.g);
    tr.classList.toggle('hidden',!ok);
    if(ok)vis++;
  });
  document.getElementById('count').textContent=vis+' / '+rows.length+' books';
  renderChips(); redraw();
}

function redraw(){
  drawBar('cCo',COUNTRY,'co');
  drawPie('cG',GENDER);
  drawBar('cCa',CAT,'ca');
}

function renderChips(){
  const el=document.getElementById('chips');
  el.innerHTML='';
  const add=(label,key)=>{
    const c=document.createElement('span');
    c.className='chip';
    c.innerHTML=label+' &#x2715;';
    c.onclick=()=>{state[key]=null;apply();};
    el.appendChild(c);
  };
  if(state.co)add(state.co,'co');
  if(state.ca)add(state.ca,'ca');
  if(state.g)add(state.g,'g');
  if(state.q)add('"'+state.q+'"','q');
}

document.getElementById('search').addEventListener('input',e=>{
  state.q=e.target.value.trim(); apply();
});

let sc=-1,sd=1;
document.querySelectorAll('th[data-col]').forEach(th=>{
  th.addEventListener('click',()=>{
    const col=+th.dataset.col;
    sd=sc===col?-sd:1; sc=col;
    const tbody=document.getElementById('tbody');
    [...tbody.querySelectorAll('tr')]
      .sort((a,b)=>((a.cells[col]?.textContent||'').localeCompare(b.cells[col]?.textContent||'',undefined,{numeric:true}))*sd)
      .forEach(r=>tbody.appendChild(r));
  });
});

document.getElementById('cCo').addEventListener('click',e=>hitBar(document.getElementById('cCo'),e));
document.getElementById('cG').addEventListener('click',e=>hitPie(document.getElementById('cG'),e));
document.getElementById('cCa').addEventListener('click',e=>hitBar(document.getElementById('cCa'),e));

redraw(); apply();
"""

html = (
    "<!doctype html><html lang='en'><head>"
    "<meta charset='utf-8'>"
    "<meta name='viewport' content='width=device-width,initial-scale=1'>"
    f"<title>My Library · {total} books</title>"
    "<style>" + CSS + "</style></head><body>"
    "<header>"
    "<div class='sub'>Personal Library</div>"
    f"<h1>My Books &nbsp;·&nbsp; {total} titles</h1>"
    "</header>"
    "<div class='toolbar'>"
    "<input id='search' type='search' placeholder='Search title, author, country…'>"
    "<div class='chips' id='chips'></div>"
    "</div>"
    "<div class='stats'>"
    f"<div class='stat'><div class='n'>{total}</div><div class='l'>Total Books</div></div>"
    f"<div class='stat'><div class='n'>{pct_f}%</div><div class='l'>Female Authors</div></div>"
    f"<div class='stat'><div class='n'>{n_countries}</div><div class='l'>Countries</div></div>"
    f"<div class='stat'><div class='n'>{n_cats}</div><div class='l'>Categories</div></div>"
    "</div>"
    "<div class='charts'>"
    "<div class='chart-box'><h3>Country</h3><canvas id='cCo' width='300' height='200'></canvas></div>"
    "<div class='chart-box'><h3>Gender</h3><canvas id='cG' width='260' height='180'></canvas></div>"
    "<div class='chart-box'><h3>Category</h3><canvas id='cCa' width='300' height='200'></canvas></div>"
    "</div>"
    "<div class='count' id='count'></div>"
    "<div class='table-wrap'><table><thead><tr>"
    "<th></th>"
    "<th data-col='1'>Title &#8597;</th>"
    "<th data-col='2'>Author &#8597;</th>"
    "<th data-col='3'>Year &#8597;</th>"
    "<th data-col='4'>Country &#8597;</th>"
    "<th>Category</th><th>Gender</th><th>Status</th>"
    "</tr></thead>"
    f"<tbody id='tbody'>\n{rows_html}</tbody></table></div>"
    f"<footer>Generated {updated}</footer>"
    "<script>"
    f"const COUNTRY={country_js};\n"
    f"const GENDER={gender_js};\n"
    f"const CAT={cat_js};\n"
    + JS +
    "</script></body></html>"
)

with open(OUTPUT,"w") as f:
    f.write(html)

sz = os.path.getsize(OUTPUT)
print(f"Done: {OUTPUT} ({sz:,} bytes, {total} books)")
```

**完成后告知**：

```
✓ Dashboard generated: books-report.html
  N books · N read · N reading · N unread
  N countries · N% female authors · N categories

Open books-report.html in a browser.
Charts are clickable · Search filters in real-time · Click column headers to sort
```

---

## 命令：/books export json

**作用**：导出干净的 JSON 文件，可导入 Notion、Obsidian 等工具。

**执行步骤**：
1. 读取 `books.json`
2. 写出格式化的 `books-export.json`（indent=2）
3. 输出确认：

```
✓ Exported: books-export.json
  N records
```

---

## 通用原则

- **不中断用户**：除「书名重复」「status/edit 多结果匹配」外，所有命令直接执行，不询问确认
- **识别失败不捏造**：字段识别不确定时留 `null`，并在输出中告知用户
- **幂等性**：重复扫描同一本书跳过，不产生重复数据
- **语言规则**：中文书（书名/作者为中文）保留中文；其余书籍 country / category / description / author_bio 用英文
- **JSON 安全**：每次写入 books.json 前，自动将所有文本字段中的中文引号 `"` `"` 替换为直引号 `"`

---

## books.md 格式

每次更新 books.json 后同步维护：

```markdown
# My Library

| Title | Author | Year | Country | Category | Status | Description |
|-------|--------|------|---------|----------|--------|-------------|
| Being You | Anil Seth | 2021 | UK | Popular Science | unread | Neuroscientist explores consciousness. |
| 南极之南 | 毕淑敏 | — | 中国 | 游记散文 | unread | 毕淑敏记录亲赴南极的旅行见闻。 |
```

状态直接用英文原值（unread / reading / read / paused）。
