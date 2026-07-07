# books · Personal Library Skill for Claude Code

Turn photos of your bookshelf into a structured, searchable library — with an interactive HTML dashboard.

**No API keys. No accounts. No internet required.**

![books-report dashboard showing cover thumbnails, charts, and search](docs/preview.png)

---

## What it does

Snap a photo of any book → Claude reads the cover → instantly added to your local library.

```
/books scan ~/Photos/bookshelf/IMG_0124.HEIC
/books scan-folder ~/Photos/bookshelf/
/books search memoir
/books status "Top End Girl" read
/books edit "Top End Girl" year 2021
/books stats
/books export json
```

---

## Commands

| Command | What it does |
|---------|-------------|
| `/books scan <image>` | Scan one photo, append to library |
| `/books scan-folder <path>` | Batch scan entire folder |
| `/books status <title> <status>` | Update reading status (`unread` / `reading` / `read` / `paused`) |
| `/books edit <title> <field> <value>` | Fix any field without rescanning |
| `/books search <keyword>` | Search by title, author, country, category, or description |
| `/books stats` | Generate interactive HTML dashboard |
| `/books export json` | Export clean JSON for Notion / Obsidian |

---

## Output files

| File | Description |
|------|-------------|
| `books.json` | Primary data store |
| `books.md` | Markdown table (paste into any note app) |
| `books-report.html` | Self-contained interactive dashboard |
| `rename-guide.txt` | Photo rename suggestions after batch scan |
| `books-export.json` | Clean JSON export |

---

## Features

- **HEIC support** — iPhone photos work directly, no manual conversion needed
- **Smart filtering** — auto-skips internal pages, travel brochures, non-book images
- **Author bio** — one-sentence author background generated alongside scan
- **Interactive dashboard** — click charts to filter, search in real-time, sort by any column
- **Cover thumbnails** — embedded as base64, no broken links, works offline
- **JSON-safe** — no curly quotes in text fields

---

## Install

```bash
npx skills add https://github.com/YOUR_USERNAME/books-skill
```

Or manually copy `SKILL.md` into your project's `.agents/skills/books/` directory.

---

## Photo tips

- Works with hand-held photos, angled shots, HEIC / JPG / PNG
- One book per photo for best results
- Front cover gives the most accurate recognition
- If the cover is unclear, Claude will leave uncertain fields as `null` rather than guess

---

## Data format

Each book record in `books.json`:

```json
{
  "id": "b001",
  "title": "Being You: A New Science of Consciousness",
  "author": "Anil Seth",
  "author_gender": "男",
  "author_bio": "英国神经科学家，萨塞克斯大学教授，研究意识本质。",
  "year": 2021,
  "country": "英国",
  "category": "科普",
  "description": "神经科学家从脑科学角度探索意识的本质",
  "status": "unread",
  "photo": "IMG_0197.HEIC",
  "added": "2026-07-07"
}
```

---

## Requirements

- [Claude Code](https://claude.ai/code)
- macOS (for HEIC conversion via `sips`) or convert photos to JPG first on other platforms
- Python 3 (pre-installed on macOS) — used only for `stats` command

---

## Made with

Built as a Claude Code skill. Scans real bookshelf photos using Claude Vision.
