# Database Architecture: Complete Guide for React Developers

**Purpose:** Understand how Calibre stores, queries, and manages data

**For:** React developers new to databases and SQL

**Time to Read:** 45-60 minutes

---

## Table of Contents

1. [Database Fundamentals](#database-fundamentals)
2. [SQLite Overview](#sqlite-overview)
3. [Calibre's Schema](#calibre-schema)
4. [The Cache Layer (In-Memory ORM)](#cache-layer)
5. [Query Patterns](#query-patterns)
6. [Full-Text Search](#full-text-search)
7. [Comparison to Modern ORMs](#comparison)

---

<a name="database-fundamentals"></a>
## 1. Database Fundamentals

### 🧠 **Mental Model: What IS a Database?**

As a React developer, you've probably used state management:

```javascript
// React state - in-memory only, lost on refresh
const [books, setBooks] = useState([
  { id: 1, title: "1984", author: "Orwell" },
  { id: 2, title: "Dune", author: "Herbert" }
]);
```

A database is like **persistent state**:
- ✅ Survives app restarts
- ✅ Handles millions of records efficiently
- ✅ Supports complex queries (filtering, joining, sorting)
- ✅ Concurrent access (multiple users)
- ✅ ACID guarantees (Atomicity, Consistency, Isolation, Durability)

🌉 **Bridge from React:**
```
useState()              →  Temporary data (RAM)
localStorage            →  Simple persistence (key-value)
IndexedDB               →  Client-side database (NoSQL)
SQLite/PostgreSQL       →  Server-side database (SQL)
```

---

### SQL Basics for React Developers

**SQL** = Structured Query Language = How you talk to databases

#### Comparison: Array Methods vs SQL

```javascript
// JavaScript Array Methods
books
  .filter(b => b.genre === 'scifi')
  .map(b => ({ id: b.id, title: b.title }))
  .sort((a, b) => a.title.localeCompare(b.title))
  .slice(0, 10)
```

```sql
-- SQL Query (same logic)
SELECT id, title
FROM books
WHERE genre = 'scifi'
ORDER BY title
LIMIT 10
```

#### Common SQL Operations

```sql
-- CREATE: Insert new book
INSERT INTO books (title, author_id)
VALUES ('1984', 42);

-- READ: Get all books
SELECT * FROM books;

-- UPDATE: Change book title
UPDATE books
SET title = 'Nineteen Eighty-Four'
WHERE id = 1;

-- DELETE: Remove book
DELETE FROM books
WHERE id = 1;
```

🌉 **Bridge to React:**
```javascript
// CRUD in React state
const [books, setBooks] = useState([]);

// CREATE
setBooks([...books, newBook]);

// READ
const book = books.find(b => b.id === 1);

// UPDATE
setBooks(books.map(b => b.id === 1 ? {...b, title: 'New'} : b));

// DELETE
setBooks(books.filter(b => b.id !== 1));
```

Same operations, different syntax!

---

<a name="sqlite-overview"></a>
## 2. SQLite Overview

### What is SQLite?

**SQLite** = A complete SQL database in a single file

🧠 **Mental Model:**
```
PostgreSQL/MySQL    →  Like a separate server (needs installation, config)
SQLite              →  Like a JSON file, but with SQL superpowers
```

**Calibre uses:** `/path/to/library/metadata.db` (single file)

### SQLite vs PostgreSQL vs MongoDB

| **Feature** | **SQLite** | **PostgreSQL** | **MongoDB** |
|------------|-----------|---------------|------------|
| **Type** | File-based SQL | Server-based SQL | Document NoSQL |
| **Setup** | Zero config | Server install | Server install |
| **Size** | ~1 MB library | ~50 MB | ~100 MB |
| **Concurrent Writes** | 1 at a time | Many | Many |
| **Use Case** | Embedded, desktop apps | Web apps, microservices | Real-time, flexible schema |
| **Best For** | Single user, local | Multi-user, production | Document storage, scaling |

**Why Calibre uses SQLite:**
- ✅ **Portable:** Library = single folder with DB file
- ✅ **Zero setup:** No database server to install
- ✅ **Fast for reads:** Excellent for single-user desktop app
- ✅ **Reliable:** Mature, battle-tested (since 2000)
- ✅ **Embedded:** Runs inside the Python app

---

### APSW (Another Python SQLite Wrapper)

**File:** Calibre uses [APSW](https://github.com/rogerbinns/apsw) to talk to SQLite

```python
# Python code talking to SQLite via APSW
import apsw

# Connect to database
connection = apsw.Connection('metadata.db')

# Execute query
cursor = connection.execute("SELECT * FROM books WHERE id = ?", (42,))
row = cursor.fetchone()

# row = (42, "1984", "George Orwell", ...)
```

🌉 **Bridge to JavaScript:**
```javascript
// Node.js talking to PostgreSQL via pg
import { Client } from 'pg';

const client = new Client({ connectionString: '...' });
await client.connect();

const result = await client.query("SELECT * FROM books WHERE id = $1", [42]);
const row = result.rows[0];
```

Same pattern!

---

<a name="calibre-schema"></a>
## 3. Calibre's Schema

### Core Tables

#### Books Table (Main entity)

```sql
CREATE TABLE books (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    sort TEXT,  -- Sortable title: "1984" → "1984", "The Great Gatsby" → "Great Gatsby, The"
    timestamp TEXT DEFAULT CURRENT_TIMESTAMP,  -- When added
    pubdate TEXT DEFAULT "2000-01-01",  -- Publication date
    series_index REAL DEFAULT 1.0,  -- Book #3 in series
    author_sort TEXT,  -- "Orwell, George"
    isbn TEXT DEFAULT "",
    path TEXT DEFAULT "",  -- Folder path: "George Orwell/1984 (1)"
    flags INTEGER DEFAULT 1,
    uuid TEXT,  -- Unique identifier
    has_cover BOOL DEFAULT 0,
    last_modified TEXT DEFAULT CURRENT_TIMESTAMP
);
```

🌉 **Bridge to Prisma:**
```prisma
model Book {
  id            Int      @id @default(autoincrement())
  title         String
  sort          String?
  timestamp     DateTime @default(now())
  pubdate       DateTime?
  seriesIndex   Float    @default(1.0)
  authorSort    String?
  isbn          String?
  path          String?
  flags         Int      @default(1)
  uuid          String   @unique
  hasCover      Boolean  @default(false)
  lastModified  DateTime @updatedAt
}
```

---

#### Authors Table

```sql
CREATE TABLE authors (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,  -- "George Orwell"
    sort TEXT,  -- "Orwell, George"
    link TEXT DEFAULT ""  -- URL to author's website
);
```

---

#### Many-to-Many: Books ↔ Authors

**Problem:** A book can have multiple authors, an author can write multiple books

**Solution:** Junction table

```sql
CREATE TABLE books_authors_link (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,  -- Foreign key to books.id
    author INTEGER NOT NULL,  -- Foreign key to authors.id
    UNIQUE(book, author)  -- Prevent duplicates
);

-- Example data:
-- book=1, author=5  → "1984" by "George Orwell"
-- book=2, author=5  → "Animal Farm" by "George Orwell"
-- book=2, author=6  → "Animal Farm" co-authored with another author
```

🌉 **Bridge to Prisma:**
```prisma
model Book {
  id      Int      @id
  authors Author[] @relation("BookAuthors")  // Many-to-many
}

model Author {
  id    Int    @id
  books Book[] @relation("BookAuthors")
}
```

Prisma hides the junction table, but it's there!

---

#### Other Tables

```sql
-- Tags (many-to-many with books)
CREATE TABLE tags (id INTEGER PRIMARY KEY, name TEXT);
CREATE TABLE books_tags_link (book INTEGER, tag INTEGER);

-- Series (one-to-many)
CREATE TABLE series (id INTEGER PRIMARY KEY, name TEXT, sort TEXT);
-- Books have series_id foreign key

-- Publishers (one-to-many)
CREATE TABLE publishers (id INTEGER PRIMARY KEY, name TEXT, sort TEXT);

-- Comments (one-to-one with books)
CREATE TABLE comments (id INTEGER PRIMARY KEY, book INTEGER, text TEXT);

-- Identifiers (one-to-many: ISBN, ASIN, etc.)
CREATE TABLE identifiers (
    id INTEGER PRIMARY KEY,
    book INTEGER,
    type TEXT,  -- 'isbn', 'asin', 'goodreads'
    val TEXT
);

-- Custom columns (user-defined metadata)
CREATE TABLE custom_column_1 (id INTEGER PRIMARY KEY, book INTEGER, value TEXT);
```

---

### Visual Schema Diagram

```
┌─────────────┐          ┌──────────────────────┐          ┌─────────────┐
│   authors   │          │ books_authors_link   │          │    books    │
├─────────────┤          ├──────────────────────┤          ├─────────────┤
│ id (PK)     │◄─────────│ author (FK)          │          │ id (PK)     │
│ name        │          │ book (FK)            │─────────►│ title       │
│ sort        │          └──────────────────────┘          │ timestamp   │
│ link        │                                            │ path        │
└─────────────┘                                            │ uuid        │
                                                           └──────┬──────┘
┌─────────────┐          ┌──────────────────────┐               │
│    tags     │          │  books_tags_link     │               │
├─────────────┤          ├──────────────────────┤               │
│ id (PK)     │◄─────────│ tag (FK)             │               │
│ name        │          │ book (FK)            │───────────────┘
└─────────────┘          └──────────────────────┘

┌─────────────┐          ┌──────────────────────┐
│   series    │          │       books          │
├─────────────┤          ├──────────────────────┤
│ id (PK)     │◄─────────│ series (FK)          │
│ name        │          │ series_index         │
└─────────────┘          └──────────────────────┘

┌─────────────┐          ┌──────────────────────┐
│  comments   │          │       books          │
├─────────────┤          ├──────────────────────┤
│ id (PK)     │          │ id (PK)              │◄────┐
│ book (FK)   │──────────┤                      │     │
│ text        │          └──────────────────────┘     │
└─────────────┘                                       │
                         ┌──────────────────────┐     │
                         │   identifiers        │     │
                         ├──────────────────────┤     │
                         │ book (FK)            │─────┘
                         │ type                 │
                         │ val                  │
                         └──────────────────────┘
```

---

<a name="cache-layer"></a>
## 4. The Cache Layer (In-Memory ORM)

### Why Cache?

**Problem:** Querying SQLite is slow (disk I/O)
**Solution:** Load entire database into RAM

**File:** [src/calibre/db/cache.py](../../src/calibre/db/cache.py)

```python
class Cache:
    '''
    An in-memory cache of metadata.db.
    All table reading/sorting/searching logic is re-implemented.
    SQLITE is simply used for persistence.
    '''

    def __init__(self, backend):
        self.backend = backend  # SQLite connection
        self.cache = {}  # In-memory data
        self.load_all_books()  # Load on startup

    def load_all_books(self):
        """Load all books from SQLite into memory"""
        for row in self.backend.conn.execute("SELECT * FROM books"):
            book_id = row[0]
            self.cache[book_id] = self.row_to_metadata(row)
```

### Cache Structure

```python
# Simplified cache structure
cache = {
    'books': {
        1: {'id': 1, 'title': '1984', 'author_ids': [5]},
        2: {'id': 2, 'title': 'Dune', 'author_ids': [12]},
        # ... all books in memory
    },
    'authors': {
        5: {'id': 5, 'name': 'George Orwell', 'sort': 'Orwell, George'},
        12: {'id': 12, 'name': 'Frank Herbert', 'sort': 'Herbert, Frank'},
        # ... all authors
    },
    'tags': { ... },
    'series': { ... },
}
```

🌉 **Bridge to React/Redux:**
```javascript
// This is like Redux state!
const state = {
  books: {
    byId: {
      1: { id: 1, title: '1984', authorIds: [5] },
      2: { id: 2, title: 'Dune', authorIds: [12] },
    },
    allIds: [1, 2],
  },
  authors: {
    byId: {
      5: { id: 5, name: 'George Orwell' },
      12: { id: 12, name: 'Frank Herbert' },
    },
    allIds: [5, 12],
  },
};
```

Same normalized structure!

---

### Read vs Write Operations

#### Fast Reads (From Cache)

```python
@read_api
def get_metadata(self, book_id):
    """Get book metadata - super fast, no disk I/O"""
    return self.cache[book_id]  # ← Instant!

@read_api
def all_book_ids(self):
    """Get all book IDs"""
    return list(self.cache.keys())  # ← Instant!

@read_api
def search(self, query):
    """Search books"""
    # Search in-memory cache, not disk
    results = [
        book for book in self.cache.values()
        if query.lower() in book['title'].lower()
    ]
    return results  # ← Fast!
```

#### Slower Writes (Cache + Disk)

```python
@write_api
def set_metadata(self, book_id, metadata):
    """Update book metadata - write to both cache and disk"""

    # 1. Update in-memory cache (fast)
    self.cache[book_id].update(metadata)

    # 2. Write to SQLite (slow, but persistent)
    self.backend.conn.execute(
        "UPDATE books SET title = ? WHERE id = ?",
        (metadata['title'], book_id)
    )
    self.backend.conn.commit()  # ← Disk write!

    # 3. Notify listeners (for UI updates)
    self.notify_listeners('metadata_changed', book_id)
```

🎯 **Remember This:** Write-Through Cache
- **Read:** Memory only (super fast)
- **Write:** Memory + Disk (slower, but safe)

---

<a name="query-patterns"></a>
## 5. Query Patterns

### Simple Queries

```python
# Get single book
@read_api
def get_metadata(self, book_id):
    if book_id not in self.cache:
        raise NoSuchBook(book_id)
    return self.cache[book_id]

# Get all books
@read_api
def all_book_ids(self):
    return list(self.cache.keys())

# Get books with tag
@read_api
def books_with_tag(self, tag_name):
    tag_id = self.tag_name_to_id(tag_name)
    return [
        book_id for book_id, book in self.cache.items()
        if tag_id in book.get('tag_ids', [])
    ]
```

🌉 **Bridge to Prisma:**
```typescript
// Get single book
const book = await prisma.book.findUnique({ where: { id: bookId } });

// Get all books
const books = await prisma.book.findMany();

// Get books with tag
const books = await prisma.book.findMany({
  where: { tags: { some: { name: tagName } } }
});
```

---

### Complex Queries with Joins

**SQL Way (what SQLite does):**
```sql
-- Get books with author names
SELECT
    books.id,
    books.title,
    authors.name AS author_name
FROM books
JOIN books_authors_link ON books.id = books_authors_link.book
JOIN authors ON books_authors_link.author = authors.id
WHERE books.title LIKE '%Dune%';
```

**Calibre Cache Way (in-memory joins):**
```python
def get_books_with_authors(self, title_query):
    results = []
    for book_id, book in self.cache.items():
        if title_query.lower() in book['title'].lower():
            # Manual join in Python
            author_names = [
                self.authors_cache[author_id]['name']
                for author_id in book.get('author_ids', [])
            ]
            results.append({
                'id': book_id,
                'title': book['title'],
                'authors': author_names
            })
    return results
```

🎯 **Key Insight:** Calibre does joins in Python, not SQL!

---

### Sorting and Pagination

```python
def get_sorted_books(self, sort_by='timestamp', limit=50, offset=0):
    # 1. Get all books
    books = list(self.cache.values())

    # 2. Sort (using custom sort key)
    books.sort(key=lambda b: b.get(sort_by, ''))

    # 3. Paginate
    return books[offset:offset + limit]
```

🌉 **Bridge to SQL:**
```sql
SELECT * FROM books
ORDER BY timestamp
LIMIT 50 OFFSET 0;
```

Same logic!

---

<a name="full-text-search"></a>
## 6. Full-Text Search

### FTS5 (SQLite Full-Text Search)

**File:** [src/calibre/db/fts/](../../src/calibre/db/fts/)

Calibre uses SQLite's FTS5 extension for fast text search:

```sql
-- Create FTS virtual table
CREATE VIRTUAL TABLE books_fts USING fts5(
    title,
    authors,
    tags,
    content=books  -- Mirror of books table
);

-- Search across all text fields
SELECT * FROM books_fts
WHERE books_fts MATCH 'orwell AND dystopia'
ORDER BY rank;  -- Ranked by relevance
```

**Python API:**
```python
def search_books(self, query):
    """Full-text search across books"""
    cursor = self.conn.execute(
        "SELECT book_id FROM books_fts WHERE books_fts MATCH ?",
        (query,)
    )
    book_ids = [row[0] for row in cursor]
    return [self.cache[id] for id in book_ids]
```

🌉 **Bridge to Modern Search:**
```javascript
// ElasticSearch/Algolia equivalent
const results = await index.search('orwell AND dystopia', {
  attributesToSearchOn: ['title', 'authors', 'tags'],
  ranking: ['typo', 'words', 'proximity'],
});
```

---

### Search Query Language

Calibre supports advanced search syntax:

```
title:"nineteen eighty" AND author:orwell
genre:scifi AND NOT publisher:"Penguin"
series:"Foundation" AND series_index:>=3
pubdate:2020..2024
rating:>4
```

**Implementation:**
```python
def parse_search_query(query):
    """Parse and execute complex search queries"""
    # Parse query into AST
    ast = SearchQueryParser.parse(query)

    # Execute search
    results = set()
    for condition in ast.conditions:
        field, operator, value = condition
        results &= self.search_field(field, operator, value)

    return results
```

---

<a name="comparison"></a>
## 7. Comparison to Modern ORMs

### Calibre vs Prisma

| **Feature** | **Calibre Cache** | **Prisma** |
|------------|------------------|-----------|
| **Approach** | In-memory cache | Query builder |
| **Reads** | Instant (RAM) | Disk I/O |
| **Writes** | Write-through | Direct to DB |
| **Type Safety** | Python dicts | TypeScript types |
| **Migrations** | Manual SQL | Auto-generated |
| **Relations** | Manual joins | Auto-resolved |
| **Query Language** | Python code | Prisma DSL |

### Example Comparison

**Calibre:**
```python
# Get book with authors and tags
book = cache.get_metadata(42)
book['authors'] = [cache.get_author(id) for id in book['author_ids']]
book['tags'] = [cache.get_tag(id) for id in book['tag_ids']]
```

**Prisma:**
```typescript
// Same query
const book = await prisma.book.findUnique({
  where: { id: 42 },
  include: { authors: true, tags: true }  // Auto-join
});
```

**Trade-offs:**
- Calibre: Faster (in memory), more manual
- Prisma: Easier, type-safe, disk I/O overhead

---

## 🎯 **Key Takeaways**

1. **SQLite is just a file**
   - `metadata.db` contains everything
   - No server needed
   - Perfect for desktop apps

2. **The cache is the ORM**
   - Entire database loaded into RAM
   - Reads are instant
   - Writes go to both cache and disk

3. **Joins happen in Python**
   - Not in SQL
   - Faster for small datasets
   - More flexible

4. **Full-text search is powerful**
   - FTS5 virtual table
   - Advanced query syntax
   - Ranked results

5. **It's normalized data**
   - Just like Redux state
   - Many-to-many with junction tables
   - Same principles as Prisma/TypeORM

---

## 📖 **Further Reading**

- **[BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md)** - How the server works
- **[DATA_FLOW_GUIDE.md](./DATA_FLOW_GUIDE.md)** - Follow data through the system
- **[SQLite Documentation](https://www.sqlite.org/docs.html)** - Official SQLite docs
- **[FTS5 Guide](https://www.sqlite.org/fts5.html)** - Full-text search

---

## ✅ **Self-Check Questions**

1. What's the difference between SQLite and PostgreSQL?
2. Why does Calibre cache the entire database in memory?
3. How are many-to-many relationships stored (e.g., books ↔ authors)?
4. What's a write-through cache?
5. How is Calibre's cache similar to Redux state?

**Answers:**
1. SQLite = file-based, zero config; PostgreSQL = server-based, multi-user
2. Fast reads (no disk I/O), better for desktop single-user apps
3. Junction tables (books_authors_link) with foreign keys
4. Updates go to both cache (fast) and disk (persistent)
5. Normalized data structure with IDs referencing other entities

---

**Next:** [REWRITE_ANALYSIS.md](./REWRITE_ANALYSIS.md) - Can we rebuild this with React/Next.js?
