# Architecture Overview: The Complete System

**Purpose:** Understand Calibre's complete architecture from 30,000 feet

**For:** React developers transitioning to full-stack thinking

**Time to Read:** 60 minutes

**Prerequisites:** Read [README.md](./README.md) first

---

## Table of Contents

1. [The Big Picture](#big-picture)
2. [Three-Tier Architecture](#three-tier)
3. [Desktop vs Web: Two Interfaces, One Core](#desktop-vs-web)
4. [Data Architecture](#data-architecture)
5. [Plugin System](#plugin-system)
6. [Comparison to Modern Architectures](#comparison)
7. [Design Decisions & Trade-offs](#design-decisions)

---

<a name="big-picture"></a>
## 1. The Big Picture

### What IS Calibre?

Calibre is a **hybrid desktop/web application** for managing e-book libraries:

```
┌─────────────────────────────────────────────────────────────────┐
│                         CALIBRE                                  │
│          The Complete E-book Management System                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────┐         ┌──────────────────────┐      │
│  │  Desktop Application │         │   Web Application    │      │
│  │      (PyQt6)         │         │   (Browser-based)    │      │
│  │                      │         │                      │      │
│  │  ┌────────────────┐ │         │  ┌────────────────┐  │      │
│  │  │ Main Window    │ │         │  │ Mobile UI      │  │      │
│  │  │ Book Library   │ │         │  │ (HTML5)        │  │      │
│  │  │ E-book Viewer  │ │         │  │                │  │      │
│  │  │ Editor         │ │         │  │ Browse Library │  │      │
│  │  │ Conversion     │ │         │  │ Read Books     │  │      │
│  │  │ Device Sync    │ │         │  │ Download Books │  │      │
│  │  └────────────────┘ │         │  └────────────────┘  │      │
│  │                      │         │                      │      │
│  │  Local UI            │         │  Remote Access       │      │
│  │  Full features       │         │  Essential features  │      │
│  │  High performance    │         │  Any device          │      │
│  └──────────┬───────────┘         └──────────┬───────────┘      │
│             │                                │                  │
│             └────────────┬───────────────────┘                  │
│                          ▼                                      │
│              ┌────────────────────────┐                         │
│              │     CORE LIBRARY       │                         │
│              │  (Shared Python Code)  │                         │
│              │                        │                         │
│              │  • Database Layer      │ ← [cache.py][1]         │
│              │  • E-book Processing   │ ← [ebooks/][2]          │
│              │  • Metadata Management │ ← [metadata/][3]        │
│              │  • Format Conversion   │ ← [conversion/][4]      │
│              │  • Plugin System       │ ← [customize/][5]       │
│              └───────────┬────────────┘                         │
│                          ▼                                      │
│              ┌────────────────────────┐                         │
│              │    DATA STORAGE        │                         │
│              │                        │                         │
│              │  ┌──────────────────┐  │                         │
│              │  │ metadata.db      │  │ ← SQLite database       │
│              │  │ (SQLite)         │  │                         │
│              │  └──────────────────┘  │                         │
│              │                        │                         │
│              │  ┌──────────────────┐  │                         │
│              │  │ Book Files       │  │ ← Actual e-books        │
│              │  │ /Author/Book/    │  │                         │
│              │  │   book.epub      │  │                         │
│              │  │   cover.jpg      │  │                         │
│              │  │   metadata.opf   │  │                         │
│              │  └──────────────────┘  │                         │
│              └────────────────────────┘                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

[1]: ../../src/calibre/db/cache.py
[2]: ../../src/calibre/ebooks/
[3]: ../../src/calibre/ebooks/metadata/
[4]: ../../src/calibre/ebooks/conversion/
[5]: ../../src/calibre/customize/
```

---

### 🧠 Mental Model: The Restaurant Analogy

Think of Calibre as a **library** (the physical kind):

| **Component** | **Restaurant** | **Calibre** | **Code** |
|--------------|---------------|-------------|----------|
| **Front Desk** | Where you order | Desktop GUI / Web UI | [gui2/][6], [srv/][7] |
| **Kitchen** | Where food is prepared | Core library (processing) | [ebooks/][2] |
| **Storage** | Walk-in freezer | Database + File system | [db/][8], files |
| **Recipes** | How to cook | Conversion plugins | [conversion/][4] |
| **Waiters** | Bring you food | API / IPC layer | [srv/routes.py][9] |
| **Menu** | What's available | Library catalog | [cache.py][1] |

[6]: ../../src/calibre/gui2/
[7]: ../../src/calibre/srv/
[8]: ../../src/calibre/db/
[9]: ../../src/calibre/srv/routes.py

**Key Insight:** The kitchen (core library) doesn't care if the order came from the front desk (desktop) or phone (web). Same ingredients, same recipes, different entry points.

---

<a name="three-tier"></a>
## 2. Three-Tier Architecture

Calibre follows classic **three-tier architecture**:

```
┌─────────────────────────────────────────────────────────────┐
│  PRESENTATION TIER                                           │
│  (What users see and interact with)                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Desktop (PyQt6)              Web (HTML/JS)                  │
│  ┌─────────────────┐          ┌─────────────────┐           │
│  │ QTableView      │          │ <table>         │           │
│  │ QDialog         │          │ <form>          │           │
│  │ QMenuBar        │          │ <nav>           │           │
│  │ Custom Widgets  │          │ React-like SPA  │           │
│  └─────────────────┘          └─────────────────┘           │
│                                                              │
│  Files: gui2/**/*.py          Files: srv/ajax.py            │
│        gui2/**/*.ui                  resources/*.html       │
│                                                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  LOGIC TIER                                                  │
│  (Business logic, rules, processing)                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  E-book Processing          Metadata Management              │
│  ┌─────────────────┐        ┌─────────────────┐            │
│  │ Parse EPUB      │        │ Extract from    │            │
│  │ Convert formats │        │ Amazon API      │            │
│  │ Generate covers │        │ Google Books    │            │
│  │ Edit content    │        │ Goodreads       │            │
│  └─────────────────┘        └─────────────────┘            │
│                                                              │
│  Files: ebooks/conversion/   Files: ebooks/metadata/        │
│        ebooks/oeb/                  ebooks/metadata/sources/│
│                                                              │
│  Search & Catalog           Device Management                │
│  ┌─────────────────┐        ┌─────────────────┐            │
│  │ FTS5 search     │        │ Kindle driver   │            │
│  │ Tag management  │        │ Kobo driver     │            │
│  │ Series tracking │        │ USB detection   │            │
│  └─────────────────┘        └─────────────────┘            │
│                                                              │
│  Files: db/search.py         Files: devices/               │
│        db/fts/                                              │
│                                                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  DATA TIER                                                   │
│  (Persistent storage and retrieval)                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  In-Memory Cache            SQLite Database                  │
│  ┌─────────────────┐        ┌─────────────────┐            │
│  │ Dict of all     │   ←→   │ books table     │            │
│  │ books, authors, │        │ authors table   │            │
│  │ tags, series    │        │ tags table      │            │
│  │                 │        │ ... 20+ tables  │            │
│  │ Fast reads      │        │ Persistent      │            │
│  │ (< 1ms)         │        │ storage         │            │
│  └─────────────────┘        └─────────────────┘            │
│                                                              │
│  Files: db/cache.py          File: metadata.db              │
│        db/backend.py         Location: /library/            │
│                                                              │
│  Filesystem                                                  │
│  ┌─────────────────────────────────────────┐                │
│  │ /library/                                │                │
│  │   Author Name/                           │                │
│  │     Book Title (1)/                      │                │
│  │       book.epub                          │                │
│  │       cover.jpg                          │                │
│  │       metadata.opf                       │                │
│  │     Book Title (2)/                      │                │
│  │       book.mobi                          │                │
│  └─────────────────────────────────────────┘                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

### 🌉 Bridge to Modern Web Architecture

**Calibre (Desktop):**
```
Presentation: PyQt6 (like React components)
Logic:        Python modules (like services/)
Data:         SQLite + Cache (like Prisma + Redis)
```

**Modern Next.js App:**
```
Presentation: React components (app/components/)
Logic:        Server actions / API routes (app/api/)
Data:         Prisma + PostgreSQL
```

**Same pattern, different technologies!**

---

<a name="desktop-vs-web"></a>
## 3. Desktop vs Web: Two Interfaces, One Core

### Architectural Separation

```
┌────────────────────────────────────────────────────────────┐
│                   DESKTOP PATH                              │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  User clicks book                                           │
│         │                                                   │
│         ▼                                                   │
│  Qt Signal emitted: book_clicked(book_id)                  │
│  [gui2/library/views.py:BookView][10]                      │
│         │                                                   │
│         ▼                                                   │
│  Slot handler: on_book_clicked(book_id)                    │
│  [gui2/main.py:MainWindow][11]                             │
│         │                                                   │
│         ▼                                                   │
│  Call core library: db.get_metadata(book_id)               │
│  [db/cache.py:Cache.get_metadata][12]                      │
│         │                                                   │
│         ▼                                                   │
│  Return metadata dict                                       │
│         │                                                   │
│         ▼                                                   │
│  Update Qt widgets: update_book_details(metadata)          │
│  [gui2/book_details.py][13]                                │
│         │                                                   │
│         ▼                                                   │
│  User sees book details                                     │
│                                                             │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                     WEB PATH                                │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  User clicks book (browser)                                 │
│         │                                                   │
│         ▼                                                   │
│  JavaScript: fetch('/ajax/book/42')                        │
│  [resources/viewer.js][14] (or custom SPA)                 │
│         │                                                   │
│         ▼                                                   │
│  HTTP request arrives at server                             │
│  [srv/loop.py:ServerLoop][15]                              │
│         │                                                   │
│         ▼                                                   │
│  Route to endpoint: ajax_book(ctx, rd, 42)                 │
│  [srv/ajax.py][16]                                         │
│         │                                                   │
│         ▼                                                   │
│  Call core library: ctx.db.get_metadata(42)                │
│  [db/cache.py:Cache.get_metadata][12]  ← SAME CODE!        │
│         │                                                   │
│         ▼                                                   │
│  Return metadata dict                                       │
│         │                                                   │
│         ▼                                                   │
│  Serialize to JSON: json(ctx, rd, endpoint, metadata)      │
│  [srv/routes.py:json][17]                                  │
│         │                                                   │
│         ▼                                                   │
│  HTTP response: {"id": 42, "title": "1984", ...}          │
│         │                                                   │
│         ▼                                                   │
│  JavaScript updates DOM: renderBookDetails(json)           │
│         │                                                   │
│         ▼                                                   │
│  User sees book details                                     │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

[10]: ../../src/calibre/gui2/library/views.py
[11]: ../../src/calibre/gui2/main.py
[12]: ../../src/calibre/db/cache.py
[13]: ../../src/calibre/gui2/book_details.py
[14]: ../../resources/viewer.js
[15]: ../../src/calibre/srv/loop.py
[16]: ../../src/calibre/srv/ajax.py
[17]: ../../src/calibre/srv/routes.py

---

### 💡 Aha Moment: Code Reuse

**The same function serves both interfaces!**

```python
# File: src/calibre/db/cache.py

@read_api
def get_metadata(self, book_id):
    '''
    Get book metadata.

    Called by:
    1. Desktop GUI (direct Python call)
    2. Web API (via HTTP endpoint)
    3. CLI tools (command-line)
    4. Plugins (3rd-party extensions)

    Same code, different entry points!
    '''
    if book_id in self._metadata_cache:
        return self._metadata_cache[book_id]

    # ... query database ...

    return metadata
```

**This is brilliant design!**
- ✅ No code duplication
- ✅ Consistent behavior
- ✅ Single source of truth
- ✅ Easier testing
- ✅ Bug fixes apply everywhere

🌉 **Bridge to Modern:**
```typescript
// This is like having a shared tRPC router:

// Server function (Next.js)
export async function getBookMetadata(bookId: number) {
  return await prisma.book.findUnique({ where: { id: bookId } });
}

// Used by:
// 1. Server component: const book = await getBookMetadata(42)
// 2. API route: return Response.json(await getBookMetadata(42))
// 3. tRPC: t.procedure.query(() => getBookMetadata(42))
```

---

<a name="data-architecture"></a>
## 4. Data Architecture

### The Write-Through Cache Pattern

**Problem:** SQLite is slow (disk I/O). Database operations block the UI.

**Solution:** Keep entire database in RAM, write through to disk.

```
┌──────────────────────────────────────────────────────────┐
│  USER REQUEST                                             │
└─────────────────┬────────────────────────────────────────┘
                  │
      ┌───────────┴────────────┐
      │                        │
      ▼                        ▼
   READ PATH              WRITE PATH
      │                        │
      ▼                        ▼
┌──────────────┐        ┌──────────────┐
│ IN-MEMORY    │        │ IN-MEMORY    │
│ CACHE        │        │ CACHE        │
│              │        │              │
│ Check cache  │        │ 1. Update    │
│ ├─ HIT?      │        │    cache     │
│ │   └─ Return│        │              │
│ └─ MISS?     │        │ 2. Write to  │
│     └─ Query │        │    SQLite    │
│        SQLite│        │              │
│              │        │ 3. Notify    │
│              │        │    listeners │
└──────────────┘        └──────────────┘
      │                        │
      ▼                        ▼
┌─────────────────────────────────────┐
│         SQLite Database              │
│         (metadata.db)                │
│                                      │
│  Persistent storage                  │
│  Survives restarts                   │
│  Source of truth                     │
└─────────────────────────────────────┘
```

**Code:** [src/calibre/db/cache.py](../../src/calibre/db/cache.py)

```python
class Cache:
    '''
    In-memory cache of the entire database.

    Like having the entire Prisma query result cached in Redis,
    but even faster (plain Python dicts).
    '''

    def __init__(self, backend):
        self.backend = backend  # SQLite connection
        self._metadata_cache = {}  # Dict[int, dict]
        self._authors_cache = {}
        self._tags_cache = {}
        # ... more caches

        # Load all data on startup
        self.reload_from_db()

    def reload_from_db(self):
        '''Load entire database into memory'''
        # This happens once at startup
        for row in self.backend.get_all_books():
            book_id = row[0]
            self._metadata_cache[book_id] = self.row_to_dict(row)

        # Same for authors, tags, series, etc.
        # Result: Everything is in RAM!

    @read_api
    def get_metadata(self, book_id):
        '''
        Read from cache (FAST - no disk I/O)
        '''
        return self._metadata_cache.get(book_id)
        # < 0.1ms - just a dict lookup!

    @write_api
    def set_metadata(self, book_id, metadata):
        '''
        Write-through cache pattern:
        1. Update cache (fast)
        2. Write to SQLite (slow but persistent)
        3. Notify listeners (UI updates)
        '''
        # 1. Update cache immediately
        self._metadata_cache[book_id].update(metadata)

        # 2. Write to disk (async, doesn't block)
        with self.write_lock:
            self.backend.update_book(book_id, metadata)
            self.backend.conn.commit()

        # 3. Notify anyone listening
        self.notify_listeners('metadata_changed', book_id)
```

---

### Database Schema (Normalized)

**File:** `metadata.db` (SQLite)

```
┌──────────┐
│  books   │──┐
├──────────┤  │
│ id (PK)  │  │
│ title    │  │
│ path     │  │
│ ...      │  │
└──────────┘  │
              │
    ┌─────────┴──────────────────┐
    │                            │
    ▼                            ▼
┌─────────────────┐      ┌─────────────────┐
│books_authors_   │      │books_tags_link  │
│     link        │      │                 │
├─────────────────┤      ├─────────────────┤
│ book (FK)       │      │ book (FK)       │
│ author (FK)     │      │ tag (FK)        │
└────┬────────────┘      └────┬────────────┘
     │                        │
     ▼                        ▼
┌──────────┐             ┌──────────┐
│ authors  │             │   tags   │
├──────────┤             ├──────────┤
│ id (PK)  │             │ id (PK)  │
│ name     │             │ name     │
│ sort     │             │          │
└──────────┘             └──────────┘
```

**Why normalized?**
- Avoid data duplication
- Maintain referential integrity
- Enable complex queries
- Standard relational design

🌉 **Bridge to Prisma:**
```prisma
model Book {
  id      Int      @id
  title   String
  authors Author[] @relation("BookAuthors")  // Many-to-many
  tags    Tag[]    @relation("BookTags")
}

model Author {
  id    Int    @id
  name  String
  books Book[] @relation("BookAuthors")
}

model Tag {
  id    Int    @id
  name  String
  books Book[] @relation("BookTags")
}
```

Same structure!

---

<a name="plugin-system"></a>
## 5. Plugin System

**Location:** [src/calibre/customize/](../../src/calibre/customize/)

Calibre has a powerful plugin architecture:

```
┌────────────────────────────────────────────────────────┐
│                   PLUGIN TYPES                          │
├────────────────────────────────────────────────────────┤
│                                                         │
│  Input Plugins        Output Plugins                    │
│  ┌──────────────┐    ┌──────────────┐                  │
│  │ EPUB Input   │    │ MOBI Output  │                  │
│  │ PDF Input    │    │ AZW3 Output  │                  │
│  │ MOBI Input   │    │ EPUB Output  │                  │
│  └──────────────┘    └──────────────┘                  │
│                                                         │
│  Metadata Plugins    Device Plugins                     │
│  ┌──────────────┐    ┌──────────────┐                  │
│  │ Amazon       │    │ Kindle       │                  │
│  │ Google Books │    │ Kobo         │                  │
│  │ Goodreads    │    │ Nook         │                  │
│  └──────────────┘    └──────────────┘                  │
│                                                         │
│  UI Plugins          News Plugins                       │
│  ┌──────────────┐    ┌──────────────┐                  │
│  │ Custom menus │    │ NYT recipe   │                  │
│  │ Extra tools  │    │ Guardian     │                  │
│  └──────────────┘    └──────────────┘                  │
│                                                         │
└────────────────────────────────────────────────────────┘
```

**Example Plugin:**

**File:** [src/calibre/customize/__init__.py](../../src/calibre/customize/__init__.py)

```python
class Plugin:
    '''
    Base class for all plugins.

    Like creating a WordPress plugin or VS Code extension.
    '''

    name = 'My Plugin'
    version = (1, 0, 0)
    author = 'John Doe'

    def initialize(self):
        '''Called when plugin loads'''
        pass

    def run(self, *args):
        '''Main plugin logic'''
        pass

class InputFormatPlugin(Plugin):
    '''Convert from a format to internal OEB format'''

    file_types = {'epub', 'mobi'}  # Supported formats

    def convert(self, stream, options):
        '''
        Convert e-book file to internal format.

        stream: File-like object
        options: User settings
        '''
        # Parse EPUB
        # Extract content, metadata, images
        # Return OEB book object
        pass
```

**Plugin Discovery:**

Plugins can be:
1. **Built-in:** [src/calibre/customize/builtins.py](../../src/calibre/customize/builtins.py)
2. **User-installed:** `~/.config/calibre/plugins/`
3. **Loaded dynamically** at runtime

---

<a name="comparison"></a>
## 6. Comparison to Modern Architectures

### Calibre vs Next.js App

| **Aspect** | **Calibre** | **Next.js 15** |
|-----------|-------------|----------------|
| **Frontend** | PyQt6 (desktop) + HTML (web) | React 19 (web) |
| **Backend** | Custom HTTP server | Node.js built-in |
| **Database** | SQLite + In-memory cache | PostgreSQL + Prisma |
| **API** | Custom @endpoint decorator | File-based routing |
| **State** | Qt signals/slots + cache | React state + TanStack Query |
| **Async** | select() event loop | async/await + event loop |
| **Build** | setuptools | Turbopack |
| **Deploy** | Standalone binary | Vercel / Docker |

---

### Calibre vs Electron App

| **Aspect** | **Calibre** | **Electron** |
|-----------|-------------|--------------|
| **Desktop UI** | Native Qt widgets | Chromium + HTML/CSS |
| **Performance** | Very fast (native) | Good (web-based) |
| **Bundle Size** | ~100MB | ~200MB+ |
| **Memory** | Low (~100MB) | Higher (~300MB+) |
| **Look & Feel** | Native OS look | Custom web UI |
| **Development** | Python + Qt | JavaScript + HTML |
| **Cross-platform** | Windows, Mac, Linux | Windows, Mac, Linux |

---

<a name="design-decisions"></a>
## 7. Design Decisions & Trade-offs

### Why Python?

**Chosen because:**
- ✅ Excellent for text processing (e-books are text!)
- ✅ Rich ecosystem (lxml, BeautifulSoup, Pillow)
- ✅ Cross-platform
- ✅ Easy to read and maintain
- ✅ Good for gluing C libraries

**Trade-offs:**
- ⚠️ Slower than C++ (but fast enough)
- ⚠️ GIL limits multi-threading (but I/O-bound anyway)

---

### Why Qt (not Electron)?

**Chosen because:**
- ✅ Native performance
- ✅ Small binary size
- ✅ True native widgets
- ✅ Mature and stable (since 1995)
- ✅ Low memory usage

**Trade-offs:**
- ⚠️ Harder to develop (vs web tech)
- ⚠️ Smaller developer pool
- ⚠️ Qt Designer learning curve

**For modern rewrite:** Consider Electron/Tauri for easier development, but accept larger bundle.

---

### Why SQLite (not PostgreSQL)?

**Chosen because:**
- ✅ Zero configuration (no server!)
- ✅ Portable (library = single folder)
- ✅ Fast for single-user
- ✅ ACID compliant
- ✅ Embedded in app

**Trade-offs:**
- ⚠️ One writer at a time
- ⚠️ Not ideal for multi-user
- ⚠️ Limited concurrent writes

**Perfect for:** Desktop apps, personal libraries
**Not for:** Multi-user web apps (use PostgreSQL)

---

### Why In-Memory Cache?

**Chosen because:**
- ✅ Blazing fast reads (< 1ms)
- ✅ Simple to implement
- ✅ Perfect for desktop (single user)
- ✅ No network overhead

**Trade-offs:**
- ⚠️ Higher memory usage (~100MB for 10k books)
- ⚠️ Startup time (load database)
- ⚠️ Not scalable to web (multiple users)

**For modern rewrite:** Use Redis or PostgreSQL directly for multi-user.

---

## 🎯 Key Architectural Insights

### 1. Separation of Concerns

```
Presentation: How it looks (PyQt6 / HTML)
Logic:        What it does (ebooks/, db/)
Data:         Where it's stored (SQLite)
```

Clean separation = easier to:
- Test each layer independently
- Swap UI (desktop → web)
- Change database (SQLite → PostgreSQL)

---

### 2. Single Responsibility

Each module has ONE job:
- `db/cache.py` - Cache management
- `ebooks/conversion/` - Format conversion
- `srv/routes.py` - HTTP routing
- `gui2/library/` - Book list display

This is **SOLID principles** in action.

---

### 3. Dependency Injection

The `Context` object:

```python
class Context:
    def __init__(self, db, user, config):
        self.db = db        # Injected!
        self.user = user    # Injected!
        self.config = config # Injected!

@endpoint('/api/books')
def get_books(ctx, rd):
    # ctx.db is ready to use - no global state!
    books = ctx.db.all_book_ids()
    return books
```

**Benefits:**
- Easy to test (inject mock db)
- No global variables
- Explicit dependencies

---

### 4. Event-Driven Architecture

**File:** [src/calibre/db/listeners.py](../../src/calibre/db/listeners.py)

```python
class EventDispatcher:
    '''
    Notify interested parties when things change.

    Like Redux actions or React Context updates.
    '''

    def __init__(self):
        self.listeners = defaultdict(list)

    def add_listener(self, event, callback):
        '''Register interest in an event'''
        self.listeners[event].append(callback)

    def dispatch(self, event, *args):
        '''Fire event to all listeners'''
        for callback in self.listeners[event]:
            callback(*args)

# Usage:
dispatcher.add_listener('book_added', update_ui)
dispatcher.add_listener('book_added', rebuild_search_index)
dispatcher.add_listener('book_added', sync_to_device)

# When book added:
dispatcher.dispatch('book_added', book_id)
# → All three listeners get called!
```

**This is the Observer pattern** - decouples components.

---

## 📖 Further Reading

**Deep Dives:**
- [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) - Server internals
- [DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md) - Data layer
- [FRONTEND_ARCHITECTURE.md](./FRONTEND_ARCHITECTURE.md) - PyQt6 GUI
- [CODE_TOURS.md](./CODE_TOURS.md) - Step-by-step code walkthroughs

**Comparisons:**
- [REWRITE_ANALYSIS.md](./REWRITE_ANALYSIS.md) - React/Next.js feasibility
- [TECH_STACK_RESEARCH.md](./TECH_STACK_RESEARCH.md) - Current vs modern tech

**Practical:**
- [HOW_TO_GUIDE.md](./HOW_TO_GUIDE.md) - Common tasks
- [FIRST_CONTRIBUTIONS.md](./FIRST_CONTRIBUTIONS.md) - Your first PR

---

## ✅ Self-Check: Do You Understand?

1. **What are the three tiers?**
   - Presentation (GUI/Web), Logic (Core), Data (SQLite)

2. **How do desktop and web share code?**
   - Both call the same core library functions

3. **Why is the cache so fast?**
   - Everything is in RAM (Python dicts)

4. **What's the write-through pattern?**
   - Update cache + SQLite on every write

5. **How do plugins work?**
   - Inherit from base Plugin class, discovered at runtime

---

**Next:** Choose your path:
- 🔧 **Technical:** [CODE_TOURS.md](./CODE_TOURS.md) - Follow actual code
- 🎨 **Frontend:** [FRONTEND_ARCHITECTURE.md](./FRONTEND_ARCHITECTURE.md) - Learn PyQt6
- 🗄️ **Backend:** [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) - Deep dive server
- 🎯 **Practical:** [HOW_TO_GUIDE.md](./HOW_TO_GUIDE.md) - Start coding!
