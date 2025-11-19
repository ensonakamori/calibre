# How-To Guide: Common Tasks & Recipes

**Purpose:** Step-by-step guides for common Calibre development tasks

**For:** React developers ready to write code

**Format:** Cookbook style - copy/paste recipes

---

## Table of Contents

### Web/API Tasks
1. [Add a New API Endpoint](#new-endpoint)
2. [Add Query Parameters to an Endpoint](#query-params)
3. [Implement Authentication](#auth)
4. [Send Real-Time Updates via WebSocket](#websocket)

### Database Tasks
5. [Add a Custom Column](#custom-column)
6. [Create a Database Index](#index)
7. [Implement Full-Text Search](#fts)
8. [Write a Complex Query](#complex-query)

### Desktop GUI Tasks
9. [Add a New Menu Item](#menu-item)
10. [Create a Dialog Window](#dialog)
11. [Add a Keyboard Shortcut](#keyboard-shortcut)
12. [Respond to Book Selection](#book-selection)

### E-book Processing
13. [Parse EPUB Metadata](#parse-epub)
14. [Generate a Cover Image](#generate-cover)
15. [Add a Conversion Plugin](#conversion-plugin)

### Testing & Debugging
16. [Write a Unit Test](#unit-test)
17. [Debug with pdb](#debug-pdb)
18. [Add Logging](#logging)

---

<a name="new-endpoint"></a>
## 1. Add a New API Endpoint

**Goal:** Create `GET /api/book/{book_id}/formats` to list available formats

### Step 1: Define the Endpoint

**File:** [src/calibre/srv/ajax.py](../../src/calibre/srv/ajax.py)

```python
from calibre.srv.routes import endpoint, json

@endpoint(
    '/ajax/book/{book_id}/formats',  # URL pattern
    types={'book_id': int},           # Type validation
    methods=('GET', 'HEAD'),          # Allowed methods
    auth_required=True,               # Require login
    cache_control=('public', 1)       # Cache for 1 hour
)
def ajax_book_formats(ctx, rd, book_id):
    '''
    Get available formats for a book.

    Returns:
    {
      "book_id": 42,
      "formats": ["EPUB", "MOBI", "PDF"],
      "paths": {
        "EPUB": "/library/Author/Book/book.epub",
        "MOBI": "/library/Author/Book/book.mobi"
      }
    }
    '''

    # Get book from database
    try:
        db = ctx.db
        formats = db.formats(book_id)  # Returns: ['EPUB', 'MOBI']
    except Exception as e:
        from calibre.srv.errors import HTTPNotFound
        raise HTTPNotFound(f'Book {book_id} not found or no formats')

    # Get file paths for each format
    paths = {}
    for fmt in formats:
        path = db.format_abspath(book_id, fmt)
        if path:
            paths[fmt] = path

    # Build response
    response = {
        'book_id': book_id,
        'formats': formats,
        'paths': paths,
        'count': len(formats)
    }

    # Return JSON (auto-serialized by @endpoint decorator)
    return response
```

### Step 2: Test the Endpoint

**Terminal:**
```bash
# Start server
calibre-server ~/test-library

# Test endpoint
curl http://localhost:8080/ajax/book/1/formats

# Expected output:
# {
#   "book_id": 1,
#   "formats": ["EPUB", "MOBI"],
#   "paths": {
#     "EPUB": "/home/user/test-library/Author/Book/book.epub",
#     "MOBI": "/home/user/test-library/Author/Book/book.mobi"
#   },
#   "count": 2
# }
```

### Step 3: Use from Frontend (JavaScript)

```javascript
// Browser/React code
async function getBookFormats(bookId) {
  const response = await fetch(`/ajax/book/${bookId}/formats`);
  const data = await response.json();

  console.log(`Book ${bookId} has ${data.count} formats:`, data.formats);
  return data;
}

// Usage in React
function BookFormats({ bookId }) {
  const [formats, setFormats] = useState(null);

  useEffect(() => {
    getBookFormats(bookId).then(setFormats);
  }, [bookId]);

  if (!formats) return <div>Loading...</div>;

  return (
    <div>
      <h3>Available Formats:</h3>
      <ul>
        {formats.formats.map(fmt => (
          <li key={fmt}>
            {fmt}
            <a href={`/get/${fmt}/${bookId}`}>Download</a>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

<a name="query-params"></a>
## 2. Add Query Parameters to an Endpoint

**Goal:** Accept `?limit=50&offset=0` for pagination

### Implementation

**File:** [src/calibre/srv/ajax.py](../../src/calibre/srv/ajax.py)

```python
@endpoint('/ajax/books', methods=('GET',))
def ajax_books_list(ctx, rd):
    '''
    List books with pagination.

    Query params:
    - limit: Max books to return (default: 50)
    - offset: Skip first N books (default: 0)
    - sort: Sort field (default: 'timestamp')
    - order: 'asc' or 'desc' (default: 'desc')

    Example: /ajax/books?limit=20&offset=40&sort=title&order=asc
    '''

    # Parse query parameters
    # rd.query is a dict of query params
    limit = int(rd.query.get('limit', 50))
    offset = int(rd.query.get('offset', 0))
    sort_by = rd.query.get('sort', 'timestamp')
    order = rd.query.get('order', 'desc')

    # Validate
    if limit > 200:
        limit = 200  # Cap at 200
    if limit < 1:
        limit = 1

    # Get books from database
    db = ctx.db
    all_book_ids = db.search('')  # Get all books

    # Sort
    if sort_by == 'timestamp':
        # Sort by when added
        book_ids = sorted(
            all_book_ids,
            key=lambda bid: db.field_for('timestamp', bid),
            reverse=(order == 'desc')
        )
    elif sort_by == 'title':
        # Sort by title
        book_ids = sorted(
            all_book_ids,
            key=lambda bid: db.field_for('title', bid).lower()
        )

    # Paginate
    paginated_ids = book_ids[offset:offset + limit]

    # Get metadata for each book
    books = []
    for book_id in paginated_ids:
        metadata = db.get_metadata(book_id)
        books.append({
            'id': book_id,
            'title': metadata.get('title'),
            'authors': metadata.get('authors', []),
            'timestamp': metadata.get('timestamp').isoformat()
        })

    # Build response
    return {
        'books': books,
        'total': len(all_book_ids),
        'limit': limit,
        'offset': offset,
        'has_more': (offset + limit) < len(all_book_ids)
    }
```

**Usage:**
```bash
# First page
curl "http://localhost:8080/ajax/books?limit=10&offset=0"

# Second page
curl "http://localhost:8080/ajax/books?limit=10&offset=10"

# Sort by title
curl "http://localhost:8080/ajax/books?sort=title&order=asc"
```

**React Implementation:**
```typescript
function BookList() {
  const [books, setBooks] = useState([]);
  const [page, setPage] = useState(0);
  const limit = 20;

  useEffect(() => {
    const offset = page * limit;
    fetch(`/ajax/books?limit=${limit}&offset=${offset}`)
      .then(res => res.json())
      .then(data => {
        setBooks(data.books);
      });
  }, [page]);

  return (
    <div>
      <BookGrid books={books} />
      <button onClick={() => setPage(p => p - 1)} disabled={page === 0}>
        Previous
      </button>
      <button onClick={() => setPage(p => p + 1)}>
        Next
      </button>
    </div>
  );
}
```

---

<a name="auth"></a>
## 3. Implement Authentication

**Goal:** Require login for certain endpoints

### Step 1: Understand Current Auth

**File:** [src/calibre/srv/auth.py](../../src/calibre/srv/auth.py)

Calibre uses **session-based authentication** with cookies.

```python
def check_session(rd):
    '''
    Check if request has valid session.

    Returns: username or None
    '''
    session_id = rd.cookies.get('session')
    if not session_id:
        return None

    # Look up session in database
    user = get_user_for_session(session_id)
    return user
```

### Step 2: Create Login Endpoint

**File:** [src/calibre/srv/ajax.py](../../src/calibre/srv/ajax.py)

```python
@endpoint('/ajax/login', methods=('POST',), auth_required=False)
def ajax_login(ctx, rd):
    '''
    Authenticate user and create session.

    POST body:
    {
      "username": "admin",
      "password": "secret123"
    }
    '''
    import json

    # Parse request body
    body = rd.read()
    data = json.loads(body)

    username = data.get('username')
    password = data.get('password')

    # Validate credentials
    user_db = ctx.user_manager
    if user_db.authenticate(username, password):
        # Create session
        session_id = create_session(username)

        # Set cookie
        rd.outcookie['session'] = session_id
        rd.outcookie['session']['path'] = '/'
        rd.outcookie['session']['httponly'] = True
        rd.outcookie['session']['max-age'] = 86400 * 30  # 30 days

        return {
            'success': True,
            'username': username,
            'session_id': session_id
        }
    else:
        from calibre.srv.errors import HTTPUnauthorized
        raise HTTPUnauthorized('Invalid credentials')
```

### Step 3: Protect an Endpoint

```python
@endpoint(
    '/ajax/admin/users',
    auth_required=True,  # ← Requires authentication!
    methods=('GET',)
)
def ajax_admin_users(ctx, rd):
    '''Only authenticated users can access'''

    # ctx.username is available (set by auth middleware)
    if ctx.username != 'admin':
        from calibre.srv.errors import HTTPForbidden
        raise HTTPForbidden('Admin only')

    # Return user list
    users = ctx.user_manager.all_users()
    return {'users': users}
```

### Step 4: Frontend Login

```javascript
// Login function
async function login(username, password) {
  const response = await fetch('/ajax/login', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ username, password }),
    credentials: 'include',  // Important! Include cookies
  });

  if (response.ok) {
    const data = await response.json();
    console.log('Logged in as:', data.username);
    return data;
  } else {
    throw new Error('Login failed');
  }
}

// React component
function LoginForm() {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await login(username, password);
      window.location.href = '/library';  // Redirect on success
    } catch (err) {
      setError(err.message);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={username}
        onChange={e => setUsername(e.target.value)}
        placeholder="Username"
      />
      <input
        type="password"
        value={password}
        onChange={e => setPassword(e.target.value)}
        placeholder="Password"
      />
      <button type="submit">Login</button>
      {error && <div className="error">{error}</div>}
    </form>
  );
}
```

---

<a name="websocket"></a>
## 4. Send Real-Time Updates via WebSocket

**Goal:** Notify connected clients when a book is added

### Server Side

**File:** [src/calibre/srv/web_socket.py](../../src/calibre/srv/web_socket.py)

```python
class WebSocketHandler:
    '''Manages WebSocket connections'''

    def __init__(self):
        self.clients = []  # List of connected WebSocket clients

    def add_client(self, ws):
        '''New WebSocket connection'''
        self.clients.append(ws)

    def remove_client(self, ws):
        '''WebSocket disconnected'''
        self.clients.remove(ws)

    def broadcast(self, message):
        '''Send message to all connected clients'''
        import json
        data = json.dumps(message)

        for client in self.clients:
            try:
                client.send(data)
            except Exception:
                # Client disconnected
                self.remove_client(client)

# Global instance
ws_handler = WebSocketHandler()

# When book is added:
def on_book_added(book_id, metadata):
    '''Called when new book added to library'''

    # Broadcast to all connected clients
    ws_handler.broadcast({
        'type': 'book_added',
        'book_id': book_id,
        'title': metadata.get('title'),
        'authors': metadata.get('authors', []),
        'timestamp': metadata.get('timestamp').isoformat()
    })
```

### Client Side (JavaScript)

```javascript
// Connect to WebSocket
const ws = new WebSocket('ws://localhost:8080/ws');

ws.onopen = () => {
  console.log('WebSocket connected');
};

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);

  if (message.type === 'book_added') {
    console.log('New book added:', message.title);

    // Update UI
    addBookToList(message);

    // Show notification
    showNotification(`New book: ${message.title}`);
  }
};

ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};

ws.onclose = () => {
  console.log('WebSocket disconnected');
  // Reconnect after delay
  setTimeout(() => {
    // Reconnect logic
  }, 5000);
};

// React hook for WebSocket
function useWebSocket(url) {
  const [message, setMessage] = useState(null);
  const ws = useRef(null);

  useEffect(() => {
    ws.current = new WebSocket(url);

    ws.current.onmessage = (event) => {
      setMessage(JSON.parse(event.data));
    };

    return () => {
      ws.current?.close();
    };
  }, [url]);

  return message;
}

// Usage in component
function BookList() {
  const [books, setBooks] = useState([]);
  const wsMessage = useWebSocket('ws://localhost:8080/ws');

  useEffect(() => {
    if (wsMessage?.type === 'book_added') {
      // Fetch the new book and add to list
      setBooks(prev => [...prev, wsMessage]);
    }
  }, [wsMessage]);

  return (
    <div>
      {books.map(book => (
        <BookCard key={book.book_id} book={book} />
      ))}
    </div>
  );
}
```

---

<a name="custom-column"></a>
## 5. Add a Custom Column

**Goal:** Add "Reading Status" column to track if you've read a book

### Step 1: Define Custom Column

**Via GUI:**
1. Preferences → Add your own columns
2. Click "Add custom column"
3. Fill in:
   - **Lookup name:** `reading_status`
   - **Column heading:** Reading Status
   - **Column type:** Text, fixed set of values
   - **Values:** `unread, reading, finished`

**Via API:**

**File:** [src/calibre/db/fields.py](../../src/calibre/db/fields.py)

```python
from calibre.db.fields import create_custom_column

def add_reading_status_column(db):
    '''Add custom column for reading status'''

    db.create_custom_column(
        label='reading_status',
        name='Reading Status',
        datatype='enumeration',  # Fixed set of values
        display={
            'enum_values': ['unread', 'reading', 'finished'],
            'enum_colors': ['gray', 'blue', 'green'],
            'use_decorations': True
        },
        is_multiple=False
    )
```

### Step 2: Set Value for a Book

```python
@write_api
def set_reading_status(self, book_id, status):
    '''Set reading status for a book'''

    # Validate status
    valid_statuses = ['unread', 'reading', 'finished']
    if status not in valid_statuses:
        raise ValueError(f'Invalid status: {status}')

    # Update custom column
    self.set_custom(book_id, status, label='reading_status')

# Usage:
db.set_reading_status(42, 'reading')
```

### Step 3: Query by Custom Column

```python
# Find all books currently being read
reading_books = db.search('reading_status:reading')

# Find finished books
finished_books = db.search('reading_status:finished')
```

### Step 4: API Endpoint

```python
@endpoint('/ajax/book/{book_id}/status', methods=('GET', 'POST'))
def ajax_book_status(ctx, rd, book_id):
    '''Get or set reading status'''

    db = ctx.db

    if rd.method == 'GET':
        # Get current status
        status = db.get_custom(book_id, label='reading_status')
        return {'book_id': book_id, 'status': status}

    elif rd.method == 'POST':
        # Set new status
        import json
        data = json.loads(rd.read())
        new_status = data.get('status')

        db.set_custom(book_id, new_status, label='reading_status')

        return {'success': True, 'status': new_status}
```

---

<a name="menu-item"></a>
## 9. Add a New Menu Item (Desktop GUI)

**Goal:** Add "Export to CSV" menu item

### Step 1: Define Action

**File:** [src/calibre/gui2/actions/__init__.py](../../src/calibre/gui2/actions/__init__.py)

```python
from PyQt6.QtWidgets import QAction
from PyQt6.QtGui import QIcon

class ExportCSVAction(QAction):
    '''Action for exporting library to CSV'''

    def __init__(self, gui):
        QAction.__init__(
            self,
            QIcon.ic('document-save.png'),  # Icon
            'Export to CSV',                 # Text
            gui                              # Parent
        )

        # Set tooltip
        self.setToolTip('Export library metadata to CSV file')

        # Set keyboard shortcut
        self.setShortcut('Ctrl+E')

        # Connect to handler
        self.triggered.connect(self.export_csv)

        self.gui = gui

    def export_csv(self):
        '''Handler when menu item clicked'''
        from PyQt6.QtWidgets import QFileDialog
        import csv

        # Ask user for filename
        filename, _ = QFileDialog.getSaveFileName(
            self.gui,
            'Export to CSV',
            'library.csv',
            'CSV files (*.csv)'
        )

        if not filename:
            return  # User cancelled

        # Get all books
        db = self.gui.current_db
        book_ids = db.all_book_ids()

        # Write CSV
        with open(filename, 'w', newline='', encoding='utf-8') as f:
            writer = csv.writer(f)

            # Header
            writer.writerow(['ID', 'Title', 'Authors', 'Tags'])

            # Rows
            for book_id in book_ids:
                metadata = db.get_metadata(book_id)
                writer.writerow([
                    book_id,
                    metadata.get('title'),
                    ', '.join(metadata.get('authors', [])),
                    ', '.join(metadata.get('tags', []))
                ])

        # Show success message
        from PyQt6.QtWidgets import QMessageBox
        QMessageBox.information(
            self.gui,
            'Export Complete',
            f'Exported {len(book_ids)} books to {filename}'
        )
```

### Step 2: Register Action

**File:** [src/calibre/gui2/main.py](../../src/calibre/gui2/main.py)

```python
class MainWindow(QMainWindow):
    def __init__(self):
        # ... existing code ...

        # Create actions
        self.create_actions()

    def create_actions(self):
        '''Create all menu actions'''

        # ... existing actions ...

        # Add our new action
        self.export_csv_action = ExportCSVAction(self)

    def create_menus(self):
        '''Create menu bar'''

        # File menu
        file_menu = self.menuBar().addMenu('&File')

        # Add existing items
        file_menu.addAction(self.add_books_action)
        file_menu.addAction(self.edit_metadata_action)

        # Add separator
        file_menu.addSeparator()

        # Add our new menu item
        file_menu.addAction(self.export_csv_action)

        # Add to toolbar too
        self.toolbar.addAction(self.export_csv_action)
```

**Result:** Menu now shows:
```
File
├── Add books...
├── Edit metadata...
├── ─────────────
├── Export to CSV    (Ctrl+E) ← New!
```

---

<a name="unit-test"></a>
## 16. Write a Unit Test

**Goal:** Test the custom column functionality

### Create Test File

**File:** `src/calibre/db/tests/test_custom_columns.py`

```python
import unittest
from calibre.db.tests.base import BaseTest

class CustomColumnTest(BaseTest):
    '''Test custom column functionality'''

    def test_create_custom_column(self):
        '''Test creating a custom column'''

        db = self.init_db()

        # Create custom column
        db.create_custom_column(
            label='test_column',
            name='Test Column',
            datatype='text',
            is_multiple=False
        )

        # Verify it exists
        columns = db.field_metadata.custom_field_metadata()
        self.assertIn('#test_column', columns)

    def test_set_custom_value(self):
        '''Test setting custom column value'''

        db = self.init_db()

        # Create column
        db.create_custom_column(
            label='status',
            name='Status',
            datatype='text'
        )

        # Add a book
        book_id = db.create_book_entry({
            'title': 'Test Book',
            'authors': ['Test Author']
        })

        # Set custom value
        db.set_custom(book_id, 'reading', label='status')

        # Verify
        value = db.get_custom(book_id, label='status')
        self.assertEqual(value, 'reading')

    def test_search_custom_column(self):
        '''Test searching by custom column'''

        db = self.init_db()

        # Create column and add books
        db.create_custom_column(label='genre', name='Genre', datatype='text')

        book1 = db.create_book_entry({'title': 'Scifi Book'})
        book2 = db.create_book_entry({'title': 'Mystery Book'})

        db.set_custom(book1, 'scifi', label='genre')
        db.set_custom(book2, 'mystery', label='genre')

        # Search
        results = db.search('genre:scifi')

        # Verify
        self.assertIn(book1, results)
        self.assertNotIn(book2, results)

if __name__ == '__main__':
    unittest.main()
```

### Run Tests

```bash
# Run all tests
python setup.py test

# Run specific test file
python -m pytest src/calibre/db/tests/test_custom_columns.py

# Run specific test method
python -m pytest src/calibre/db/tests/test_custom_columns.py::CustomColumnTest::test_set_custom_value

# With verbose output
python -m pytest -v src/calibre/db/tests/test_custom_columns.py
```

---

<a name="debug-pdb"></a>
## 17. Debug with pdb

**Goal:** Step through code to understand how it works

### Add Breakpoint

```python
# File: src/calibre/srv/ajax.py

@endpoint('/ajax/book/{book_id}')
def ajax_book(ctx, rd, book_id):
    '''Get book metadata'''

    # Add breakpoint here
    import pdb; pdb.set_trace()  # ← Debugger will stop here

    # When this line executes, you'll get an interactive prompt
    metadata = ctx.db.get_metadata(book_id)
    return metadata
```

### Run with Debugger

```bash
# Start server
python src/calibre/debug.py --serve-books ~/library

# Make request (will hit breakpoint)
curl http://localhost:8080/ajax/book/1
```

**Interactive Debugging:**

```python
# When breakpoint hits:
> /path/to/ajax.py(42)ajax_book()
-> metadata = ctx.db.get_metadata(book_id)

# Commands:
(Pdb) book_id           # Print variable
42

(Pdb) ctx.db            # Inspect object
<calibre.db.cache.Cache object at 0x...>

(Pdb) type(ctx.db)      # Check type
<class 'calibre.db.cache.Cache'>

(Pdb) dir(ctx.db)       # See all methods/attributes
['get_metadata', 'set_metadata', 'all_book_ids', ...]

(Pdb) n                 # Next line (step over)
(Pdb) s                 # Step into function
(Pdb) c                 # Continue execution
(Pdb) l                 # List source code around current line
(Pdb) w                 # Where am I? (stack trace)
(Pdb) h                 # Help
```

### VS Code Debugging

**File:** `.vscode/launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Calibre Server",
      "type": "python",
      "request": "launch",
      "program": "${workspaceFolder}/src/calibre/debug.py",
      "args": ["--serve-books", "/path/to/test-library"],
      "console": "integratedTerminal",
      "justMyCode": false  // Step into library code
    }
  ]
}
```

**Usage:**
1. Set breakpoint in VS Code (click left of line number)
2. Press F5 to start debugging
3. Make request to endpoint
4. VS Code stops at breakpoint
5. Use debug toolbar to step through

---

## 🎯 Quick Reference

### Common Patterns

**API Endpoint Template:**
```python
@endpoint('/api/resource/{id}', types={'id': int})
def api_handler(ctx, rd, id):
    data = ctx.db.get_thing(id)
    return {'result': data}
```

**Database Query:**
```python
book_id = db.create_book_entry(metadata)
metadata = db.get_metadata(book_id)
db.set_metadata(book_id, new_metadata)
book_ids = db.search('author:orwell')
```

**Qt GUI Action:**
```python
class MyAction(QAction):
    def __init__(self, gui):
        QAction.__init__(self, 'Label', gui)
        self.triggered.connect(self.handler)

    def handler(self):
        # Do something
        pass
```

---

## 📖 Further Reading

- [CODE_TOURS.md](./CODE_TOURS.md) - Step-by-step code walkthroughs
- [FIRST_CONTRIBUTIONS.md](./FIRST_CONTRIBUTIONS.md) - Your first PR
- [TESTING_GUIDE.md](./TESTING_GUIDE.md) - Testing best practices
- [DEBUGGING_GUIDE.md](./DEBUGGING_GUIDE.md) - Advanced debugging

---

**Questions?** Check [FAQ.md](./FAQ.md) or ask on the forum!
