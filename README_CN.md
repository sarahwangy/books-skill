# Books — 个人书库 Claude Code Skill

[English README →](./README.md)

拍一张书架照片，书就进库了。

扫描封面 → Claude Vision 识别书名、作者、国家、类别、作者简介 → 自动存入本地 JSON → 生成可交互的 HTML 统计报表，支持四种主题配色、一键打印 PDF，部署到 Vercel 即可在手机和电脑同步查看。

---

## 功能特点

- **HEIC 自动处理** — iPhone 拍的照片直接用，macOS `sips` 内置转换，无需安装额外工具
- **批量扫描** — 指定文件夹，一次处理整个书架。超过 20 张会先询问是否试跑前 20 张确认准确率
- **作者简介自动生成** — 每次扫描同时生成一句话作者背景介绍
- **交互式统计报表** — 点击图表筛选，实时搜索，列标题排序，多个筛选条件叠加生效
- **四种视觉主题** — Classic / Dark / Warm / Ocean，生成报表时选择，整套配色（背景、图表、标签）一起切换
- **打印 / PDF** — 浏览器内一键导出，打印 CSS 自动隐藏图表，只保留干净的书单表格
- **Vercel 部署** — 一条命令发布到固定 URL，手机书签一次，永久可访问
- **封面缩略图内嵌** — base64 编码写入 HTML，离线也能看，不依赖本地路径
- **不止书籍** — 同样适用于酒标、名片、植物标签、桌游盒子，任何有封面的物品都可归档

---

## 准确率 & 成本说明

**识别准确率约 80–90%**

- 清晰正面封面：准确率很高
- 倾斜拍摄、书脊、低对比度封面：部分字段可能留空（`null`）
- 不确定的字段**不会猜测**，留空并告知用户

**影响准确率的因素：**
- 模糊或残缺封面 → 书名 / 作者可能有误
- 冷门作者 → 出版年和国家通常靠推断，非确认
- 内页、收据、宣传册 → 自动标记为非书籍，跳过

**速度 & token 消耗：**
- 每张图片约 1–3 秒
- 超过 20 张时，skill 会询问是否先试跑前 20 张——新文件夹建议这样做
- 大批量（100 张以上）建议分批处理

---

## 安装

### 手动安装（推荐）

```bash
# 项目级别（仅当前项目）
mkdir -p .agents/skills/books
cp SKILL.md .agents/skills/books/

# 全局（所有项目）
mkdir -p ~/.claude/skills/books
cp SKILL.md ~/.claude/skills/books/
```

### 直接 clone

```bash
git clone https://github.com/sarahwangy/books-skill .agents/skills/books
```

安装后在 Claude Code 中输入 `/books scan <图片路径>` 即可使用。

---

## 命令列表

| 命令 | 说明 |
|------|------|
| `/books scan <图片>` | 扫描单张照片，追加到书库 |
| `/books scan-folder <路径>` | 批量扫描文件夹（超 20 张先询问） |
| `/books isbn <ISBN>` | 用 ISBN 号查 Open Library，比封面识别更准确 |
| `/books list [状态]` | 终端快速列出书库，可按状态过滤 |
| `/books status <书名> <状态>` | 更新阅读状态 |
| `/books edit <书名> <字段> <新值>` | 修正任意字段，无需重新扫描 |
| `/books search <关键词>` | 搜索书名、作者、国家、类别、描述 |
| `/books stats [主题]` | 生成交互式 HTML 统计报表 |
| `/books present [主题]` | 生成数据驱动的 HTML 幻灯片演示，内容来自真实书库 |
| `/books deploy` | 部署报表到 Vercel，生成永久 URL |
| `/books rate <书名> <1-5>` | 给书打星（1–5 分）|
| `/books note <书名> <笔记>` | 添加私人备注，可追加或覆盖 |
| `/books suggest` | 根据书库模式推荐下一本读什么 |
| `/books export json` | 导出 JSON，可导入 Notion / Obsidian |

**状态可选值**：`unread`（未读）· `reading`（在读）· `read`（已读）· `paused`（暂停）

---

## 统计报表主题

| 主题 | 配色风格 |
|------|---------|
| **Classic** | 暖米色背景，钢蓝 accent（默认）|
| **Dark** | 深海军蓝背景，琥珀 accent |
| **Warm** | 象牙色背景，赭石 accent |
| **Ocean** | 浅青色背景，深石板 accent |

---

## 数据格式

`books.json` 单条记录：

```json
{
  "id": "b015",
  "title": "Being You: A New Science of Consciousness",
  "author": "Anil Seth",
  "author_gender": "M",
  "year": 2021,
  "country": "UK",
  "category": "Popular Science",
  "description": "Neuroscientist challenges our understanding of self and reality through brain science.",
  "author_bio": "British neuroscientist at the University of Sussex researching the nature of consciousness.",
  "status": "unread",
  "photo": "IMG_0197.HEIC",
  "added": "2026-07-07"
}
```

**语言规则**：中文书（书名 / 作者为中文）所有字段保留中文；其他书籍的 country、category、description、author_bio 统一用英文。

---

## 输出文件

| 文件 | 说明 |
|------|------|
| `books.json` | 主数据文件，所有操作的数据源 |
| `books.md` | Markdown 表格，可粘贴到任意笔记 app |
| `books-report.html` | 自包含交互式报表 |
| `books-export.json` | 干净导出，供 Notion / Obsidian 导入 |
| `rename-guide.txt` | 批量扫描后的图片重命名建议 |

---

## 后续路线图

- `/books reading-log` — 按月显示阅读节奏，时间轴可视化
- 封面扫描 + ISBN 交叉核对 — 扫描后用 ISBN 校验，提升 title/year 准确率
- 多设备同步 — 目前 books.json 存本地；`/books deploy` 解决只读访问，未来版本考虑双向同步

---

## 环境要求

| 依赖 | 用途 |
|------|------|
| [Claude Code](https://claude.ai/code) | 运行 skill |
| macOS | HEIC 转换（`sips` 内置）；JPG/PNG 在任意系统可用 |
| Python 3 | 生成 `books-report.html`（macOS 预装）|
| Node.js | 仅 `/books deploy` 需要 |
| Vercel 账号（免费）| 仅 `/books deploy` 需要 |

---

## License

MIT
