# Technology Stack Deep Dive

Comprehensive guide to every technology used in Calibre, explaining why it was chosen, how it's used, and how it compares to modern alternatives.

## Table of Contents

1. [Overview](#overview)
2. [Python 3.10+](#python-310)
3. [PyQt6](#pyqt6)
4. [SQLite](#sqlite)
5. [Custom HTTP Server](#custom-http-server)
6. [XML/HTML Processing](#xmlhtml-processing)
7. [Image Processing](#image-processing)
8. [E-book Formats](#e-book-formats)
9. [Build System](#build-system)
10. [Development Tools](#development-tools)
11. [Technology Comparison Matrix](#technology-comparison-matrix)

---

## Overview

### Calibre's Technology Philosophy

Calibre's technology choices prioritize:

1. **Zero Dependencies for Users**: Users shouldn't need to install anything extra
2. **Cross-Platform**: Works identically on Windows, macOS, Linux
3. **Performance**: Fast even with 100,000+ books
4. **Stability**: Proven, battle-tested libraries
5. **Offline-First**: Works without internet connection

**Status**: ✅ All technologies current as of November 2025 (see [TECH_STACK_RESEARCH.md](TECH_STACK_RESEARCH.md))

---

## Python 3.10+

### Why Python?

**Chosen for**:
- Cross-platform (Windows, macOS, Linux)
- Rich ecosystem for e-book processing
- Easy to extend with plugins
- Good GUI framework support (PyQt6)
- Excellent text processing

**Current Version in Calibre**: Python 3.10+
**Latest Python**: 3.13 (November 2025)
**Status**: ✅ CURRENT (3.10 supported until October 2026)

### Key Python Features Used

#### 1. Type Hints (✅ CURRENT)

```python
from typing import List, Dict, Optional, Union

def get_book_metadata(
    book_id: int,
    include_authors: bool = True
) -> Dict[str, any]:
    """
    Get book metadata

    Args:
        book_id: Book ID to fetch
        include_authors: Whether to include author list

    Returns:
        Dictionary with book metadata

    Type hints help:
    - IDEs provide better autocomplete
    - Static analysis catches bugs
    - Documentation is self-evident
    """
    pass
```

**🔗 Code References**:
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py) - Extensive type hints

**💭 TypeScript Comparison**:
```typescript
// Very similar to TypeScript:
function getBookMetadata(
  bookId: number,
  includeAuthors: boolean = true
): Record<string, any> {
  // ...
}
```

#### 2. Context Managers (✅ CURRENT)

```python
# Automatic resource cleanup
with open('book.txt', 'r') as f:
    content = f.read()
# File automatically closed, even if exception occurs

# Database transactions
with db.transaction():
    db.execute('UPDATE books SET ...')
    db.execute('INSERT INTO authors ...')
# Auto-commit on success, auto-rollback on exception

# Custom context manager
class DatabaseLock:
    def __enter__(self):
        self.conn.execute('BEGIN EXCLUSIVE')
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            self.conn.commit()
        else:
            self.conn.rollback()

with DatabaseLock():
    # Critical section
    pass
```

**💭 Try-Finally Comparison**:
```javascript
// JavaScript equivalent (no built-in context managers):
const f = fs.openSync('book.txt', 'r')
try {
  const content = fs.readFileSync(f, 'utf8')
} finally {
  fs.closeSync(f)  // Always runs
}

// Or modern async:
await using file = await fs.open('book.txt')  // TC39 proposal
```

#### 3. Decorators (✅ CURRENT)

See [PATTERNS_AND_CONVENTIONS.md](PATTERNS_AND_CONVENTIONS.md#decorator-pattern) for extensive examples.

```python
@endpoint('/api/books/{book_id}', types={'book_id': int})
@require_auth
@rate_limit(requests=100, per_seconds=60)
def ajax_get_book(ctx, rd, book_id):
    return {'book': get_book(book_id)}

# Decorators wrap functions, adding functionality
# Calibre uses them for:
# - API routing
# - Database access control
# - Caching
# - Performance monitoring
```

#### 4. Properties (✅ CURRENT)

```python
class BookMetadata:
    def __init__(self, data):
        self._title = data['title']
        self._rating = data.get('rating', 0)

    @property
    def title(self):
        """Get title"""
        return self._title

    @title.setter
    def title(self, value):
        """Set title with validation"""
        if not value or not value.strip():
            raise ValueError("Title cannot be empty")
        self._title = value.strip()

    @property
    def display_title(self):
        """Computed property (read-only)"""
        return self._title.upper()

# Usage:
book = BookMetadata({'title': 'Great Book'})
print(book.title)  # "Great Book" (calls getter)
book.title = " New Title "  # Calls setter, validates & strips
print(book.display_title)  # "NEW TITLE" (computed)
```

**💭 JavaScript Getter/Setter**:
```javascript
class BookMetadata {
  constructor(data) {
    this._title = data.title
  }

  get title() {
    return this._title
  }

  set title(value) {
    if (!value || !value.trim()) {
      throw new Error('Title cannot be empty')
    }
    this._title = value.trim()
  }

  get displayTitle() {
    return this._title.toUpperCase()
  }
}
```

Very similar!

### Python vs JavaScript for Calibre

| Feature | Python | JavaScript/Node.js |
|---------|--------|-------------------|
| **Desktop GUI** | ✅ PyQt6 (native) | ⚠️ Electron (heavy) |
| **Binary Parsing** | ✅ Excellent (`struct`) | ⚠️ Good (Buffers) |
| **C Extensions** | ✅ Easy (`ctypes`, Cython) | ⚠️ Harder (N-API) |
| **Packaging** | ✅ Single executable | ⚠️ Large bundles |
| **Text Processing** | ✅ Excellent | ✅ Excellent |
| **Web Server** | ✅ Many options | ✅ Many options |
| **Type Safety** | ⚠️ Optional (type hints) | ⚠️ Optional (TypeScript) |
| **Speed** | ⚠️ Slower than compiled | ⚠️ Slower than compiled |

**Verdict**: Python is the right choice for Calibre's use case (desktop app, binary processing, e-book formats).

---

## PyQt6

### What is PyQt6?

PyQt6 is **Python bindings for Qt 6**, a mature, cross-platform GUI framework.

**Version in Calibre**: 6.8.1 (Latest, Qt 6.8 LTS)
**Status**: ✅ CURRENT

### Qt Framework Layers

```
PyQt6 (Python bindings)
    ↓
Qt 6.8 (C++ framework)
    ↓
Platform APIs (Win32, Cocoa, X11/Wayland)
```

### Why PyQt6 Over Alternatives?

| Framework | Pros | Cons | Status |
|-----------|------|------|--------|
| **PyQt6** | Native look, mature, fast | Commercial license for commercial apps | ✅ Using |
| **PySide6** | Same API, LGPL license | Slightly behind PyQt6 updates | ⚠️ Alternative |
| **Tkinter** | Built-in, simple | Looks outdated, limited widgets | ❌ Too basic |
| **wxPython** | Native widgets | Smaller community, older | ❌ Less polished |
| **Kivy** | Modern, touch-first | Non-native look | ❌ Wrong use case |
| **Electron** | Web technologies | 200MB+ overhead, slow | ❌ Too heavy |

**Verdict**: PyQt6 is the best choice for a feature-rich desktop application.

### Core PyQt6 Concepts

#### Widgets (Components)

```python
from PyQt6.QtWidgets import (
    QWidget, QVBoxLayout, QHBoxLayout,
    QPushButton, QLabel, QLineEdit, QTableView
)

class BookDetailsWidget(QWidget):
    """Custom widget showing book details"""

    def __init__(self, book_data, parent=None):
        super().__init__(parent)
        self.book_data = book_data
        self.setup_ui()

    def setup_ui(self):
        """Build the UI"""
        # Create layout
        layout = QVBoxLayout()

        # Add widgets
        title_label = QLabel(f"<h2>{self.book_data['title']}</h2>")
        layout.addWidget(title_label)

        author_label = QLabel(f"by {', '.join(self.book_data['authors'])}")
        layout.addWidget(author_label)

        # Set layout
        self.setLayout(layout)
```

**🔗 Code References**:
- [src/calibre/gui2/widgets.py](../../src/calibre/gui2/widgets.py) - Custom widgets
- [src/calibre/gui2/library/views.py](../../src/calibre/gui2/library/views.py) - Book list view

**💭 React Comparison**:
```javascript
// Very similar to React components:
function BookDetailsWidget({ bookData }) {
  return (
    <div>
      <h2>{bookData.title}</h2>
      <p>by {bookData.authors.join(', ')}</p>
    </div>
  )
}
```

Key differences:
- PyQt6: Imperative (create, configure, add to layout)
- React: Declarative (describe what you want)

#### Signals and Slots (Event System)

```python
from PyQt6.QtCore import pyqtSignal, QObject

class BookListModel(QObject):
    """Model emitting signals on changes"""

    # Define signals (like events)
    bookAdded = pyqtSignal(int)  # Emits book_id
    bookUpdated = pyqtSignal(int, dict)  # book_id, changes
    selectionChanged = pyqtSignal(list)  # selected book_ids

    def add_book(self, book_data):
        book_id = self.db.add_book(book_data)
        self.bookAdded.emit(book_id)  # Emit signal
        return book_id

class BookListView(QWidget):
    """View listening to model signals"""

    def __init__(self, model):
        super().__init__()
        self.model = model

        # Connect signals to slots (handlers)
        self.model.bookAdded.connect(self.on_book_added)
        self.model.bookUpdated.connect(self.on_book_updated)

    def on_book_added(self, book_id):
        """Slot (handler) for bookAdded signal"""
        print(f"Book {book_id} added, refreshing list...")
        self.refresh()

    def on_book_updated(self, book_id, changes):
        """Slot for bookUpdated signal"""
        print(f"Book {book_id} updated: {changes}")
        self.update_book_in_list(book_id)
```

**Signal Types**:
```python
# No arguments
clicked = pyqtSignal()

# Single argument
textChanged = pyqtSignal(str)

# Multiple arguments
bookUpdated = pyqtSignal(int, dict)

# Overloaded signals (multiple signatures)
valueChanged = pyqtSignal([int], [str])
```

**💭 React Comparison**:
```javascript
// React uses callbacks (props):
function BookList({ onBookAdded, onBookUpdated }) {
  const handleAdd = (bookData) => {
    const bookId = db.addBook(bookData)
    onBookAdded(bookId)  // Call parent's callback
  }

  return <button onClick={handleAdd}>Add Book</button>
}

// Or event emitters:
const events = new EventEmitter()
events.on('bookAdded', handleBookAdded)
events.emit('bookAdded', bookId)
```

Signals/slots are more powerful:
- Type-safe (Qt validates signal types)
- Can connect multiple slots to one signal
- Can connect across threads
- Automatic disconnect on object destruction

#### Layouts

```python
# Vertical layout (stack vertically)
layout = QVBoxLayout()
layout.addWidget(QLabel("Title:"))
layout.addWidget(QLineEdit())
layout.addWidget(QPushButton("Save"))

# Horizontal layout (side by side)
layout = QHBoxLayout()
layout.addWidget(QPushButton("Cancel"))
layout.addWidget(QPushButton("OK"))

# Grid layout (rows & columns)
layout = QGridLayout()
layout.addWidget(QLabel("Title:"), 0, 0)
layout.addWidget(QLineEdit(), 0, 1)
layout.addWidget(QLabel("Author:"), 1, 0)
layout.addWidget(QLineEdit(), 1, 1)

# Form layout (label-field pairs)
layout = QFormLayout()
layout.addRow("Title:", QLineEdit())
layout.addRow("Author:", QLineEdit())
layout.addRow("Rating:", QSpinBox())

# Nested layouts
main_layout = QVBoxLayout()

# Top section
top_layout = QHBoxLayout()
top_layout.addWidget(QLabel("Search:"))
top_layout.addWidget(QLineEdit())
main_layout.addLayout(top_layout)

# Middle section (table)
main_layout.addWidget(QTableView())

# Bottom section (buttons)
button_layout = QHBoxLayout()
button_layout.addWidget(QPushButton("Add"))
button_layout.addWidget(QPushButton("Delete"))
main_layout.addLayout(button_layout)
```

**💭 React/Flexbox Comparison**:
```javascript
// QVBoxLayout = flex-direction: column
<div style={{ display: 'flex', flexDirection: 'column' }}>
  <Label>Title:</Label>
  <Input />
  <Button>Save</Button>
</div>

// QHBoxLayout = flex-direction: row
<div style={{ display: 'flex', flexDirection: 'row' }}>
  <Button>Cancel</Button>
  <Button>OK</Button>
</div>

// QGridLayout = CSS Grid
<div style={{ display: 'grid', gridTemplateColumns: 'auto 1fr' }}>
  <Label>Title:</Label>
  <Input />
  <Label>Author:</Label>
  <Input />
</div>
```

Very similar concepts!

#### Model-View Architecture

See [PATTERNS_AND_CONVENTIONS.md](PATTERNS_AND_CONVENTIONS.md#model-view-pattern) for detailed examples.

```python
# Model: Manages data
class BookTableModel(QAbstractTableModel):
    def rowCount(self, parent=None):
        return len(self.books)

    def data(self, index, role=Qt.ItemDataRole.DisplayRole):
        return self.books[index.row()][index.column()]

# View: Displays data
table_view = QTableView()
table_view.setModel(model)

# User edits cell → View notifies Model → Model updates data → Model notifies View → View refreshes
```

**💭 React Comparison**:
```javascript
// Similar to React state management:
const [books, setBooks] = useState([])

function BookTable() {
  return (
    <table>
      {books.map(book => (
        <tr key={book.id}>
          <td>{book.title}</td>
          <td>{book.author}</td>
        </tr>
      ))}
    </table>
  )
}

// User edits → setBooks() → Component re-renders
```

### PyQt6 vs Web Technologies

| Feature | PyQt6 | React + Electron | Web App |
|---------|-------|------------------|---------|
| **Native Look** | ✅ Matches OS | ⚠️ Close but not perfect | ❌ Web look |
| **Performance** | ✅ Fast (C++) | ⚠️ Good (V8) | ⚠️ Depends on browser |
| **App Size** | ✅ ~50MB | ❌ ~200MB | ✅ ~5MB (server-side) |
| **Offline** | ✅ Full access | ✅ Full access | ❌ Limited |
| **System Integration** | ✅ Deep | ⚠️ Limited | ❌ Very limited |
| **Development Speed** | ⚠️ Moderate | ✅ Fast (hot reload) | ✅ Fast |
| **Styling** | ⚠️ QSS (like CSS) | ✅ CSS/Tailwind | ✅ CSS/Tailwind |
| **Learning Curve** | ⚠️ Steep | ✅ Familiar (web devs) | ✅ Familiar |

**Verdict**: PyQt6 is right for Calibre (desktop app needing native performance and offline access).

---

## SQLite

### Why SQLite?

**Chosen for**:
- Zero configuration (no server setup)
- Single file database (easy backups)
- Fast for read-heavy workloads
- Full SQL support
- Cross-platform
- Public domain (no licensing issues)

**Version in Calibre**: 3.50.4 (via APSW wrapper)
**Latest SQLite**: 3.51.0
**Status**: ⚠️ SLIGHTLY OUTDATED (minor version behind, non-critical)

### SQLite vs Other Databases

| Database | Use Case | Calibre Fit? |
|----------|----------|--------------|
| **SQLite** | Embedded, single-user | ✅ PERFECT |
| **PostgreSQL** | Multi-user, server-based | ❌ Overkill |
| **MySQL** | Web apps, server-based | ❌ Overkill |
| **MongoDB** | Document store, NoSQL | ❌ Wrong paradigm |
| **Redis** | Cache, in-memory | ⚠️ Could complement SQLite |

**Why not PostgreSQL?**
- Calibre is single-user (one person's library)
- No need for concurrent writes from multiple clients
- SQLite is faster for this use case
- No server setup required

### APSW (Another Python SQLite Wrapper)

Calibre uses **APSW** instead of standard `sqlite3` module:

```python
# Standard library sqlite3:
import sqlite3
conn = sqlite3.connect('library.db')

# APSW (what Calibre uses):
import apsw
conn = apsw.Connection('library.db')
```

**Why APSW?**

| Feature | APSW | sqlite3 |
|---------|------|---------|
| **SQLite Version** | Latest | Older (bundled with Python) |
| **Virtual Tables** | ✅ Full support | ⚠️ Limited |
| **Blob I/O** | ✅ Streaming | ❌ Load all to memory |
| **Backup API** | ✅ Supported | ❌ Not exposed |
| **Custom Functions** | ✅ Easy | ⚠️ Harder |
| **Performance** | ✅ Faster | ⚠️ Good |

**🔗 Code References**:
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py#L50) - APSW usage

### SQLite Features Calibre Uses

#### 1. Full-Text Search (FTS5)

```python
# Create FTS5 virtual table
conn.execute('''
    CREATE VIRTUAL TABLE books_fts USING fts5(
        title,
        authors,
        comments,
        content='books',
        content_rowid='id'
    )
''')

# Search with highlighting
results = conn.execute('''
    SELECT
        books.id,
        books.title,
        snippet(books_fts, 0, '<mark>', '</mark>', '...', 50) as highlight
    FROM books_fts
    JOIN books ON books_fts.rowid = books.id
    WHERE books_fts MATCH ?
    ORDER BY rank
''', ('quantum physics',))

# Returns:
# [
#   (42, 'Quantum Physics 101', '<mark>Quantum</mark> <mark>Physics</mark> for beginners'),
#   ...
# ]
```

**🔗 Code References**:
- [src/calibre/db/fts.py](../../src/calibre/db/fts.py) - FTS5 implementation

**💭 Elasticsearch Comparison**:
```javascript
// Similar to Elasticsearch:
const results = await elastic.search({
  index: 'books',
  body: {
    query: {
      multi_match: {
        query: 'quantum physics',
        fields: ['title', 'authors', 'comments']
      }
    },
    highlight: {
      fields: { title: {}, comments: {} }
    }
  }
})
```

FTS5 is surprisingly powerful for embedded use!

#### 2. JSON Support

```python
# Store JSON in SQLite (3.38+)
conn.execute('''
    CREATE TABLE books (
        id INTEGER PRIMARY KEY,
        title TEXT,
        metadata JSON  -- JSON column
    )
''')

conn.execute('''
    INSERT INTO books (title, metadata) VALUES (
        'Great Book',
        json('{"tags": ["fiction", "bestseller"], "rating": 4.5}')
    )
''')

# Query JSON fields
books = conn.execute('''
    SELECT title, json_extract(metadata, '$.rating') as rating
    FROM books
    WHERE json_extract(metadata, '$.rating') > 4.0
''').fetchall()

# Returns: [('Great Book', 4.5)]
```

**💭 MongoDB Comparison**:
```javascript
// MongoDB:
db.books.find({
  'metadata.rating': { $gt: 4.0 }
})

// SQLite JSON functions:
json_extract(metadata, '$.rating') > 4.0
```

SQLite can handle light JSON use cases!

#### 3. Window Functions

```python
# Ranking within groups
conn.execute('''
    SELECT
        title,
        author,
        rating,
        ROW_NUMBER() OVER (
            PARTITION BY author
            ORDER BY rating DESC
        ) as rank_for_author
    FROM books
''')

# Returns:
# title                | author        | rating | rank_for_author
# The Shining         | Stephen King  | 4.8    | 1
# It                  | Stephen King  | 4.6    | 2
# Brief History       | Hawking       | 4.9    | 1
```

Powerful for analytics!

#### 4. Common Table Expressions (CTEs)

```python
# Recursive CTEs for hierarchical data
conn.execute('''
    WITH RECURSIVE book_hierarchy(id, title, parent_id, level) AS (
        -- Base case: top-level books
        SELECT id, title, parent_id, 0
        FROM books
        WHERE parent_id IS NULL

        UNION ALL

        -- Recursive case: children
        SELECT b.id, b.title, b.parent_id, bh.level + 1
        FROM books b
        JOIN book_hierarchy bh ON b.parent_id = bh.id
    )
    SELECT * FROM book_hierarchy
    ORDER BY level, title
''')
```

### SQLite Performance Tips

```python
# 1. Use indexes
conn.execute('CREATE INDEX idx_books_title ON books(title)')
conn.execute('CREATE INDEX idx_authors_name ON authors(name)')

# 2. Use transactions for bulk inserts
conn.execute('BEGIN TRANSACTION')
for book in books:
    conn.execute('INSERT INTO books VALUES (?, ?)', book)
conn.execute('COMMIT')

# Without transaction: 5000ms
# With transaction: 50ms (100x faster!)

# 3. Use prepared statements
stmt = conn.cursor().execute('SELECT * FROM books WHERE id = ?')
for book_id in book_ids:
    stmt.execute([book_id])  # Reuses prepared statement

# 4. Enable WAL mode for concurrent reads
conn.execute('PRAGMA journal_mode = WAL')
# Allows readers while writer is active

# 5. Analyze query performance
conn.execute('EXPLAIN QUERY PLAN SELECT * FROM books WHERE title LIKE ?')
```

**🔗 Code References**:
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py#L200) - Performance optimizations

---

## Custom HTTP Server

### Why Custom Server?

Calibre implements its own HTTP server instead of using frameworks like Flask or FastAPI.

**Reasons**:
1. **No dependencies**: Calibre is standalone
2. **Precise control**: Exactly the features needed, no bloat
3. **Performance**: Optimized for Calibre's use case
4. **Stability**: No framework update breakage

**Status**: ✅ CURRENT (actively maintained, battle-tested)

### Architecture

[src/calibre/srv/loop.py](../../src/calibre/srv/loop.py#L392):
```python
import select
import socket

class ServerLoop:
    """
    Custom HTTP server using select() for async I/O

    Architecture:
    1. Accept connections
    2. Use select() to wait for I/O
    3. Read requests
    4. Dispatch to handlers
    5. Write responses
    """

    def __init__(self, host='0.0.0.0', port=8080):
        self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server_socket.bind((host, port))
        self.server_socket.listen(100)
        self.server_socket.setblocking(False)

        self.connections = {}  # sock → connection state

    def serve_forever(self):
        """Main event loop"""
        while True:
            # Get sockets ready for I/O
            readable, writable, errored = select.select(
                [self.server_socket] + list(self.connections.keys()),
                [sock for sock, conn in self.connections.items() if conn.has_data_to_write()],
                [],
                timeout=1.0
            )

            # Accept new connections
            if self.server_socket in readable:
                client_sock, addr = self.server_socket.accept()
                client_sock.setblocking(False)
                self.connections[client_sock] = Connection(client_sock)

            # Read from ready sockets
            for sock in readable:
                if sock == self.server_socket:
                    continue

                conn = self.connections[sock]
                try:
                    conn.read_data()
                    if conn.request_complete():
                        self.handle_request(conn)
                except Exception as e:
                    self.close_connection(sock)

            # Write to ready sockets
            for sock in writable:
                conn = self.connections[sock]
                try:
                    conn.write_data()
                    if conn.response_complete():
                        self.close_connection(sock)
                except Exception:
                    self.close_connection(sock)
```

**How It Works**:

```
1. Client connects → ServerSocket accepts → New Connection object created

2. select() waits for activity:
   - Readable: Client sent data (request)
   - Writable: Can send data to client (response)

3. Read request:
   GET /api/books HTTP/1.1
   Host: localhost:8080

4. Parse request → Find route → Call handler

5. Handler returns response:
   HTTP/1.1 200 OK
   Content-Type: application/json
   {"books": [...]}

6. Write response → Close connection (or keep-alive)
```

**💭 Modern Comparison**:

| Feature | Calibre Custom | Express.js | FastAPI |
|---------|----------------|------------|---------|
| **Dependencies** | ✅ Zero | ❌ Many | ❌ Many |
| **Performance** | ✅ Optimized | ✅ Good | ✅ Excellent |
| **Features** | ⚠️ Basic | ✅ Rich ecosystem | ✅ Rich ecosystem |
| **WebSocket** | ⚠️ Possible but not implemented | ✅ Easy (Socket.IO) | ✅ Easy (built-in) |
| **Middleware** | ⚠️ Manual | ✅ Rich ecosystem | ✅ Built-in |
| **Async** | ⚠️ select()-based | ✅ async/await | ✅ async/await |

**When Custom Server Makes Sense**:
- ✅ Embedded application (like Calibre)
- ✅ Specific requirements
- ✅ No external dependencies desired
- ❌ Web-first applications
- ❌ Need rich ecosystem

### Routing System

[src/calibre/srv/routes.py](../../src/calibre/srv/routes.py#L57):
```python
# Decorator-based routing (similar to Flask/FastAPI)
@endpoint('/api/books/{book_id}', types={'book_id': int})
def ajax_get_book(ctx, rd, book_id):
    book = ctx.db.get_metadata(book_id)
    return {'book': book}

# Pattern matching
class Router:
    def __init__(self):
        self.routes = []

    def add_route(self, pattern, handler, types):
        # Convert pattern to regex
        # /api/books/{book_id} → ^/api/books/(?P<book_id>[^/]+)$
        regex = self.pattern_to_regex(pattern)
        self.routes.append((regex, handler, types))

    def find_route(self, path):
        """Find matching route for path"""
        for regex, handler, types in self.routes:
            match = regex.match(path)
            if match:
                # Extract path parameters
                params = match.groupdict()
                # Convert types
                for key, expected_type in types.items():
                    if key in params:
                        params[key] = expected_type(params[key])
                return handler, params
        return None, None
```

**💭 Express.js Comparison**:
```javascript
// Express routing:
app.get('/api/books/:bookId', (req, res) => {
  const bookId = parseInt(req.params.bookId)
  const book = db.getMetadata(bookId)
  res.json({ book })
})

// Very similar to Calibre's @endpoint decorator!
```

---

## XML/HTML Processing

### lxml (✅ CURRENT)

**Version**: 6.0.1
**Status**: ✅ CURRENT

E-books use XML formats (EPUB, AZW3, etc.). Calibre needs fast, robust XML processing.

```python
from lxml import etree

# Parse EPUB content.opf
tree = etree.parse('content.opf')

# XPath queries
title = tree.xpath('//dc:title/text()', namespaces={
    'dc': 'http://purl.org/dc/elements/1.1/'
})[0]

authors = tree.xpath('//dc:creator/text()', namespaces={
    'dc': 'http://purl.org/dc/elements/1.1/'
})

# Modify XML
title_elem = tree.xpath('//dc:title')[0]
title_elem.text = 'New Title'

# Write back
tree.write('content.opf', encoding='utf-8', xml_declaration=True)
```

**Why lxml Over Alternatives?**

| Library | Speed | Features | Calibre Choice |
|---------|-------|----------|----------------|
| **lxml** | ✅ Fastest | ✅ XPath, XSLT | ✅ Using |
| **xml.etree** | ⚠️ Slower | ⚠️ Basic | ❌ Too limited |
| **BeautifulSoup** | ❌ Slowest | ✅ Forgiving | ⚠️ For HTML only |
| **minidom** | ❌ Slow | ⚠️ DOM API | ❌ Outdated |

### BeautifulSoup (⚠️ OUTDATED)

**Version in Calibre**: 4.12.2
**Latest Version**: 4.14.2
**Status**: ⚠️ OUTDATED (should upgrade)

**Used for**: HTML parsing (more forgiving than lxml for messy HTML)

```python
from bs4 import BeautifulSoup

# Parse messy HTML (from web scraping)
html = "<html><body><p>Unclosed paragraph<div>Malformed</html>"

soup = BeautifulSoup(html, 'html.parser')

# BeautifulSoup fixes it automatically!
print(soup.prettify())
# <html>
#  <body>
#   <p>Unclosed paragraph</p>
#   <div>Malformed</div>
#  </body>
# </html>

# Find elements
paragraphs = soup.find_all('p')
links = soup.select('a[href]')  # CSS selectors

# Extract text
text = soup.get_text()
```

**💭 Web Scraping Comparison**:
```javascript
// Cheerio (Node.js):
const $ = cheerio.load(html)
const paragraphs = $('p')
const links = $('a[href]')
const text = $.text()
```

Very similar API!

---

## Image Processing

### Pillow (🚨 MAJOR VERSION BEHIND)

**Version in Calibre**: 10.3.0
**Latest Version**: 12.0.0
**Status**: 🚨 MAJOR VERSION BEHIND (should upgrade to 12.x)

**Used for**: Book cover processing, format conversions

```python
from PIL import Image

# Open cover image
img = Image.open('cover.jpg')

# Resize while maintaining aspect ratio
img.thumbnail((600, 800), Image.Resampling.LANCZOS)

# Convert format
img.save('cover.webp', 'WEBP', quality=85)

# Get dimensions
width, height = img.size

# Crop
cropped = img.crop((10, 10, 590, 790))

# Apply filters
from PIL import ImageFilter
blurred = img.filter(ImageFilter.BLUR)
sharpened = img.filter(ImageFilter.SHARPEN)

# Add text (for thumbnails)
from PIL import ImageDraw, ImageFont
draw = ImageDraw.Draw(img)
font = ImageFont.truetype('arial.ttf', 36)
draw.text((10, 10), 'SAMPLE', fill='red', font=font)
```

**Why Pillow?**

| Library | Features | Performance | Status |
|---------|----------|-------------|--------|
| **Pillow** | ✅ Comprehensive | ✅ Fast (C core) | ✅ Standard |
| **OpenCV** | ✅ Advanced | ✅ Fastest | ⚠️ Overkill for Calibre |
| **imageio** | ⚠️ Basic | ⚠️ Slower | ❌ Limited |
| **Wand** | ✅ ImageMagick wrapper | ⚠️ External dependency | ❌ Extra dependency |

**💡 Upgrade Recommended**: Pillow 12.x has:
- Better WebP support
- Performance improvements
- Security fixes
- New image formats

---

## E-book Formats

Calibre supports 20+ formats. Here are the main ones:

### EPUB (✅ CURRENT STANDARD)

**Format**: ZIP container with XHTML + CSS + metadata
**Status**: ✅ Industry standard

```
book.epub (ZIP file)
├─ mimetype                 # "application/epub+zip"
├─ META-INF/
│  └─ container.xml        # Points to content.opf
├─ OEBPS/
│  ├─ content.opf          # Metadata (XML)
│  ├─ toc.ncx              # Table of contents
│  ├─ chapter1.xhtml       # Content (XHTML)
│  ├─ chapter2.xhtml
│  ├─ styles.css           # Styles
│  └─ cover.jpg            # Cover image
```

**Processing**:
```python
import zipfile
from lxml import etree

# Open EPUB
with zipfile.ZipFile('book.epub', 'r') as zf:
    # Read metadata
    opf_content = zf.read('OEBPS/content.opf')
    tree = etree.fromstring(opf_content)

    # Extract title
    title = tree.xpath('//dc:title/text()', namespaces={
        'dc': 'http://purl.org/dc/elements/1.1/'
    })[0]

    # Extract chapter HTML
    chapter1 = zf.read('OEBPS/chapter1.xhtml').decode('utf-8')
```

### PDF (✅ WIDELY USED)

**Status**: ✅ Universal but not ideal for e-reading

**Challenges**:
- Fixed layout (doesn't reflow)
- Hard to extract text
- Large file sizes

**Calibre's PDF handling**:
```python
# Extract text (uses pdftotext or PyPDF2)
import subprocess
text = subprocess.check_output(['pdftotext', 'book.pdf', '-']).decode('utf-8')

# Or using PyPDF2:
import PyPDF2
with open('book.pdf', 'rb') as f:
    reader = PyPDF2.PdfReader(f)
    text = ''.join(page.extract_text() for page in reader.pages)
```

### MOBI/AZW3 (⚠️ PROPRIETARY)

**Status**: ⚠️ Amazon proprietary, being phased out in favor of EPUB

**Format**: Binary, similar to EPUB but Amazon-specific

---

## Build System

### pyproject.toml (✅ CURRENT)

Calibre uses modern Python packaging:

[pyproject.toml](../../pyproject.toml):
```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "calibre"
version = "7.0.0"
requires-python = ">=3.10"

dependencies = [
    "PyQt6>=6.8.0",
    "lxml>=6.0.0",
    "Pillow>=10.0.0",
    # ... more
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "black>=23.0.0",
    "mypy>=1.0.0"
]
```

**Building**:
```bash
# Development install
pip install -e .

# Build wheel
python -m build

# Install
pip install dist/calibre-7.0.0-py3-none-any.whl
```

**💭 JavaScript Comparison**:
```json
// package.json:
{
  "name": "calibre",
  "version": "7.0.0",
  "dependencies": {
    "react": "^18.0.0",
    "next": "^14.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "eslint": "^8.0.0"
  }
}
```

Very similar!

---

## Development Tools

### Testing: pytest (✅ CURRENT)

```python
# test_book_metadata.py
import pytest

def test_book_title_required():
    with pytest.raises(ValueError):
        BookMetadata(title='', authors=['Author'])

def test_rating_validation():
    book = BookMetadata('Title', ['Author'])
    book.set_rating(4.5)  # ✅ OK
    assert book.rating == 4.5

    with pytest.raises(ValueError):
        book.set_rating(6.0)  # ❌ Out of range

@pytest.fixture
def sample_book():
    return BookMetadata('Test Book', ['Test Author'])

def test_with_fixture(sample_book):
    assert sample_book.title == 'Test Book'
```

### Linting: flake8 / pylint

```bash
# Check code style
flake8 src/calibre/

# Type checking
mypy src/calibre/

# Format code
black src/calibre/
```

### Debugging: pdb

```python
# Set breakpoint
import pdb; pdb.set_trace()

# Or Python 3.7+:
breakpoint()

# Commands:
# n = next line
# s = step into function
# c = continue
# p variable = print variable
# l = list code
```

---

## Technology Comparison Matrix

### Calibre Stack vs Modern Web Stack

| Layer | Calibre | Modern Web Alternative |
|-------|---------|----------------------|
| **Language** | Python 3.10+ | TypeScript/JavaScript |
| **GUI** | PyQt6 (desktop) | React + Electron |
| **Database** | SQLite + APSW | PostgreSQL/MongoDB |
| **ORM** | Custom cache layer | Prisma/TypeORM |
| **HTTP Server** | Custom (select-based) | Express/Fastify |
| **API Framework** | Custom decorators | Next.js API routes |
| **Testing** | pytest | Jest/Vitest |
| **Type Checking** | mypy (optional) | TypeScript (required) |
| **Package Manager** | pip | npm/pnpm |
| **Build Tool** | setuptools | Vite/Webpack |

### When to Choose Calibre's Stack

✅ **Use Calibre's approach when**:
- Building desktop application
- Need offline-first
- Working with binary formats
- Need OS integration
- Performance critical
- No server needed

❌ **Use web stack when**:
- Building web application
- Need real-time collaboration
- Multiple concurrent users
- Rapid iteration
- Rich ecosystem needed

---

## Summary

### Key Technologies

1. **Python 3.10+** - ✅ CURRENT, right choice for desktop app
2. **PyQt6 6.8.1** - ✅ CURRENT (latest), excellent for native GUI
3. **SQLite 3.50.4** - ⚠️ Slightly outdated but fine for Calibre's use
4. **Custom HTTP** - ✅ CURRENT, appropriate for embedded server
5. **lxml 6.0.1** - ✅ CURRENT, fast XML processing
6. **BeautifulSoup 4.12.2** - ⚠️ OUTDATED, should upgrade to 4.14.2
7. **Pillow 10.3.0** - 🚨 MAJOR VERSION BEHIND, should upgrade to 12.x

### Upgrade Priority

1. 🚨 **High**: Pillow 10.3.0 → 12.0.0 (security, features)
2. ⚠️ **Medium**: BeautifulSoup 4.12.2 → 4.14.2 (bug fixes)
3. ⚠️ **Low**: SQLite 3.50.4 → 3.51.0 (minor improvements)

### Learning Path

1. **Start with**: Python basics, type hints
2. **Then learn**: PyQt6 widgets and signals/slots
3. **Then understand**: SQLite and database patterns
4. **Finally master**: E-book formats and custom server

### Next Steps

- Try the exercises in [EXERCISES.md](EXERCISES.md)
- Read source code with new understanding
- Contribute improvements to outdated dependencies!

---

*See [TECH_STACK_RESEARCH.md](TECH_STACK_RESEARCH.md) for version audit and [PATTERNS_AND_CONVENTIONS.md](PATTERNS_AND_CONVENTIONS.md) for code patterns.*
