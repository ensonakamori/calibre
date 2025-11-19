# Frequently Asked Questions

**Purpose:** Quick answers to common questions from React developers

**Updated:** November 19, 2025

---

## General Questions

### Q: I'm a React developer - can I contribute to Calibre?

**A:** Absolutely! While Calibre is Python/Qt, the concepts are similar:
- Components → Widgets
- State → Qt Signals/Slots + Database Cache
- API Routes → @endpoint decorators
- Database → SQLite with ORM

Start with our [React Developer Guide](./README.md) and you'll be productive within a week.

**Quick Start Path:**
1. Read [GETTING_STARTED.md](./GETTING_STARTED.md) (30 min)
2. Follow [CODE_TOURS.md](./CODE_TOURS.md) (2 hours)
3. Try [FIRST_CONTRIBUTIONS.md](./FIRST_CONTRIBUTIONS.md) (1 week)

---

### Q: Do I need to learn Python before contributing?

**A:** Not really! Python is **very similar** to JavaScript:

```javascript
// JavaScript
const books = data.filter(b => b.author === 'Orwell');
books.forEach(book => console.log(book.title));
```

```python
# Python (almost the same!)
books = [b for b in data if b.author == 'Orwell']
for book in books:
    print(book.title)
```

**Key Differences:**
- Indentation matters (no braces `{}`)
- `snake_case` not `camelCase`
- `self` instead of `this`

That's it! You can read Python code immediately.

**Resources:**
- [Python for JavaScript Developers](https://www.valentinog.com/blog/python-js/)
- [Real Python Tutorial](https://realpython.com/python-first-steps/)

---

### Q: Can Calibre be rewritten in React/Next.js?

**A:** Yes! See our detailed [REWRITE_ANALYSIS.md](./REWRITE_ANALYSIS.md).

**TL;DR:**
- **Web Interface:** ✅ Easy to rewrite (6-12 months)
- **Desktop App:** ⚠️ Complex (18-24 months, use Electron/Tauri)
- **E-book Processing:** ❌ Keep Python (no JS equivalent)

**Recommended:** Hybrid approach
- Modernize web interface with Next.js
- Keep desktop PyQt6 (works great!)
- Share backend logic

---

## Technical Questions

### Q: Why does Calibre use SQLite instead of PostgreSQL?

**A:** SQLite is **perfect for desktop apps**:

✅ **Advantages:**
- Zero configuration (no server!)
- Portable (library = single folder)
- Fast for single user
- Entire database in one file

⚠️ **Trade-offs:**
- Not great for multiple users
- One writer at a time

For **web app** with many users → use PostgreSQL
For **desktop app** (Calibre) → SQLite is ideal

**File:** [metadata.db](../../metadata.db) in your library folder

---

### Q: How does the in-memory cache work?

**A:** Like Redux + Prisma combined:

```
User Request
     ↓
Check Cache (RAM) ← Super fast (< 1ms)
     ├─ HIT? → Return immediately
     └─ MISS? → Query SQLite → Cache it → Return
```

**Benefits:**
- Reads are instant (cached)
- Writes update both cache + SQLite
- Entire library in RAM (~100MB for 10k books)

**Code:** [src/calibre/db/cache.py](../../src/calibre/db/cache.py)

**Like:**
```typescript
// React Query (caches API responses)
const { data } = useQuery(['books'], fetchBooks);
// → Fast on subsequent calls (cached!)
```

---

### Q: What's the difference between the desktop and web UI?

**A:** Same core, different interfaces:

| Feature | Desktop (PyQt6) | Web (Browser) |
|---------|----------------|---------------|
| **UI Framework** | Native Qt widgets | HTML/CSS/JS |
| **Features** | ALL features | Essential features |
| **Performance** | Very fast | Good |
| **Use Case** | Power users, offline | Remote access, any device |
| **Code Location** | [gui2/](../../src/calibre/gui2/) | [srv/](../../src/calibre/srv/) |

Both call the **same backend functions**:
```python
# Same function serves both!
db.get_metadata(book_id)
```

See: [ARCHITECTURE_OVERVIEW.md](./ARCHITECTURE_OVERVIEW.md#desktop-vs-web)

---

### Q: How do I add a new API endpoint?

**A:** Three steps:

```python
# 1. File: src/calibre/srv/ajax.py
from calibre.srv.routes import endpoint

@endpoint('/api/my-endpoint/{param}', types={'param': int})
def my_handler(ctx, rd, param):
    '''Handle the request'''
    result = ctx.db.do_something(param)
    return {'result': result}

# 2. Test it
# curl http://localhost:8080/api/my-endpoint/42

# 3. Use from React
// fetch('/api/my-endpoint/42').then(res => res.json())
```

**Full Guide:** [HOW_TO_GUIDE.md](./HOW_TO_GUIDE.md#new-endpoint)

---

### Q: How do I debug Python code?

**A:** Just like JavaScript, but use `pdb`:

```python
# Add breakpoint
import pdb; pdb.set_trace()

# When it hits:
(Pdb) variable_name      # Print variable
(Pdb) n                  # Next line
(Pdb) s                  # Step into
(Pdb) c                  # Continue
(Pdb) l                  # List code
```

**Or use VS Code:**
- Set breakpoint (click line number)
- Press F5
- Step through code visually

**Guide:** [HOW_TO_GUIDE.md](./HOW_TO_GUIDE.md#debug-pdb)

---

### Q: Where is the book data stored?

**A:** Two places:

**1. Metadata (in database):**
```
/library/metadata.db  ← SQLite database
  └─ books table
  └─ authors table
  └─ tags table
  └─ ... 20+ more tables
```

**2. Actual Files (on filesystem):**
```
/library/
  Author Name/
    Book Title (1)/
      book.epub       ← The actual e-book
      cover.jpg       ← Cover image
      metadata.opf    ← OPF metadata
```

**Why separate?**
- Database = Fast search, indexing
- Filesystem = Actual book files (large)

---

## Development Questions

### Q: How do I run tests?

**A:**

```bash
# All tests
python setup.py test

# Specific file
python -m pytest src/calibre/db/tests/test_cache.py

# Specific test
python -m pytest src/calibre/db/tests/test_cache.py::TestCache::test_get_metadata

# With output
python -m pytest -v src/calibre/db/tests/
```

**Write your own:**

```python
import unittest
from calibre.db.tests.base import BaseTest

class MyTest(BaseTest):
    def test_something(self):
        db = self.init_db()
        # Test logic here
        self.assertEqual(1 + 1, 2)
```

**Guide:** [TESTING_GUIDE.md](./TESTING_GUIDE.md) (to be created)

---

### Q: Where should I put my code?

**A:** Follow these patterns:

| **What** | **Where** | **Example** |
|----------|-----------|-------------|
| API endpoint | `srv/ajax.py` or `srv/books.py` | [srv/ajax.py](../../src/calibre/srv/ajax.py) |
| Database method | `db/cache.py` | [db/cache.py](../../src/calibre/db/cache.py) |
| E-book processing | `ebooks/` | [ebooks/](../../src/calibre/ebooks/) |
| Desktop GUI | `gui2/` | [gui2/](../../src/calibre/gui2/) |
| Utility function | `utils/` | [utils/](../../src/calibre/utils/) |
| Tests | `*/tests/test_*.py` | [db/tests/](../../src/calibre/db/tests/) |

**Guide:** [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md)

---

### Q: How do I make changes to the GUI?

**A:** Calibre uses **Qt Designer** for UI:

```bash
# 1. Open .ui file in Qt Designer
designer src/calibre/gui2/preferences/look_feel.ui

# 2. Make visual changes
# (drag widgets, edit properties)

# 3. Rebuild (converts .ui to .py)
python setup.py build

# 4. Run Calibre to see changes
calibre
```

**UI Files:** [src/calibre/gui2/**/*.ui](../../src/calibre/gui2/)
**Generated:** `*_ui.py` (don't edit these!)

**React Equivalent:**
```
.ui file     →  JSX template
Qt Designer  →  Figma / visual editor
Widgets      →  React components
```

---

## Architecture Questions

### Q: What is the @endpoint decorator?

**A:** It registers a URL route handler (like Next.js API routes):

```python
@endpoint(
    '/api/books/{book_id}',  # URL pattern
    types={'book_id': int},  # Type validation
    methods=('GET', 'POST'), # Allowed methods
    auth_required=True       # Require login
)
def handle_books(ctx, rd, book_id):
    # ctx = context (has db, user, config)
    # rd = request data (headers, cookies)
    # book_id = extracted from URL, type-checked
    pass
```

**Next.js Equivalent:**
```typescript
// app/api/books/[id]/route.ts
export async function GET(
  req: Request,
  { params }: { params: { id: string } }
) { ... }
```

**Code:** [src/calibre/srv/routes.py:57](../../src/calibre/srv/routes.py#L57)

---

### Q: What is Qt and PyQt6?

**A:**

**Qt** = Cross-platform GUI framework (C++)
**PyQt6** = Python bindings for Qt

Think of it as:
- Qt → Like React Native (cross-platform UI)
- PyQt6 → Like react-native-web (bridge to Python)

**Widgets** = Components
```python
# PyQt6 (like React)
class BookList(QWidget):
    def __init__(self):
        super().__init__()
        self.setup_ui()

    def setup_ui(self):
        layout = QVBoxLayout()
        button = QPushButton("Add Book")
        button.clicked.connect(self.on_add_book)
        layout.addWidget(button)
```

```javascript
// React equivalent
function BookList() {
  const handleAddBook = () => { ... };

  return (
    <div>
      <button onClick={handleAddBook}>Add Book</button>
    </div>
  );
}
```

---

### Q: How does the plugin system work?

**A:** Plugins **extend Calibre's functionality**:

```python
from calibre.customize import Plugin

class MyPlugin(Plugin):
    name = 'My Custom Plugin'
    version = (1, 0, 0)

    def run(self, path):
        # Do something with the book file
        pass
```

**Types of Plugins:**
- Input/Output formats (EPUB → MOBI)
- Metadata sources (fetch from Amazon, Goodreads)
- Device drivers (Kindle, Kobo)
- UI enhancements (new menu items, tools)

**Location:** [src/calibre/customize/](../../src/calibre/customize/)

**Like:**
- WordPress plugins
- VS Code extensions
- Browser extensions

---

## Rewrite Questions

### Q: Should I rewrite Calibre in React/Next.js?

**A:** Depends on your goals!

**FOR:**
- ✅ Modern developer experience
- ✅ Easier to hire React devs
- ✅ Beautiful modern UI possible
- ✅ Mobile-friendly web interface
- ✅ Hot reload, better tooling

**AGAINST:**
- ⚠️ Massive effort (18-24 months full rewrite)
- ⚠️ E-book processing has no JS equivalent
- ⚠️ Current PyQt6 is fast and works great
- ⚠️ Need to maintain both during transition

**Recommended Approach:**
1. **Phase 1:** Modernize **web interface only** with Next.js (6-12 months)
2. **Phase 2:** Evaluate if desktop rewrite is worth it
3. **Phase 3:** Maybe Electron, or keep hybrid

**Full Analysis:** [REWRITE_ANALYSIS.md](./REWRITE_ANALYSIS.md)

---

### Q: What would a modern stack look like?

**A:**

```
Frontend:     React 19 + TypeScript
Framework:    Next.js 15 (App Router)
Styling:      Tailwind CSS
State:        Zustand + TanStack Query
Backend:      Next.js API Routes + tRPC
Database:     Prisma + PostgreSQL (or SQLite)
Auth:         NextAuth.js
Testing:      Vitest + Playwright
Desktop:      Keep PyQt6 or use Electron/Tauri
Processing:   Keep Python (expose as API)
```

**Time:** 6-12 months (web only), 18-24 months (full app)
**Team:** 2-4 developers

---

## Contributing Questions

### Q: How do I get my PR merged?

**A:**

**Required:**
1. ✅ Tests pass
2. ✅ Code follows style guide
3. ✅ No breaking changes (or documented)
4. ✅ Addresses real user need

**Recommended:**
5. ✅ Add tests for new code
6. ✅ Update documentation
7. ✅ Clear commit messages

**Process:**
1. Submit PR
2. Automated checks run
3. Maintainer reviews (1-7 days)
4. Address feedback
5. Approval + merge!

**Guide:** [FIRST_CONTRIBUTIONS.md](./FIRST_CONTRIBUTIONS.md)

---

### Q: What's a good first contribution?

**A:** Start small, build confidence:

**Level 1 (⭐☆☆☆☆):** Documentation
- Fix typos
- Add docstrings
- Improve comments

**Level 2 (⭐⭐☆☆☆):** Type hints
- Add missing type annotations
- Improve IDE experience

**Level 3 (⭐⭐⭐☆☆):** Tests
- Add unit tests for existing code
- Increase code coverage

**Level 4 (⭐⭐⭐⭐☆):** Features
- Add new API endpoint
- Add search filter
- Improve error messages

**Examples:** [FIRST_CONTRIBUTIONS.md](./FIRST_CONTRIBUTIONS.md#quick-wins)

---

### Q: Where can I get help?

**A:**

- **Documentation:** [docs/learning/](.) - Start here!
- **Forum:** [MobileRead](https://www.mobileread.com/forums/forumdisplay.php?f=166)
- **GitHub:** Comment on issues/PRs
- **Bug Tracker:** [Launchpad](https://bugs.launchpad.net/calibre)

**Best Practices:**
- Search existing issues first
- Provide minimal reproduction steps
- Include error messages
- Describe what you tried

---

## Learning Path Questions

### Q: What should I learn first?

**A:** Follow this path:

**Week 1: Understanding**
- [README.md](./README.md) - Overview
- [GETTING_STARTED.md](./GETTING_STARTED.md) - Setup
- [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md) - Navigate codebase

**Week 2: Architecture**
- [ARCHITECTURE_OVERVIEW.md](./ARCHITECTURE_OVERVIEW.md) - System design
- [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) - Server
- [DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md) - Data

**Week 3: Practice**
- [CODE_TOURS.md](./CODE_TOURS.md) - Follow requests
- [HOW_TO_GUIDE.md](./HOW_TO_GUIDE.md) - Common tasks
- [FIRST_CONTRIBUTIONS.md](./FIRST_CONTRIBUTIONS.md) - Start contributing!

**Week 4: Contribute!**
- Pick a good first issue
- Make changes
- Submit PR
- Celebrate! 🎉

---

### Q: I'm overwhelmed - where do I start?

**A:** Don't try to learn everything!

**If you want to...**

**...add web features:** → [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) only
**...understand data:** → [DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md) only
**...modify desktop UI:** → [FRONTEND_ARCHITECTURE.md](./FRONTEND_ARCHITECTURE.md) (to be created)
**...just contribute:** → [FIRST_CONTRIBUTIONS.md](./FIRST_CONTRIBUTIONS.md) only

**Start small.** One document. One task. One PR.

---

## Performance Questions

### Q: Why is Calibre so fast?

**A:** Three reasons:

**1. In-Memory Cache**
- Entire database in RAM
- Reads are instant (< 1ms)

**2. Native Code**
- Qt is C++ (compiled, not interpreted)
- Python for logic, C++ for heavy lifting

**3. Smart Design**
- Lazy loading
- Efficient data structures
- Optimized algorithms

**Comparison:**
- Calibre (cached): ~1ms to get book
- Typical web app: ~50ms (network + DB)

---

### Q: Can I make Calibre faster?

**A:** It's already very optimized, but:

**Possible Improvements:**
- Use modern Python 3.13 (JIT compiler)
- Optimize specific slow queries
- Add more caching layers
- Use Rust for performance-critical parts

**Where to focus:**
- E-book conversion (already multi-threaded)
- Full-text search (already uses FTS5)
- Large library loading (already optimized)

**Most impact:** Improving **perceived** performance (progress bars, async operations, better UX)

---

## Miscellaneous

### Q: What license is Calibre?

**A:** **GPL v3** (open source)

**Means:**
- ✅ Free to use, modify, distribute
- ✅ Can use in commercial products
- ⚠️ Must share source code if distributing
- ⚠️ Derived works must also be GPL v3

**File:** [LICENSE](../../LICENSE)

---

### Q: How big is Calibre?

**A:**

- **Python Code:** ~600,000 lines
- **Desktop GUI:** ~300,000 lines
- **E-book Processing:** ~150,000 lines
- **Backend/Database:** ~80,000 lines
- **Tests:** ~50,000 lines

**Files:** ~2,000 Python files
**Contributors:** 100+
**Since:** 2006 (19 years!)

---

### Q: Who maintains Calibre?

**A:**

**Creator:** Kovid Goyal (since 2006)
**Contributors:** 100+ developers worldwide
**Community:** Thousands of users, plugin developers

**It's a mature, actively maintained project!**

---

## Still Have Questions?

**Ask in:**
- [MobileRead Forum](https://www.mobileread.com/forums/forumdisplay.php?f=166)
- GitHub Issues (for code questions)
- This documentation (submit PR to add FAQ!)

**Remember:** No question is too basic. We all started as beginners!

---

**Updated:** November 19, 2025 | **Next Review:** February 2026
