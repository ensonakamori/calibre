# Patterns and Conventions in Calibre

A comprehensive guide to code patterns, conventions, and best practices used throughout the Calibre codebase. This document helps React developers understand Calibre's architectural patterns and coding standards.

## Table of Contents

1. [Overview](#overview)
2. [Naming Conventions](#naming-conventions)
3. [Code Organization Patterns](#code-organization-patterns)
4. [Design Patterns](#design-patterns)
5. [Database Patterns](#database-patterns)
6. [GUI Patterns](#gui-patterns)
7. [API Patterns](#api-patterns)
8. [Error Handling Patterns](#error-handling-patterns)
9. [Testing Patterns](#testing-patterns)
10. [Performance Patterns](#performance-patterns)
11. [Security Patterns](#security-patterns)
12. [Pattern Status Reference](#pattern-status-reference)

---

## Overview

### Pattern Categories

Calibre uses several categories of patterns:

```
Architecture Patterns
├─ MVC (Model-View-Controller)
├─ Three-tier architecture
└─ Plugin architecture

Design Patterns
├─ Decorator (✅ current)
├─ Observer (Signals/Slots) (✅ current)
├─ Factory (✅ current)
├─ Singleton (⚠️ use sparingly)
└─ Command (not implemented, see EXERCISES.md)

Database Patterns
├─ Repository pattern (✅ current)
├─ Write-through cache (✅ current)
└─ Query object pattern (✅ current)

API Patterns
├─ Decorator-based routing (✅ current)
├─ Type validation (✅ current)
└─ JSON serialization (✅ current)
```

### Pattern Status Legend

- ✅ **CURRENT**: Modern, recommended pattern
- ⚠️ **OUTDATED**: Works but has better alternatives
- 🚨 **DEPRECATED**: Should not be used in new code
- 🔄 **TRANSITIONING**: Being modernized
- 💡 **RECOMMENDED**: Best practice for new code

---

## Naming Conventions

### Python Naming (✅ CURRENT)

**Following PEP 8 standards**:

```python
# Modules: lowercase with underscores
# src/calibre/srv/http_request.py ✅
# src/calibre/srv/HttpRequest.py ❌

# Classes: PascalCase
class BookMetadata:  # ✅
    pass

class book_metadata:  # ❌
    pass

# Functions/methods: snake_case
def get_book_metadata(book_id):  # ✅
    pass

def getBookMetadata(bookId):  # ❌ (camelCase is JavaScript style)
    pass

# Constants: UPPER_CASE
MAX_RESULTS = 100  # ✅
maxResults = 100   # ❌

# Private methods: _leading_underscore
def _internal_helper(self):  # ✅
    pass

# "Really private" (name mangling): __double_underscore
def __dont_call_directly(self):  # ✅
    pass
```

**🔗 Code References**:
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py) - Excellent example of PEP 8 compliance
- [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py) - Consistent naming throughout

**💭 React Comparison**:
```javascript
// React/JavaScript conventions:
class BookMetadata { }      // PascalCase classes (same)
function getBookMetadata() {} // camelCase functions (different!)
const MAX_RESULTS = 100     // UPPER_CASE constants (same)
const _internalHelper = () => {} // _private convention (same)

// Key difference: Python uses snake_case for functions,
// JavaScript uses camelCase
```

---

### File Naming (✅ CURRENT)

```
Python modules: snake_case.py
├─ http_request.py ✅
├─ cache.py ✅
└─ HttpRequest.py ❌

Qt UI files: snake_case.ui
├─ book_details.ui ✅
└─ BookDetails.ui ❌

Test files: test_*.py
├─ test_cache.py ✅
├─ cache_test.py ⚠️ (works but inconsistent)
└─ cacheTest.py ❌
```

**🔗 Examples**:
- [src/calibre/srv/loop.py](../../src/calibre/srv/loop.py)
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py)
- [src/calibre/gui2/library/views.py](../../src/calibre/gui2/library/views.py)

---

### Variable Naming (✅ CURRENT)

```python
# Descriptive names (✅ CURRENT)
book_id = 42
author_names = ['Stephen King']
is_valid = True
has_cover = False

# Avoid abbreviations unless well-known (⚠️)
idx = 0      # ⚠️ Better: index = 0
cnt = 10     # ⚠️ Better: count = 10
db = get_db() # ✅ OK - universally understood

# Loop variables (✅ CURRENT)
for book in books:  # ✅ Descriptive
    pass

for i, book in enumerate(books):  # ✅ When index needed
    pass

for i in range(10):  # ✅ OK for numeric loops
    pass

# Type hints (💡 RECOMMENDED for new code)
def get_book(book_id: int) -> dict:  # ✅
    pass

def get_book(book_id):  # ⚠️ Works but missing type info
    pass
```

**🔗 Code References**:
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py#L200) - Good variable naming
- [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py#L100) - Type hints usage

---

### API Endpoint Naming (✅ CURRENT)

```python
# REST-style naming
@endpoint('/api/books')  # ✅ List all books (GET)
@endpoint('/api/books/{id}')  # ✅ Get single book (GET)
@endpoint('/api/books', methods=['POST'])  # ✅ Create book
@endpoint('/api/books/{id}', methods=['PUT'])  # ✅ Update book
@endpoint('/api/books/{id}', methods=['DELETE'])  # ✅ Delete book

# Nested resources
@endpoint('/api/books/{book_id}/authors')  # ✅
@endpoint('/api/books/{book_id}/formats')  # ✅

# Actions (when not CRUD)
@endpoint('/api/books/{id}/convert')  # ✅ POST to trigger conversion
@endpoint('/api/books/{id}/download')  # ✅ GET to download
@endpoint('/api/search')  # ✅ GET with query params

# Function naming for endpoints
def ajax_get_books(ctx, rd):  # ✅ ajax_ prefix + descriptive
    pass

def ajax_update_book(ctx, rd, book_id):  # ✅
    pass

def book(ctx, rd):  # ❌ Too vague
    pass
```

**🔗 Code References**:
- [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py#L400) - API endpoint examples

**💭 React/Next.js Comparison**:
```javascript
// Next.js file-based routing:
pages/api/books/index.ts       → /api/books
pages/api/books/[id].ts        → /api/books/{id}
pages/api/books/[id]/authors.ts → /api/books/{id}/authors

// Calibre uses decorator-based routing (like Flask/FastAPI)
// Single file, multiple endpoints
```

---

## Code Organization Patterns

### Module Organization (✅ CURRENT)

**Standard module structure**:

```python
"""
Module docstring explaining purpose

This module handles book metadata operations.
"""

# 1. Standard library imports
import os
import sys
from datetime import datetime
from typing import List, Dict, Optional

# 2. Third-party imports
from PyQt6.QtCore import QObject, pyqtSignal
import lxml.etree as ET

# 3. Local imports
from calibre.db import FIELD_MAP
from calibre.srv.errors import HTTPNotFound
from calibre.utils.date import parse_date

# 4. Constants
MAX_RESULTS = 100
DEFAULT_LIMIT = 50

# 5. Module-level functions (if any)
def parse_query(query_string):
    pass

# 6. Classes
class BookMetadata:
    """Class docstring"""

    def __init__(self):
        pass

    def method(self):
        pass

# 7. Main execution (if script)
if __name__ == '__main__':
    main()
```

**🔗 Code References**:
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py#L1) - Good example of module organization
- [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py#L1) - Import organization

---

### Package Structure (✅ CURRENT)

```
calibre/
├─ __init__.py          # Package initialization
├─ constants.py         # Shared constants
├─ db/                  # Database layer
│  ├─ __init__.py
│  ├─ cache.py         # Main cache class
│  ├─ write.py         # Write operations
│  └─ search.py        # Search operations
├─ srv/                 # Server layer
│  ├─ __init__.py
│  ├─ loop.py          # Server loop
│  ├─ routes.py        # API routes
│  └─ errors.py        # HTTP errors
└─ gui2/                # GUI layer
   ├─ __init__.py
   ├─ main.py          # Main window
   └─ widgets.py       # Custom widgets
```

**Key principles**:
1. **Clear separation of concerns**: Database, server, GUI are separate packages
2. **Shallow hierarchies**: Rarely more than 3 levels deep
3. **`__init__.py` imports**: Export main classes for convenient imports

**Example** [src/calibre/db/__init__.py](../../src/calibre/db/__init__.py):
```python
# Export main classes
from calibre.db.cache import Cache
from calibre.db.errors import NoSuchBook

# Now users can do:
# from calibre.db import Cache
# Instead of:
# from calibre.db.cache import Cache
```

**💭 React Comparison**:
```javascript
// React app structure (similar principles):
src/
├─ index.ts           // Entry point
├─ constants.ts       // Shared constants
├─ database/          // Database layer
│  ├─ index.ts       // Export main functions
│  ├─ cache.ts
│  └─ queries.ts
├─ api/               // API layer
│  ├─ index.ts
│  ├─ routes.ts
│  └─ errors.ts
└─ components/        // UI layer
   ├─ index.ts
   └─ BookList.tsx

// Same principles: separation of concerns, clear boundaries
```

---

### Class Organization (✅ CURRENT)

**Standard class structure**:

```python
class BookMetadata:
    """
    Represents book metadata

    Attributes:
        title: Book title
        authors: List of author names
        rating: Rating from 0-5
    """

    # 1. Class variables
    DEFAULT_RATING = 0

    # 2. Constructor
    def __init__(self, title: str, authors: List[str]):
        """Initialize book metadata"""
        self.title = title
        self.authors = authors
        self.rating = self.DEFAULT_RATING

    # 3. Special methods (__str__, __repr__, etc.)
    def __str__(self):
        return f"{self.title} by {', '.join(self.authors)}"

    def __repr__(self):
        return f"BookMetadata(title={self.title!r}, authors={self.authors!r})"

    # 4. Properties
    @property
    def display_title(self):
        """Title formatted for display"""
        return self.title.upper()

    # 5. Public methods (alphabetically)
    def add_author(self, author: str):
        """Add an author to the book"""
        if author not in self.authors:
            self.authors.append(author)

    def set_rating(self, rating: float):
        """Set the book rating"""
        if not 0 <= rating <= 5:
            raise ValueError("Rating must be between 0 and 5")
        self.rating = rating

    # 6. Private methods
    def _validate(self):
        """Internal validation logic"""
        if not self.title:
            raise ValueError("Title cannot be empty")
```

**🔗 Code References**:
- [src/calibre/ebooks/metadata/book/base.py](../../src/calibre/ebooks/metadata/book/base.py) - Metadata class structure

---

## Design Patterns

### Decorator Pattern (✅ CURRENT)

**Used extensively in Calibre for:**
- API endpoint registration
- Database access control
- Caching
- Type validation

**Example: Endpoint Decorator**

[src/calibre/srv/routes.py](../../src/calibre/srv/routes.py#L57):
```python
def endpoint(route, methods=None, types=None):
    """
    Decorator to register API endpoint

    Args:
        route: URL pattern with optional {parameters}
        methods: HTTP methods (default: ['GET'])
        types: Parameter type validation

    Example:
        @endpoint('/api/books/{book_id}', types={'book_id': int})
        def ajax_get_book(ctx, rd, book_id):
            return {'book_id': book_id}
    """
    def decorator(func):
        # Register route
        func.route = route
        func.methods = methods or ['GET']
        func.types = types or {}

        # Return wrapped function
        @wraps(func)
        def wrapper(ctx, rd, **kwargs):
            # Validate types
            for key, expected_type in func.types.items():
                if key in kwargs:
                    kwargs[key] = expected_type(kwargs[key])

            # Call original function
            return func(ctx, rd, **kwargs)

        return wrapper
    return decorator
```

**Usage**:
```python
@endpoint('/api/books/{book_id}/rating', types={'book_id': int}, methods=['POST'])
def ajax_update_rating(ctx, rd, book_id):
    rating = rd.request_body_json.get('rating')
    ctx.db.set_metadata(book_id, {'rating': rating})
    return {'success': True}
```

**💡 Why This Pattern?**
1. **Separation of concerns**: Routing logic separate from business logic
2. **Reusability**: Same decorator for all endpoints
3. **Declarative**: Clear what the function does from decorator
4. **Type safety**: Automatic parameter validation

**💭 React Comparison**:
```javascript
// Higher-Order Component (HOC) in React:
function withAuth(Component) {
  return function AuthenticatedComponent(props) {
    const { isAuthenticated } = useAuth()
    if (!isAuthenticated) return <Login />
    return <Component {...props} />
  }
}

// Usage:
const ProtectedPage = withAuth(DashboardPage)

// Similar to Calibre's decorator pattern:
// - Wraps existing functionality
// - Adds cross-cutting concerns
// - Declarative
```

---

### Database Access Decorators (✅ CURRENT)

[src/calibre/db/cache.py](../../src/calibre/db/cache.py#L100):
```python
def read_api(func):
    """
    Decorator for read-only database operations

    - Automatically handles database locking
    - Provides error handling
    - Logs operation
    """
    @wraps(func)
    def wrapper(self, *args, **kwargs):
        with self.read_lock:
            try:
                return func(self, *args, **kwargs)
            except Exception as e:
                logger.error(f"Error in {func.__name__}: {e}")
                raise
    return wrapper

def write_api(func):
    """
    Decorator for write database operations

    - Acquires write lock
    - Manages transactions
    - Invalidates cache
    - Logs changes
    """
    @wraps(func)
    def wrapper(self, *args, **kwargs):
        with self.write_lock:
            try:
                result = func(self, *args, **kwargs)
                self.commit()
                self._dirty.clear()  # Invalidate cache
                return result
            except Exception as e:
                self.rollback()
                logger.error(f"Error in {func.__name__}: {e}")
                raise
    return wrapper
```

**Usage**:
```python
class Cache:
    @read_api
    def get_metadata(self, book_id):
        """Get book metadata (read-only)"""
        return self.conn.execute(
            'SELECT * FROM books WHERE id = ?', (book_id,)
        ).fetchone()

    @write_api
    def set_metadata(self, book_id, metadata):
        """Update book metadata (write)"""
        self.conn.execute(
            'UPDATE books SET title = ? WHERE id = ?',
            (metadata['title'], book_id)
        )
```

**💡 Benefits**:
1. **Thread safety**: Automatic locking
2. **Transaction management**: Auto commit/rollback
3. **Cache invalidation**: Automatic
4. **Error handling**: Centralized
5. **Logging**: Automatic

---

### Observer Pattern (Signals/Slots) (✅ CURRENT)

**PyQt6's signal/slot mechanism** is a type-safe implementation of the Observer pattern.

**🔗 Code References**:
- [src/calibre/gui2/library/views.py](../../src/calibre/gui2/library/views.py#L200) - Signals in action

**Example**:
```python
from PyQt6.QtCore import QObject, pyqtSignal

class BookListModel(QObject):
    """Model for book list"""

    # Define signals
    bookAdded = pyqtSignal(int)  # Emits book_id
    bookUpdated = pyqtSignal(int, dict)  # Emits book_id, changes
    bookDeleted = pyqtSignal(int)

    def add_book(self, book_data):
        """Add a book"""
        book_id = self.db.add_book(book_data)
        self.bookAdded.emit(book_id)  # Notify observers
        return book_id

    def update_book(self, book_id, changes):
        """Update a book"""
        self.db.set_metadata(book_id, changes)
        self.bookUpdated.emit(book_id, changes)  # Notify observers

class BookListView(QWidget):
    """View for book list"""

    def __init__(self, model):
        super().__init__()
        self.model = model

        # Connect signals to slots
        self.model.bookAdded.connect(self.on_book_added)
        self.model.bookUpdated.connect(self.on_book_updated)
        self.model.bookDeleted.connect(self.on_book_deleted)

    def on_book_added(self, book_id):
        """Handle book added signal"""
        print(f"Book {book_id} was added")
        self.refresh_list()

    def on_book_updated(self, book_id, changes):
        """Handle book updated signal"""
        print(f"Book {book_id} was updated: {changes}")
        self.refresh_book(book_id)
```

**💡 Benefits**:
1. **Loose coupling**: Model doesn't know about views
2. **Multiple observers**: Many views can listen to one model
3. **Type safety**: PyQt6 validates signal types at runtime
4. **Thread safety**: Signals work across threads

**💭 React Comparison**:
```javascript
// React: Props + callbacks
function BookList({ onBookAdded, onBookUpdated }) {
  // Parent passes callbacks
  const handleAdd = (bookData) => {
    const bookId = addBook(bookData)
    onBookAdded(bookId)  // Notify parent
  }

  return <div>...</div>
}

// Or: Event emitter (similar to signals)
const eventBus = new EventEmitter()
eventBus.on('bookAdded', handleBookAdded)
eventBus.emit('bookAdded', bookId)

// Or: React Context + useEffect (for global state)
const BookContext = createContext()
function BookProvider({ children }) {
  const [books, setBooks] = useState([])
  // All children can subscribe to changes
  return <BookContext.Provider value={{books, setBooks}}>
    {children}
  </BookContext.Provider>
}
```

---

### Factory Pattern (✅ CURRENT)

**Used for creating objects based on runtime conditions**

**Example: Widget Factory**

```python
class WidgetFactory:
    """Factory for creating appropriate widgets based on field type"""

    @staticmethod
    def create_editor(field_name, field_type, value):
        """
        Create appropriate editor widget for field

        Args:
            field_name: Name of field (e.g., 'title', 'rating')
            field_type: Type of field (str, int, float, date)
            value: Current value

        Returns:
            QWidget: Appropriate editor widget
        """
        if field_type == str:
            return QLineEdit(value)
        elif field_type == int:
            editor = QSpinBox()
            editor.setValue(value)
            return editor
        elif field_type == float and field_name == 'rating':
            return StarRatingWidget(value)
        elif field_type == datetime:
            editor = QDateEdit()
            editor.setDate(value)
            return editor
        elif field_name == 'comments':
            return QTextEdit(value)
        else:
            return QLineEdit(str(value))

# Usage:
editor = WidgetFactory.create_editor('title', str, 'The Great Gatsby')
# → Returns QLineEdit

rating_editor = WidgetFactory.create_editor('rating', float, 4.5)
# → Returns StarRatingWidget
```

**💡 Benefits**:
1. **Centralized creation logic**: One place to change widget types
2. **Runtime flexibility**: Choose widget based on data
3. **Easier testing**: Can mock factory
4. **Extensibility**: Easy to add new types

---

### Singleton Pattern (⚠️ USE SPARINGLY)

**Used in Calibre for global resources (database connection, settings)**

**Example**:
```python
class DatabaseManager:
    """
    Singleton database manager

    ⚠️ Note: Singletons can make testing difficult.
    Consider dependency injection instead.
    """
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance

    def __init__(self):
        if self._initialized:
            return

        self.conn = None
        self._initialized = True

    def connect(self, db_path):
        if self.conn is None:
            self.conn = sqlite3.connect(db_path)

# Usage:
db1 = DatabaseManager()
db2 = DatabaseManager()
assert db1 is db2  # Same instance
```

**⚠️ Problems with Singletons**:
1. **Global state**: Hard to reason about
2. **Testing difficulty**: Can't easily mock or reset
3. **Hidden dependencies**: Not clear from function signature
4. **Thread safety**: Requires careful synchronization

**💡 Better Alternative: Dependency Injection**:
```python
# Instead of singleton:
class BookManager:
    def get_book(self, book_id):
        db = DatabaseManager()  # ❌ Hidden dependency
        return db.query(...)

# Use dependency injection:
class BookManager:
    def __init__(self, db):  # ✅ Explicit dependency
        self.db = db

    def get_book(self, book_id):
        return self.db.query(...)

# Usage:
db = create_database()
book_manager = BookManager(db)  # Clear dependencies
```

**🔗 Code References**:
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py) - Uses dependency injection, not singleton

---

## Database Patterns

### Repository Pattern (✅ CURRENT)

**Abstracts database access into a repository class**

[src/calibre/db/cache.py](../../src/calibre/db/cache.py#L138):
```python
class Cache:
    """
    Repository for book metadata

    Provides high-level interface to database operations.
    Hides SQL implementation details.
    """

    def __init__(self, conn):
        self.conn = conn
        self._cache = {}

    # Read methods
    @read_api
    def all_book_ids(self):
        """Get set of all book IDs"""
        if 'all_ids' not in self._cache:
            self._cache['all_ids'] = {
                row[0] for row in
                self.conn.execute('SELECT id FROM books')
            }
        return self._cache['all_ids']

    @read_api
    def get_metadata(self, book_id):
        """Get book metadata"""
        return self.conn.execute(
            'SELECT * FROM books WHERE id = ?', (book_id,)
        ).fetchone()

    @read_api
    def authors(self, book_id):
        """Get list of author names for book"""
        return [
            row[0] for row in self.conn.execute('''
                SELECT authors.name
                FROM authors
                JOIN books_authors_link bal ON authors.id = bal.author
                WHERE bal.book = ?
                ORDER BY bal.id
            ''', (book_id,))
        ]

    # Write methods
    @write_api
    def set_metadata(self, book_id, metadata):
        """Update book metadata"""
        for key, value in metadata.items():
            if key == 'title':
                self.conn.execute(
                    'UPDATE books SET title = ? WHERE id = ?',
                    (value, book_id)
                )
            elif key == 'rating':
                self.conn.execute(
                    'UPDATE books SET rating = ? WHERE id = ?',
                    (value, book_id)
                )

        # Invalidate cache
        self._cache.pop('all_ids', None)

    @write_api
    def set_authors(self, book_id, authors):
        """Set authors for book"""
        # Remove existing links
        self.conn.execute(
            'DELETE FROM books_authors_link WHERE book = ?',
            (book_id,)
        )

        # Add new authors
        for author_name in authors:
            # Get or create author
            author_id = self._get_or_create_author(author_name)

            # Link to book
            self.conn.execute(
                'INSERT INTO books_authors_link (book, author) VALUES (?, ?)',
                (book_id, author_id)
            )
```

**💡 Benefits**:
1. **Abstraction**: Business logic doesn't see SQL
2. **Testability**: Can mock repository
3. **Maintainability**: Database changes isolated
4. **Caching**: Can add transparent caching

**💭 React/Prisma Comparison**:
```javascript
// Prisma ORM (similar to repository pattern):
class BookRepository {
  constructor(private prisma: PrismaClient) {}

  async getAllBookIds(): Promise<number[]> {
    const books = await this.prisma.book.findMany({
      select: { id: true }
    })
    return books.map(b => b.id)
  }

  async getMetadata(bookId: number) {
    return await this.prisma.book.findUnique({
      where: { id: bookId },
      include: { authors: true }
    })
  }

  async setMetadata(bookId: number, metadata: any) {
    return await this.prisma.book.update({
      where: { id: bookId },
      data: metadata
    })
  }
}

// Same pattern: high-level interface, hides implementation
```

---

### Write-Through Cache Pattern (✅ CURRENT)

**Calibre's database uses a write-through cache for performance**

[src/calibre/db/cache.py](../../src/calibre/db/cache.py#L200):
```python
class Cache:
    """
    Write-through cache for book metadata

    Data flow:
    1. Read: Check memory cache → Check SQLite → Update cache
    2. Write: Update SQLite → Update memory cache
    """

    def __init__(self, conn):
        self.conn = conn
        self._metadata_cache = {}  # book_id → metadata
        self._authors_cache = {}   # book_id → [author names]
        self._dirty = set()        # IDs that need refresh

    @read_api
    def get_metadata(self, book_id):
        """Get metadata with caching"""
        # Check cache
        if book_id in self._metadata_cache and book_id not in self._dirty:
            return self._metadata_cache[book_id]

        # Cache miss or dirty - fetch from database
        row = self.conn.execute(
            'SELECT * FROM books WHERE id = ?', (book_id,)
        ).fetchone()

        # Update cache
        self._metadata_cache[book_id] = row
        self._dirty.discard(book_id)

        return row

    @write_api
    def set_metadata(self, book_id, metadata):
        """Update metadata (write-through)"""
        # Write to database first
        self.conn.execute(
            'UPDATE books SET title = ?, rating = ? WHERE id = ?',
            (metadata.get('title'), metadata.get('rating'), book_id)
        )

        # Update cache
        if book_id in self._metadata_cache:
            self._metadata_cache[book_id].update(metadata)
        else:
            # Fetch and cache
            self._metadata_cache[book_id] = self.get_metadata(book_id)

        # Mark as clean (just updated)
        self._dirty.discard(book_id)

    def invalidate(self, book_id):
        """Mark cache entry as dirty"""
        self._dirty.add(book_id)
```

**💡 Cache Strategy Comparison**:

```
Write-Through (✅ Calibre uses this)
├─ Write: Update DB → Update cache
├─ Pros: Cache always consistent, simple
└─ Cons: Write latency includes cache update

Write-Behind (Alternative)
├─ Write: Update cache → Queue DB write
├─ Pros: Fast writes
└─ Cons: Risk of data loss, complex

Write-Around (Alternative)
├─ Write: Update DB only, invalidate cache
├─ Pros: Avoids cache pollution
└─ Cons: Next read is slow
```

**💭 React/Redis Comparison**:
```javascript
// Similar pattern with Redis cache:
class CachedBookRepository {
  constructor(
    private db: Database,
    private redis: RedisClient
  ) {}

  async getMetadata(bookId: number) {
    // Try cache first
    const cached = await this.redis.get(`book:${bookId}`)
    if (cached) {
      return JSON.parse(cached)
    }

    // Cache miss - fetch from DB
    const metadata = await this.db.query(
      'SELECT * FROM books WHERE id = ?', [bookId]
    )

    // Update cache
    await this.redis.setex(
      `book:${bookId}`,
      3600,  // TTL: 1 hour
      JSON.stringify(metadata)
    )

    return metadata
  }

  async setMetadata(bookId: number, metadata: any) {
    // Write-through: Update DB first
    await this.db.query(
      'UPDATE books SET title = ?, rating = ? WHERE id = ?',
      [metadata.title, metadata.rating, bookId]
    )

    // Then update cache
    await this.redis.setex(
      `book:${bookId}`,
      3600,
      JSON.stringify(metadata)
    )
  }
}
```

---

### Query Object Pattern (✅ CURRENT)

**Encapsulates complex queries in objects**

[src/calibre/db/search.py](../../src/calibre/db/search.py):
```python
class SearchQuery:
    """
    Represents a search query

    Builds SQL from high-level search terms.
    """

    def __init__(self, db):
        self.db = db
        self.conditions = []
        self.parameters = []

    def where_title_contains(self, text):
        """Add title search condition"""
        self.conditions.append('books.title LIKE ?')
        self.parameters.append(f'%{text}%')
        return self  # Fluent interface

    def where_author_is(self, author_name):
        """Add author condition"""
        self.conditions.append('''
            EXISTS (
                SELECT 1 FROM authors
                JOIN books_authors_link bal ON authors.id = bal.author
                WHERE bal.book = books.id
                AND authors.name = ?
            )
        ''')
        self.parameters.append(author_name)
        return self

    def where_rating_gte(self, min_rating):
        """Add minimum rating condition"""
        self.conditions.append('books.rating >= ?')
        self.parameters.append(min_rating)
        return self

    def execute(self):
        """Execute the query"""
        if not self.conditions:
            # No conditions - return all books
            sql = 'SELECT id FROM books ORDER BY title'
            return self.db.conn.execute(sql).fetchall()

        # Build WHERE clause
        where_clause = ' AND '.join(self.conditions)
        sql = f'''
            SELECT DISTINCT books.id
            FROM books
            WHERE {where_clause}
            ORDER BY books.title
        '''

        return self.db.conn.execute(sql, self.parameters).fetchall()

# Usage (fluent interface):
results = (SearchQuery(db)
    .where_title_contains('python')
    .where_rating_gte(4.0)
    .execute())

# Or step by step:
query = SearchQuery(db)
query.where_author_is('Stephen King')
query.where_rating_gte(4.5)
results = query.execute()
```

**💡 Benefits**:
1. **Composable**: Build queries programmatically
2. **Type-safe**: Better than string concatenation
3. **Testable**: Easy to mock
4. **Readable**: Fluent interface is clear
5. **Reusable**: Can save and reuse queries

**💭 Modern ORM Comparison**:
```javascript
// Prisma query builder (similar pattern):
const results = await prisma.book.findMany({
  where: {
    title: { contains: 'python' },
    rating: { gte: 4.0 }
  },
  orderBy: { title: 'asc' }
})

// TypeORM query builder:
const results = await getRepository(Book)
  .createQueryBuilder('book')
  .where('book.title LIKE :title', { title: '%python%' })
  .andWhere('book.rating >= :rating', { rating: 4.0 })
  .orderBy('book.title', 'ASC')
  .getMany()

// Same pattern: Composable, type-safe, readable
```

---

## GUI Patterns

### Model-View Pattern (✅ CURRENT)

**PyQt6 uses Model-View architecture** (similar to React's unidirectional data flow)

**🔗 Code References**:
- [src/calibre/gui2/library/models.py](../../src/calibre/gui2/library/models.py) - Table models
- [src/calibre/gui2/library/views.py](../../src/calibre/gui2/library/views.py) - Views

**Example**:
```python
from PyQt6.QtCore import QAbstractTableModel, Qt, pyqtSignal

class BookListModel(QAbstractTableModel):
    """
    Model for book list (data layer)

    Responsibilities:
    - Store book data
    - Handle data changes
    - Notify views of changes
    """

    # Signal when data changes
    dataChanged = pyqtSignal()

    def __init__(self, db):
        super().__init__()
        self.db = db
        self.books = []
        self.load_books()

    def load_books(self):
        """Load books from database"""
        self.books = [
            self.db.get_metadata(book_id)
            for book_id in self.db.all_book_ids()
        ]
        self.dataChanged.emit()

    # Required QAbstractTableModel methods
    def rowCount(self, parent=None):
        return len(self.books)

    def columnCount(self, parent=None):
        return 3  # Title, Authors, Rating

    def data(self, index, role=Qt.ItemDataRole.DisplayRole):
        """Get data for cell"""
        if not index.isValid():
            return None

        book = self.books[index.row()]
        col = index.column()

        if role == Qt.ItemDataRole.DisplayRole:
            if col == 0:
                return book.title
            elif col == 1:
                return ', '.join(self.db.authors(book.id))
            elif col == 2:
                return book.rating or 0

        return None

    def headerData(self, section, orientation, role=Qt.ItemDataRole.DisplayRole):
        """Get column headers"""
        if role == Qt.ItemDataRole.DisplayRole and orientation == Qt.Orientation.Horizontal:
            return ['Title', 'Authors', 'Rating'][section]
        return None

    # Custom methods
    def update_rating(self, row, new_rating):
        """Update book rating"""
        book = self.books[row]
        self.db.set_metadata(book.id, {'rating': new_rating})
        book.rating = new_rating

        # Notify views
        top_left = self.index(row, 2)  # Rating column
        bottom_right = top_left
        self.dataChanged.emit(top_left, bottom_right)


class BookListView(QTableView):
    """
    View for book list (presentation layer)

    Responsibilities:
    - Display data from model
    - Handle user interactions
    - Send signals on user actions
    """

    # Signal when user changes rating
    ratingChanged = pyqtSignal(int, float)  # row, new_rating

    def __init__(self, model):
        super().__init__()
        self.setModel(model)

        # Configure view
        self.setSelectionBehavior(QTableView.SelectionBehavior.SelectRows)
        self.setAlternatingRowColors(True)
        self.setSortingEnabled(True)

        # Connect signals
        model.dataChanged.connect(self.on_data_changed)

    def on_data_changed(self):
        """Handle model data change"""
        self.viewport().update()  # Refresh display

    def mouseDoubleClickEvent(self, event):
        """Handle double-click to edit rating"""
        index = self.indexAt(event.pos())
        if index.column() == 2:  # Rating column
            self.edit_rating(index.row())
        super().mouseDoubleClickEvent(event)

    def edit_rating(self, row):
        """Show rating editor"""
        # ... show dialog to edit rating ...
        new_rating = rating_dialog.get_rating()

        # Emit signal (controller will handle)
        self.ratingChanged.emit(row, new_rating)


class MainWindow(QMainWindow):
    """
    Controller: Connects model and view
    """

    def __init__(self, db):
        super().__init__()

        # Create model and view
        self.model = BookListModel(db)
        self.view = BookListView(self.model)

        # Connect view signals to controller methods
        self.view.ratingChanged.connect(self.on_rating_changed)

        # Set as central widget
        self.setCentralWidget(self.view)

    def on_rating_changed(self, row, new_rating):
        """Handle rating change from view"""
        # Update model
        self.model.update_rating(row, new_rating)

        # Could do other things here:
        # - Show notification
        # - Update statistics
        # - Sync to server
```

**Data Flow**:
```
User Action (View)
    ↓
Signal emitted
    ↓
Controller handles signal
    ↓
Updates Model
    ↓
Model emits dataChanged
    ↓
View refreshes display
```

**💭 React Comparison**:
```javascript
// Similar unidirectional data flow in React:

// Model (state management):
const useBookStore = create((set) => ({
  books: [],
  loadBooks: async (db) => {
    const books = await db.getAllBooks()
    set({ books })
  },
  updateRating: (bookId, rating) => {
    set((state) => ({
      books: state.books.map(book =>
        book.id === bookId ? { ...book, rating } : book
      )
    }))
  }
}))

// View (component):
function BookListView() {
  const { books, updateRating } = useBookStore()

  const handleRatingChange = (bookId, newRating) => {
    // Update local state
    updateRating(bookId, newRating)

    // Persist to backend
    api.updateBookRating(bookId, newRating)
  }

  return (
    <table>
      {books.map(book => (
        <BookRow
          key={book.id}
          book={book}
          onRatingChange={(rating) => handleRatingChange(book.id, rating)}
        />
      ))}
    </table>
  )
}

// Data flow:
// User clicks → handleRatingChange → updateRating (store) → Component re-renders
```

**Both patterns** separate concerns and use unidirectional data flow!

---

### Delegate Pattern (Custom Renderers) (✅ CURRENT)

**Custom cell rendering in Qt tables/lists**

```python
from PyQt6.QtWidgets import QStyledItemDelegate, QStyleOptionViewItem
from PyQt6.QtCore import Qt, QRect
from PyQt6.QtGui import QPainter

class StarRatingDelegate(QStyledItemDelegate):
    """
    Custom delegate to render rating as stars

    Responsibilities:
    - Paint star rating in table cell
    - Handle editor creation for editing
    """

    def paint(self, painter: QPainter, option: QStyleOptionViewItem, index):
        """
        Custom paint method

        Called by Qt to render each cell in the rating column.
        """
        # Get rating value (0-5)
        rating = index.data(Qt.ItemDataRole.DisplayRole)
        if rating is None:
            return

        # Calculate display
        full_stars = int(rating)
        has_half = (rating % 1) >= 0.5

        # Draw background
        painter.save()
        if option.state & QStyle.StateFlag.State_Selected:
            painter.fillRect(option.rect, option.palette.highlight())

        # Draw stars
        star_size = 16
        x = option.rect.x() + 5
        y = option.rect.y() + (option.rect.height() - star_size) // 2

        for i in range(5):
            if i < full_stars:
                star_char = '★'
            elif i == full_stars and has_half:
                star_char = '⯪'
            else:
                star_char = '☆'

            painter.drawText(x + i * star_size, y, star_size, star_size,
                           Qt.AlignmentFlag.AlignCenter, star_char)

        painter.restore()

    def createEditor(self, parent, option, index):
        """Create editor widget for cell"""
        from calibre.gui2.widgets import StarRatingWidget
        editor = StarRatingWidget(parent=parent)
        editor.setAutoFillBackground(True)
        return editor

    def setEditorData(self, editor, index):
        """Load current value into editor"""
        rating = index.data(Qt.ItemDataRole.DisplayRole)
        editor.set_rating(rating)

    def setModelData(self, editor, model, index):
        """Save editor value back to model"""
        rating = editor.get_rating()
        model.setData(index, rating, Qt.ItemDataRole.EditRole)

# Usage:
table_view = QTableView()
table_view.setItemDelegateForColumn(2, StarRatingDelegate())  # Column 2 = Rating
```

**💭 React Comparison**:
```javascript
// React: Custom cell renderer
function BookTable({ books }) {
  const columns = [
    { header: 'Title', accessor: 'title' },
    { header: 'Author', accessor: 'author' },
    {
      header: 'Rating',
      accessor: 'rating',
      // Custom cell renderer (like Qt delegate)
      Cell: ({ value }) => (
        <StarRating rating={value} onChange={handleRatingChange} />
      )
    }
  ]

  return <Table data={books} columns={columns} />
}

function StarRating({ rating, onChange }) {
  const fullStars = Math.floor(rating)
  const hasHalf = (rating % 1) >= 0.5

  return (
    <div>
      {[...Array(5)].map((_, i) => {
        if (i < fullStars) return <span key={i}>★</span>
        if (i === fullStars && hasHalf) return <span key={i}>⯪</span>
        return <span key={i}>☆</span>
      })}
    </div>
  )
}
```

Similar concepts: custom rendering logic for specific columns/cells!

---

## API Patterns

### RESTful Endpoint Design (✅ CURRENT)

**Calibre follows REST principles for API endpoints**

**Resource-based URLs**:
```python
# Books collection
@endpoint('/api/books')  # GET: List all books
@endpoint('/api/books', methods=['POST'])  # POST: Create book
@endpoint('/api/books/{book_id}', types={'book_id': int})  # GET: Get single book
@endpoint('/api/books/{book_id}', types={'book_id': int}, methods=['PUT'])  # PUT: Update book
@endpoint('/api/books/{book_id}', types={'book_id': int}, methods=['DELETE'])  # DELETE: Delete book

# Nested resources
@endpoint('/api/books/{book_id}/authors', types={'book_id': int})  # GET: Book's authors
@endpoint('/api/books/{book_id}/formats', types={'book_id': int})  # GET: Book's formats

# Actions (not pure REST, but pragmatic)
@endpoint('/api/books/{book_id}/convert', types={'book_id': int}, methods=['POST'])
# POST /api/books/42/convert {"to_format": "EPUB"}

@endpoint('/api/books/{book_id}/download', types={'book_id': int})
# GET /api/books/42/download?format=EPUB
```

**HTTP Status Codes** (✅ CURRENT):
```python
# Success codes
return {'data': ...}  # 200 OK (implicit)
return {'data': ...}, 201  # 201 Created (explicit)

# Error codes
raise HTTPNotFound('Book not found')  # 404
raise HTTPBadRequest('Invalid rating')  # 400
raise HTTPForbidden('Access denied')  # 403
raise HTTPServerError('Database error')  # 500
```

**💡 REST Principles**:
1. **Resources**: URLs represent resources (books, authors)
2. **HTTP methods**: Use GET/POST/PUT/DELETE correctly
3. **Stateless**: Each request is independent
4. **Standard responses**: Consistent JSON structure
5. **HTTP semantics**: Use status codes correctly

---

### Request/Response Format (✅ CURRENT)

**Standard JSON response format**:

```python
# Success response
{
    "data": {...},
    "meta": {
        "timestamp": "2025-11-19T10:30:00Z",
        "version": "1.0"
    }
}

# List response with pagination
{
    "data": [...],
    "meta": {
        "total": 1000,
        "page": 1,
        "per_page": 50,
        "pages": 20
    }
}

# Error response
{
    "error": {
        "message": "Book not found",
        "code": "BOOK_NOT_FOUND",
        "details": {
            "book_id": 999
        }
    }
}
```

**Implementation**:
```python
def format_success_response(data, meta=None):
    """Format successful API response"""
    response = {'data': data}
    if meta:
        response['meta'] = meta
    return response

def format_error_response(message, code=None, details=None, status=400):
    """Format error API response"""
    error = {'message': message}
    if code:
        error['code'] = code
    if details:
        error['details'] = details

    return {'error': error}, status

# Usage:
@endpoint('/api/books')
def ajax_get_books(ctx, rd):
    books = get_all_books(ctx.db)

    return format_success_response(
        data=books,
        meta={
            'total': len(books),
            'timestamp': datetime.utcnow().isoformat() + 'Z'
        }
    )

@endpoint('/api/books/{book_id}', types={'book_id': int})
def ajax_get_book(ctx, rd, book_id):
    if book_id not in ctx.db.all_book_ids():
        return format_error_response(
            message=f'Book {book_id} not found',
            code='BOOK_NOT_FOUND',
            details={'book_id': book_id},
            status=404
        )

    book = ctx.db.get_metadata(book_id)
    return format_success_response(data=book)
```

---

## Error Handling Patterns

### Exception Hierarchy (✅ CURRENT)

**Calibre defines custom exceptions for different error types**

[src/calibre/srv/errors.py](../../src/calibre/srv/errors.py):
```python
class HTTPError(Exception):
    """Base class for all HTTP errors"""
    status_code = 500

    def __init__(self, message='Internal Server Error', details=None):
        self.message = message
        self.details = details or {}
        super().__init__(message)

    def to_dict(self):
        return {
            'error': {
                'message': self.message,
                'code': self.__class__.__name__,
                'details': self.details
            }
        }

class HTTPNotFound(HTTPError):
    """404 Not Found"""
    status_code = 404

    def __init__(self, message='Not Found', details=None):
        super().__init__(message, details)

class HTTPBadRequest(HTTPError):
    """400 Bad Request"""
    status_code = 400

    def __init__(self, message='Bad Request', details=None):
        super().__init__(message, details)

class HTTPForbidden(HTTPError):
    """403 Forbidden"""
    status_code = 403

    def __init__(self, message='Forbidden', details=None):
        super().__init__(message, details)

class HTTPServerError(HTTPError):
    """500 Internal Server Error"""
    status_code = 500

# Usage:
@endpoint('/api/books/{book_id}', types={'book_id': int})
def ajax_get_book(ctx, rd, book_id):
    if book_id not in ctx.db.all_book_ids():
        raise HTTPNotFound(
            f'Book {book_id} not found',
            details={'book_id': book_id}
        )

    try:
        book = ctx.db.get_metadata(book_id)
        return {'data': book}
    except DatabaseError as e:
        logger.error(f'Database error: {e}')
        raise HTTPServerError(
            'Failed to fetch book',
            details={'error': str(e)}
        )
```

**💡 Benefits**:
1. **Type safety**: Can catch specific exceptions
2. **Automatic status codes**: Each exception knows its HTTP status
3. **Consistent errors**: All errors have same format
4. **Traceable**: Easy to log and debug

**💭 React/Express Comparison**:
```javascript
// Express middleware for error handling:
class HTTPError extends Error {
  constructor(message, statusCode = 500, details = {}) {
    super(message)
    this.statusCode = statusCode
    this.details = details
  }

  toJSON() {
    return {
      error: {
        message: this.message,
        code: this.constructor.name,
        details: this.details
      }
    }
  }
}

class NotFoundError extends HTTPError {
  constructor(message = 'Not Found', details) {
    super(message, 404, details)
  }
}

// Express error handler:
app.use((err, req, res, next) => {
  if (err instanceof HTTPError) {
    res.status(err.statusCode).json(err.toJSON())
  } else {
    res.status(500).json({ error: { message: 'Internal Server Error' } })
  }
})

// Usage in route:
app.get('/api/books/:id', async (req, res) => {
  const book = await db.getBook(req.params.id)
  if (!book) {
    throw new NotFoundError(`Book ${req.params.id} not found`, {
      bookId: req.params.id
    })
  }
  res.json({ data: book })
})
```

Same pattern: custom exception classes with HTTP status codes!

---

### Try-Except Patterns (✅ CURRENT)

**Best practices for error handling**:

```python
# Pattern 1: Catch specific exceptions (✅ CURRENT)
try:
    book = ctx.db.get_metadata(book_id)
except NoSuchBook:
    raise HTTPNotFound(f'Book {book_id} not found')
except DatabaseError as e:
    logger.error(f'Database error: {e}')
    raise HTTPServerError('Database operation failed')

# Pattern 2: Catch and re-raise (✅ CURRENT)
try:
    result = complex_operation()
except Exception as e:
    logger.exception('Operation failed')  # Logs full traceback
    raise  # Re-raise same exception

# Pattern 3: Multiple except blocks (✅ CURRENT)
try:
    data = json.loads(request_body)
    validate_schema(data)
    save_to_database(data)
except json.JSONDecodeError:
    raise HTTPBadRequest('Invalid JSON')
except ValidationError as e:
    raise HTTPBadRequest(f'Validation failed: {e}')
except DatabaseError:
    raise HTTPServerError('Failed to save data')

# Pattern 4: Finally for cleanup (✅ CURRENT)
file_handle = None
try:
    file_handle = open(file_path, 'r')
    data = file_handle.read()
    process(data)
except IOError as e:
    logger.error(f'File error: {e}')
    raise
finally:
    if file_handle:
        file_handle.close()  # Always runs

# Pattern 5: Context managers (💡 RECOMMENDED)
# Better than try/finally
with open(file_path, 'r') as f:
    data = f.read()
    process(data)
# File automatically closed

# Pattern 6: Suppress specific exceptions (⚠️ USE CAREFULLY)
from contextlib import suppress

with suppress(KeyError):
    del my_dict['key']  # Ignore if key doesn't exist

# Equivalent to:
try:
    del my_dict['key']
except KeyError:
    pass

# ❌ ANTI-PATTERNS:

# Don't catch all exceptions without re-raising
try:
    dangerous_operation()
except Exception:  # ❌ Swallows all errors, hard to debug
    pass

# Don't use bare except
try:
    operation()
except:  # ❌ Catches even KeyboardInterrupt, SystemExit
    pass

# Don't raise generic exceptions
raise Exception('Something went wrong')  # ❌ Use specific exception class

# Don't return error codes (use exceptions)
def get_book(book_id):
    if book_id not in books:
        return None, 'Not found'  # ❌ Makes error handling messy
    return book, None
```

---

## Testing Patterns

### Unit Test Structure (✅ CURRENT)

```python
import unittest
from unittest.mock import Mock, patch, MagicMock

class TestBookMetadata(unittest.TestCase):
    """
    Test cases for BookMetadata class

    Structure:
    - setUp: Run before each test
    - tearDown: Run after each test
    - test_*: Individual test methods
    """

    def setUp(self):
        """Set up test fixtures"""
        self.db = Mock()  # Mock database
        self.book_id = 1
        self.metadata = {
            'title': 'Test Book',
            'rating': 4.5
        }

    def tearDown(self):
        """Clean up after test"""
        # Close connections, delete temp files, etc.
        pass

    def test_get_metadata_returns_correct_data(self):
        """Test that get_metadata returns correct book data"""
        # Arrange
        self.db.get_metadata.return_value = self.metadata

        # Act
        result = self.db.get_metadata(self.book_id)

        # Assert
        self.assertEqual(result['title'], 'Test Book')
        self.assertEqual(result['rating'], 4.5)
        self.db.get_metadata.assert_called_once_with(self.book_id)

    def test_get_metadata_raises_on_invalid_id(self):
        """Test that get_metadata raises for invalid book ID"""
        # Arrange
        self.db.get_metadata.side_effect = NoSuchBook('Book not found')

        # Act & Assert
        with self.assertRaises(NoSuchBook):
            self.db.get_metadata(999)

    def test_set_rating_validates_range(self):
        """Test that set_rating validates rating is 0-5"""
        book = BookMetadata('Title', ['Author'])

        # Valid ratings
        book.set_rating(0)
        book.set_rating(5)
        book.set_rating(3.5)

        # Invalid ratings
        with self.assertRaises(ValueError):
            book.set_rating(-1)

        with self.assertRaises(ValueError):
            book.set_rating(6)

    @patch('calibre.db.cache.sqlite3.connect')
    def test_database_connection(self, mock_connect):
        """Test database connection with patching"""
        # Arrange
        mock_conn = MagicMock()
        mock_connect.return_value = mock_conn

        # Act
        cache = Cache('/path/to/db')

        # Assert
        mock_connect.assert_called_once_with('/path/to/db')

# Run tests
if __name__ == '__main__':
    unittest.main()
```

**💭 Jest (JavaScript) Comparison**:
```javascript
describe('BookMetadata', () => {
  let db, bookId, metadata

  beforeEach(() => {
    db = mock<Database>()
    bookId = 1
    metadata = { title: 'Test Book', rating: 4.5 }
  })

  afterEach(() => {
    // Cleanup
  })

  it('should return correct book data', () => {
    // Arrange
    db.getMetadata.mockReturnValue(metadata)

    // Act
    const result = db.getMetadata(bookId)

    // Assert
    expect(result.title).toBe('Test Book')
    expect(result.rating).toBe(4.5)
    expect(db.getMetadata).toHaveBeenCalledWith(bookId)
  })

  it('should raise error for invalid book ID', () => {
    // Arrange
    db.getMetadata.mockImplementation(() => {
      throw new NotFoundError('Book not found')
    })

    // Act & Assert
    expect(() => db.getMetadata(999)).toThrow(NotFoundError)
  })
})
```

Very similar patterns!

---

## Performance Patterns

### Lazy Loading (✅ CURRENT)

**Load data only when needed**:

```python
class BookMetadata:
    """Book metadata with lazy-loaded properties"""

    def __init__(self, db, book_id):
        self.db = db
        self.book_id = book_id

        # Eager loading (loaded immediately)
        self.title = db.get_title(book_id)

        # Lazy loading (loaded on first access)
        self._authors = None
        self._comments = None
        self._cover = None

    @property
    def authors(self):
        """Lazy-load authors"""
        if self._authors is None:
            self._authors = self.db.authors(self.book_id)
        return self._authors

    @property
    def comments(self):
        """Lazy-load comments (large text)"""
        if self._comments is None:
            self._comments = self.db.comments(self.book_id)
        return self._comments

    @property
    def cover(self):
        """Lazy-load cover image (large binary)"""
        if self._cover is None:
            self._cover = self.db.cover(self.book_id)
        return self._cover

# Usage:
book = BookMetadata(db, 1)
print(book.title)  # ✅ Already loaded
print(book.authors)  # ✅ Loads now (on first access)
print(book.authors)  # ✅ Uses cached value
```

**💡 When to Use**:
- Large data (images, long text)
- Rarely accessed fields
- Expensive computations
- Nested relationships

---

### Batch Operations (✅ CURRENT)

**Process multiple items in single operation**:

```python
# ❌ BAD: N queries
def update_ratings_individually(db, book_ratings):
    """Update ratings one at a time (slow!)"""
    for book_id, rating in book_ratings.items():
        db.execute(
            'UPDATE books SET rating = ? WHERE id = ?',
            (rating, book_id)
        )
        db.commit()  # Commit after each update

# ✅ GOOD: Single transaction
def update_ratings_batch(db, book_ratings):
    """Update all ratings in one transaction (fast!)"""
    db.execute('BEGIN TRANSACTION')

    try:
        for book_id, rating in book_ratings.items():
            db.execute(
                'UPDATE books SET rating = ? WHERE id = ?',
                (rating, book_id)
            )
        db.commit()  # Single commit
    except Exception:
        db.rollback()
        raise

# ✅ EVEN BETTER: Bulk update with executemany
def update_ratings_bulk(db, book_ratings):
    """Bulk update using executemany (fastest!)"""
    data = [(rating, book_id) for book_id, rating in book_ratings.items()]

    db.executemany(
        'UPDATE books SET rating = ? WHERE id = ?',
        data
    )
    db.commit()

# Benchmark:
book_ratings = {i: random.uniform(0, 5) for i in range(1000)}

# Method 1: 5000ms
# Method 2: 500ms (10x faster!)
# Method 3: 100ms (50x faster!)
```

---

## Security Patterns

### SQL Injection Prevention (✅ CURRENT)

**Always use parameter binding**:

```python
# ❌ VULNERABLE: SQL injection
def search_books_unsafe(db, title):
    """NEVER DO THIS!"""
    query = f"SELECT * FROM books WHERE title LIKE '%{title}%'"
    return db.execute(query).fetchall()

# Attacker input:
# title = "'; DROP TABLE books; --"
# Result: SQL INJECTION ATTACK!

# ✅ SAFE: Parameterized query
def search_books_safe(db, title):
    """Use parameter binding"""
    query = "SELECT * FROM books WHERE title LIKE ?"
    return db.execute(query, (f'%{title}%',)).fetchall()

# Parameterization prevents injection:
# title = "'; DROP TABLE books; --"
# Result: Searches for literal string "'; DROP TABLE books; --"
```

**💡 Rules**:
1. **ALWAYS** use `?` placeholders
2. **NEVER** use f-strings or `%` formatting for SQL
3. **Use** `execute(query, params)` pattern
4. **Libraries** handle escaping automatically

---

## Pattern Status Reference

### ✅ CURRENT - Recommended Patterns

These patterns are modern, well-maintained, and recommended for new code:

- **Decorator pattern** for API endpoints
- **Repository pattern** for database access
- **Write-through cache** for performance
- **Model-View pattern** for GUI
- **Exception hierarchy** for error handling
- **Type hints** for documentation
- **Context managers** for resource management
- **Parameterized queries** for SQL safety
- **Batch operations** for performance
- **Lazy loading** for optimization

### ⚠️ OUTDATED - Use with Caution

These patterns work but have better modern alternatives:

- **Singleton pattern** → Use dependency injection
- **String formatting** (`%s`) → Use f-strings
- **Manual resource cleanup** → Use context managers
- **Bare except** → Catch specific exceptions

### 🚨 DEPRECATED - Avoid in New Code

These patterns should not be used:

- **exec() / eval()** for dynamic code → Use proper data structures
- **Global variables** for state → Pass explicitly
- **Thread unsafe operations** → Use locks or asyncio
- **Raw SQL without parameters** → Use parameter binding

---

## Summary

### Key Takeaways for React Developers

1. **Decorators ≈ Higher-Order Components**
   - Both wrap functionality
   - Both add cross-cutting concerns

2. **Signals/Slots ≈ Props/Callbacks**
   - Both for component communication
   - Both type-safe (in their own ways)

3. **Model-View ≈ State Management**
   - Both separate data from presentation
   - Both use unidirectional data flow

4. **Repository ≈ Data Layer**
   - Both abstract database access
   - Both provide high-level API

5. **Naming Conventions Different**
   - Python: `snake_case` functions
   - JavaScript: `camelCase` functions
   - Both: `PascalCase` classes

### Next Steps

1. Read through actual Calibre code using these patterns
2. Try the exercises in [EXERCISES.md](EXERCISES.md)
3. Contribute code following these conventions
4. Share feedback on what's unclear!

---

*This guide is a living document. Please suggest improvements!*
