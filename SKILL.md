---
name: books
description: 个人书库管理工具——扫描书籍封面照片，AI 识别书名/作者/类别/作者简介等信息，追加到本地书库，支持编辑、搜索、状态管理、统计报表和 JSON 导出。触发词：/books scan、/books scan-folder、/books edit、/books search、/books status、/books stats、/books export。
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

books.json 中每条记录的字段：

```json
{
  "id": "uuid-v4",
  "title": "书名",
  "title_original": "原文书名（外文书才填）",
  "author": "作者姓名",
  "author_gender": "女 | 男 | 机构 | 男女合著 | 未知",
  "year": 2020,
  "country": "国家/地区",
  "category": "类别",
  "description": "一句话内容描述",
  "author_bio": "作者一句话简介（身份背景，非书籍内容）",
  "status": "unread | reading | read | paused",
  "photo": "原始照片文件名",
  "added": "YYYY-MM-DD"
}
```

**类别参考**（不限于此，根据实际内容判断）：
个人理财、自我成长、心理、育儿/心理、文学随笔、游记散文、科普、商业、历史、哲学、设计、技术、小说、儿童读物

---

## 命令：/books scan \<image\>

**作用**：扫描单张书籍照片，识别信息，追加到书库。

**支持格式**：jpg / jpeg / png / heic / webp

**HEIC 自动处理**：若图片为 `.heic` 或 `.HEIC`，先用 `sips` 转换为 jpg（Mac 自带，无需安装）：
```bash
sips -s format jpeg "<input.heic>" --out /tmp/books-scan-tmp.jpg
```
转换后读取 `/tmp/books-scan-tmp.jpg`，完成后删除临时文件。

**执行步骤**：
1. 若为 HEIC 格式，先执行上方转换步骤
2. 用 Claude Vision 读取图片，识别以下字段：
   - 书名（中文 / 外文）
   - 作者
   - 出版年（封面或版权页可见时）
   - 国家/地区（根据作者 + 出版社推断）
   - 类别（根据书名 + 副标题推断）
   - 一句话内容描述（根据书名 + 副标题推断）
   - 作者简介 `author_bio`：一句话说明作者身份背景（非书籍内容），例如「英国神经科学家，萨塞克斯大学教授，研究意识本质。」
3. 读取现有 `books.json`（不存在则创建空数组）
4. 检查是否已有同书名记录 → 若有，告知用户「已存在，跳过」，不重复添加
5. 生成新记录，`status` 默认 `"unread"`，`added` 为今日日期
6. 追加到 `books.json`，同步更新 `books.md`
7. 输出确认：

```
✓ 已添加
  书名：《xxx》
  作者：xxx（女 · 英国）
  作者简介：xxx
  类别：xxx
  出版年：xxxx
  描述：xxx
  状态：未读

书库现有 N 本书。
```

**识别不确定时**：字段留空或标 `null`，不要猜测或捏造信息。告知用户哪些字段未能识别。

---

## 命令：/books scan-folder \<path\>

**作用**：批量扫描指定文件夹内所有图片（jpg / jpeg / png / heic / webp）。

**执行步骤**：
1. 列出文件夹内所有图片文件
2. HEIC 文件先用 `sips` 转为 jpg，识别后删除临时文件
3. 逐张识别（与 `/books scan` 相同逻辑）
4. 跳过已存在的书目（书名匹配）
4. 全部完成后，一次性写入 `books.json` 和 `books.md`
5. 生成 `rename-guide.txt`，格式：

```
原文件名 → 建议文件名
IMG_3291.jpg → 个人理财_Barefoot-Investor_ScottPape.jpg
IMG_3292.jpg → 心理_真希望我父母读过这本书_PhilippaPerry.jpg
```

6. 输出总结：

```
扫描完成
  共识别：N 本
  已跳过（重复）：N 本
  识别失败：N 张（列出文件名）

新增书目：
  · 《xxx》— 作者（类别）
  · 《xxx》— 作者（类别）

重命名建议已保存至 rename-guide.txt
```

---

## 命令：/books status \<书名关键词\> \<状态\>

**作用**：更新某本书的阅读状态。

状态可选值：`unread`（未读）| `reading`（在读）| `read`（已读）| `paused`（暂停）

**执行步骤**：
1. 在 `books.json` 中模糊匹配书名（不区分大小写，支持关键词）
2. 若匹配到唯一结果，直接更新状态
3. 若匹配到多个结果，列出让用户确认编号
4. 更新 `books.json` 和 `books.md`
5. 输出确认：

```
✓ 已更新
  《xxx》→ 在读
```

**示例**：
```
/books status barefoot read
/books status 认知觉醒 reading
```

---

## 命令：/books edit \<书名关键词\> \<字段\> \<新值\>

**作用**：修正书库中某本书的某个字段，不需要重新扫描。

**可编辑字段**：
`title` / `author` / `author_gender` / `year` / `country` / `category` / `description` / `author_bio` / `status` / `photo`

**执行步骤**：
1. 在 `books.json` 中模糊匹配书名
2. 若匹配到唯一结果，显示当前字段值，更新并确认
3. 若匹配到多个结果，列出让用户选编号
4. 更新 `books.json` 和 `books.md`
5. 输出确认：

```
✓ 已更新
  《Top End Girl》
  year: null → 2021
```

**示例**：
```
/books edit "Top End Girl" year 2021
/books edit barefoot author_bio 澳洲理财顾问，以水桶理财法闻名全国
/books edit "Big Feelings" category 心理
/books edit 南极 description 毕淑敏亲赴南极的旅行散文，记录极地风光与人生思考
```

**注意**：
- description 和 author_bio 字段禁止使用中文引号，使用直引号或不加引号，防止 JSON 损坏
- year 字段传入整数，无出版年传 `null`
- 编辑不会触发重新识别，只更新指定字段

---

## 命令：/books search \<关键词\>

**作用**：搜索书库，支持书名、作者、类别、描述的模糊匹配。

**执行步骤**：
1. 读取 `books.json`
2. 在 title、author、category、description 字段中做关键词匹配（不区分大小写）
3. 输出 Markdown 表格，按相关度排序：

```
搜索「理财」，找到 N 本：

| 书名 | 作者 | 类别 | 状态 |
|------|------|------|------|
| 《The Barefoot Investor》| Scott Pape | 个人理财 | 已读 |
| 《财务自由之路》| 博多·舍费尔 | 个人理财 | 未读 |
```

4. 若无结果，提示「书库中没有找到相关书目」

---

## 命令：/books stats

**作用**：生成交互式统计报表，输出自包含的 `books-report.html`。

**报表结构**

顶部控制栏：
- 实时搜索框：跨书名、作者、国家、类别同时模糊匹配，结果即时过滤
- 活跃筛选标签：显示当前生效的所有筛选条件，每个单独 ✕ 清除，多个时出现「清除全部」

统计数字（4格）：
- 书库总量
- 女性作者占比（%）
- 涵盖国家/地区数
- 书籍类别数

交互图表（3列，Canvas 绘制）：
- **国家/地区分布**横向条形图 — 点击任意条形筛选该国书籍；再次点击取消
- **作者性别占比**饼图 — 点击扇形筛选女/男作者书籍；再次点击取消
- **类别分布**横向条形图 — 点击任意条形筛选该类别；再次点击取消
- 选中项高亮，未选中项变淡，视觉反馈明确

书目表格：
- 第一列嵌入**封面缩略图**（base64 内嵌，无需外部路径）
- 列字段：封面 / 书名 / 作者 / 出版年 / 国家 / 类别 / 性别 / 状态
- 点击列标题升序/降序排列，排序与筛选同时生效
- 点击类别标签（绿色）→ 按类别筛选
- 点击性别标签（红/蓝）→ 按性别筛选
- 筛选后显示「N / 总数」计数

**封面图片处理**：
- 每张书籍照片用 `sips` 缩放到 200px 宽后转 base64 内嵌
- HEIC 先转 jpg，临时文件用后删除
- 图片以 `object-fit: cover` 裁剪为统一卡片尺寸（48×65px）

**执行方式**：运行以下 Python 脚本生成报表（用 Bash 工具执行，路径替换为当前工作目录）。

```python
import json, base64, os, subprocess, tempfile

BOOKS_JSON = "books.json"   # 当前目录
OUTPUT    = "books-report.html"
IMG_DIR   = ""              # 照片所在目录（与 photo 字段拼接）

with open(BOOKS_JSON) as f:
    books = json.load(f)

status_map = {"unread":"未读","reading":"在读","read":"已读","paused":"暂停"}

def encode_photo(photo):
    if not photo:
        return None
    src = os.path.join(IMG_DIR, photo) if IMG_DIR else photo
    if not os.path.exists(src):
        return None
    tmp = tempfile.mktemp(suffix=".jpg")
    try:
        subprocess.run(["sips","-s","format","jpeg",src,"--out",tmp,"-Z","200"],
                       capture_output=True)
        with open(tmp,"rb") as f:
            return base64.b64encode(f.read()).decode()
    finally:
        if os.path.exists(tmp): os.remove(tmp)

# Stats
total   = len(books)
female  = sum(1 for b in books if b.get("author_gender") == "女")
pct_f   = round(female/total*100) if total else 0
countries = len({b.get("country") for b in books if b.get("country")})
cats    = len({b.get("category") for b in books if b.get("category")})
status_counts = {s: sum(1 for b in books if b.get("status")==s) for s in ["read","reading","unread","paused"]}

# Chart data
from collections import Counter
country_counts = Counter(b.get("country","") for b in books if b.get("country"))
cat_counts     = Counter(b.get("category","") for b in books if b.get("category"))
gender_counts  = Counter(b.get("author_gender","") for b in books if b.get("author_gender"))

country_data = [{"label":k,"val":v} for k,v in country_counts.most_common()]
cat_data     = [{"label":k,"val":v} for k,v in cat_counts.most_common()]
gender_data  = [{"label":k,"val":v} for k,v in gender_counts.most_common()]

COLORS = ["#8B4513","#C4955A","#7A8C6E","#4A7B9D","#8B7AAF","#6B8F71","#B8775A","#5A6B8B"]

def js_arr(data):
    return json.dumps([{**d,"col":COLORS[i%len(COLORS)]} for i,d in enumerate(data)], ensure_ascii=False)

# Table rows
rows = ""
for b in books:
    b64 = encode_photo(b.get("photo",""))
    img = f'<img src="data:image/jpeg;base64,{b64}" alt="">' if b64 else '<div class="no-img"></div>'
    gcls = "tag-f" if b.get("author_gender")=="女" else "tag-m"
    bio  = b.get("author_bio","") or ""
    rows += f'''<tr data-country="{b.get("country","")}" data-cat="{b.get("category","")}" data-gender="{b.get("author_gender","")}" data-title="{b.get("title","")}" data-author="{b.get("author","")}">
  <td class="td-cover" title="{bio}">{img}</td>
  <td class="td-title">{b.get("title","")}</td>
  <td class="td-author">{b.get("author","")}<br><span class="bio">{bio}</span></td>
  <td class="td-year">{b.get("year") or "—"}</td>
  <td>{b.get("country","")}</td>
  <td><span class="tag tag-cat">{b.get("category","")}</span></td>
  <td><span class="tag {gcls}">{b.get("author_gender","")}</span></td>
  <td><span class="tag tag-status">{status_map.get(b.get("status","unread"),"未读")}</span></td>
</tr>'''

# [HTML template — Claude fills stats, js_arr(country_data), js_arr(cat_data), js_arr(gender_data), rows]
# Full HTML is identical to the reference implementation in books-report.html.
# Key variable substitutions:
#   {total}, {pct_f}, {countries}, {cats}
#   {js_arr(country_data)}, {js_arr(gender_data)}, {js_arr(cat_data)}
#   {rows}
html = REPORT_TEMPLATE.format(
    total=total, pct_f=pct_f, countries=countries, cats=cats,
    country_data=js_arr(country_data),
    gender_data=js_arr(gender_data),
    cat_data=js_arr(cat_data),
    rows=rows,
    updated="2026-07-07"
)

with open(OUTPUT,"w") as f:
    f.write(html)
print(f"Done: {OUTPUT} ({os.path.getsize(OUTPUT)} bytes)")
```

`REPORT_TEMPLATE` 是完整 HTML 字符串，其中图表、搜索框、筛选逻辑均已固化。Claude 执行此脚本时直接复用，不重新写 HTML/JS 逻辑。

**完成后告知**：
```
✓ 统计报表已生成：books-report.html
  共 N 本书 · 已读 N · 在读 N · 未读 N
  涵盖 N 个国家/地区 · 女性作者 N%

用浏览器打开 books-report.html 查看完整交互报表。
支持：点击图表筛选 · 搜索框过滤 · 列排序
```

---

## 命令：/books export json

**作用**：导出干净的 JSON 文件，可导入其他工具（Notion、Obsidian 数据库等）。

**执行步骤**：
1. 读取 `books.json`
2. 输出格式化的 `books-export.json`（与 books.json 格式相同，美化缩进）
3. 输出确认：

```
✓ 已导出：books-export.json
  共 N 条记录
```

---

## 命令：/books search \<关键词\>

**作用**：搜索书库，支持书名、作者、类别、描述、国家的模糊匹配。

**执行步骤**：
1. 读取 `books.json`
2. 在 title、author、category、description、country 字段中做关键词匹配（不区分大小写）
3. 输出 Markdown 表格，按书库原顺序：

```
搜索「回忆录」，找到 6 本：

| 书名 | 作者 | 国家 | 类别 | 状态 |
|------|------|------|------|------|
| Top End Girl | Miranda Tapsell | 澳大利亚 | 回忆录 | 未读 |
```

4. 若无结果，提示「书库中没有找到相关书目」

**示例**：
```
/books search 英国        # 按国家搜
/books search 回忆录      # 按类别搜
/books search Annie Duke  # 按作者搜
/books search 冲浪        # 按描述内容搜
```

---

## 通用原则

- **不中断用户**：除「书名重复」和「status 命令多结果匹配」外，所有命令直接执行，不询问确认
- **识别失败不捏造**：字段识别不确定时留 `null`，并在输出中告知用户
- **幂等性**：重复扫描同一本书会跳过，不产生重复数据
- **中英文混合**：书名、作者保留原文，description 统一用中文
- **JSON 安全**：description 等文本字段禁止使用中文引号（`"` `"`），改用直引号或省略引号，否则 books.json 会损坏

---

## books.md 格式

每次更新 books.json 后同步维护此文件：

```markdown
# 我的书库

共 N 本 · 更新于 YYYY-MM-DD

| 书名 | 作者 | 出版年 | 国家 | 类别 | 状态 | 描述 |
|------|------|--------|------|------|------|------|
| 《The Barefoot Investor》| Scott Pape | 2016 | 澳大利亚 | 个人理财 | 已读 | 澳洲国民理财书 |
```

状态显示：`unread` → 未读，`reading` → 在读，`read` → 已读，`paused` → 暂停
