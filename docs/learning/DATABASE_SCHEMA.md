# Database Schema Documentation

Complete reference for Calibre's SQLite database schema, explaining every table, column, relationship, and index.

## Table of Contents

1. [Overview](#overview)
2. [Core Tables](#core-tables)
3. [Relationship Tables](#relationship-tables)
4. [Metadata Tables](#metadata-tables)
5. [Custom Columns](#custom-columns)
6. [Full-Text Search](#full-text-search)
7. [Indexes and Performance](#indexes-and-performance)
8. [Schema Migrations](#schema-migrations)
9. [SQL Examples](#sql-examples)

---

## Overview

### Database Structure

Calibre uses **SQLite** as its database engine with the following characteristics:

- **Single file**: `metadata.db` in library folder
- **Version**: SQLite 3.50.4+ (via APSW wrapper)
- **Size**: Typically 5-50 MB for 1,000-10,000 books
- **Encoding**: UTF-8
- **Journal Mode**: WAL (Write-Ahead Logging) for concurrent reads

### Architecture Pattern

Calibre's schema follows a **normalized relational design**:

```
Core Data (books table)
    ↓
Many-to-Many Relationships (via link tables)
    ├─ Books ←→ Authors
    ├─ Books ←→ Tags
    ├─ Books ←→ Publishers
    ├─ Books ←→ Series
    ├─ Books ←→ Languages
    └─ Books ←→ Identifiers

Dependent Data (one-to-many)
    ├─ Books → Comments
    ├─ Books → Data (file formats)
    └─ Books → Custom Columns
```

### Entity-Relationship Diagram

```
┌──────────┐       ┌─────────────────────┐       ┌─────────┐
│ authors  │←──────│ books_authors_link  │──────→│  books  │
└──────────┘       └─────────────────────┘       └─────────┘
                                                       ↑
┌──────────┐       ┌─────────────────────┐           │
│   tags   │←──────│   books_tags_link   │───────────┘
└──────────┘       └─────────────────────┘           │
                                                       │
┌──────────┐       ┌─────────────────────┐           │
│ series   │←──────│  books_series_link  │───────────┘
└──────────┘       └─────────────────────┘           │
                                                       │
┌──────────┐       ┌─────────────────────┐           │
│publishers│←──────│books_publishers_link│───────────┘
└──────────┘       └─────────────────────┘           │
                                                       │
┌──────────┐                                          │
│ comments │──────────────────────────────────────────┘
└──────────┘                                          │
                                                       │
┌──────────┐                                          │
│   data   │──────────────────────────────────────────┘
└──────────┘                (formats: EPUB, PDF, etc.)
```

---

## Core Tables

### books

**Purpose**: Central table storing core book metadata

**🔗 Code Reference**: [src/calibre/db/tables.py](../../src/calibre/db/tables.py)

```sql
CREATE TABLE books (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    title           TEXT NOT NULL DEFAULT 'Unknown' COLLATE NOCASE,
    sort            TEXT COLLATE NOCASE,
    timestamp       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    pubdate         TIMESTAMP,
    series_index    REAL NOT NULL DEFAULT 1.0,
    author_sort     TEXT COLLATE NOCASE,
    isbn            TEXT DEFAULT "",
    lccn            TEXT DEFAULT "",
    path            TEXT NOT NULL DEFAULT "",
    flags           INTEGER NOT NULL DEFAULT 1,
    uuid            TEXT,
    has_cover       BOOL DEFAULT 0,
    last_modified   TIMESTAMP NOT NULL DEFAULT "2000-01-01 00:00:00+00:00"
);
```

**Columns Explained**:

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `id` | INTEGER | Primary key, auto-increment | `42` |
| `title` | TEXT | Book title | `"The Great Gatsby"` |
| `sort` | TEXT | Sortable title (articles removed) | `"Great Gatsby, The"` |
| `timestamp` | TIMESTAMP | Date added to library | `2025-01-15 10:30:00` |
| `pubdate` | TIMESTAMP | Publication date | `1925-04-10 00:00:00` |
| `series_index` | REAL | Position in series | `1.0`, `2.5` |
| `author_sort` | TEXT | Sortable author names | `"Fitzgerald, F. Scott"` |
| `isbn` | TEXT | ISBN-10 or ISBN-13 | `"9780743273565"` |
| `lccn` | TEXT | Library of Congress Control Number | `"2004110402"` |
| `path` | TEXT | Relative path to book folder | `"F. Scott Fitzgerald/The Great Gatsby (42)"` |
| `flags` | INTEGER | Bit flags for book state | `1` = visible, `2` = marked, etc. |
| `uuid` | TEXT | Unique identifier (UUID4) | `"a1b2c3d4-e5f6-4a5b-8c9d-0e1f2a3b4c5d"` |
| `has_cover` | BOOL | Whether book has cover image | `1` (true) or `0` (false) |
| `last_modified` | TIMESTAMP | Last metadata change | `2025-11-19 14:22:00` |

**Design Notes**:

- `COLLATE NOCASE`: Case-insensitive sorting
- `series_index` is REAL: Allows fractional indexes (1.5, 2.5 for inserted books)
- `path` is relative to library root
- `flags` uses bit masking for multiple boolean states

**💭 Prisma Schema Equivalent**:
```prisma
model Book {
  id            Int       @id @default(autoincrement())
  title         String    @default("Unknown")
  sort          String?
  timestamp     DateTime  @default(now())
  pubdate       DateTime?
  seriesIndex   Float     @default(1.0)
  authorSort    String?
  isbn          String    @default("")
  lccn          String    @default("")
  path          String    @default("")
  flags         Int       @default(1)
  uuid          String?
  hasCover      Boolean   @default(false)
  lastModified  DateTime  @default(now())

  authors       BookAuthor[]
  tags          BookTag[]
  publishers    BookPublisher[]
  series        BookSeries[]
  comments      Comment?
  data          Data[]
  identifiers   Identifier[]

  @@index([title])
  @@index([authorSort])
  @@index([timestamp])
}
```

### data

**Purpose**: Stores book file format information (EPUB, PDF, etc.)

```sql
CREATE TABLE data (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    book        INTEGER NOT NULL,
    format      TEXT NOT NULL COLLATE NOCASE,
    uncompressed_size INTEGER NOT NULL,
    name        TEXT NOT NULL,

    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE
);
```

**Columns Explained**:

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `id` | INTEGER | Primary key | `1` |
| `book` | INTEGER | Foreign key to books.id | `42` |
| `format` | TEXT | Format type (uppercase) | `"EPUB"`, `"PDF"`, `"MOBI"` |
| `uncompressed_size` | INTEGER | Size in bytes (uncompressed) | `2457600` (2.4 MB) |
| `name` | TEXT | Filename without extension | `"The Great Gatsby"` |

**Example Data**:

```sql
INSERT INTO data VALUES (1, 42, 'EPUB', 1500000, 'The Great Gatsby');
INSERT INTO data VALUES (2, 42, 'PDF', 3000000, 'The Great Gatsby');
INSERT INTO data VALUES (3, 43, 'MOBI', 1200000, 'To Kill a Mockingbird');
```

**File Storage**:

Actual files are stored in:
```
{library_path}/
└─ {author_name}/
   └─ {book_title} ({book_id})/
      ├─ cover.jpg
      ├─ metadata.opf
      ├─ {name}.epub
      ├─ {name}.pdf
      └─ {name}.mobi
```

Example:
```
Calibre Library/
└─ F. Scott Fitzgerald/
   └─ The Great Gatsby (42)/
      ├─ cover.jpg
      ├─ metadata.opf
      ├─ The Great Gatsby.epub
      └─ The Great Gatsby.pdf
```

### comments

**Purpose**: Stores book descriptions/summaries

```sql
CREATE TABLE comments (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    book    INTEGER NOT NULL,
    text    TEXT NOT NULL COLLATE NOCASE,

    UNIQUE(book),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE
);
```

**Columns Explained**:

| Column | Type | Description |
|--------|------|-------------|
| `id` | INTEGER | Primary key |
| `book` | INTEGER | Foreign key to books.id (unique) |
| `text` | TEXT | Description/summary (HTML allowed) |

**Example**:

```sql
INSERT INTO comments VALUES (
    1,
    42,
    '<p>A classic American novel set in the Jazz Age...</p>'
);
```

**Design Notes**:

- One-to-one relationship (one book → one comment)
- `UNIQUE(book)` enforces one comment per book
- HTML is allowed for formatting
- Large text fields (can be several KB)

---

## Relationship Tables

### books_authors_link

**Purpose**: Many-to-many relationship between books and authors

```sql
CREATE TABLE books_authors_link (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    book    INTEGER NOT NULL,
    author  INTEGER NOT NULL,

    UNIQUE(book, author),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(author) REFERENCES authors(id) ON DELETE CASCADE
);
```

**Example Data**:

```sql
-- Book 42 written by authors 10 and 11 (co-authors)
INSERT INTO books_authors_link VALUES (1, 42, 10);
INSERT INTO books_authors_link VALUES (2, 42, 11);

-- Book 43 written by author 10
INSERT INTO books_authors_link VALUES (3, 43, 10);
```

**Query Examples**:

```sql
-- Get all authors for book 42
SELECT a.name
FROM authors a
JOIN books_authors_link bal ON a.id = bal.author
WHERE bal.book = 42
ORDER BY bal.id;  -- Preserves author order

-- Get all books by author 10
SELECT b.title
FROM books b
JOIN books_authors_link bal ON b.id = bal.book
WHERE bal.author = 10;

-- Get book count per author
SELECT a.name, COUNT(bal.book) as book_count
FROM authors a
LEFT JOIN books_authors_link bal ON a.id = bal.author
GROUP BY a.id
ORDER BY book_count DESC;
```

### authors

**Purpose**: Stores unique author names

```sql
CREATE TABLE authors (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    name    TEXT NOT NULL COLLATE NOCASE,
    sort    TEXT COLLATE NOCASE,
    link    TEXT NOT NULL DEFAULT "",

    UNIQUE(name)
);
```

**Columns Explained**:

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `id` | INTEGER | Primary key | `10` |
| `name` | TEXT | Author display name | `"F. Scott Fitzgerald"` |
| `sort` | TEXT | Sortable name | `"Fitzgerald, F. Scott"` |
| `link` | TEXT | URL to author page (optional) | `"https://..."` |

### books_tags_link

**Purpose**: Many-to-many relationship between books and tags

```sql
CREATE TABLE books_tags_link (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    book    INTEGER NOT NULL,
    tag     INTEGER NOT NULL,

    UNIQUE(book, tag),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(tag) REFERENCES tags(id) ON DELETE CASCADE
);
```

### tags

**Purpose**: Stores unique tag names (genres, categories, etc.)

```sql
CREATE TABLE tags (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    name    TEXT NOT NULL COLLATE NOCASE,

    UNIQUE(name)
);
```

**Example Data**:

```sql
INSERT INTO tags VALUES (1, 'Fiction');
INSERT INTO tags VALUES (2, 'Classic');
INSERT INTO tags VALUES (3, 'American Literature');

-- Tag book 42 with multiple tags
INSERT INTO books_tags_link VALUES (1, 42, 1);  -- Fiction
INSERT INTO books_tags_link VALUES (2, 42, 2);  -- Classic
INSERT INTO books_tags_link VALUES (3, 42, 3);  -- American Literature
```

### books_publishers_link

**Purpose**: Many-to-many relationship between books and publishers

```sql
CREATE TABLE books_publishers_link (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    book        INTEGER NOT NULL,
    publisher   INTEGER NOT NULL,

    UNIQUE(book),  -- Note: Actually one-to-one in practice
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(publisher) REFERENCES publishers(id) ON DELETE CASCADE
);
```

### publishers

**Purpose**: Stores unique publisher names

```sql
CREATE TABLE publishers (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    name    TEXT NOT NULL COLLATE NOCASE,
    sort    TEXT COLLATE NOCASE,

    UNIQUE(name)
);
```

### books_series_link

**Purpose**: Links books to series (with position index)

```sql
CREATE TABLE books_series_link (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    book    INTEGER NOT NULL,
    series  INTEGER NOT NULL,

    UNIQUE(book),  -- One book → one series
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(series) REFERENCES series(id) ON DELETE CASCADE
);
```

**Design Note**: The series position is actually stored in `books.series_index`, not in the link table.

### series

**Purpose**: Stores unique series names

```sql
CREATE TABLE series (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    name    TEXT NOT NULL COLLATE NOCASE,
    sort    TEXT COLLATE NOCASE,

    UNIQUE(name)
);
```

**Example**:

```sql
-- Create series
INSERT INTO series VALUES (1, 'Harry Potter', 'Harry Potter');

-- Link books to series (with positions in books.series_index)
INSERT INTO books_series_link VALUES (1, 100, 1);  -- Book 100
UPDATE books SET series_index = 1.0 WHERE id = 100;  -- First book

INSERT INTO books_series_link VALUES (2, 101, 1);  -- Book 101
UPDATE books SET series_index = 2.0 WHERE id = 101;  -- Second book
```

---

## Metadata Tables

### identifiers

**Purpose**: Stores external identifiers (ISBN, Amazon ASIN, Goodreads ID, etc.)

```sql
CREATE TABLE identifiers (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    book    INTEGER NOT NULL,
    type    TEXT NOT NULL DEFAULT "isbn" COLLATE NOCASE,
    val     TEXT NOT NULL COLLATE NOCASE,

    UNIQUE(book, type),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE
);
```

**Columns Explained**:

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `id` | INTEGER | Primary key | `1` |
| `book` | INTEGER | Foreign key to books.id | `42` |
| `type` | TEXT | Identifier type | `"isbn"`, `"asin"`, `"goodreads"`, `"google"` |
| `val` | TEXT | Identifier value | `"9780743273565"` |

**Example Data**:

```sql
-- Book 42 has multiple identifiers
INSERT INTO identifiers VALUES (1, 42, 'isbn', '9780743273565');
INSERT INTO identifiers VALUES (2, 42, 'goodreads', '4671');
INSERT INTO identifiers VALUES (3, 42, 'amazon', 'B000FC0PQA');
```

### languages

**Purpose**: Stores unique language codes (ISO 639-2)

```sql
CREATE TABLE languages (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    lang_code   TEXT NOT NULL COLLATE NOCASE,

    UNIQUE(lang_code)
);
```

### books_languages_link

**Purpose**: Links books to languages

```sql
CREATE TABLE books_languages_link (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    book        INTEGER NOT NULL,
    lang_code   INTEGER NOT NULL,

    UNIQUE(book, lang_code),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(lang_code) REFERENCES languages(id) ON DELETE CASCADE
);
```

**Example**:

```sql
-- English language
INSERT INTO languages VALUES (1, 'eng');

-- Book 42 is in English
INSERT INTO books_languages_link VALUES (1, 42, 1);
```

### ratings

**Purpose**: Stores book ratings (deprecated, now in books table)

**Note**: Modern Calibre stores ratings directly in `books` table. This table exists for backward compatibility.

---

## Custom Columns

Calibre supports user-defined metadata fields (custom columns).

### custom_columns

**Purpose**: Defines custom column metadata

```sql
CREATE TABLE custom_columns (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    label           TEXT NOT NULL,
    name            TEXT NOT NULL,
    datatype        TEXT NOT NULL,
    mark_for_delete BOOL DEFAULT 0 NOT NULL,
    editable        BOOL DEFAULT 1 NOT NULL,
    display         TEXT DEFAULT "{}" NOT NULL,
    is_multiple     BOOL DEFAULT 0 NOT NULL,
    normalized      BOOL NOT NULL,

    UNIQUE(label)
);
```

**Columns Explained**:

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `label` | TEXT | Unique identifier | `"my_rating"`, `"read_date"` |
| `name` | TEXT | Display name | `"My Rating"`, `"Date Read"` |
| `datatype` | TEXT | Data type | `"int"`, `"float"`, `"text"`, `"datetime"`, `"bool"` |
| `is_multiple` | BOOL | Can have multiple values | `1` for tags, `0` for single value |
| `normalized` | BOOL | Uses link table | `1` = link table, `0` = direct column |
| `display` | TEXT | JSON with display options | `"{\"number_format\": \"0.0\"}"` |

**Example**:

```sql
-- Add custom column: "Read Date"
INSERT INTO custom_columns VALUES (
    1,
    'read_date',
    'Date Read',
    'datetime',
    0,  -- not marked for delete
    1,  -- editable
    '{}',
    0,  -- single value
    0   -- not normalized (direct column)
);
```

This creates:

```sql
-- Dynamic table created automatically
CREATE TABLE custom_column_1 (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    book    INTEGER NOT NULL,
    value   TIMESTAMP,

    UNIQUE(book),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE
);
```

### Custom Column Naming

Custom columns use the pattern: `custom_column_{id}`

**Non-normalized (direct)**: `custom_column_1`
```sql
CREATE TABLE custom_column_1 (
    id      INTEGER PRIMARY KEY,
    book    INTEGER NOT NULL,
    value   {TYPE},
    UNIQUE(book)
);
```

**Normalized (with link table)**: `custom_column_2` + `custom_column_2_link`

```sql
-- Values table
CREATE TABLE custom_column_2 (
    id      INTEGER PRIMARY KEY,
    value   TEXT NOT NULL COLLATE NOCASE,
    UNIQUE(value)
);

-- Link table
CREATE TABLE books_custom_column_2_link (
    id      INTEGER PRIMARY KEY,
    book    INTEGER NOT NULL,
    value   INTEGER NOT NULL,
    UNIQUE(book, value),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(value) REFERENCES custom_column_2(id) ON DELETE CASCADE
);
```

---

## Full-Text Search

### books_fts

**Purpose**: Virtual table for full-text search using FTS5

```sql
CREATE VIRTUAL TABLE books_fts USING fts5(
    title,
    authors,
    tags,
    series,
    publisher,
    comments,
    identifiers,
    content='',
    tokenize='unicode61 remove_diacritics 2'
);
```

**How It Works**:

1. FTS5 creates internal tables (`books_fts_*`) for indexing
2. When book metadata changes, FTS index is updated
3. Search queries use `MATCH` operator

**Search Examples**:

```sql
-- Simple search
SELECT * FROM books_fts
WHERE books_fts MATCH 'quantum physics';

-- Phrase search
SELECT * FROM books_fts
WHERE books_fts MATCH '"science fiction"';

-- Boolean search
SELECT * FROM books_fts
WHERE books_fts MATCH 'python AND (programming OR development)';

-- Field-specific search
SELECT * FROM books_fts
WHERE books_fts MATCH 'authors:stephen AND title:dark';

-- With ranking and highlighting
SELECT
    books.id,
    books.title,
    snippet(books_fts, 0, '<mark>', '</mark>', '...', 50) as highlight,
    rank
FROM books_fts
JOIN books ON books_fts.rowid = books.id
WHERE books_fts MATCH 'quantum'
ORDER BY rank
LIMIT 20;
```

**🔗 Code Reference**: [src/calibre/db/fts.py](../../src/calibre/db/fts.py)

---

## Indexes and Performance

### Primary Indexes

```sql
-- Books table indexes
CREATE INDEX books_idx ON books (title COLLATE NOCASE);
CREATE INDEX books_author_sort_idx ON books (author_sort COLLATE NOCASE);
CREATE INDEX books_timestamp_idx ON books (timestamp);
CREATE INDEX books_pubdate_idx ON books (pubdate);
CREATE INDEX books_series_index_idx ON books (series_index);

-- Authors indexes
CREATE INDEX authors_idx ON authors (name COLLATE NOCASE);
CREATE INDEX authors_sort_idx ON authors (sort COLLATE NOCASE);

-- Tags indexes
CREATE INDEX tags_idx ON tags (name COLLATE NOCASE);

-- Data (formats) indexes
CREATE INDEX data_idx ON data (format);
CREATE INDEX formats_book_idx ON data (book);

-- Link table indexes (for JOIN performance)
CREATE INDEX books_authors_link_aidx ON books_authors_link (author);
CREATE INDEX books_authors_link_bidx ON books_authors_link (book);
CREATE INDEX books_tags_link_aidx ON books_tags_link (tag);
CREATE INDEX books_tags_link_bidx ON books_tags_link (book);
```

**Index Usage Guidelines**:

1. **Always index foreign keys** (for JOIN performance)
2. **Index frequently filtered columns** (title, author_sort, timestamp)
3. **Use COLLATE NOCASE** for case-insensitive searches
4. **Avoid over-indexing** (indexes slow down writes)

### Performance Analysis

```sql
-- Check if query uses index
EXPLAIN QUERY PLAN
SELECT * FROM books WHERE title = 'Great Gatsby';
-- Output: SEARCH TABLE books USING INDEX books_idx (title=?)

-- Check table statistics
SELECT * FROM sqlite_stat1;

-- Rebuild indexes (if database is slow)
REINDEX;

-- Analyze tables (update statistics)
ANALYZE;
```

---

## Schema Migrations

Calibre includes a schema versioning system:

### metadata_dirtied

**Purpose**: Tracks schema version and dirty state

```sql
CREATE TABLE metadata_dirtied (
    id      INTEGER PRIMARY KEY,
    book    INTEGER,

    UNIQUE(book)
);
```

### preferences

**Purpose**: Stores schema version and preferences

```sql
CREATE TABLE preferences (
    id      INTEGER PRIMARY KEY,
    key     TEXT NOT NULL,
    val     TEXT NOT NULL,

    UNIQUE(key)
);

-- Store schema version
INSERT INTO preferences VALUES (1, 'database_version', '1.0');
```

**Migration Process**:

1. Check `database_version` from preferences
2. If old version, run migrations
3. Migrations are in [src/calibre/db/migrations.py](../../src/calibre/db/migrations.py)

**Example Migration**:

```python
def migrate_to_v2(conn):
    """Add rating column to books table"""
    conn.execute('ALTER TABLE books ADD COLUMN rating REAL')
    conn.execute("UPDATE preferences SET val = '2.0' WHERE key = 'database_version'")
```

---

## SQL Examples

### Complex Queries

#### 1. Get Books with All Metadata

```sql
SELECT
    b.id,
    b.title,
    GROUP_CONCAT(DISTINCT a.name, ', ') as authors,
    GROUP_CONCAT(DISTINCT t.name, ', ') as tags,
    s.name as series,
    b.series_index,
    p.name as publisher,
    b.pubdate,
    b.rating,
    c.text as comments,
    GROUP_CONCAT(DISTINCT d.format, ', ') as formats
FROM books b
LEFT JOIN books_authors_link bal ON b.id = bal.book
LEFT JOIN authors a ON bal.author = a.id
LEFT JOIN books_tags_link btl ON b.id = btl.book
LEFT JOIN tags t ON btl.tag = t.id
LEFT JOIN books_series_link bsl ON b.id = bsl.book
LEFT JOIN series s ON bsl.series = s.id
LEFT JOIN books_publishers_link bpl ON b.id = bpl.book
LEFT JOIN publishers p ON bpl.publisher = p.id
LEFT JOIN comments c ON b.id = c.book
LEFT JOIN data d ON b.id = d.book
WHERE b.id = 42
GROUP BY b.id;
```

#### 2. Find Books by Multiple Criteria

```sql
-- Books by Stephen King, tagged 'Horror', with rating > 4
SELECT DISTINCT b.id, b.title, b.rating
FROM books b
JOIN books_authors_link bal ON b.id = bal.book
JOIN authors a ON bal.author = a.id
JOIN books_tags_link btl ON b.id = btl.book
JOIN tags t ON btl.tag = t.id
WHERE a.name LIKE '%Stephen King%'
  AND t.name = 'Horror'
  AND b.rating > 4
ORDER BY b.rating DESC;
```

#### 3. Series with Book Count

```sql
SELECT
    s.name as series_name,
    COUNT(bsl.book) as book_count,
    MIN(b.pubdate) as first_published,
    MAX(b.pubdate) as last_published
FROM series s
JOIN books_series_link bsl ON s.id = bsl.series
JOIN books b ON bsl.book = b.id
GROUP BY s.id
HAVING book_count > 1
ORDER BY book_count DESC;
```

#### 4. Books Without Covers

```sql
SELECT id, title, author_sort
FROM books
WHERE has_cover = 0
ORDER BY title;
```

#### 5. Most Popular Tags

```sql
SELECT
    t.name,
    COUNT(btl.book) as usage_count
FROM tags t
JOIN books_tags_link btl ON t.id = btl.tag
GROUP BY t.id
ORDER BY usage_count DESC
LIMIT 20;
```

#### 6. Recent Additions

```sql
SELECT
    b.id,
    b.title,
    GROUP_CONCAT(a.name, ', ') as authors,
    b.timestamp as added_date
FROM books b
LEFT JOIN books_authors_link bal ON b.id = bal.book
LEFT JOIN authors a ON bal.author = a.id
WHERE b.timestamp > datetime('now', '-30 days')
GROUP BY b.id
ORDER BY b.timestamp DESC;
```

#### 7. Books with Multiple Formats

```sql
SELECT
    b.id,
    b.title,
    COUNT(d.format) as format_count,
    GROUP_CONCAT(d.format, ', ') as formats
FROM books b
JOIN data d ON b.id = d.book
GROUP BY b.id
HAVING format_count > 1
ORDER BY format_count DESC;
```

---

## Database Maintenance

### Vacuum

Reclaim unused space and defragment:

```sql
VACUUM;
```

**When to run**: After deleting many books

### Integrity Check

Verify database integrity:

```sql
PRAGMA integrity_check;
-- Should return: ok

-- Quick check (faster):
PRAGMA quick_check;
```

### Statistics

Update query optimizer statistics:

```sql
ANALYZE;
```

**When to run**: After major changes (import/delete many books)

### Backup

```sql
-- Using SQLite backup API (via Python):
import apsw

source = apsw.Connection('metadata.db')
dest = apsw.Connection('metadata_backup.db')

with dest.backup('main', source, 'main') as backup:
    backup.step()  # Copy all pages
```

---

## Summary

### Key Tables

1. **books**: Core metadata (title, pubdate, flags)
2. **authors, tags, series, publishers**: Normalized entities
3. **books_*_link**: Many-to-many relationships
4. **data**: File formats (EPUB, PDF, etc.)
5. **comments**: Book descriptions
6. **identifiers**: External IDs (ISBN, etc.)
7. **custom_column_***: User-defined fields
8. **books_fts**: Full-text search index

### Schema Principles

1. **Normalization**: Avoid data duplication
2. **Foreign Keys**: Ensure referential integrity
3. **Indexes**: Optimize common queries
4. **COLLATE NOCASE**: Case-insensitive searches
5. **ON DELETE CASCADE**: Clean up orphaned records

### Best Practices

1. **Always use prepared statements** (prevent SQL injection)
2. **Use transactions** for multiple writes
3. **Index foreign keys** for JOIN performance
4. **Run ANALYZE** after bulk operations
5. **Backup before schema changes**

### Next Steps

- See [DATABASE_ARCHITECTURE.md](DATABASE_ARCHITECTURE.md) for conceptual overview
- See [HOW_TO_GUIDE.md](HOW_TO_GUIDE.md) for practical SQL examples
- See [EXERCISES.md](EXERCISES.md#exercise-22-create-a-database-query-with-joins) for hands-on practice

---

*Complete schema reference for Calibre's SQLite database. All tables, columns, relationships, and indexes documented.*
