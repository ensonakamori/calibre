# Project Structure: Complete Directory Guide

**Purpose:** Understand Calibre's codebase organization with hyperlinks to actual code

**For:** React developers navigating a Python desktop application

**Time to Read:** 30 minutes

---

## Table of Contents

1. [Repository Overview](#repository-overview)
2. [Source Code Organization](#source-code)
3. [Key Directories Deep Dive](#key-directories)
4. [How to Navigate](#navigation)
5. [Where to Find Things](#find-things)

---

<a name="repository-overview"></a>
## 1. Repository Overview

### Top-Level Structure

```
calibre/
├── src/calibre/              ← Main application code ⭐
├── recipes/                  ← News download recipes
├── resources/                ← Static assets (images, CSS, JS)
├── setup/                    ← Build scripts
├── manual/                   ← Documentation source
├── imgsrc/                   ← Icon generation
├── pyproject.toml            ← Dependencies & config
├── setup.py                  ← Build configuration
└── README.md                 ← Project readme
```

**🔗 Key Files:**
- **[pyproject.toml](../../pyproject.toml)** - All dependencies and project metadata
- **[setup.py](../../setup.py)** - Build system entry point
- **[README.md](../../README.md)** - Project overview

---

<a name="source-code"></a>
## 2. Source Code Organization

### Primary Source Directory: `src/calibre/`

**Location:** [src/calibre/](../../src/calibre/)

This is where **ALL** the application code lives (~600k lines of Python).

```
src/calibre/
├── __init__.py               ← Package initialization
├── constants.py              ← App-wide constants
├── customize/                ← Plugin system
├── db/                       ← Database layer ⭐
├── devices/                  ← E-reader device drivers
├── ebooks/                   ← E-book processing ⭐
├── gui2/                     ← Desktop GUI (PyQt6) ⭐
├── library/                  ← Library management
├── srv/                      ← Content server (web) ⭐
├── utils/                    ← Utility functions
└── web/                      ← Web scraping utilities
```

**🧠 Mental Model for React Developers:**
```
src/calibre/
├── gui2/        →  Like your React components folder
├── srv/         →  Like your Next.js app/api/ folder
├── db/          →  Like your Prisma schema + queries
├── ebooks/      →  Business logic (unique to Calibre)
└── utils/       →  Like your lib/ or utils/ folder
```

---

<a name="key-directories"></a>
## 3. Key Directories Deep Dive

### 📂 `src/calibre/gui2/` - Desktop GUI (PyQt6)

**Purpose:** Entire desktop application interface
**Lines of Code:** ~300,000
**Tech:** PyQt6 (Qt 6 Python bindings)

**Structure:**
```
gui2/
├── main.py                   ← Main window
├── library/                  ← Book library view
│   ├── views.py             ← Table/grid views
│   └── models.py            ← Data models
├── viewer/                   ← E-book viewer
├── preferences/              ← Settings UI
├── dialogs/                  ← Popup dialogs
├── convert/                  ← Conversion UI
├── store/                    ← E-book store integration
└── **/*.ui                   ← Qt Designer XML files
```

**🔗 Key Files to Explore:**

**Main Application:**
- [gui2/main.py](../../src/calibre/gui2/main.py) - Main window class
- [gui2/ui.py](../../src/calibre/gui2/ui.py) - UI initialization
- [gui2/__init__.py](../../src/calibre/gui2/__init__.py) - GUI package entry

**Library Views:**
- [gui2/library/views.py](../../src/calibre/gui2/library/views.py) - Book list table view
- [gui2/library/models.py](../../src/calibre/gui2/library/models.py) - Qt model for book data
- [gui2/library/delegates.py](../../src/calibre/gui2/library/delegates.py) - Custom cell renderers

**E-book Viewer:**
- [gui2/viewer/main.py](../../src/calibre/gui2/viewer/main.py) - Viewer window
- [gui2/viewer/toc.py](../../src/calibre/gui2/viewer/toc.py) - Table of contents
- [gui2/viewer/annotations.py](../../src/calibre/gui2/viewer/annotations.py) - Highlights & notes

**Dialogs:**
- [gui2/dialogs/confirm_delete.py](../../src/calibre/gui2/dialogs/confirm_delete.py) - Deletion confirmation
- [gui2/dialogs/book_details.py](../../src/calibre/gui2/dialogs/book_details.py) - Book metadata editor
- [gui2/dialogs/tag_editor.py](../../src/calibre/gui2/dialogs/tag_editor.py) - Tag management

**Settings:**
- [gui2/preferences/main.py](../../src/calibre/gui2/preferences/main.py) - Preferences dialog
- [gui2/preferences/look_feel.py](../../src/calibre/gui2/preferences/look_feel.py) - UI customization
- [gui2/preferences/plugins.py](../../src/calibre/gui2/preferences/plugins.py) - Plugin management

🌉 **Bridge to React:**
```
gui2/main.py           →  Like App.tsx (root component)
gui2/library/views.py  →  Like BookList.tsx component
gui2/dialogs/          →  Like components/modals/
gui2/**/*.ui files     →  Like .jsx files (but XML)
```

**📚 Learn More:**
- [FRONTEND_ARCHITECTURE.md](./FRONTEND_ARCHITECTURE.md) - GUI deep dive (to be created)
- [PyQt6 Tutorial](https://www.riverbankcomputing.com/static/Docs/PyQt6/tutorial.html)

---

### 📂 `src/calibre/srv/` - Content Server (Web Interface)

**Purpose:** HTTP server for remote library access
**Lines of Code:** ~50,000
**Tech:** Custom async HTTP server + templating

**Structure:**
```
srv/
├── loop.py                   ← Event loop & HTTP server ⭐
├── routes.py                 ← Routing system ⭐
├── handler.py                ← Request handler
├── http_request.py           ← Request parser
├── http_response.py          ← Response builder
├── ajax.py                   ← AJAX endpoints
├── books.py                  ← Book API endpoints
├── metadata.py               ← Metadata endpoints
├── content.py                ← Static file serving
├── auth.py                   ← Authentication
├── users.py                  ← User management
├── web_socket.py             ← WebSocket support
└── tests/                    ← Server tests
```

**🔗 Key Files to Explore:**

**Core Server:**
- [srv/loop.py](../../src/calibre/srv/loop.py) - Main event loop (select-based)
  - `ServerLoop` class - Main server class
  - `serve_forever()` - Event loop that handles connections
- [srv/routes.py](../../src/calibre/srv/routes.py) - Routing system
  - `@endpoint` decorator - Defines routes (line ~57)
  - `Route` class - Route matching logic (line ~110)
  - `Router` class - Route registry (line ~224)
- [srv/handler.py](../../src/calibre/srv/handler.py) - Request handling

**HTTP Layer:**
- [srv/http_request.py](../../src/calibre/srv/http_request.py) - Parse HTTP requests
- [srv/http_response.py](../../src/calibre/srv/http_response.py) - Build HTTP responses
  - `RequestData` class - Request wrapper
  - HTTP header handling

**API Endpoints:**
- [srv/ajax.py](../../src/calibre/srv/ajax.py) - AJAX API (JSON responses)
- [srv/books.py](../../src/calibre/srv/books.py) - Book-related endpoints
  - Book listing, details, download
- [srv/metadata.py](../../src/calibre/srv/metadata.py) - Metadata operations
  - Edit book metadata remotely
- [srv/content.py](../../src/calibre/srv/content.py) - Serve book content
  - Stream EPUB/MOBI files
  - Serve cover images

**Security & Auth:**
- [srv/auth.py](../../src/calibre/srv/auth.py) - Authentication system
  - Session management
  - Cookie-based auth
- [srv/users.py](../../src/calibre/srv/users.py) - User database

**Real-time:**
- [srv/web_socket.py](../../src/calibre/srv/web_socket.py) - WebSocket server
  - Live updates when books added/changed

🌉 **Bridge to Next.js:**
```
srv/loop.py        →  Node.js HTTP server
srv/routes.py      →  Next.js app/api/ routes
srv/@endpoint      →  export async function GET()
srv/ajax.py        →  API route handlers
srv/auth.py        →  NextAuth.js / auth middleware
```

**📚 Learn More:**
- [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) - Server deep dive
- [DATA_FLOW_GUIDE.md](./DATA_FLOW_GUIDE.md) - Request lifecycle

---

### 📂 `src/calibre/db/` - Database Layer

**Purpose:** SQLite database management & in-memory cache
**Lines of Code:** ~30,000
**Tech:** APSW (SQLite), custom ORM

**Structure:**
```
db/
├── __init__.py               ← Package exports
├── cache.py                  ← In-memory cache (ORM) ⭐
├── backend.py                ← SQLite backend
├── legacy.py                 ← Old database format support
├── fields.py                 ← Field definitions
├── write.py                  ← Write operations
├── search.py                 ← Search queries
├── categories.py             ← Tag/category management
├── fts/                      ← Full-text search
│   ├── __init__.py
│   └── search.py
├── tables.py                 ← Table definitions
├── locking.py                ← Thread-safe locks
├── listeners.py              ← Event system
└── cli/                      ← Command-line tools
```

**🔗 Key Files to Explore:**

**Core Cache:**
- [db/cache.py](../../src/calibre/db/cache.py) - **THE** database layer ⭐
  - `Cache` class (line ~138) - Main in-memory cache
  - `@read_api` decorator - Read operations
  - `@write_api` decorator - Write operations
  - `get_metadata()` - Get book data
  - `set_metadata()` - Update book data
  - `all_book_ids()` - List all books
  - `search()` - Search books

**SQLite Backend:**
- [db/backend.py](../../src/calibre/db/backend.py) - Low-level SQLite operations
  - Connection management
  - Raw SQL queries
  - Transaction handling

**Field System:**
- [db/fields.py](../../src/calibre/db/fields.py) - Field type definitions
  - `Field` base class
  - `TextField`, `IntField`, `DateField`, etc.
  - Custom column support

**Search:**
- [db/search.py](../../src/calibre/db/search.py) - Search query parser
  - Parse: `title:"1984" AND author:orwell`
  - Execute complex queries
- [db/fts/search.py](../../src/calibre/db/fts/search.py) - Full-text search (FTS5)
  - SQLite FTS5 integration
  - Ranked search results

**Write Operations:**
- [db/write.py](../../src/calibre/db/write.py) - Database writes
  - Add books
  - Update metadata
  - Delete books

**Events:**
- [db/listeners.py](../../src/calibre/db/listeners.py) - Event system
  - `EventDispatcher` class
  - Notify when books change
  - Used for UI updates

**Thread Safety:**
- [db/locking.py](../../src/calibre/db/locking.py) - Read/write locks
  - Thread-safe database access
  - Prevents corruption

🌉 **Bridge to Prisma:**
```
db/cache.py          →  Prisma Client
db/backend.py        →  Raw SQL queries (.$queryRaw)
db/fields.py         →  Prisma schema fields
db/search.py         →  .findMany({ where: {...} })
db/fts/              →  Full-text search extension
db/listeners.py      →  Prisma middleware / events
```

**📚 Learn More:**
- [DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md) - Database deep dive

---

### 📂 `src/calibre/ebooks/` - E-book Processing

**Purpose:** Parse, convert, and manipulate e-book formats
**Lines of Code:** ~80,000
**Tech:** lxml, BeautifulSoup, Pillow, custom parsers

**Structure:**
```
ebooks/
├── conversion/               ← Format conversion pipeline
│   ├── cli.py               ← CLI interface
│   └── plugins/             ← Input/output plugins
│       ├── epub_input.py
│       ├── mobi_input.py
│       ├── pdf_input.py
│       └── ...
├── metadata/                 ← Metadata extraction
│   ├── book/                ← Base metadata classes
│   ├── sources/             ← Online sources (Amazon, Google Books)
│   └── opf2.py              ← OPF file handling
├── oeb/                      ← Open eBook (EPUB) handling
│   ├── parse.py             ← Parse EPUB
│   ├── transforms/          ← Content transformations
│   └── polish/              ← EPUB editing/polishing
├── txt/                      ← Plain text handling
├── rtf/                      ← RTF format
├── pdf/                      ← PDF handling
├── covers.py                 ← Cover image generation
└── chardet.py                ← Character encoding detection
```

**🔗 Key Files to Explore:**

**Conversion Pipeline:**
- [ebooks/conversion/cli.py](../../src/calibre/ebooks/conversion/cli.py) - Conversion CLI
- [ebooks/conversion/config.py](../../src/calibre/ebooks/conversion/config.py) - Conversion settings
- [ebooks/conversion/plugins/](../../src/calibre/ebooks/conversion/plugins/) - Format plugins

**EPUB Handling:**
- [ebooks/oeb/parse.py](../../src/calibre/ebooks/oeb/parse.py) - Parse EPUB files
  - Extract EPUB contents
  - Parse OPF (metadata)
  - Read NCX (table of contents)
- [ebooks/oeb/polish/main.py](../../src/calibre/ebooks/oeb/polish/main.py) - Edit EPUBs
  - Modify EPUB contents
  - Fix common issues

**Metadata:**
- [ebooks/metadata/book/base.py](../../src/calibre/ebooks/metadata/book/base.py) - `Metadata` class
  - Book metadata model
  - Title, authors, tags, etc.
- [ebooks/metadata/opf2.py](../../src/calibre/ebooks/metadata/opf2.py) - OPF parser/writer
  - Read metadata from EPUB
  - Write OPF files
- [ebooks/metadata/sources/](../../src/calibre/ebooks/metadata/sources/) - Online metadata
  - Amazon plugin
  - Google Books plugin
  - Goodreads, etc.

**Cover Generation:**
- [ebooks/covers.py](../../src/calibre/ebooks/covers.py) - Generate cover images
  - Create covers from text
  - Resize/crop existing covers
  - Uses Pillow (PIL)

**Format-Specific:**
- [ebooks/mobi/](../../src/calibre/ebooks/mobi/) - MOBI/AZW format
- [ebooks/pdf/](../../src/calibre/ebooks/pdf/) - PDF handling (LIMITED)
- [ebooks/txt/](../../src/calibre/ebooks/txt/) - Plain text
- [ebooks/rtf/](../../src/calibre/ebooks/rtf/) - Rich Text Format

🌉 **Bridge to JavaScript:**
```
ebooks/metadata/   →  Like cheerio/jsdom for web scraping
ebooks/oeb/        →  Like epub.js (but with write capability)
ebooks/covers.py   →  Like sharp or canvas for images
ebooks/conversion/ →  No JS equivalent (complex format conversion)
```

**📚 Learn More:**
- E-book formats are complex - **keep this in Python** if rewriting!
- Could expose as microservice API

---

### 📂 `src/calibre/utils/` - Utility Functions

**Purpose:** Shared utilities used throughout the app
**Lines of Code:** ~40,000

**Structure:**
```
utils/
├── config.py                 ← Configuration management
├── date.py                   ← Date/time utilities
├── icu.py                    ← International text handling
├── img.py                    ← Image processing
├── logging.py                ← Logging setup
├── network.py                ← Network utilities
├── serialize.py              ← JSON/MessagePack serialization
├── search_query_parser.py    ← Search query parsing
├── smtp.py                   ← Email sending
└── ...
```

**🔗 Key Files:**
- [utils/config.py](../../src/calibre/utils/config.py) - Config system
- [utils/date.py](../../src/calibre/utils/date.py) - Date parsing/formatting
- [utils/icu.py](../../src/calibre/utils/icu.py) - Unicode text sorting
- [utils/serialize.py](../../src/calibre/utils/serialize.py) - JSON/msgpack helpers

---

### 📂 `resources/` - Static Assets

**Purpose:** Images, CSS, JavaScript, fonts
**Location:** [resources/](../../resources/)

**Structure:**
```
resources/
├── images/                   ← PNG/SVG icons
│   ├── lt.png               ← Calibre logo
│   └── ...
├── content-server/           ← Web UI assets
│   ├── index.html           ← Main HTML template
│   └── ...
├── fonts/                    ← Embedded fonts
├── viewer.html               ← E-book viewer HTML
├── viewer.js                 ← E-book viewer JS
├── editor.js                 ← EPUB editor JS
├── mathjax/                  ← Math rendering
└── ...
```

**🔗 Key Files:**
- [resources/content-server/index.html](../../resources/content-server/index.html) - Web UI template
- [resources/viewer.html](../../resources/viewer.html) - E-book viewer (HTML+JS)
- [resources/images/](../../resources/images/) - All UI icons

🌉 **Bridge to Next.js:**
```
resources/           →  public/ folder
resources/images/    →  public/images/
resources/**/*.js    →  Like bundled client-side JS
```

---

### 📂 `recipes/` - News Download Recipes

**Purpose:** Python scripts to download news from websites
**Lines of Code:** ~200,000 (1,600+ recipes!)
**Location:** [recipes/](../../recipes/)

**Structure:**
```
recipes/
├── newyorker.recipe
├── economist.recipe
├── nytimes.recipe
└── ... (1,600+ more)
```

**Example Recipe:**
```python
# recipes/newyorker.recipe
from calibre.web.feeds.news import BasicNewsRecipe

class NewYorker(BasicNewsRecipe):
    title = 'The New Yorker'
    url = 'https://www.newyorker.com'

    def get_browser(self):
        # Custom browser setup
        pass

    def parse_index(self):
        # Scrape article list
        pass
```

**🔗 Explore:**
- [recipes/](../../recipes/) - Browse all recipes
- [src/calibre/web/feeds/news.py](../../src/calibre/web/feeds/news.py) - Recipe base class

---

### 📂 `setup/` - Build Scripts

**Purpose:** Build, test, and release scripts
**Location:** [setup/](../../setup/)

**Structure:**
```
setup/
├── build.py                  ← Main build script
├── commands.py               ← Build commands
├── test.py                   ← Test runner
├── translations.py           ← i18n handling
└── ...
```

**🔗 Key Files:**
- [setup/build.py](../../setup/build.py) - Build C extensions
- [setup/test.py](../../setup/test.py) - Run tests
- [setup/translations.py](../../setup/translations.py) - Translation workflow

---

<a name="navigation"></a>
## 4. How to Navigate

### Finding Features

**"Where is X implemented?"**

| **Feature** | **Location** | **File** |
|------------|-------------|----------|
| Book list display | `gui2/library/` | [views.py](../../src/calibre/gui2/library/views.py) |
| Add book to library | `gui2/` + `db/` | [add.py](../../src/calibre/gui2/add.py), [db/cache.py](../../src/calibre/db/cache.py) |
| Convert EPUB to MOBI | `ebooks/conversion/` | [plugins/](../../src/calibre/ebooks/conversion/plugins/) |
| Web server API | `srv/` | [routes.py](../../src/calibre/srv/routes.py), [ajax.py](../../src/calibre/srv/ajax.py) |
| Full-text search | `db/fts/` | [search.py](../../src/calibre/db/fts/search.py) |
| E-book viewer | `gui2/viewer/` | [main.py](../../src/calibre/gui2/viewer/main.py) |
| Metadata download | `ebooks/metadata/sources/` | [amazon.py](../../src/calibre/ebooks/metadata/sources/amazon.py), etc. |
| Device sync | `devices/` | [Various device drivers](../../src/calibre/devices/) |

---

### Code Search Tips

**Using grep:**
```bash
# Find where a function is defined
grep -r "def get_metadata" src/calibre/

# Find where a class is used
grep -r "from calibre.db import Cache" src/

# Find all endpoints
grep -r "@endpoint" src/calibre/srv/
```

**Using your editor:**
- VS Code: `Ctrl+Shift+F` (search in files)
- "Go to Definition" (F12) works with Python

---

<a name="find-things"></a>
## 5. Where to Find Things

### Common Tasks

**"I want to..."**

| **Task** | **Start Here** |
|---------|---------------|
| Understand the GUI | [gui2/main.py](../../src/calibre/gui2/main.py) → [FRONTEND_ARCHITECTURE.md](./FRONTEND_ARCHITECTURE.md) |
| Understand the web server | [srv/loop.py](../../src/calibre/srv/loop.py) → [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) |
| Understand the database | [db/cache.py](../../src/calibre/db/cache.py) → [DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md) |
| Add a new API endpoint | [srv/ajax.py](../../src/calibre/srv/ajax.py) → [HOW_TO_GUIDE.md](./HOW_TO_GUIDE.md) |
| Fix a bug in book conversion | [ebooks/conversion/](../../src/calibre/ebooks/conversion/) → Search error logs |
| Add a news recipe | [recipes/](../../recipes/) → Copy existing recipe |
| Modify metadata fields | [db/fields.py](../../src/calibre/db/fields.py) → [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) |

---

## File Naming Conventions

**Python modules:**
- `snake_case.py` - All Python files
- `__init__.py` - Package initialization
- `*_ui.py` - Auto-generated from Qt `.ui` files (**don't edit!**)

**Qt Designer files:**
- `*.ui` - XML files defining UI layouts
- Compiled to `*_ui.py` during build

**Test files:**
- `test_*.py` or `*_test.py`
- Located in `tests/` subdirectories

---

## Import Patterns

**How modules are imported:**

```python
# Absolute imports (preferred)
from calibre.db.cache import Cache
from calibre.srv.routes import endpoint
from calibre.ebooks.metadata import Metadata

# Relative imports (within package)
from .cache import Cache
from ..utils import config

# Conditional imports (for optional deps)
try:
    from PyQt6.QtWidgets import QApplication
except ImportError:
    # Fallback or skip
    pass
```

---

## Generated Files (Don't Edit!)

⚠️ **These files are auto-generated during build:**

- `**/*_ui.py` - Generated from `.ui` files
- `resources/*.rcc` - Compiled resources
- `build/` directory - All build artifacts
- `*.pyc` files - Python bytecode

**Always edit the source:**
- For UI: Edit `.ui` files with Qt Designer
- For resources: Edit source files, then rebuild

---

## 🎯 Quick Reference

### Most Important Files (Start Here)

1. **[src/calibre/gui2/main.py](../../src/calibre/gui2/main.py)** - Desktop app entry point
2. **[src/calibre/srv/loop.py](../../src/calibre/srv/loop.py)** - Web server
3. **[src/calibre/db/cache.py](../../src/calibre/db/cache.py)** - Database layer
4. **[src/calibre/ebooks/metadata/book/base.py](../../src/calibre/ebooks/metadata/book/base.py)** - Metadata model
5. **[pyproject.toml](../../pyproject.toml)** - Dependencies

### File Count by Directory

| Directory | Python Files | Total Lines |
|-----------|--------------|-------------|
| `gui2/` | ~800 | ~300,000 |
| `ebooks/` | ~500 | ~150,000 |
| `srv/` | ~30 | ~50,000 |
| `db/` | ~20 | ~30,000 |
| `utils/` | ~100 | ~40,000 |
| `devices/` | ~50 | ~30,000 |

**Total:** ~1,500 Python files, ~600,000 lines

---

## 📖 Next Steps

1. **Explore key files:** Click the hyperlinks above
2. **Read architecture docs:**
   - [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md)
   - [DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md)
   - [DATA_FLOW_GUIDE.md](./DATA_FLOW_GUIDE.md)
3. **Take a code tour:** [CODE_TOURS.md](./CODE_TOURS.md)
4. **Make your first change:** [FIRST_CONTRIBUTIONS.md](./FIRST_CONTRIBUTIONS.md)

---

**Found this helpful?** All hyperlinks point to actual code. Click around and explore!
