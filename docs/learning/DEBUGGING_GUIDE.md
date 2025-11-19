# Calibre Debugging Guide

> **For React developers**: This guide bridges Python debugging tools with familiar JavaScript/React debugging workflows. Think of `pdb` as Chrome DevTools console, `logging` as `console.log`, and `cProfile` as the Performance tab.

---

## Table of Contents

1. [Overview](#overview)
2. [Debugging Philosophy](#debugging-philosophy)
3. [Python Debugging Tools](#python-debugging-tools)
4. [Interactive Debugging with pdb](#interactive-debugging-with-pdb)
5. [VS Code Debugging Setup](#vs-code-debugging-setup)
6. [PyQt6 GUI Debugging](#pyqt6-gui-debugging)
7. [Database Query Debugging](#database-query-debugging)
8. [HTTP Server Debugging](#http-server-debugging)
9. [Performance Profiling](#performance-profiling)
10. [Memory Debugging](#memory-debugging)
11. [Logging Best Practices](#logging-best-practices)
12. [Common Debugging Patterns](#common-debugging-patterns)
13. [Troubleshooting Guide](#troubleshooting-guide)

---

## Overview

Debugging in Python/Calibre is different from JavaScript/React debugging, but the core principles are the same: **understand the state**, **trace the execution**, and **isolate the problem**.

### Quick Comparison

| Task | JavaScript/React | Python/Calibre |
|------|-----------------|----------------|
| **Breakpoints** | Chrome DevTools | `pdb`, VS Code debugger |
| **Console logging** | `console.log()` | `print()`, `logging` |
| **React component state** | React DevTools | PyQt6 object inspector |
| **Network inspection** | Network tab | `logging` in `routes.py` |
| **Performance profiling** | Performance tab | `cProfile`, `line_profiler` |
| **Memory leaks** | Heap snapshots | `tracemalloc`, `objgraph` |
| **Error stack traces** | Browser console | Terminal output, logs |

---

## Debugging Philosophy

### The Calibre Debugging Mindset

```python
# ❌ BAD: Random print statements
def process_book(book_id):
    print("here")  # What does this tell you?
    book = db.get_metadata(book_id)
    print("got book")  # Still vague
    return book

# ✅ GOOD: Structured logging with context
def process_book(book_id):
    logger.debug(f"Processing book_id={book_id}")

    book = db.get_metadata(book_id)
    logger.debug(f"Retrieved book: {book.title} (id={book_id})")

    if book is None:
        logger.warning(f"Book {book_id} not found in database")

    return book
```

**React comparison**: Just like you use React DevTools to inspect component state and props, use Python's debugger to inspect object attributes and local variables.

### Debugging Layers

Calibre has three main debugging layers:

1. **Application Layer** (GUI, HTTP server)
   - [src/calibre/gui2/main.py](../../src/calibre/gui2/main.py) - Main window
   - [src/calibre/srv/loop.py](../../src/calibre/srv/loop.py) - HTTP server

2. **Business Logic Layer** (Book processing, metadata)
   - [src/calibre/ebooks/](../../src/calibre/ebooks/) - Format conversions
   - [src/calibre/library/](../../src/calibre/library/) - Library operations

3. **Data Layer** (Database, cache)
   - [src/calibre/db/cache.py](../../src/calibre/db/cache.py) - Database cache
   - [src/calibre/db/tables.py](../../src/calibre/db/tables.py) - SQL operations

---

## Python Debugging Tools

### 1. Built-in `pdb` (Python Debugger)

**React equivalent**: Chrome DevTools debugger

```python
import pdb

def convert_book(book_id, output_format):
    """Convert book to different format"""
    book = get_metadata(book_id)

    # Set breakpoint - execution will pause here
    pdb.set_trace()

    # Now you can inspect variables, step through code
    converter = get_converter(output_format)
    result = converter.convert(book)
    return result
```

**Modern alternative** (Python 3.7+):
```python
def convert_book(book_id, output_format):
    book = get_metadata(book_id)

    # Built-in breakpoint() - cleaner syntax
    breakpoint()

    converter = get_converter(output_format)
    result = converter.convert(book)
    return result
```

### 2. `ipdb` (IPython Debugger)

**Better than pdb**: Syntax highlighting, tab completion, better history

```bash
# Install
pip install ipdb

# Use
import ipdb; ipdb.set_trace()
```

### 3. `debugpy` (VS Code Remote Debugging)

Used by VS Code for debugging. See [VS Code Setup](#vs-code-debugging-setup) below.

### 4. Post-mortem Debugging

Debug crashes without modifying code:

```python
import pdb
import sys

def main():
    try:
        # Your code that might crash
        result = risky_operation()
    except Exception:
        # Drop into debugger at exception point
        pdb.post_mortem(sys.exc_info()[2])
        raise

# Or use it globally
if __name__ == '__main__':
    import pdb
    try:
        main()
    except:
        pdb.post_mortem()
```

---

## Interactive Debugging with pdb

### Basic Commands

| Command | Shortcut | Description | React DevTools Equivalent |
|---------|----------|-------------|---------------------------|
| `list` | `l` | Show current code | Source tab |
| `next` | `n` | Step over (next line) | Step over button |
| `step` | `s` | Step into function | Step into button |
| `continue` | `c` | Continue execution | Resume button |
| `break` | `b` | Set breakpoint | Click line number |
| `print` | `p` | Print variable | Console |
| `pp` | `pp` | Pretty-print variable | Console |
| `where` | `w` | Show stack trace | Call stack panel |
| `up` | `u` | Go up stack frame | Click parent in stack |
| `down` | `d` | Go down stack frame | Click child in stack |
| `quit` | `q` | Exit debugger | Stop button |

### Example Debugging Session

Let's debug the book metadata update in [src/calibre/db/cache.py](../../src/calibre/db/cache.py#L138):

```python
# File: src/calibre/db/cache.py
def set_metadata(self, book_id, metadata, ignore_errors=False):
    """Update book metadata"""
    import pdb; pdb.set_trace()  # START DEBUGGING HERE

    if not isinstance(book_id, int):
        raise ValueError(f"book_id must be int, got {type(book_id)}")

    # Update fields
    self.set_field('title', book_id, metadata.title)
    self.set_field('authors', book_id, metadata.authors)

    return True
```

**Debugging session**:
```
(Pdb) # Execution paused at breakpoint
(Pdb) p book_id
123

(Pdb) p metadata
<BookMetadata title='1984' authors=['George Orwell']>

(Pdb) p metadata.title
'1984'

(Pdb) pp metadata.__dict__
{'title': '1984',
 'authors': ['George Orwell'],
 'rating': 4.5,
 'tags': ['Fiction', 'Dystopian'],
 'pubdate': datetime.datetime(1949, 6, 8)}

(Pdb) n  # Execute next line
> /calibre/db/cache.py(147)
-> self.set_field('title', book_id, metadata.title)

(Pdb) s  # Step into set_field()
> /calibre/db/cache.py(89)set_field()
-> def set_field(self, field, book_id, value):

(Pdb) w  # Show where we are in the stack
  /calibre/gui2/actions/edit_metadata.py(234)
-> cache.set_metadata(book_id, metadata)
  /calibre/db/cache.py(145)set_metadata()
-> self.set_field('title', book_id, metadata.title)
> /calibre/db/cache.py(89)set_field()
-> def set_field(self, field, book_id, value):

(Pdb) c  # Continue execution
```

### Conditional Breakpoints

Only break when a condition is true:

```python
def process_books(book_ids):
    for book_id in book_ids:
        # Only break for book_id == 42
        if book_id == 42:
            breakpoint()

        process_book(book_id)
```

Or using pdb commands:
```
(Pdb) break cache.py:147, book_id == 42
Breakpoint 1 at /calibre/db/cache.py:147
```

### Advanced pdb Commands

```python
# List breakpoints
(Pdb) break
Num Type         Disp Enb   Where
1   breakpoint   keep yes   at /calibre/db/cache.py:147

# Disable/enable breakpoints
(Pdb) disable 1
(Pdb) enable 1

# Clear breakpoints
(Pdb) clear 1

# Execute Python code
(Pdb) !book_ids = [1, 2, 3]
(Pdb) !print(f"Processing {len(book_ids)} books")
Processing 3 books

# Call functions
(Pdb) !self.get_metadata(123).title
'1984'
```

---

## VS Code Debugging Setup

### Configuration File

Create `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Calibre GUI",
            "type": "python",
            "request": "launch",
            "program": "${workspaceFolder}/src/calibre/gui2/main.py",
            "console": "integratedTerminal",
            "justMyCode": false,
            "env": {
                "CALIBRE_DEVELOP_FROM": "${workspaceFolder}/src",
                "PYTHONPATH": "${workspaceFolder}/src"
            }
        },
        {
            "name": "Python: Content Server",
            "type": "python",
            "request": "launch",
            "program": "${workspaceFolder}/src/calibre/srv/standalone.py",
            "args": [
                "--port", "8080",
                "--with-library", "/path/to/test/library"
            ],
            "console": "integratedTerminal",
            "justMyCode": false
        },
        {
            "name": "Python: Current File",
            "type": "python",
            "request": "launch",
            "program": "${file}",
            "console": "integratedTerminal",
            "justMyCode": false
        },
        {
            "name": "Python: Pytest Current File",
            "type": "python",
            "request": "launch",
            "module": "pytest",
            "args": [
                "${file}",
                "-v"
            ],
            "console": "integratedTerminal",
            "justMyCode": false
        },
        {
            "name": "Python: Attach to Process",
            "type": "python",
            "request": "attach",
            "processId": "${command:pickProcess}",
            "justMyCode": false
        }
    ]
}
```

### Using VS Code Debugger

1. **Set breakpoints**: Click left of line number (red dot appears)
2. **Start debugging**: Press `F5` or click "Run and Debug"
3. **Debug toolbar appears**:
   - Continue (F5)
   - Step Over (F10)
   - Step Into (F11)
   - Step Out (Shift+F11)
   - Restart (Ctrl+Shift+F5)
   - Stop (Shift+F5)

4. **Panels**:
   - **Variables**: See all local/global variables (like React DevTools state)
   - **Watch**: Add expressions to monitor
   - **Call Stack**: See execution path
   - **Breakpoints**: Manage all breakpoints

### Debug Console

**React equivalent**: Browser console

```python
# In Debug Console (while paused at breakpoint):
book_id
# Output: 123

metadata.title
# Output: '1984'

[book.title for book in db.all_books()]
# Output: ['1984', 'Brave New World', 'Fahrenheit 451']

# Call methods
db.get_metadata(123).rating
# Output: 4.5
```

### Logpoints (Non-breaking Breakpoints)

**Like `console.log()` but without modifying code**:

1. Right-click line number
2. Select "Add Logpoint"
3. Enter expression: `Book ID: {book_id}, Title: {metadata.title}`
4. Code runs without stopping, but logs to Debug Console

---

## PyQt6 GUI Debugging

### 1. Widget Inspector

**React DevTools equivalent**: Inspect component hierarchy

```python
from PyQt6.QtWidgets import QApplication

def debug_widget_tree(widget, indent=0):
    """Print widget hierarchy (like React DevTools)"""
    print('  ' * indent + f'{widget.__class__.__name__} ({widget.objectName()})')
    for child in widget.children():
        if isinstance(child, QWidget):
            debug_widget_tree(child, indent + 1)

# Usage in main window
app = QApplication.instance()
main_window = app.activeWindow()
debug_widget_tree(main_window)

# Output:
# MainWindow (main_window)
#   QWidget (central_widget)
#     QVBoxLayout ()
#       BookList (book_list)
#       QHBoxLayout ()
#         QPushButton (add_button)
#         QPushButton (remove_button)
```

**See in action**: [src/calibre/gui2/main.py](../../src/calibre/gui2/main.py#L56)

### 2. Signal/Slot Debugging

**Signals are like React events** - debug them similarly:

```python
from PyQt6.QtCore import QObject, pyqtSignal

class BookListDebug(QObject):
    # Enable signal debugging
    bookSelected = pyqtSignal(int)

    def __init__(self):
        super().__init__()
        # Log all signal emissions
        self.bookSelected.connect(self._debug_signal)

    def _debug_signal(self, book_id):
        import traceback
        print(f"Signal bookSelected emitted with book_id={book_id}")
        print("Call stack:")
        traceback.print_stack()

    def select_book(self, book_id):
        print(f"Selecting book {book_id}")
        self.bookSelected.emit(book_id)  # This will log above

# Or use Qt's built-in debugging
import os
os.environ['QT_LOGGING_RULES'] = 'qt.qpa.*=true'
```

**Example in Calibre**: [src/calibre/gui2/library/views.py](../../src/calibre/gui2/library/views.py#L234)

### 3. Event Debugging

Debug mouse/keyboard events:

```python
from PyQt6.QtCore import QEvent
from PyQt6.QtWidgets import QWidget

class DebugWidget(QWidget):
    def event(self, event):
        """Intercept ALL events for debugging"""
        print(f"Event: {event.type()} ({QEvent.Type(event.type()).name})")
        return super().event(event)

    def mousePressEvent(self, event):
        """Debug mouse clicks"""
        print(f"Mouse clicked at ({event.x()}, {event.y()})")
        print(f"Button: {event.button()}")
        print(f"Modifiers: {event.modifiers()}")
        super().mousePressEvent(event)
```

### 4. Layout Debugging

**React equivalent**: Inspecting flexbox/grid layouts

```python
def debug_layout(layout, indent=0):
    """Print layout structure"""
    print('  ' * indent + f'{layout.__class__.__name__}')
    for i in range(layout.count()):
        item = layout.itemAt(i)
        if item.widget():
            print('  ' * (indent + 1) + f'Widget: {item.widget().__class__.__name__}')
        elif item.layout():
            debug_layout(item.layout(), indent + 1)

# Find layout issues
def check_layout_constraints(widget):
    """Check for common layout problems"""
    if not widget.layout():
        print(f"WARNING: {widget} has no layout")

    if widget.minimumSize().width() > widget.maximumSize().width():
        print(f"ERROR: {widget} min size > max size")

    for child in widget.children():
        if isinstance(child, QWidget):
            check_layout_constraints(child)
```

### 5. Performance Debugging (GUI)

**React equivalent**: React Profiler

```python
from PyQt6.QtCore import QElapsedTimer

class PerformanceDebugMixin:
    """Mixin to track widget render performance"""

    def paintEvent(self, event):
        timer = QElapsedTimer()
        timer.start()

        super().paintEvent(event)

        elapsed = timer.elapsed()
        if elapsed > 16:  # Slower than 60 FPS
            print(f"WARNING: {self.__class__.__name__} paint took {elapsed}ms")
```

**Example**: [src/calibre/gui2/library/views.py](../../src/calibre/gui2/library/views.py) - book list rendering

---

## Database Query Debugging

### 1. SQL Query Logging

**React equivalent**: Redux Logger middleware

Enable SQL logging in [src/calibre/db/cache.py](../../src/calibre/db/cache.py):

```python
class Cache:
    def __init__(self, db_path):
        self.conn = apsw.Connection(db_path)

        # Enable SQL tracing (like Redux logger)
        if DEBUG:
            self.conn.setexectrace(self._trace_sql)

    def _trace_sql(self, cursor, sql, bindings):
        """Log all SQL queries"""
        print(f"\n--- SQL Query ---")
        print(f"SQL: {sql}")
        if bindings:
            print(f"Bindings: {bindings}")

        # Measure execution time
        import time
        start = time.time()

        def trace_result(cursor, sql, bindings):
            elapsed = (time.time() - start) * 1000
            print(f"Took: {elapsed:.2f}ms")
            print(f"Rows affected: {cursor.getdescription()}")

        return trace_result
```

**Real usage**: Set environment variable:
```bash
export CALIBRE_DEBUG_SQL=1
calibre-debug
```

### 2. Query Explain Plans

**Like "Analyze" in PostgreSQL**:

```python
def explain_query(db, sql):
    """Show query execution plan"""
    cursor = db.conn.cursor()

    # SQLite's EXPLAIN QUERY PLAN
    explain_sql = f"EXPLAIN QUERY PLAN {sql}"
    results = cursor.execute(explain_sql).fetchall()

    print("\n--- Query Plan ---")
    for row in results:
        print(row)

    # Check for performance issues
    plan_text = ' '.join(str(r) for r in results)
    if 'SCAN' in plan_text and 'USING INDEX' not in plan_text:
        print("⚠️  WARNING: Full table scan detected!")

# Usage
explain_query(db, """
    SELECT b.id, b.title
    FROM books b
    JOIN books_authors_link bal ON b.id = bal.book
    WHERE bal.author = 5
""")

# Output:
# --- Query Plan ---
# (0, 0, 0, 'SEARCH books_authors_link USING INDEX books_authors_link_aidx (author=?)')
# (0, 0, 0, 'SEARCH books USING INTEGER PRIMARY KEY (rowid=?)')
```

**Example queries**: [src/calibre/db/tables.py](../../src/calibre/db/tables.py#L89)

### 3. Transaction Debugging

```python
class TransactionDebugger:
    """Debug transaction boundaries"""

    def __init__(self, conn):
        self.conn = conn
        self.transaction_depth = 0

    def begin(self):
        self.transaction_depth += 1
        print(f"BEGIN TRANSACTION (depth={self.transaction_depth})")
        self.conn.cursor().execute('BEGIN')

    def commit(self):
        print(f"COMMIT (depth={self.transaction_depth})")
        self.conn.cursor().execute('COMMIT')
        self.transaction_depth -= 1

    def rollback(self):
        print(f"ROLLBACK (depth={self.transaction_depth})")
        self.conn.cursor().execute('ROLLBACK')
        self.transaction_depth -= 1

    def __enter__(self):
        self.begin()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            self.rollback()
        else:
            self.commit()

# Usage
with TransactionDebugger(db.conn):
    db.set_field('title', 123, 'New Title')
    db.set_field('authors', 123, ['New Author'])
# Output:
# BEGIN TRANSACTION (depth=1)
# COMMIT (depth=1)
```

### 4. Cache Debugging

**Like React Query DevTools**:

```python
class CacheDebugger:
    """Debug database cache hits/misses"""

    def __init__(self, cache):
        self.cache = cache
        self.stats = {'hits': 0, 'misses': 0, 'writes': 0}

    def get_metadata(self, book_id):
        if book_id in self.cache._metadata_cache:
            self.stats['hits'] += 1
            print(f"✅ Cache HIT for book {book_id}")
        else:
            self.stats['misses'] += 1
            print(f"❌ Cache MISS for book {book_id}")

        return self.cache.get_metadata(book_id)

    def print_stats(self):
        total = self.stats['hits'] + self.stats['misses']
        if total > 0:
            hit_rate = (self.stats['hits'] / total) * 100
            print(f"\n--- Cache Statistics ---")
            print(f"Hits: {self.stats['hits']}")
            print(f"Misses: {self.stats['misses']}")
            print(f"Hit Rate: {hit_rate:.1f}%")
            print(f"Writes: {self.stats['writes']}")
```

**See cache implementation**: [src/calibre/db/cache.py](../../src/calibre/db/cache.py#L138)

---

## HTTP Server Debugging

### 1. Request/Response Logging

**Like browser Network tab**:

```python
# In src/calibre/srv/loop.py
class DebugLoop(ServerLoop):
    def handle_request(self, conn, request):
        """Log all HTTP requests"""
        print(f"\n--- HTTP Request ---")
        print(f"Method: {request.method}")
        print(f"Path: {request.path}")
        print(f"Headers: {dict(request.headers)}")
        if request.body:
            print(f"Body: {request.body[:200]}...")  # First 200 chars

        # Time the request
        import time
        start = time.time()

        response = super().handle_request(conn, request)

        elapsed = (time.time() - start) * 1000
        print(f"\n--- HTTP Response ---")
        print(f"Status: {response.status_code}")
        print(f"Time: {elapsed:.2f}ms")
        if elapsed > 100:
            print(f"⚠️  SLOW REQUEST: {request.path}")

        return response
```

**Enable via environment variable**:
```bash
export CALIBRE_DEBUG_HTTP=1
calibre-server
```

### 2. Endpoint Debugging

Debug specific routes in [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py):

```python
from functools import wraps

def debug_endpoint(func):
    """Decorator to debug endpoint calls"""
    @wraps(func)
    def wrapper(ctx, rd, *args, **kwargs):
        print(f"\n--- Endpoint: {func.__name__} ---")
        print(f"Args: {args}")
        print(f"Kwargs: {kwargs}")
        print(f"Context: {ctx}")
        print(f"Request data: {rd.__dict__}")

        try:
            result = func(ctx, rd, *args, **kwargs)
            print(f"Result: {result}")
            return result
        except Exception as e:
            print(f"❌ ERROR: {e}")
            import traceback
            traceback.print_exc()
            raise

    return wrapper

# Usage
@endpoint('/api/books/{book_id}', types={'book_id': int})
@debug_endpoint
def ajax_get_book(ctx, rd, book_id):
    return ctx.db.get_metadata(book_id)
```

### 3. WebSocket Debugging

Debug WebSocket connections:

```python
class DebugWebSocket:
    """Debug WebSocket messages"""

    def __init__(self, ws):
        self.ws = ws
        self.message_count = 0

    def send(self, message):
        self.message_count += 1
        print(f"→ WS Send #{self.message_count}: {message[:100]}...")
        self.ws.send(message)

    def receive(self):
        message = self.ws.receive()
        self.message_count += 1
        print(f"← WS Receive #{self.message_count}: {message[:100]}...")
        return message
```

---

## Performance Profiling

### 1. Function-Level Profiling with cProfile

**React equivalent**: Chrome Performance tab

```python
import cProfile
import pstats

def profile_function(func):
    """Decorator to profile a function"""
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()

        result = func(*args, **kwargs)

        profiler.disable()

        # Print stats
        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')
        stats.print_stats(20)  # Top 20 functions

        return result

    return wrapper

# Usage
@profile_function
def process_library():
    """Process all books in library"""
    for book_id in db.all_book_ids():
        book = db.get_metadata(book_id)
        # ... processing

# Output:
#    ncalls  tottime  percall  cumtime  percall filename:lineno(function)
#      100    0.050    0.001    2.500    0.025 cache.py:147(get_metadata)
#      100    0.100    0.001    2.000    0.020 tables.py:89(execute_query)
#      500    1.800    0.004    1.800    0.004 {method 'execute' of 'apsw.Cursor'}
```

**Profile entire script**:
```bash
python -m cProfile -o profile.stats script.py

# Analyze results
python -c "import pstats; p = pstats.Stats('profile.stats'); p.sort_stats('cumulative'); p.print_stats(20)"
```

### 2. Line-Level Profiling with line_profiler

**More granular than cProfile** - see which lines are slow:

```bash
# Install
pip install line_profiler
```

```python
# Add @profile decorator (no import needed!)
@profile
def convert_book(book_id):
    book = db.get_metadata(book_id)           # Line 1
    converter = get_converter('epub')          # Line 2
    result = converter.convert(book)           # Line 3
    save_result(result)                        # Line 4
    return result

# Run with line profiler
kernprof -l -v script.py

# Output:
# Line #      Hits         Time  Per Hit   % Time  Line Contents
# ==============================================================
#      1         1        2.3     2.3      0.1     book = db.get_metadata(book_id)
#      2         1        0.1     0.1      0.0     converter = get_converter('epub')
#      3         1     2100.5  2100.5     99.5     result = converter.convert(book)
#      4         1        8.2     8.2      0.4     save_result(result)
#
# ⚠️ Line 3 takes 99.5% of time!
```

### 3. Memory Profiling with memory_profiler

**Find memory leaks**:

```bash
pip install memory_profiler
```

```python
from memory_profiler import profile

@profile
def load_large_library():
    books = []
    for book_id in range(10000):
        book = db.get_metadata(book_id)
        books.append(book)  # Memory grows here
    return books

# Run
python -m memory_profiler script.py

# Output:
# Line #    Mem usage    Increment   Line Contents
# ================================================
#      1     50.0 MiB     50.0 MiB   @profile
#      2                             def load_large_library():
#      3     50.0 MiB      0.0 MiB       books = []
#      4    250.5 MiB    200.5 MiB       for book_id in range(10000):
#      5    250.5 MiB      0.0 MiB           book = db.get_metadata(book_id)
#      6    250.5 MiB      0.0 MiB           books.append(book)
#      7    250.5 MiB      0.0 MiB       return books
```

### 4. Flame Graphs (Visual Profiling)

**Like Chrome Performance flame charts**:

```bash
# Install py-spy
pip install py-spy

# Profile running process
sudo py-spy record -o profile.svg --pid $(pgrep -f calibre)

# Profile script
py-spy record -o profile.svg -- python script.py

# Open profile.svg in browser - interactive flame graph!
```

### 5. Real-World Profiling Example

Profile the book conversion in [src/calibre/ebooks/conversion/](../../src/calibre/ebooks/conversion/):

```python
import cProfile
import pstats
from calibre.ebooks.conversion.plumber import Plumber

def profile_conversion():
    """Profile MOBI to EPUB conversion"""
    profiler = cProfile.Profile()
    profiler.enable()

    # Convert book
    plumber = Plumber('input.mobi', 'output.epub', log)
    plumber.run()

    profiler.disable()

    # Analyze results
    stats = pstats.Stats(profiler)
    stats.sort_stats('cumulative')

    print("\n=== Top 20 Slowest Functions ===")
    stats.print_stats(20)

    print("\n=== Functions that call 'parse_html' ===")
    stats.print_callers('parse_html')

    print("\n=== Functions called by 'convert_images' ===")
    stats.print_callees('convert_images')

profile_conversion()
```

---

## Memory Debugging

### 1. Track Memory Allocation with tracemalloc

**Built into Python 3.4+**:

```python
import tracemalloc

def debug_memory():
    """Track memory allocations"""
    tracemalloc.start()

    # Snapshot before
    snapshot1 = tracemalloc.take_snapshot()

    # Code that might leak memory
    books = []
    for i in range(10000):
        book = create_book(i)
        books.append(book)

    # Snapshot after
    snapshot2 = tracemalloc.take_snapshot()

    # Compare snapshots
    top_stats = snapshot2.compare_to(snapshot1, 'lineno')

    print("\n=== Top 10 Memory Allocations ===")
    for stat in top_stats[:10]:
        print(stat)

    # Output:
    # cache.py:147: size=15.2 MiB (+15.0 MiB), count=10000 (+10000), average=1.6 KiB
    # tables.py:89: size=5.1 MiB (+5.0 MiB), count=50000 (+50000), average=107 B

debug_memory()
```

### 2. Find Reference Cycles with objgraph

**Find objects that aren't garbage collected**:

```bash
pip install objgraph
```

```python
import objgraph

def debug_reference_leak():
    """Find objects that leak references"""

    # Create some objects
    books = create_many_books(1000)

    # Show most common types
    print("\n=== Most Common Objects ===")
    objgraph.show_most_common_types(limit=10)
    # Output:
    # dict         15234
    # tuple         8567
    # list          3421
    # BookMetadata  1000  ← Our objects

    # Find reference cycles
    print("\n=== Reference Chain to BookMetadata ===")
    book = books[0]
    objgraph.show_chain(
        objgraph.find_backref_chain(book, objgraph.is_proper_module),
        filename='chain.png'
    )
    # Creates visual graph showing what's holding references
```

### 3. Memory Leak Detection

**Track object creation over time**:

```python
import gc
import sys

def find_memory_leaks():
    """Detect objects that never get freed"""

    # Force garbage collection
    gc.collect()

    # Count objects before
    before = len(gc.get_objects())

    # Run suspicious code
    for i in range(100):
        process_book(i)

    # Force garbage collection again
    gc.collect()

    # Count objects after
    after = len(gc.get_objects())

    if after > before + 1000:  # More than 1000 new objects
        print(f"⚠️  MEMORY LEAK: {after - before} objects not freed")

        # Find what leaked
        new_objects = gc.get_objects()[before:after]
        types = {}
        for obj in new_objects:
            t = type(obj).__name__
            types[t] = types.get(t, 0) + 1

        print("\nLeaked object types:")
        for t, count in sorted(types.items(), key=lambda x: x[1], reverse=True)[:10]:
            print(f"  {t}: {count}")
```

### 4. Monitor Process Memory

```python
import psutil
import os

def monitor_memory(func):
    """Decorator to monitor memory usage"""
    def wrapper(*args, **kwargs):
        process = psutil.Process(os.getpid())

        # Memory before
        mem_before = process.memory_info().rss / 1024 / 1024  # MB
        print(f"Memory before: {mem_before:.1f} MB")

        result = func(*args, **kwargs)

        # Memory after
        mem_after = process.memory_info().rss / 1024 / 1024
        print(f"Memory after: {mem_after:.1f} MB")
        print(f"Memory increase: {mem_after - mem_before:.1f} MB")

        return result

    return wrapper

@monitor_memory
def load_library():
    books = db.all_books()
    return books

# Output:
# Memory before: 45.2 MB
# Memory after: 245.8 MB
# Memory increase: 200.6 MB
```

---

## Logging Best Practices

### 1. Calibre Logging Setup

**React equivalent**: Structured console logging

Calibre uses Python's `logging` module. See [src/calibre/__init__.py](../../src/calibre/__init__.py):

```python
import logging

# Create logger for your module
logger = logging.getLogger('calibre.mymodule')

def process_book(book_id):
    logger.debug(f"Starting to process book {book_id}")

    try:
        book = db.get_metadata(book_id)
        logger.info(f"Retrieved book: {book.title}")

        if not book.has_cover():
            logger.warning(f"Book {book_id} missing cover")

        result = convert_book(book)
        logger.info(f"Conversion successful: {book_id} → {result.format}")

        return result

    except Exception as e:
        logger.error(f"Failed to process book {book_id}: {e}", exc_info=True)
        raise
```

### 2. Log Levels

**Choose appropriate level**:

```python
# DEBUG: Detailed diagnostic info (only in development)
logger.debug(f"Cache hit for book_id={book_id}, metadata={metadata}")

# INFO: Confirmation that things are working
logger.info(f"Converted {count} books to EPUB")

# WARNING: Something unexpected, but app still works
logger.warning(f"Book {book_id} has no ISBN, using title for lookup")

# ERROR: Serious problem, but app can continue
logger.error(f"Failed to download cover for {book_id}: {e}")

# CRITICAL: Serious error, app might not continue
logger.critical(f"Database corruption detected at {db_path}")
```

### 3. Structured Logging

**Better than string formatting**:

```python
# ❌ BAD: Hard to parse
logger.info(f"User {user} converted {book} from {input_fmt} to {output_fmt} in {elapsed}s")

# ✅ GOOD: Structured, parseable
logger.info(
    "Book converted",
    extra={
        'user': user,
        'book_id': book.id,
        'input_format': input_fmt,
        'output_format': output_fmt,
        'elapsed_seconds': elapsed,
        'file_size_mb': file_size / 1024 / 1024
    }
)

# Can be parsed by log aggregators (Datadog, Splunk, etc.)
```

### 4. Context Managers for Logging

```python
import logging
import time
from contextlib import contextmanager

@contextmanager
def log_operation(operation_name):
    """Log operation start, end, and duration"""
    logger = logging.getLogger('calibre')

    logger.info(f"Starting: {operation_name}")
    start = time.time()

    try:
        yield
        elapsed = time.time() - start
        logger.info(f"Completed: {operation_name} in {elapsed:.2f}s")
    except Exception as e:
        elapsed = time.time() - start
        logger.error(f"Failed: {operation_name} after {elapsed:.2f}s", exc_info=True)
        raise

# Usage
with log_operation("Convert 100 books to EPUB"):
    for book_id in book_ids:
        convert_book(book_id)

# Output:
# INFO: Starting: Convert 100 books to EPUB
# INFO: Completed: Convert 100 books to EPUB in 45.23s
```

### 5. Debug Mode

**Enable verbose logging**:

```python
import logging
import os

# In your main script
if os.environ.get('CALIBRE_DEBUG'):
    logging.basicConfig(
        level=logging.DEBUG,
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            logging.FileHandler('calibre-debug.log'),
            logging.StreamHandler()
        ]
    )
else:
    logging.basicConfig(level=logging.INFO)

# Now run with debug mode
# CALIBRE_DEBUG=1 python script.py
```

### 6. Rotating Log Files

**Prevent logs from filling disk**:

```python
from logging.handlers import RotatingFileHandler

logger = logging.getLogger('calibre')
handler = RotatingFileHandler(
    'calibre.log',
    maxBytes=10 * 1024 * 1024,  # 10 MB
    backupCount=5  # Keep 5 old log files
)
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
handler.setFormatter(formatter)
logger.addHandler(handler)

# Creates: calibre.log, calibre.log.1, calibre.log.2, etc.
```

---

## Common Debugging Patterns

### 1. Binary Search Debugging

**Find which commit introduced a bug**:

```bash
git bisect start
git bisect bad          # Current commit is bad
git bisect good v5.0.0  # v5.0.0 was good

# Git checks out middle commit
# Test if bug exists
git bisect bad  # or "git bisect good"

# Repeat until git finds the first bad commit
git bisect reset  # Done
```

### 2. Print Debugging (Strategic)

**React equivalent**: `console.log` with strategy

```python
# ❌ BAD: Random prints
def process():
    print("here")
    x = calculate()
    print("got x")
    return x

# ✅ GOOD: Strategic prints with context
def process():
    print(f"\n{'='*50}")
    print(f"process() called from {__name__}")
    print(f"{'='*50}")

    x = calculate()
    print(f"calculate() returned: {x!r} (type: {type(x).__name__})")

    if x is None:
        print("⚠️  WARNING: calculate() returned None")
        import traceback
        traceback.print_stack()

    return x
```

### 3. Assertion Debugging

**Catch bugs early**:

```python
def set_rating(self, book_id, rating):
    """Set book rating (0-5)"""

    # Assertions for development (removed in production with -O flag)
    assert isinstance(book_id, int), f"book_id must be int, got {type(book_id)}"
    assert 0 <= rating <= 5, f"rating must be 0-5, got {rating}"
    assert self.has_id(book_id), f"book_id {book_id} doesn't exist"

    self._set_field('rating', book_id, rating)

# Run with assertions enabled (default)
python script.py

# Run with assertions disabled (production)
python -O script.py
```

### 4. Exception Debugging

**Get detailed error info**:

```python
import sys
import traceback

def detailed_exception():
    """Print detailed exception information"""
    exc_type, exc_value, exc_tb = sys.exc_info()

    print("\n=== Exception Details ===")
    print(f"Type: {exc_type.__name__}")
    print(f"Message: {exc_value}")
    print(f"\nTraceback:")

    # Print full traceback
    traceback.print_tb(exc_tb)

    # Print local variables at exception point
    print("\n=== Local Variables ===")
    frame = exc_tb.tb_frame
    for key, value in frame.f_locals.items():
        print(f"  {key} = {value!r}")

# Usage
try:
    risky_operation()
except Exception:
    detailed_exception()
    raise
```

### 5. Timing Debugging

**Find slow code**:

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(name):
    """Time a block of code"""
    start = time.time()
    yield
    elapsed = (time.time() - start) * 1000
    print(f"{name}: {elapsed:.2f}ms")

# Usage
with timer("Load all books"):
    books = db.all_books()  # Load all books: 1234.56ms

with timer("Filter by author"):
    filtered = [b for b in books if 'Orwell' in b.authors]  # Filter by author: 23.45ms
```

### 6. Conditional Debugging

**Debug only specific cases**:

```python
DEBUG_BOOK_ID = 42  # Only debug this book

def process_book(book_id):
    if book_id == DEBUG_BOOK_ID:
        breakpoint()  # Only break for book 42

    # Or use logging
    if book_id == DEBUG_BOOK_ID:
        logger.setLevel(logging.DEBUG)
    else:
        logger.setLevel(logging.WARNING)

    # Process book...
```

---

## Troubleshooting Guide

### Common Issues and Solutions

#### 1. "Module not found" Error

```python
# ERROR: ModuleNotFoundError: No module named 'calibre'

# Solution 1: Set PYTHONPATH
export PYTHONPATH=/path/to/calibre/src:$PYTHONPATH

# Solution 2: Install in development mode
cd /path/to/calibre
pip install -e .

# Solution 3: Add to sys.path in code
import sys
sys.path.insert(0, '/path/to/calibre/src')
```

#### 2. PyQt6 GUI Not Showing

```python
# Problem: GUI window doesn't appear

# Solution: Check if QApplication exists
from PyQt6.QtWidgets import QApplication
import sys

app = QApplication.instance()
if app is None:
    app = QApplication(sys.argv)

window = MainWindow()
window.show()

# MUST call exec() to start event loop
sys.exit(app.exec())  # Don't forget this!
```

#### 3. Database Locked Error

```python
# ERROR: apsw.BusyError: database is locked

# Solution 1: Close other connections
db.conn.close()

# Solution 2: Increase timeout
db.conn.setbusytimeout(5000)  # 5 seconds

# Solution 3: Use WAL mode (better concurrency)
db.conn.cursor().execute('PRAGMA journal_mode=WAL')
```

#### 4. "Segmentation Fault" (Crash)

```bash
# Crashes are often caused by:
# 1. PyQt6 C++ object deleted while Python still references it
# 2. Invalid memory access in C extension

# Debug with gdb
gdb python
(gdb) run script.py
# When it crashes:
(gdb) backtrace  # Show C stack trace

# Or use faulthandler
python -X faulthandler script.py
```

#### 5. Import Circular Dependencies

```python
# ERROR: ImportError: cannot import name 'X' from partially initialized module

# Problem: A imports B, B imports A

# Solution: Use lazy imports
def process_book(book_id):
    from calibre.db.cache import Cache  # Import here, not at top
    cache = Cache(db_path)
    # ...
```

#### 6. Signal/Slot Connection Issues

```python
# Problem: Signal emitted but slot not called

# Solution 1: Check connection return value
result = self.bookSelected.connect(self.on_book_selected)
if not result:
    print("ERROR: Connection failed!")

# Solution 2: Use lambda to debug
self.bookSelected.connect(lambda book_id: print(f"Signal emitted: {book_id}"))

# Solution 3: Check if object was deleted
if self.sender() is None:
    print("ERROR: Sender was deleted!")
```

#### 7. Performance Issues

```python
# Problem: GUI freezing

# Solution 1: Move work to background thread
from PyQt6.QtCore import QThread

class WorkerThread(QThread):
    def run(self):
        # Do heavy work here
        process_books()

thread = WorkerThread()
thread.start()

# Solution 2: Use QTimer to process incrementally
from PyQt6.QtCore import QTimer

def process_next_book():
    book_id = book_queue.pop(0)
    process_book(book_id)
    if book_queue:
        QTimer.singleShot(0, process_next_book)  # Schedule next

QTimer.singleShot(0, process_next_book)
```

---

## Quick Reference

### Debugging Commands Cheatsheet

```python
# Set breakpoint
breakpoint()              # Python 3.7+
import pdb; pdb.set_trace()  # Python 3.6 and earlier

# pdb commands
n          # Next line
s          # Step into function
c          # Continue execution
l          # List code
p var      # Print variable
pp var     # Pretty-print variable
w          # Show stack trace
u          # Up stack frame
d          # Down stack frame
q          # Quit debugger

# Logging
import logging
logger = logging.getLogger(__name__)
logger.debug("Debug message")
logger.info("Info message")
logger.warning("Warning message")
logger.error("Error message")
logger.critical("Critical message")

# Profiling
python -m cProfile script.py
python -m cProfile -o profile.stats script.py

# Memory debugging
import tracemalloc
tracemalloc.start()
snapshot = tracemalloc.take_snapshot()
```

### VS Code Keyboard Shortcuts

| Action | Windows/Linux | macOS |
|--------|---------------|-------|
| Start debugging | F5 | F5 |
| Step over | F10 | F10 |
| Step into | F11 | F11 |
| Step out | Shift+F11 | Shift+F11 |
| Continue | F5 | F5 |
| Toggle breakpoint | F9 | F9 |
| Debug console | Ctrl+Shift+Y | Cmd+Shift+Y |

---

## Next Steps

Now that you understand debugging:

1. **Practice**: Use debugger on a real Calibre feature
2. **Read**: [TESTING_GUIDE.md](./TESTING_GUIDE.md) - Write tests to prevent bugs
3. **Read**: [DEVELOPMENT_WORKFLOW.md](./DEVELOPMENT_WORKFLOW.md) - Professional workflow
4. **Read**: [SECURITY_GUIDE.md](./SECURITY_GUIDE.md) - Debug security issues

---

## Additional Resources

### Official Documentation
- [Python pdb docs](https://docs.python.org/3/library/pdb.html)
- [VS Code debugging](https://code.visualstudio.com/docs/editor/debugging)
- [PyQt6 debugging](https://doc.qt.io/qt-6/debug.html)
- [cProfile docs](https://docs.python.org/3/library/profile.html)

### Calibre-Specific
- [src/calibre/debug.py](../../src/calibre/debug.py) - Calibre's debug utilities
- [src/calibre/utils/logging.py](../../src/calibre/utils/logging.py) - Logging utilities

### Tools
- [pdb++](https://github.com/pdbpp/pdbpp) - Enhanced pdb
- [ipdb](https://github.com/gotcha/ipdb) - IPython debugger
- [pudb](https://github.com/inducer/pudb) - Visual debugger
- [py-spy](https://github.com/benfred/py-spy) - Sampling profiler

---

**Pro tip**: The best debugging tool is **prevention**. Write tests ([TESTING_GUIDE.md](./TESTING_GUIDE.md)), use type hints, and add assertions to catch bugs early!
