# Data Flow Guide: Request-to-Response Lifecycle

**Purpose:** Follow a request through Calibre's entire system

**For:** Understanding how all the pieces connect

**Time to Read:** 30 minutes

---

## Table of Contents

1. [Simple Request Flow](#simple-flow)
2. [Complete Request Lifecycle](#complete-lifecycle)
3. [Database Query Flow](#database-flow)
4. [File Upload Flow](#file-upload)
5. [Real-Time Updates](#real-time)

---

<a name="simple-flow"></a>
## 1. Simple Request Flow

### Example: Get Book Details

**User Action:** Click on a book in the web interface

**Complete Journey:**

```
┌─────────┐
│ Browser │  User clicks book
└────┬────┘
     │
     │ 1. HTTP GET /api/books/42
     ▼
┌─────────────────┐
│  Network        │  TCP/IP packet
└────┬────────────┘
     │
     │ 2. Arrives at server
     ▼
┌─────────────────┐
│  Loop.py        │  Accept connection
│  (HTTP Server)  │  Parse HTTP request
└────┬────────────┘
     │
     │ 3. Route to handler
     ▼
┌─────────────────┐
│  Routes.py      │  Match: /api/books/{book_id}
│  (Router)       │  Extract: book_id = 42
└────┬────────────┘
     │
     │ 4. Call endpoint function
     ▼
┌─────────────────┐
│  Books.py       │  def get_book(ctx, rd, book_id):
│  (Endpoint)     │      return ctx.db.get_metadata(42)
└────┬────────────┘
     │
     │ 5. Query database
     ▼
┌─────────────────┐
│  Cache.py       │  Check in-memory cache
│  (DB Layer)     │  Return book data
└────┬────────────┘
     │
     │ 6. If not in cache
     ▼
┌─────────────────┐
│  Backend.py     │  Query SQLite:
│  (SQLite)       │  SELECT * FROM books WHERE id=42
└────┬────────────┘
     │
     │ 7. Return data
     ▼
┌─────────────────┐
│  Books.py       │  Format as JSON
│  (Endpoint)     │  Add HTTP headers
└────┬────────────┘
     │
     │ 8. Send response
     ▼
┌─────────────────┐
│  Loop.py        │  HTTP/1.1 200 OK
│  (HTTP Server)  │  Content-Type: application/json
└────┬────────────┘  {"id": 42, "title": "1984", ...}
     │
     │ 9. Over network
     ▼
┌─────────┐
│ Browser │  Receive JSON
└─────────┘  Update UI
```

**Total Time:**
- Best case (cached): **< 1ms**
- Cache miss (SQLite): **< 10ms**
- Network latency: **+ 20-100ms**

---

<a name="complete-lifecycle"></a>
## 2. Complete Request Lifecycle

### Code-Level Walkthrough

**Step 1: Browser Makes Request**

```javascript
// Frontend (React)
async function fetchBook(bookId) {
  const response = await fetch(`http://localhost:8080/api/books/${bookId}`);
  const book = await response.json();
  return book;
}
```

**Step 2: Server Receives Connection**

**File:** [src/calibre/srv/loop.py:90-100](../../src/calibre/srv/loop.py#L90-L100)

```python
# Main event loop
def serve_forever(self):
    while True:
        # Wait for socket events
        readable, writable, errored = select.select(
            self.read_sockets,
            self.write_sockets,
            self.error_sockets,
            timeout=1.0
        )

        # New connection?
        for sock in readable:
            if sock is self.listen_socket:
                client_sock, addr = sock.accept()
                self.handle_new_connection(client_sock, addr)
```

**Step 3: Parse HTTP Request**

**File:** [src/calibre/srv/http_request.py](../../src/calibre/srv/http_request.py)

```python
def parse_request(raw_data):
    # Parse: "GET /api/books/42 HTTP/1.1\r\n..."
    lines = raw_data.split(b'\r\n')

    # First line: method, path, version
    method, path, version = lines[0].split(b' ')

    # Parse headers
    headers = {}
    for line in lines[1:]:
        if b':' in line:
            key, value = line.split(b':', 1)
            headers[key.strip()] = value.strip()

    return RequestData(
        method=method.decode(),
        path=path.decode(),
        headers=headers
    )
```

**Step 4: Route Matching**

**File:** [src/calibre/srv/routes.py:269-284](../../src/calibre/srv/routes.py#L269-L284)

```python
def find_route(self, path):
    # path = ['api', 'books', '42']
    path_parts = path.strip('/').split('/')

    # Try each route pattern
    for route in self.routes:
        # Pattern: /api/books/{book_id}
        # Match: ['api', 'books', {book_id}]
        args = route.matches(path_parts)
        if args is not False:
            return route.endpoint, args

    raise HTTPNotFound()

# Result: endpoint=get_book, args=(42,)
```

**Step 5: Authentication (if required)**

**File:** [src/calibre/srv/auth.py](../../src/calibre/srv/auth.py)

```python
def authenticate(request_data, endpoint):
    if not endpoint.auth_required:
        return None

    # Check session cookie
    session_id = request_data.cookies.get('session')
    if not session_id:
        raise HTTPUnauthorized()

    # Validate session
    user = get_user_from_session(session_id)
    if not user:
        raise HTTPUnauthorized()

    return user
```

**Step 6: Call Endpoint Function**

**File:** [src/calibre/srv/books.py](../../src/calibre/srv/books.py) (example)

```python
@endpoint('/api/books/{book_id}', types={'book_id': int})
def get_book(ctx, request_data, book_id):
    # ctx = Context (has db, user, config)
    # request_data = RequestData (headers, cookies, etc.)
    # book_id = 42 (parsed from URL, type-checked)

    try:
        metadata = ctx.db.get_metadata(book_id)
    except NoSuchBook:
        raise HTTPNotFound(f'No book with id: {book_id}')

    # Format as JSON
    return json(ctx, request_data, endpoint, metadata)
```

**Step 7: Database Query**

**File:** [src/calibre/db/cache.py](../../src/calibre/db/cache.py)

```python
@read_api
def get_metadata(self, book_id):
    # 1. Check cache first
    if book_id in self._metadata_cache:
        return self._metadata_cache[book_id]

    # 2. Cache miss - query SQLite
    with self.read_lock:
        row = self.backend.get_book(book_id)
        if row is None:
            raise NoSuchBook(book_id)

    # 3. Convert row to metadata dict
    metadata = self.row_to_metadata(row)

    # 4. Store in cache
    self._metadata_cache[book_id] = metadata

    return metadata
```

**Step 8: SQLite Query**

**File:** [src/calibre/db/backend.py](../../src/calibre/db/backend.py)

```python
def get_book(self, book_id):
    cursor = self.conn.execute('''
        SELECT
            id, title, sort, timestamp, pubdate,
            series_index, author_sort, isbn, path,
            flags, uuid, has_cover, last_modified
        FROM books
        WHERE id = ?
    ''', (book_id,))

    row = cursor.fetchone()
    return row
```

**Step 9: Build HTTP Response**

**File:** [src/calibre/srv/http_response.py](../../src/calibre/srv/http_response.py)

```python
def build_response(status_code, headers, body):
    # Status line
    response = f"HTTP/1.1 {status_code} {STATUS_MESSAGES[status_code]}\r\n"

    # Headers
    for key, value in headers.items():
        response += f"{key}: {value}\r\n"

    # Blank line
    response += "\r\n"

    # Body
    if body:
        response += body

    return response.encode('utf-8')

# Example output:
# HTTP/1.1 200 OK\r\n
# Content-Type: application/json\r\n
# Content-Length: 234\r\n
# \r\n
# {"id": 42, "title": "1984", ...}
```

**Step 10: Send to Client**

```python
def send_response(client_socket, response_data):
    # Send all data
    client_socket.sendall(response_data)

    # Close connection (or keep-alive)
    client_socket.close()
```

**Step 11: Browser Receives**

```javascript
// Response arrives
fetch(...).then(response => {
  // Parse JSON
  return response.json();
}).then(book => {
  // Update state
  setBook(book);
  // React re-renders
});
```

---

<a name="database-flow"></a>
## 3. Database Query Flow

### Complex Query: Books by Author with Tags

**User wants:** All books by "George Orwell" with tag "dystopia"

**SQL (how you'd think to do it):**
```sql
SELECT books.*
FROM books
JOIN books_authors_link ON books.id = books_authors_link.book
JOIN authors ON books_authors_link.author = authors.id
JOIN books_tags_link ON books.id = books_tags_link.book
JOIN tags ON books_tags_link.tag = tags.id
WHERE authors.name = 'George Orwell'
  AND tags.name = 'dystopia';
```

**Calibre (how it actually works):**
```python
def books_by_author_with_tag(author_name, tag_name):
    # 1. Find author ID
    author_id = None
    for aid, author in self.authors_cache.items():
        if author['name'] == author_name:
            author_id = aid
            break

    if not author_id:
        return []

    # 2. Find tag ID
    tag_id = None
    for tid, tag in self.tags_cache.items():
        if tag['name'] == tag_name:
            tag_id = tid
            break

    if not tag_id:
        return []

    # 3. Find books with this author AND tag (in-memory)
    results = []
    for book_id, book in self.books_cache.items():
        has_author = author_id in book.get('author_ids', [])
        has_tag = tag_id in book.get('tag_ids', [])

        if has_author and has_tag:
            results.append(book)

    return results
```

**Why this is fast:**
- ✅ All data in RAM (no disk I/O)
- ✅ Simple Python loops (fast for small data)
- ✅ No SQL parsing overhead

**When it's slow:**
- 🔴 Very large libraries (100k+ books)
- 🔴 Complex filters

---

<a name="file-upload"></a>
## 4. File Upload Flow

### Example: Upload New E-book

**Complete Flow:**

```
User selects file
     │
     ▼
┌──────────────────┐
│  Browser         │  1. Read file as ArrayBuffer
│  (React)         │  2. Create FormData
└────┬─────────────┘     const formData = new FormData()
     │                   formData.append('file', file)
     │
     │ 3. POST /api/books/upload
     ▼
┌──────────────────┐
│  Server          │  4. Parse multipart form data
│  (upload.py)     │  5. Save temp file
└────┬─────────────┘     /tmp/calibre_upload_abc123.epub
     │
     │ 6. Extract metadata
     ▼
┌──────────────────┐
│  Metadata        │  7. Parse EPUB/MOBI
│  Extraction      │  8. Extract: title, author, cover
└────┬─────────────┘
     │
     │ 9. Add to database
     ▼
┌──────────────────┐
│  Database        │  10. INSERT INTO books
│  (cache.py)      │  11. INSERT INTO authors (if new)
└────┬─────────────┘  12. Link author to book
     │
     │ 13. Move file to library
     ▼
┌──────────────────┐
│  File System     │  14. Create folder: /Author/Title/
│                  │  15. Move: book.epub
└────┬─────────────┘  16. Extract cover: cover.jpg
     │
     │ 17. Run plugins
     ▼
┌──────────────────┐
│  Plugins         │  18. Post-import hooks
│                  │  19. Fetch additional metadata
└────┬─────────────┘
     │
     │ 20. Notify listeners
     ▼
┌──────────────────┐
│  Events          │  21. Emit "book_added" event
│                  │  22. Desktop GUI updates
└────┬─────────────┘  23. WebSocket notify web clients
     │
     │ 24. Return response
     ▼
┌──────────────────┐
│  Browser         │  25. Show success message
│                  │  26. Refresh book list
└──────────────────┘
```

---

<a name="real-time"></a>
## 5. Real-Time Updates

### WebSocket Flow: Book Added in Desktop → Update Web UI

**Scenario:** User adds book in desktop app, web browser should update instantly

```
Desktop App                    Server                     Web Browser
     │                           │                              │
     │ 1. Add book              │                              │
     ├──────────────────────────>│                              │
     │                           │                              │
     │                           │ 2. Update database           │
     │                           │    (cache + SQLite)          │
     │                           │                              │
     │                           │ 3. Emit event                │
     │                           │    dispatch('book_added', 42)│
     │                           │                              │
     │                           │ 4. Send WebSocket message    │
     │                           ├─────────────────────────────>│
     │                           │  {"event": "book_added",     │
     │                           │   "book_id": 42}             │
     │                           │                              │
     │                           │                              │ 5. Receive WS
     │                           │                              │
     │                           │                              │ 6. Fetch book
     │                           │<─────────────────────────────┤
     │                           │  GET /api/books/42           │
     │                           │                              │
     │                           │ 7. Return book data          │
     │                           ├─────────────────────────────>│
     │                           │  {"id": 42, "title": ...}    │
     │                           │                              │
     │                           │                              │ 8. Update UI
     │                           │                              │    (new book appears!)
```

**Code:**

**Server (Python):**
```python
# File: src/calibre/srv/web_socket.py
class WebSocketHandler:
    def __init__(self):
        self.clients = []

    def on_book_added(self, book_id):
        # Notify all connected clients
        message = json.dumps({
            'event': 'book_added',
            'book_id': book_id
        })
        for client in self.clients:
            client.send(message)
```

**Client (React):**
```javascript
// WebSocket connection
const ws = new WebSocket('ws://localhost:8080/ws');

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);

  if (data.event === 'book_added') {
    // Fetch new book
    const book = await fetchBook(data.book_id);

    // Update state
    setBooks(prev => [...prev, book]);
  }
};
```

---

## 🎯 Key Takeaways

1. **Requests flow through layers**
   - Network → Server → Router → Endpoint → Database → Response

2. **Cache makes reads fast**
   - In-memory access (< 1ms)
   - SQLite fallback (< 10ms)

3. **Writes update both cache and disk**
   - Write-through pattern
   - Ensures consistency

4. **Events enable real-time**
   - WebSockets for instant updates
   - Observer pattern for decoupling

5. **It's just functions calling functions**
   - No magic, just layers
   - Each layer has a specific job

---

## 📖 Further Reading

- **[BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md)** - Server implementation details
- **[DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md)** - Database patterns
- **[CODE_TOURS.md](./CODE_TOURS.md)** - Step through actual code

---

**Ready to trace a request yourself?** Use the Python debugger to step through a real request!
