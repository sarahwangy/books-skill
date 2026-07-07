---
name: books
description: 个人书库管理工具——扫描书籍封面照片，AI 识别书名/作者/类别/作者简介等信息，追加到本地书库，支持编辑、搜索、状态管理、统计报表和 JSON 导出。触发词：/books scan、/books scan-folder、/books isbn、/books list、/books edit、/books search、/books status、/books stats、/books present、/books deploy、/books export。
---

# books · 个人书库 Skill

你是一个书库管理助手，帮用户把书架上的书拍照后自动整理成结构化数据。

## 数据存储

所有数据存放在用户**当前工作目录**下的 `my_lovely_library/` 文件夹中：

- `my_lovely_library/books.json` — 主数据文件，所有书籍的结构化数据
- `my_lovely_library/books.md` — 人类可读的 Markdown 表格
- `my_lovely_library/books-report.html` — 统计报表（由 `/books stats` 生成）
- `my_lovely_library/books-present.html` — 幻灯片演示（由 `/books present` 生成）
- `my_lovely_library/books-export.json` — 导出文件（由 `/books export` 生成）
- `my_lovely_library/rename-guide.txt` — 批量扫描后的重命名建议（由 `/books scan-folder` 生成）

**首次运行时**：若 `my_lovely_library/` 不存在，自动执行 `mkdir -p my_lovely_library` 创建，无需用户手动操作。

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
  "rating": null,
  "notes": null,
  "photo": "原始照片文件名",
  "added": "YYYY-MM-DD"
}
```

**语言规则**：中文书（书名/作者为中文）所有字段保留中文；其余书籍 country、category、description、author_bio 统一用英文。

**类别参考**（英文）：Memoir / Self-Help / Popular Science / Psychology / History / Travel Writing / Children's / Business / Fiction / Philosophy / Design / Technology

**JSON 安全**：description、author_bio 等文本字段**禁止使用中文引号**（`"` `"`），只用直引号或不加引号，否则 my_lovely_library/books.json 损坏无法读取。

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
3. 确保 `my_lovely_library/` 目录存在（`mkdir -p my_lovely_library`），读取 `my_lovely_library/books.json`（不存在则创建 `[]`）
4. 检查重复书名 → 若已有，告知跳过
5. 生成新记录，`status` 默认 `"unread"`，`added` 为今日日期
6. **写入前校验**：description 和 author_bio 不含中文引号，若有则替换为直引号
8. 追加到 `my_lovely_library/books.json`，同步更新 `my_lovely_library/books.md`
8. 输出确认：

```
✓ Added
  Title:    Being You
  Author:   Anil Seth (M · UK)
  Bio:      British neuroscientist at University of Sussex researching consciousness.
  Category: Popular Science · 2021
  Status:   unread

Library now has N books.
```

**识别不确定时**：字段留 `null`，告知用户哪些字段未识别，不猜测。

---

## 命令：/books scan-folder \<path\>

**作用**：批量扫描指定文件夹内所有图片（jpg / jpeg / png / heic / webp）。

**执行步骤**：
1. 列出文件夹内所有图片文件，统计总数
2. **若超过 20 张，先询问用户**：

```
Found N images. Run a trial scan of the first 20 to check accuracy before processing all?
  1. Yes, trial first 20
  2. No, scan all N now
```

3. 确保 `my_lovely_library/` 目录存在（`mkdir -p my_lovely_library`）
4. HEIC 文件先转 jpg，识别后删除临时文件
4. 逐张识别（逻辑同 `/books scan`），每张约 1–3 秒
4. 跳过已存在书目（书名匹配）
5. 写入前对所有 description / author_bio 校验中文引号，自动替换
6. 一次性写入 `my_lovely_library/books.json` 和 `my_lovely_library/books.md`
7. 生成 `my_lovely_library/rename-guide.txt`：

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

Rename guide saved to my_lovely_library/rename-guide.txt
```

---

## 命令：/books isbn \<ISBN\>

**作用**：通过 ISBN 号码从 Open Library 查询书籍元数据，准确率高于封面识别，追加到书库。

**ISBN 来源**：书背面条形码下方的 13 位数字（EAN-13），或书名页的 10 位旧 ISBN。

**执行步骤**：

1. 用 curl 查询 Open Library API（**必须加 `-L` 跟随重定向**，ISBN 端点会 302 跳转）：

```bash
curl -sL "https://openlibrary.org/isbn/<ISBN>.json"
```

2. 解析返回 JSON，提取以下字段：
   - `title` → 书名
   - `authors[0].key` → 作者 key（格式 `/authors/OL12345A`），需再发一次请求：
     ```bash
     curl -sL "https://openlibrary.org/authors/OL12345A.json"
     ```
     从中提取 `name`；`bio` 字段在 Open Library 中经常为空，**不依赖它**
   - `publish_date` → 出版年（提取年份数字，如 `"2021"` 或 `"October 5, 2021"` 都取四位数字）
   - `publishers[0]` → 出版社（用于推断国家，非直接字段）
   - `subjects` → 可选，辅助判断 category

3. Open Library **不提供**以下字段，由 Claude 根据已知书名/作者知识补全：
   - `country` — 根据作者国籍/出版商推断
   - `category` — 根据书名和 subjects 判断
   - `description` — Claude 写一句话内容描述
   - `author_bio` — Claude 写一句话作者简介（Open Library bio 经常为空，Claude 始终自行撰写）
   - `author_gender` — Claude 根据作者姓名和知识判断

4. 确保 `my_lovely_library/` 目录存在（`mkdir -p my_lovely_library`）
5. 读取 `my_lovely_library/books.json`，检查是否已有同名书 → 已有则告知跳过

6. 组装完整记录，`status` 默认 `"unread"`，`added` 为今日，`photo` 为 `null`

7. **写入前校验**：description / author_bio 不含中文引号，有则自动替换

7. 追加到 `my_lovely_library/books.json`，同步更新 `my_lovely_library/books.md`

9. 输出确认：

```
✓ Added via ISBN (Open Library)
  ISBN:     9780008669713
  Title:    Being You: A New Science of Consciousness
  Author:   Anil Seth (M · UK)
  Bio:      British neuroscientist at the University of Sussex researching consciousness.
  Category: Popular Science · 2021
  Status:   unread

  Fields from Open Library: title, author, year
  Fields inferred by Claude: country, category, description, author_bio, author_gender

Library now has N books.
```

**API 返回为空或 404 时**：
```
✗ ISBN 9780000000000 not found in Open Library.
  Try /books scan with a cover photo instead.
```

**示例**：
```
/books isbn 9780008669713
/books isbn 0385737951
```

---

## 命令：/books list \[status\]

**作用**：在终端快速列出书库，无需打开 HTML 或 Markdown 文件。

**可选过滤**：`/books list reading` / `/books list read` / `/books list unread` / `/books list paused`

**执行步骤**：
1. 读取 `my_lovely_library/books.json`
2. 若有 status 参数，过滤对应状态；否则显示全部
3. 输出紧凑表格（Markdown 格式）：

```
Library · 16 books (1 reading · 15 unread)

 # | Title                                    | Author              | Category        | Status
---|------------------------------------------|---------------------|-----------------|--------
 1 | Being You                                | Anil Seth           | Popular Science | reading
 2 | Top End Girl                             | Miranda Tapsell     | Memoir          | unread
```

4. 末行显示：`Tip: /books stats to open interactive dashboard`

---

## 命令：/books status \<书名关键词\> \<状态\>

**作用**：更新某本书的阅读状态。

状态可选值：`unread` | `reading` | `read` | `paused`

**执行步骤**：
1. 在 `my_lovely_library/books.json` 中模糊匹配书名（不区分大小写）
2. 唯一匹配 → 直接更新；多个匹配 → 列出让用户确认编号
3. 确保 `my_lovely_library/` 目录存在（`mkdir -p my_lovely_library`）
4. 确保 `my_lovely_library/` 目录存在（`mkdir -p my_lovely_library`）
5. 更新 `my_lovely_library/books.json` 和 `my_lovely_library/books.md`
6. 输出确认：

```
✓ Updated
  "Top End Girl" → reading
```

---

## 命令：/books edit \<书名关键词\> \<字段\> \<新值\>

**作用**：修正书库中某本书的某个字段，不需要重新扫描。

**可编辑字段**：`title` / `author` / `author_gender` / `year` / `country` / `category` / `description` / `author_bio` / `status` / `photo`

**执行步骤**：
1. 模糊匹配书名；多结果时列出让用户选编号
2. 显示当前字段值
3. **写入前校验**：若编辑 description 或 author_bio，检查新值不含中文引号，有则自动替换
4. 更新 `my_lovely_library/books.json` 和 `my_lovely_library/books.md`
5. 输出确认：

```
✓ Updated
  "Top End Girl"
  year: null → 2021
```

---

## 命令：/books rate \<书名关键词\> \<评分\>

**作用**：给一本书打星（1–5 分），存入书库。

**执行步骤**：
1. 模糊匹配书名；多结果时列出让用户选编号
2. 验证评分为 1–5 的整数，否则提示重新输入
3. 确保 `my_lovely_library/` 目录存在（`mkdir -p my_lovely_library`）
4. 更新 `my_lovely_library/books.json` 中该书的 `rating` 字段，同步 `my_lovely_library/books.md`
5. 输出确认：

```
✓ Rated
  "Being You" → ★★★★★ (5/5)
```

**示例**：
```
/books rate "Being You" 5
/books rate anil 4
```

---

## 命令：/books note \<书名关键词\> \<笔记内容\>

**作用**：给一本书添加私人备注（读后感、摘录、想法等），不限长度。

**执行步骤**：
1. 模糊匹配书名；多结果时列出让用户选编号
2. 若已有笔记，显示当前内容，询问"覆盖还是追加？"：

```
Current note: "读了三遍，每次都有新发现"
  1. Append to existing note
  2. Replace with new note
```

3. 确保 `my_lovely_library/` 目录存在（`mkdir -p my_lovely_library`）
4. 更新 `my_lovely_library/books.json` 中该书的 `notes` 字段（追加时用换行分隔），同步 `my_lovely_library/books.md`
5. 输出确认：

```
✓ Note saved
  "Being You"
  Note: "意识不是被动接收现实，而是大脑主动建构的预测——读完彻底改变了我对感知的理解"
```

**示例**：
```
/books note "Being You" 意识不是被动接收现实，而是大脑主动建构的预测
/books note anil "读了三遍，每次都有新发现"
```

---

## 命令：/books suggest

**作用**：根据书库现有书目的模式，推荐下一本读什么。

**执行步骤**：
1. 读取 `my_lovely_library/books.json`，分析：
   - 已读书目（`status: read`）的类别、国家、性别分布
   - 评分（`rating`）高的书的共同特征
   - 尚未读的书（`status: unread`）中是否有匹配的
2. 先检查书库内未读书目，若有强匹配则优先推荐库内书
3. 再给出 2–3 本外部推荐，说明推荐理由
4. 输出格式：

```
Based on your library (16 books · mostly Memoir & Self-Help · UK/Australia authors)

From your unread shelf:
  → "Big Feelings" — matches your psychology interest, highly rated in similar taste profiles

External picks:
  → "Crying in H Mart" by Michelle Zauner — Memoir, cross-cultural identity, similar to "From Scratch"
  → "The Body Keeps the Score" by Bessel van der Kolk — Psychology, complements your self-help reads
  → "Just Kids" by Patti Smith — Memoir, female author, artistic and introspective tone

Run /books scan to add any of these when you get a copy.
```

**注意**：推荐基于书库数据分析，不调用外部 API，全部由 Claude 根据书目模式判断。

---

## 命令：/books search \<关键词\>

**作用**：搜索书库，支持书名、作者、类别、描述、国家的模糊匹配。

**执行步骤**：
1. 读取 `my_lovely_library/books.json`
2. 在 title、author、category、description、country 字段做关键词匹配（不区分大小写）
3. 输出 Markdown 表格；无结果提示 `No books found matching "<keyword>"`

---

## 命令：/books stats \[主题\]

**作用**：生成交互式统计报表，输出自包含的 `my_lovely_library/books-report.html`，支持四种视觉风格。

**第一步：确认主题**

若用户未在命令中指定主题，展示以下选项：

```
Choose a dashboard theme:

  1. Classic  — warm beige, steel blue     (default)
  2. Dark     — deep navy, amber
  3. Warm     — ivory, terracotta
  4. Ocean    — soft teal, dark slate

Which theme? (1–4, or press Enter for Classic)
```

记录用户选择，在 Python 脚本中填入对应的 THEME 值（`classic` / `dark` / `warm` / `ocean`）。

**第二步：执行 Python 脚本**

将以下完整脚本写入 `/tmp/books_stats.py`，将 `THEME = "classic"` 替换为用户选择的主题，然后用 Bash 执行 `python3 /tmp/books_stats.py`，完成后删除临时文件。

```python
import json, base64, os, subprocess, tempfile
from collections import Counter
from datetime import date

BOOKS_JSON = "my_lovely_library/books.json"
OUTPUT     = "my_lovely_library/books-report.html"
THEME      = "classic"   # claude fills: classic | dark | warm | ocean

THEMES = {
    "classic": {
        "bg":"#f5f3ef","card":"#ffffff","border":"#e8e4de",
        "accent":"#4A7B9D","accent2":"#C4955A","text":"#2c2c2c",
        "muted":"#999999","hdr_bg":"#2c2c2c","hdr_fg":"#f5f3ef",
        "row_hover":"#faf8f5","tag_cat_bg":"#e8f0e8","tag_cat_fg":"#4a7a4a",
        "tag_f_bg":"#fce8ec","tag_f_fg":"#9a3a4a",
        "tag_m_bg":"#e8eef8","tag_m_fg":"#3a4a8a",
        "chart_colors":["#4A7B9D","#C4955A","#7A8C6E","#8B4513","#8B7AAF","#6B8F71","#B8775A","#5A6B8B"],
    },
    "dark": {
        "bg":"#0f172a","card":"#1e293b","border":"#334155",
        "accent":"#e2b96f","accent2":"#7dd3fc","text":"#e2e8f0",
        "muted":"#64748b","hdr_bg":"#020617","hdr_fg":"#e2e8f0",
        "row_hover":"#263348","tag_cat_bg":"#1e3a2f","tag_cat_fg":"#6ee7b7",
        "tag_f_bg":"#3b1a2a","tag_f_fg":"#f9a8d4",
        "tag_m_bg":"#1a2a3b","tag_m_fg":"#93c5fd",
        "chart_colors":["#e2b96f","#7dd3fc","#6ee7b7","#f9a8d4","#c4b5fd","#fca5a5","#fdba74","#a3e635"],
    },
    "warm": {
        "bg":"#fdf6ee","card":"#fffaf4","border":"#e8d8c4",
        "accent":"#c4522a","accent2":"#8a6a3a","text":"#3a2010",
        "muted":"#a08060","hdr_bg":"#3a2010","hdr_fg":"#fdf6ee",
        "row_hover":"#fef9f2","tag_cat_bg":"#f0e8d8","tag_cat_fg":"#7a5a2a",
        "tag_f_bg":"#fce8e0","tag_f_fg":"#c4522a",
        "tag_m_bg":"#e8e0d8","tag_m_fg":"#6a5040",
        "chart_colors":["#c4522a","#8a6a3a","#b87a3a","#7a4a2a","#d4824a","#6a5030","#e0a060","#5a4028"],
    },
    "ocean": {
        "bg":"#f0f7f7","card":"#ffffff","border":"#b8d8d8",
        "accent":"#2a7f7f","accent2":"#4a9a8a","text":"#1a3a3a",
        "muted":"#6a9a9a","hdr_bg":"#1a3a3a","hdr_fg":"#e8f5f5",
        "row_hover":"#e8f5f5","tag_cat_bg":"#d8eeee","tag_cat_fg":"#2a6a6a",
        "tag_f_bg":"#f0e8f0","tag_f_fg":"#7a4a7a",
        "tag_m_bg":"#d8eaf0","tag_m_fg":"#2a5a7a",
        "chart_colors":["#2a7f7f","#4a9a8a","#6abaaa","#3a6a8a","#5a8aaa","#8acaca","#2a5a6a","#7aaaaa"],
    },
}

t = THEMES.get(THEME, THEMES["classic"])

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

total       = len(books)
female      = sum(1 for b in books if b.get("author_gender") in ("女","F"))
pct_f       = round(female/total*100) if total else 0
n_countries = len({b.get("country") for b in books if b.get("country")})
n_cats      = len({b.get("category") for b in books if b.get("category")})
updated     = date.today().isoformat()

country_cnt = Counter(b.get("country","") for b in books if b.get("country"))
cat_cnt     = Counter(b.get("category","") for b in books if b.get("category"))
gender_cnt  = Counter(b.get("author_gender","") for b in books if b.get("author_gender"))
COLORS      = t["chart_colors"]

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

CSS = f"""
*{{box-sizing:border-box;margin:0;padding:0}}
body{{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;background:{t['bg']};color:{t['text']};font-size:14px}}
header{{background:{t['hdr_bg']};color:{t['hdr_fg']};padding:20px 32px}}
header .sub{{font-size:11px;opacity:.5;letter-spacing:1px;text-transform:uppercase;margin-bottom:4px}}
header h1{{font-size:22px;font-weight:600}}
.toolbar{{padding:14px 32px;display:flex;align-items:center;gap:10px;flex-wrap:wrap;background:{t['card']};border-bottom:1px solid {t['border']}}}
#search{{padding:8px 12px;border:1px solid {t['border']};border-radius:6px;font-size:13px;width:240px;outline:none;background:{t['card']};color:{t['text']}}}
#search:focus{{border-color:{t['accent']}}}
.btn-print{{padding:7px 14px;background:{t['accent']};color:{t['hdr_fg']};border:none;border-radius:6px;font-size:12px;cursor:pointer;margin-left:auto}}
.btn-print:hover{{opacity:.85}}
.chips{{display:flex;gap:6px;flex-wrap:wrap}}
.chip{{background:{t['accent']};color:{t['hdr_fg']};padding:3px 10px;border-radius:20px;font-size:12px;cursor:pointer}}
.chip:hover{{opacity:.85}}
.stats{{display:grid;grid-template-columns:repeat(4,1fr);gap:1px;background:{t['border']};margin:24px 32px 0}}
.stat{{background:{t['card']};padding:16px 20px;text-align:center}}
.stat .n{{font-size:30px;font-weight:700;color:{t['accent']}}}
.stat .l{{font-size:11px;color:{t['muted']};margin-top:2px;text-transform:uppercase;letter-spacing:.5px}}
.charts{{display:grid;grid-template-columns:repeat(3,1fr);gap:16px;padding:20px 32px}}
.chart-box{{background:{t['card']};border-radius:8px;padding:16px;border:1px solid {t['border']}}}
.chart-box h3{{font-size:11px;font-weight:600;text-transform:uppercase;letter-spacing:.8px;color:{t['muted']};margin-bottom:10px}}
canvas{{cursor:pointer;display:block;max-width:100%}}
.count{{padding:6px 32px 10px;font-size:12px;color:{t['muted']}}}
.table-wrap{{padding:0 32px 40px;overflow-x:auto}}
table{{width:100%;border-collapse:collapse;background:{t['card']};border-radius:8px;overflow:hidden;border:1px solid {t['border']}}}
th{{background:{t['bg']};font-size:11px;font-weight:600;text-transform:uppercase;letter-spacing:.5px;color:{t['muted']};padding:10px 12px;text-align:left;cursor:pointer;user-select:none;white-space:nowrap}}
th:hover{{opacity:.8}}
td{{padding:8px 12px;border-top:1px solid {t['border']};vertical-align:middle}}
.tc{{width:52px;padding:6px 8px}}
.tc img,.no-img{{width:44px;height:60px;object-fit:cover;border-radius:3px;display:block;background:{t['border']}}}
.tt{{font-weight:500;max-width:220px}}
.ta{{max-width:180px;color:{t['muted']};font-size:13px}}
.bio{{color:{t['muted']};font-size:11px;opacity:.7}}
.ty{{text-align:center;color:{t['muted']};white-space:nowrap}}
.tag{{font-size:11px;padding:2px 8px;border-radius:10px;display:inline-block;cursor:pointer;white-space:nowrap}}
.tcat{{background:{t['tag_cat_bg']};color:{t['tag_cat_fg']}}}
.tf{{background:{t['tag_f_bg']};color:{t['tag_f_fg']}}}
.tm{{background:{t['tag_m_bg']};color:{t['tag_m_fg']}}}
.tst{{background:{t['border']};color:{t['muted']};cursor:default}}
tr.hidden{{display:none}}
tr:hover td{{background:{t['row_hover']}}}
footer{{text-align:center;padding:20px;color:{t['muted']};font-size:11px}}
@media print{{
  .toolbar,.charts,.count,footer{{display:none}}
  .stats{{margin:0 0 16px}}
  .table-wrap{{padding:0}}
  header{{padding:12px 16px}}
  table{{border:1px solid #ccc;font-size:12px}}
  th,td{{padding:6px 8px}}
  .tc img,.no-img{{width:32px;height:44px}}
  tr.hidden{{display:none!important}}
  body{{background:#fff;color:#000}}
}}
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
    ctx.globalAlpha=dim?0.2:1;
    ctx.fillStyle=d.col; ctx.fillRect(pl,y,bw,rh);
    ctx.fillStyle=getComputedStyle(document.body).color;
    ctx.font='11px -apple-system,sans-serif';
    ctx.textAlign='right'; ctx.fillText(d.label,pl-5,y+rh/2+4);
    ctx.globalAlpha=dim?0.4:0.7; ctx.textAlign='left';
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
    ctx.globalAlpha=dim?0.2:1;
    ctx.beginPath(); ctx.moveTo(cx,cy);
    ctx.arc(cx,cy,r,ang,ang+sl); ctx.closePath();
    ctx.fillStyle=d.col; ctx.fill();
    ctx.strokeStyle=document.body.style.background||'#f5f3ef';
    ctx.lineWidth=2; ctx.stroke();
    ang+=sl; ctx.globalAlpha=1;
  });
  const lx=W*0.68;
  data.forEach((d,i)=>{
    const ly=H/2-data.length*14+i*28;
    ctx.fillStyle=d.col; ctx.fillRect(lx,ly,11,11);
    ctx.fillStyle=getComputedStyle(document.body).color;
    ctx.font='11px -apple-system,sans-serif';
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
document.getElementById('btn-print').addEventListener('click',()=>window.print());

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
    f"<header><div class='sub'>Personal Library · {THEME.title()} theme</div>"
    f"<h1>My Books &nbsp;·&nbsp; {total} titles</h1></header>"
    "<div class='toolbar'>"
    "<input id='search' type='search' placeholder='Search title, author, country…'>"
    "<div class='chips' id='chips'></div>"
    "<button class='btn-print' id='btn-print'>⎙ Print / PDF</button>"
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
    f"<footer>Generated {updated} · {THEME.title()} theme</footer>"
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
print(f"Done: {OUTPUT} ({sz:,} bytes, {total} books, theme={THEME})")
```

**完成后告知**：

```
✓ Dashboard generated: my_lovely_library/books-report.html
  N books · theme: Classic
  Print / PDF: click the "⎙ Print / PDF" button in the browser

Open my_lovely_library/books-report.html in a browser.
```

---

## 命令：/books present \[主题\]

**作用**：根据 `my_lovely_library/books.json` 生成一份交互式 HTML 幻灯片演示，数据来自真实书库，支持四种视觉风格。输出文件 `my_lovely_library/books-present.html`。

**第一步：确认主题**

若用户未指定主题，展示以下选项（与 `/books stats` 一致）：

```
Choose a presentation theme:

  1. Classic  — warm beige, steel blue     (default)
  2. Dark     — deep navy, amber
  3. Warm     — ivory, terracotta
  4. Ocean    — soft teal, dark slate

Which theme? (1–4, or press Enter for Classic)
```

**第二步：执行 Python 脚本**

将以下完整脚本写入 `/tmp/books_present.py`，将 `THEME = "classic"` 替换为用户选择的主题，然后执行 `python3 /tmp/books_present.py`，完成后删除临时文件。

```python
import json, os
from collections import Counter
from datetime import date

BOOKS_JSON = "my_lovely_library/books.json"
OUTPUT     = "my_lovely_library/books-present.html"
THEME      = "classic"   # claude fills: classic | dark | warm | ocean

THEMES = {
    "classic": {
        "bg_dark":    "#1e2d3d", "bg_light":  "#f8f5ef",
        "accent":     "#4A7B9D", "accent_s":  "#b8ccd8",
        "text_dark":  "#ede8dd", "text_light": "#1e2d3d",
        "muted_dark": "rgba(184,204,216,0.55)", "muted_light": "rgba(30,45,61,0.45)",
        "rule_dark":  "rgba(74,123,157,0.32)",  "rule_light":  "rgba(74,123,157,0.18)",
        "dot_on":     "#4A7B9D", "dot_off":   "rgba(74,123,157,0.3)",
    },
    "dark": {
        "bg_dark":    "#0f172a", "bg_light":  "#1e293b",
        "accent":     "#e2b96f", "accent_s":  "#cbd5e1",
        "text_dark":  "#f1f5f9", "text_light": "#f1f5f9",
        "muted_dark": "rgba(203,213,225,0.5)", "muted_light": "rgba(203,213,225,0.5)",
        "rule_dark":  "rgba(226,185,111,0.25)", "rule_light": "rgba(226,185,111,0.2)",
        "dot_on":     "#e2b96f", "dot_off":   "rgba(226,185,111,0.3)",
    },
    "warm": {
        "bg_dark":    "#2c1810", "bg_light":  "#fdf6ee",
        "accent":     "#c4522a", "accent_s":  "#c8a882",
        "text_dark":  "#f5e8d8", "text_light": "#2c1810",
        "muted_dark": "rgba(200,168,130,0.55)", "muted_light": "rgba(44,24,16,0.45)",
        "rule_dark":  "rgba(196,82,42,0.3)",  "rule_light":  "rgba(196,82,42,0.18)",
        "dot_on":     "#c4522a", "dot_off":   "rgba(196,82,42,0.3)",
    },
    "ocean": {
        "bg_dark":    "#0d2b2b", "bg_light":  "#f0f7f7",
        "accent":     "#2a7f7f", "accent_s":  "#a0c8c8",
        "text_dark":  "#e8f4f4", "text_light": "#0d2b2b",
        "muted_dark": "rgba(160,200,200,0.55)", "muted_light": "rgba(13,43,43,0.45)",
        "rule_dark":  "rgba(42,127,127,0.3)",  "rule_light":  "rgba(42,127,127,0.18)",
        "dot_on":     "#2a7f7f", "dot_off":   "rgba(42,127,127,0.3)",
    },
}

with open(BOOKS_JSON, encoding="utf-8") as f:
    books = json.load(f)

t = THEMES[THEME]
today = date.today()
this_year = str(today.year)

total       = len(books)
reading_bks = [b for b in books if b.get("status") == "reading"]
read_bks    = [b for b in books if b.get("status") == "read"]
unread_bks  = [b for b in books if b.get("status") == "unread"]
rated_bks   = [b for b in books if b.get("rating") is not None]
top_rated   = sorted(rated_bks, key=lambda b: b.get("rating") or 0, reverse=True)

cats      = Counter(b["category"] for b in books if b.get("category"))
countries = Counter(b["country"]  for b in books if b.get("country"))
female_pct = round(sum(1 for b in books if b.get("author_gender") in ("F","女")) / total * 100) if total else 0
top_cat     = cats.most_common(1)[0][0] if cats else ""
top_country = countries.most_common(1)[0][0] if countries else ""

def esc(s):
    return str(s).replace("&","&amp;").replace("<","&lt;").replace(">","&gt;")

def eyebrow(label, dark):
    rc = t["rule_dark"] if dark else t["rule_light"]
    tc = t["accent_s"]  if dark else t["accent"]
    return ('<div class="ey" style="color:' + tc + '">'
            '<span class="rule" style="background:' + rc + '"></span>'
            + esc(label) + '</div>')

def bar(val, mx, color):
    pct = round(val / mx * 100) if mx else 0
    return ('<div class="bar-wrap"><div class="bar-fill" style="width:' + str(pct)
            + '%;background:' + color + '"></div></div>')

slides = []

# 1 · Cover
slides.append(
    '<div class="slide active" style="background:' + t["bg_dark"] + ';align-items:center;text-align:center;">'
    + '<div class="cover-big" style="color:' + t["accent_s"] + '">' + str(total) + '</div>'
    + '<div class="cover-unit" style="color:' + t["muted_dark"] + '">books in my library</div>'
    + '<div class="cover-rule" style="background:' + t["rule_dark"] + '"></div>'
    + '<div class="cover-title" style="color:' + t["text_dark"] + '">My Library</div>'
    + '<div class="cover-sub" style="color:' + t["muted_dark"] + '">'
    + esc(top_cat) + '  ·  ' + esc(top_country) + '  ·  ' + this_year + '</div>'
    + '</div>'
)

# 2 · By the Numbers
def scard(num, label):
    return ('<div class="stat-card" style="border-color:' + t["rule_light"] + '">'
            + '<div class="stat-num" style="color:' + t["accent"] + '">' + esc(str(num)) + '</div>'
            + '<div class="stat-label" style="color:' + t["muted_light"] + '">' + esc(label) + '</div>'
            + '</div>')

slides.append(
    '<div class="slide" style="background:' + t["bg_light"] + '">'
    + eyebrow("By the Numbers", False)
    + '<p class="s-head" style="color:' + t["text_light"] + '">What the shelf looks like.</p>'
    + '<div class="stat-grid">'
    + scard(total, "Books")
    + scard(len(countries), "Countries")
    + scard(str(female_pct) + "%", "Female Authors")
    + scard(len(cats), "Categories")
    + '</div></div>'
)

# 3 · Reading Now (conditional)
if reading_bks:
    rows = ""
    for b in reading_bks[:6]:
        rows += ('<div class="list-row" style="border-color:' + t["rule_dark"] + '">'
                 + '<div class="list-title" style="color:' + t["text_dark"] + '">' + esc(b.get("title","")) + '</div>'
                 + '<div class="list-sub" style="color:' + t["muted_dark"] + '">' + esc(b.get("author","")) + '</div>'
                 + '</div>')
    slides.append(
        '<div class="slide" style="background:' + t["bg_dark"] + '">'
        + eyebrow("Reading Now", True)
        + '<p class="s-head" style="color:' + t["text_dark"] + '">'
        + str(len(reading_bks)) + ' book' + ('s' if len(reading_bks) != 1 else '') + ' in progress.</p>'
        + '<div class="list-col">' + rows + '</div></div>'
    )

# 4 · Top Rated (conditional)
if top_rated:
    rows = ""
    for b in top_rated[:6]:
        r = b.get("rating") or 0
        stars = "★" * r + "☆" * (5 - r)
        rows += ('<div class="list-row" style="border-color:' + t["rule_light"] + '">'
                 + '<div class="list-title" style="color:' + t["text_light"] + '">' + esc(b.get("title","")) + '</div>'
                 + '<div style="display:flex;gap:0.7rem;margin-top:0.2rem;">'
                 + '<span style="color:' + t["accent"] + ';font-size:0.85rem">' + stars + '</span>'
                 + '<span class="list-sub" style="color:' + t["muted_light"] + '">' + esc(b.get("author","")) + '</span>'
                 + '</div></div>')
    slides.append(
        '<div class="slide" style="background:' + t["bg_light"] + '">'
        + eyebrow("Top Rated", False)
        + '<p class="s-head" style="color:' + t["text_light"] + '">Highest-rated reads.</p>'
        + '<div class="list-col">' + rows + '</div></div>'
    )

# 5 · By Category
top_cats = cats.most_common(8)
mx = top_cats[0][1] if top_cats else 1
rows = ""
for cat, cnt in top_cats:
    rows += ('<div class="bar-row">'
             + '<div class="bar-label" style="color:' + t["muted_dark"] + '">' + esc(cat) + '</div>'
             + bar(cnt, mx, t["accent"])
             + '<div class="bar-count" style="color:' + t["accent_s"] + '">' + str(cnt) + '</div>'
             + '</div>')
slides.append(
    '<div class="slide" style="background:' + t["bg_dark"] + '">'
    + eyebrow("By Category", True)
    + '<p class="s-head" style="color:' + t["text_dark"] + '">What you reach for most.</p>'
    + '<div class="bar-list">' + rows + '</div></div>'
)

# 6 · Around the World
top_cs = countries.most_common(8)
mx = top_cs[0][1] if top_cs else 1
rows = ""
for country, cnt in top_cs:
    rows += ('<div class="bar-row">'
             + '<div class="bar-label" style="color:' + t["muted_light"] + '">' + esc(country) + '</div>'
             + bar(cnt, mx, t["accent"])
             + '<div class="bar-count" style="color:' + t["accent"] + '">' + str(cnt) + '</div>'
             + '</div>')
slides.append(
    '<div class="slide" style="background:' + t["bg_light"] + '">'
    + eyebrow("Around the World", False)
    + '<p class="s-head" style="color:' + t["text_light"] + '">Your shelf spans '
    + str(len(countries)) + (' country.' if len(countries) == 1 else ' countries.') + '</p>'
    + '<div class="bar-list">' + rows + '</div></div>'
)

# 7 · The Shelf
recent = sorted([b for b in books if b.get("added")], key=lambda b: b["added"], reverse=True)[:5]
rows = ""
for b in recent:
    sc = t["accent_s"] if b.get("status") == "reading" else t["muted_dark"]
    rows += ('<div class="list-row" style="border-color:' + t["rule_dark"] + '">'
             + '<div class="list-title" style="color:' + t["text_dark"] + '">' + esc(b.get("title","")) + '</div>'
             + '<div class="list-sub" style="color:' + sc + '">'
             + esc(b.get("status","")) + '  ·  ' + esc(b.get("added","")) + '</div>'
             + '</div>')
slides.append(
    '<div class="slide" style="background:' + t["bg_dark"] + '">'
    + eyebrow("The Shelf", True)
    + '<div style="display:flex;gap:4vw;align-items:baseline;margin-bottom:2rem;">'
    + '<div><div class="stat-num" style="color:' + t["accent_s"] + ';font-size:clamp(3rem,8vw,7rem);line-height:1">'
    + str(len(unread_bks)) + '</div><div class="stat-label" style="color:' + t["muted_dark"] + '">unread</div></div>'
    + '<div><div class="stat-num" style="color:' + t["accent_s"] + ';font-size:clamp(3rem,8vw,7rem);line-height:1">'
    + str(len(read_bks)) + '</div><div class="stat-label" style="color:' + t["muted_dark"] + '">finished</div></div>'
    + '</div>'
    + '<div class="list-label" style="color:' + t["muted_dark"] + '">Recently added</div>'
    + '<div class="list-col">' + rows + '</div></div>'
)

n = len(slides)
dot_on  = t["dot_on"]
dot_off = t["dot_off"]

css = (
    "*, *::before, *::after { box-sizing:border-box; margin:0; padding:0; }"
    "body { font-family:'Optima',Candara,'Gill Sans','Segoe UI',sans-serif; overflow:hidden; height:100vh; width:100vw; }"
    ".deck { position:relative; width:100vw; height:100vh; }"
    ".slide { position:absolute; inset:0; display:flex; flex-direction:column; justify-content:center;"
    "  padding:6vh 8vw; opacity:0; pointer-events:none; transition:opacity 0.45s ease; }"
    ".slide.active { opacity:1; pointer-events:auto; }"
    ".ey { display:flex; flex-direction:column; font-size:0.62rem; letter-spacing:0.22em;"
    "  text-transform:uppercase; margin-bottom:2rem; }"
    ".rule { display:block; width:2.4rem; height:1px; margin-bottom:0.55rem; }"
    ".s-head { font-family:'Palatino Linotype',Palatino,'Book Antiqua',serif;"
    "  font-size:clamp(1.4rem,3.5vw,2.6rem); font-weight:400; margin-bottom:2rem;"
    "  text-wrap:balance; max-width:32ch; line-height:1.15; }"
    ".cover-big { font-family:'Palatino Linotype',Palatino,'Book Antiqua',serif;"
    "  font-size:clamp(6rem,18vw,16rem); font-weight:400; line-height:0.85; letter-spacing:-0.02em;"
    "  animation:riseIn 0.9s cubic-bezier(0.16,1,0.3,1) both; }"
    ".cover-unit { font-size:clamp(0.7rem,1.4vw,0.9rem); letter-spacing:0.18em; text-transform:uppercase;"
    "  margin-top:0.5rem; animation:riseIn 0.9s 0.1s cubic-bezier(0.16,1,0.3,1) both; }"
    ".cover-rule { width:3rem; height:1px; margin:2rem auto;"
    "  animation:riseIn 0.9s 0.18s cubic-bezier(0.16,1,0.3,1) both; }"
    ".cover-title { font-family:'Palatino Linotype',Palatino,'Book Antiqua',serif;"
    "  font-size:clamp(1.4rem,3.5vw,2.8rem); font-weight:400;"
    "  animation:riseIn 0.9s 0.24s cubic-bezier(0.16,1,0.3,1) both; }"
    ".cover-sub { font-size:clamp(0.65rem,1.3vw,0.85rem); letter-spacing:0.14em; text-transform:uppercase;"
    "  margin-top:0.75rem; animation:riseIn 0.9s 0.32s cubic-bezier(0.16,1,0.3,1) both; }"
    "@keyframes riseIn { from{opacity:0;transform:translateY(1.5rem)} to{opacity:1;transform:translateY(0)} }"
    ".stat-grid { display:grid; grid-template-columns:repeat(4,1fr); gap:1.5vw; max-width:860px; }"
    ".stat-card { border:1px solid; border-radius:2px; padding:1.4rem 1.2rem; }"
    ".stat-num { font-family:'Palatino Linotype',Palatino,'Book Antiqua',serif;"
    "  font-size:clamp(2.2rem,5vw,4rem); font-weight:400; line-height:1; margin-bottom:0.5rem;"
    "  font-variant-numeric:tabular-nums; }"
    ".stat-label { font-size:clamp(0.6rem,1.1vw,0.75rem); letter-spacing:0.14em; text-transform:uppercase; }"
    ".list-col { display:flex; flex-direction:column; max-width:640px; }"
    ".list-row { border-top:1px solid; padding:0.65rem 0; }"
    ".list-title { font-family:'Palatino Linotype',Palatino,'Book Antiqua',serif;"
    "  font-size:clamp(0.85rem,1.8vw,1.2rem); font-weight:400; }"
    ".list-sub { font-size:clamp(0.65rem,1.2vw,0.8rem); letter-spacing:0.05em; margin-top:0.15rem; }"
    ".list-label { font-size:0.62rem; letter-spacing:0.18em; text-transform:uppercase; margin-bottom:0.75rem; }"
    ".bar-list { display:flex; flex-direction:column; gap:0.65rem; max-width:680px; width:100%; }"
    ".bar-row { display:grid; grid-template-columns:10rem 1fr 2.5rem; gap:1rem; align-items:center; }"
    ".bar-label { font-size:clamp(0.68rem,1.2vw,0.82rem); text-align:right; line-height:1.3; }"
    ".bar-wrap { height:6px; background:rgba(128,128,128,0.12); border-radius:1px; overflow:hidden; }"
    ".bar-fill { height:100%; border-radius:1px; }"
    ".bar-count { font-size:0.7rem; font-variant-numeric:tabular-nums; }"
    ".counter { position:fixed; top:1.4rem; right:2rem; font-size:0.68rem; letter-spacing:0.14em;"
    "  z-index:200; font-variant-numeric:tabular-nums; padding:0.28rem 0.6rem; border-radius:0.2rem;"
    "  background:rgba(0,0,0,0.35); color:rgba(255,255,255,0.6); }"
    ".dots { position:fixed; bottom:1.6rem; left:50%; transform:translateX(-50%);"
    "  display:flex; gap:0.45rem; z-index:200;"
    "  background:rgba(0,0,0,0.35); padding:0.45rem 0.8rem; border-radius:2rem; }"
    ".dot { width:5px; height:5px; border-radius:50%; cursor:pointer;"
    "  transition:background 0.3s,transform 0.25s; border:none; }"
    ".dot.active { transform:scale(1.45); }"
    ".btn-p,.btn-n { position:fixed; top:50%; transform:translateY(-50%);"
    "  background:rgba(0,0,0,0.3); border:none; cursor:pointer; padding:0.9rem 0.6rem;"
    "  color:rgba(255,255,255,0.4); z-index:200; transition:color 0.2s,background 0.2s; border-radius:2px; }"
    ".btn-p:hover,.btn-n:hover { color:rgba(255,255,255,0.85); background:rgba(0,0,0,0.55); }"
    ".btn-p { left:0.6rem; } .btn-n { right:0.6rem; }"
)

js = (
    "(function(){"
    "var sl=document.querySelectorAll('.slide'),"
    "ct=document.getElementById('ct'),"
    "de=document.getElementById('de'),"
    "total=sl.length,cur=0;"
    "sl.forEach(function(_,i){"
    "var b=document.createElement('button');b.className='dot'+(i===0?' active':'');"
    "b.style.background=i===0?'" + dot_on + "':'" + dot_off + "';"
    "b.setAttribute('aria-label','Slide '+(i+1));"
    "b.addEventListener('click',function(){go(i);});de.appendChild(b);});"
    "function go(n){"
    "sl[cur].classList.remove('active');de.children[cur].style.background='" + dot_off + "';"
    "de.children[cur].classList.remove('active');"
    "cur=((n%total)+total)%total;"
    "sl[cur].classList.add('active');de.children[cur].style.background='" + dot_on + "';"
    "de.children[cur].classList.add('active');"
    "ct.textContent=(cur+1)+' / '+total;}"
    "document.getElementById('bp').addEventListener('click',function(){go(cur-1);});"
    "document.getElementById('bn').addEventListener('click',function(){go(cur+1);});"
    "document.addEventListener('keydown',function(e){"
    "if(e.key==='ArrowRight'||e.key==='ArrowDown'||e.key===' '){e.preventDefault();go(cur+1);}"
    "if(e.key==='ArrowLeft'||e.key==='ArrowUp'){e.preventDefault();go(cur-1);}});"
    "var tx=null;"
    "document.addEventListener('touchstart',function(e){tx=e.touches[0].clientX;},{passive:true});"
    "document.addEventListener('touchend',function(e){"
    "if(tx===null)return;var dx=e.changedTouches[0].clientX-tx;"
    "if(Math.abs(dx)>40)go(dx<0?cur+1:cur-1);tx=null;});"
    "}());"
)

svg_prev = "<svg width='20' height='20' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='1.8' stroke-linecap='round'><polyline points='15 18 9 12 15 6'/></svg>"
svg_next = "<svg width='20' height='20' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='1.8' stroke-linecap='round'><polyline points='9 18 15 12 9 6'/></svg>"

html = (
    "<!DOCTYPE html><html lang='en'><head>"
    "<meta charset='UTF-8'><meta name='viewport' content='width=device-width,initial-scale=1'>"
    "<title>My Library — Presentation</title>"
    "<style>" + css + "</style></head><body>"
    "<span class='counter' id='ct'>1 / " + str(n) + "</span>"
    "<div class='deck'>" + "".join(slides) + "</div>"
    "<div class='dots' id='de'></div>"
    "<button class='btn-p' id='bp' aria-label='Previous'>" + svg_prev + "</button>"
    "<button class='btn-n' id='bn' aria-label='Next'>" + svg_next + "</button>"
    "<script>" + js + "</script>"
    "</body></html>"
)

with open(OUTPUT, "w", encoding="utf-8") as f:
    f.write(html)

print("Saved:", OUTPUT, "—", n, "slides")
```

**第三步：输出确认**

```
✓ Presentation saved to my_lovely_library/books-present.html
  Theme:  Classic
  Slides: 7  (Reading Now + Top Rated slides auto-added when data exists)

Open my_lovely_library/books-present.html in your browser.
← → arrow keys or click to navigate · touch swipe supported
```

**幻灯片结构**（根据书库数据动态生成）：

| 幻灯片 | 内容 | 条件 |
|--------|------|------|
| 1 | 封面 — 书库总量、主类别、年份 | 必有 |
| 2 | 统计数字 — 总数、国家数、女性作者%、类别数 | 必有 |
| 3 | 正在读 — 当前阅读中的书单 | 有 `reading` 状态书时 |
| 4 | 最高评分 — 4–5 星书目 | 有 `rating` 数据时 |
| 5 | 按类别 — 横向条形图 | 必有 |
| 6 | 环游世界 — 国家分布条形图 | 必有 |
| 7 | 书架 — 未读/已读数量 + 最新入库 | 必有 |

**示例**：
```
/books present
/books present dark
/books present warm
```

---

## 命令：/books deploy

**作用**：将 `my_lovely_library/books-report.html` 部署到 Vercel，生成可在任何设备访问的公开 URL。

**前置条件**：
- Node.js 已安装（`node -v` 验证）
- 首次运行会自动安装 Vercel CLI 并引导登录

**执行步骤**：

1. 检查 `my_lovely_library/books-report.html` 是否存在，不存在则提示先运行 `/books stats`
2. 检查并安装 Vercel CLI：
```bash
npx vercel --version 2>/dev/null || npm install -g vercel
```
3. 检查登录状态，未登录则引导：
```bash
vercel whoami 2>/dev/null || vercel login
```
4. 创建临时部署目录，将 `my_lovely_library/books-report.html` 复制为 `index.html`：
```bash
mkdir -p /tmp/books-deploy
cp my_lovely_library/books-report.html /tmp/books-deploy/index.html
```
5. 部署（固定项目名，每次覆盖同一 URL）：
```bash
cd /tmp/books-deploy && vercel deploy --yes --prod --name my-books-library
```
6. 清理临时目录：
```bash
rm -rf /tmp/books-deploy
```
7. 输出确认：

```
✓ Deployed: https://my-books-library.vercel.app

Open this URL on any device — phone, iPad, or another computer.
Bookmark it. Run /books deploy again after adding new books to update.
```

**注意**：
- 首次 deploy 约需 30 秒；后续更新约 10 秒
- 部署的是静态快照，加新书后需重新运行 `/books stats` + `/books deploy`
- 若想修改项目名，将 `--name my-books-library` 改为自定义名称（只能用小写字母和连字符）

---

## 命令：/books export json

**作用**：导出干净的 JSON 文件，可导入 Notion、Obsidian 等工具。

**执行步骤**：
1. 读取 `my_lovely_library/books.json`
2. 确保 `my_lovely_library/` 目录存在（`mkdir -p my_lovely_library`）
3. 写出格式化的 `my_lovely_library/books-export.json`（indent=2）
4. 输出确认：

```
✓ Exported: my_lovely_library/books-export.json
  N records
```

---

## 通用原则

- **不中断用户**：除「书名重复」「status/edit 多结果匹配」「stats 主题选择」外，所有命令直接执行，不询问确认
- **识别失败不捏造**：字段识别不确定时留 `null`，并在输出中告知用户
- **幂等性**：重复扫描同一本书跳过，不产生重复数据
- **语言规则**：中文书（书名/作者为中文）保留中文；其余书籍 country / category / description / author_bio 用英文
- **JSON 安全**：每次写入 my_lovely_library/books.json 前，自动将所有文本字段中的中文引号 `"` `"` 替换为直引号 `"`

---

## my_lovely_library/books.md 格式

每次更新 my_lovely_library/books.json 后同步维护：

```markdown
# My Library

| Title | Author | Year | Country | Category | Status | Description |
|-------|--------|------|---------|----------|--------|-------------|
| Being You | Anil Seth | 2021 | UK | Popular Science | unread | Neuroscientist explores consciousness. |
| 南极之南 | 毕淑敏 | — | 中国 | 游记散文 | unread | 毕淑敏记录亲赴南极的旅行见闻。 |
```

状态直接用英文原值（unread / reading / read / paused）。
