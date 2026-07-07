# Books — Personal Library Skill for Claude Code

[中文 README →](./README_CN.md)

A coding-agent skill that turns photos of your bookshelf into a structured, searchable library — complete with an interactive analytics dashboard, four visual themes, one-click PDF export, and Vercel deployment for multi-device access.

No API keys. No accounts. No internet required to scan.

---

## What This Does

**Books** solves a specific problem: you have a shelf full of books and no idea what you own, what you've read, or how your reading actually breaks down by country, gender, or category.

Point it at a photo → Claude reads the cover → the book is in your library. Batch-scan a whole shelf in one go. Open the dashboard, click a chart bar to filter by country, search by keyword, sort any column. Print the table to PDF. Deploy to Vercel so your phone and laptop see the same library.

```
/books scan ~/Photos/bookshelf/IMG_0124.HEIC
/books scan-folder ~/Photos/bookshelf/
/books isbn 9780008669713
/books list
/books search memoir
/books status "Top End Girl" read
/books edit "Top End Girl" year 2021
/books rate "Being You" 5
/books note "Being You" 意识是大脑主动建构的预测，读完彻底改变了我对感知的理解
/books suggest
/books stats
/books deploy
/books export json
```

---

## Key Features

- **iPhone HEIC support** — Photos from your camera roll work directly. No manual conversion needed; the skill calls macOS `sips` automatically.
- **Batch scanning** — Point at a folder, walk away. The skill identifies books, skips internal pages and non-book images, flags duplicates, and writes everything in one pass.
- **Author bio generation** — Every scan produces a one-sentence author background alongside the book metadata.
- **Interactive dashboard** — Click chart bars to filter, search in real time, sort any column, stack multiple filters. All state is shared — a country click + a search query work together.
- **Four visual themes** — Classic, Dark, Warm, or Ocean. Chosen at report generation time; the entire colour system (background, cards, charts, tags) switches together.
- **Print / PDF** — One button in the browser. Print CSS hides charts and controls, leaving a clean book list — correctly paginated, covers included if present.
- **Vercel deploy** — One command publishes the dashboard to a permanent URL. Same URL every time; bookmark it on your phone.
- **Cover thumbnails** — Embedded as base64 in the HTML. No broken links, no external paths, works offline.
- **JSON-safe writing** — Every write automatically strips Chinese curly quotes from text fields before touching `books.json`.
- **English-first, Chinese-respectful** — Non-Chinese books use English for all metadata fields. Chinese books stay fully Chinese.
- **Works beyond books** — The same skill handles wine labels, business cards, plant tags, and board game boxes. Books are the default; anything with a cover works.

---

## Installation

### Manual (recommended)

Copy `SKILL.md` into your project's skills directory:

```bash
# Project-level (this project only)
mkdir -p .agents/skills/books
cp SKILL.md .agents/skills/books/

# Global (all projects)
mkdir -p ~/.claude/skills/books
cp SKILL.md ~/.claude/skills/books/
```

Then invoke it in Claude Code:

```
/books scan ~/Photos/IMG_0124.HEIC
```

### Clone directly

```bash
# Project-level
git clone https://github.com/sarahwangy/books-skill .agents/skills/books

# Global
git clone https://github.com/sarahwangy/books-skill ~/.claude/skills/books
```

---

## Commands

### `/books scan <image>`

Scans a single photo and appends the book to your library.

```
/books scan ~/Photos/IMG_0124.HEIC
/books scan ./bookshelf/being-you.jpg
```

- Accepts JPG, PNG, HEIC, WEBP
- HEIC files are converted via `sips` automatically — no extra tools needed
- Recognises: title, author, year, country, category, description, author bio
- Skips duplicates (matched by title)
- Leaves uncertain fields as `null` rather than guessing

**Output:**
```
✓ Added
  Title:    Being You: A New Science of Consciousness
  Author:   Anil Seth (M · UK)
  Bio:      British neuroscientist at University of Sussex researching consciousness.
  Category: Popular Science · 2021
  Status:   unread

Library now has 16 books.
```

---

### `/books scan-folder <path>`

Batch-scans every image in a folder.

```
/books scan-folder ~/Photos/bookshelf/
```

- Processes JPG, PNG, HEIC, WEBP
- Skips non-book images (internal pages, brochures, receipts) — flags them in the summary
- Skips titles already in the library
- Writes `books.json` and `books.md` once at the end, not after every photo
- Generates `rename-guide.txt` with suggested filenames in `Category_Title_Author` format

**Output:**
```
Scan complete
  Identified:            14 books
  Skipped (duplicate):    2
  Skipped (non-book):     3 files — IMG_0121.HEIC, IMG_0122.HEIC, IMG_0174.HEIC

New additions:
  · Being You — Anil Seth (Popular Science)
  · Top End Girl — Miranda Tapsell (Memoir)
  · 南极之南 — 毕淑敏（游记散文）

Rename guide saved to rename-guide.txt
```

---

### `/books list [status]`

Quick terminal overview — no browser needed.

```
/books list
/books list reading
/books list read
/books list unread
```

**Output:**
```
Library · 16 books (1 reading · 15 unread)

 # | Title                                    | Author              | Category        | Status
---|------------------------------------------|---------------------|-----------------|--------
 1 | Being You                                | Anil Seth           | Popular Science | reading
 2 | Top End Girl                             | Miranda Tapsell     | Memoir          | unread
 3 | 南极之南                                  | 毕淑敏              | 游记散文         | unread

Tip: /books stats to open interactive dashboard
```

---

### `/books status <title keyword> <status>`

Updates reading status with fuzzy title matching.

```
/books status "Being You" reading
/books status anil read
/books status "Top End" paused
```

Status values: `unread` · `reading` · `read` · `paused`

If the keyword matches more than one book, the skill lists the matches and asks you to pick by number.

---

### `/books edit <title keyword> <field> <new value>`

Corrects any field without rescanning.

```
/books edit "Top End Girl" year 2021
/books edit anil category "Popular Science"
/books edit "Big Feelings" author_bio "American workplace emotion researcher and illustrator"
/books edit "南极之南" status read
```

Editable fields: `title` · `author` · `author_gender` · `year` · `country` · `category` · `description` · `author_bio` · `status` · `photo`

Curly-quote sanitisation runs automatically on `description` and `author_bio` before writing.

---

### `/books search <keyword>`

Searches across title, author, country, category, and description.

```
/books search UK
/books search memoir
/books search "Annie Duke"
/books search 冲浪
```

Returns a Markdown table sorted by relevance. If nothing matches: `No books found matching "keyword"`.

---

### `/books stats [theme]`

Generates `books-report.html` — a fully self-contained interactive dashboard.

```
/books stats
/books stats dark
/books stats warm
```

If no theme is given, the skill asks you to choose:

```
Choose a dashboard theme:

  1. Classic  — warm beige, steel blue     (default)
  2. Dark     — deep navy, amber
  3. Warm     — ivory, terracotta
  4. Ocean    — soft teal, dark slate

Which theme? (1–4, or press Enter for Classic)
```

The dashboard includes:
- **4 stat cards** — total books, female author %, countries, categories
- **3 interactive charts** — country bar, gender pie, category bar; click any segment to filter
- **Active filter chips** — shows current filters; click ✕ to clear individually
- **Real-time search** — filters across title, author, country, category simultaneously
- **Sortable table** — click any column header to sort ascending/descending
- **Cover thumbnails** — base64-embedded, no external paths
- **⎙ Print / PDF button** — browser print dialog with clean print CSS

The HTML file is fully self-contained. Send it to anyone; it needs no server.

---

### `/books isbn <ISBN>`

Look up a book by its ISBN number via Open Library (free, no API key). More accurate than cover recognition for title, author, and year.

```
/books isbn 9780008669713
/books isbn 0385737951
```

- ISBN is the 13-digit number under the barcode on the back cover (or the older 10-digit version)
- Fetches title, author, and year from Open Library — Claude fills in country, category, description, author_bio, and author_gender
- Clearly labels which fields came from Open Library vs Claude inference
- Falls back gracefully if the ISBN isn't in Open Library's database

**Output:**
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
```

---

### `/books rate <title keyword> <1-5>`

Rate a book 1–5 stars. Fuzzy title matching — no need for exact title.

```
/books rate "Being You" 5
/books rate anil 4
```

Rating is stored in `books.json` and shown in the stats dashboard.

---

### `/books note <title keyword> <text>`

Add a personal note to a book — reading impressions, quotes, thoughts, anything. If a note already exists, the skill asks whether to append or replace.

```
/books note "Being You" 意识是大脑主动建构的预测，不是被动接收现实
/books note anil "Read three times, something new each time"
```

---

### `/books suggest`

Recommends what to read next based on your library's patterns — categories you enjoy, countries you haven't explored, highly rated books you've already read, and unread books already on your shelf.

```
/books suggest
```

No external API needed. Claude analyses your `books.json` directly.

**Example output:**
```
Based on your library (16 books · mostly Memoir & Self-Help · UK/Australia)

From your unread shelf:
  → "Big Feelings" — matches your psychology interest

External picks:
  → "Crying in H Mart" by Michelle Zauner — cross-cultural memoir, similar to "From Scratch"
  → "The Body Keeps the Score" — Psychology, complements your self-help reads
```

---

### `/books deploy`

Publishes `books-report.html` to a permanent Vercel URL.

```
/books deploy
```

- Installs Vercel CLI automatically if not present
- Prompts Vercel login on first run (free account)
- Uses a fixed project name (`my-books-library`) so the URL never changes
- Run again after adding new books to update the live version

**Output:**
```
✓ Deployed: https://my-books-library.vercel.app

Open this URL on any device — phone, iPad, or another computer.
Bookmark it. Run /books deploy again after adding new books to update.
```

---

### `/books export json`

Exports a clean, formatted JSON file for Notion, Obsidian, or any other tool.

```
/books export json
```

Writes `books-export.json` with `indent=2` formatting. Same schema as `books.json`.

---

## Dashboard Themes

### Classic
Warm beige canvas, steel-blue accent, soft sand borders. The default. Works in any light and feels like a well-organised reading nook.

### Dark
Deep navy background (`#0f172a`), amber accent, slate card surfaces. Chart colours shift to warm yellows and teals. Comfortable for evening reading sessions.

### Warm
Ivory canvas, terracotta accent, dusty borders. Earth tones throughout — tags in sand and burnt orange. Feels like a handwritten book journal.

### Ocean
Soft teal background, dark slate accent, clean white cards. Calm and airy. Chart colours run through teal and aquamarine.

---

## Output Files

| File | Description |
|------|-------------|
| `books.json` | Primary data store — source of truth |
| `books.md` | Markdown table — paste into any note app |
| `books-report.html` | Self-contained interactive dashboard |
| `books-export.json` | Clean export for Notion / Obsidian / other tools |
| `rename-guide.txt` | Photo rename suggestions after batch scan |

---

## Data Format

Each record in `books.json`:

```json
{
  "id": "b015",
  "title": "Being You: A New Science of Consciousness",
  "title_original": null,
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

**Language rule:** Chinese books (Chinese title or author) keep all fields in Chinese. All other books use English for `country`, `category`, `description`, and `author_bio`.

**Status values:** `unread` · `reading` · `read` · `paused`

**Gender values:** `F` · `M` · `Group` · `Mixed` · `Unknown` (Chinese books: `女` · `男` · `机构`)

---

## Architecture

The entire skill lives in a single `SKILL.md` file. No helper scripts, no dependencies to install, no build step.

| Component | How it works |
|-----------|-------------|
| Vision recognition | Claude reads the cover image directly — no external OCR service |
| HEIC conversion | macOS `sips` (built-in, zero install) |
| Stats generation | A Python script embedded in `SKILL.md`; Claude writes it to `/tmp`, runs it, then deletes it |
| Base64 encoding | Python `base64` module — no line-break issues from shell `base64` |
| Deployment | Vercel CLI (`npx vercel`) — installed on first use |
| Charts | HTML5 Canvas API — no chart library dependency |
| Data | `books.json` flat file — no database, no schema migration |

---

## Roadmap

Features under consideration, roughly in priority order:

- **`/books reading-log`** — Monthly reading pace, visible as a timeline.
- **`/books isbn` + cover verify** — After a cover scan, optionally cross-check the detected ISBN against Open Library to confirm title/year accuracy.
- **Multi-device sync** — Right now `books.json` lives locally. Running `/books deploy` gets you read-only access anywhere; a future version could sync edits back.

---

## Accuracy & Cost

**Recognition accuracy: ~80–90%**

- Clear, well-lit front covers: very high accuracy
- Angled shots, spines, or low-contrast covers: some fields may be left as `null`
- Uncertain fields are never guessed — they're left blank and reported to you

**What affects accuracy:**
- Blurry or partial covers → title/author may be wrong
- Lesser-known authors → year and country often inferred, not confirmed
- Internal pages, receipts, brochures → flagged as non-book and skipped

**Cost & speed:**
- Each image takes approximately 1–3 seconds
- For large batches (20+ images), the skill will ask if you want to trial-scan the first 20 before committing to the full set — recommended for a new folder
- Token usage scales with image count; very large batches (100+) are best split across multiple sessions

---

## Requirements

| Requirement | Used for |
|-------------|----------|
| [Claude Code](https://claude.ai/code) | Running the skill |
| macOS | HEIC conversion via `sips` (JPG/PNG work on any OS) |
| Python 3 | Generating `books-report.html` (pre-installed on macOS) |
| Node.js | `/books deploy` only — not needed for scanning or stats |
| Vercel account (free) | `/books deploy` only |

---

## License

MIT — use it, modify it, share it.
