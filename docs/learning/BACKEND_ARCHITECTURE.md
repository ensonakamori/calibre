# Backend Architecture: Complete Guide for React Developers

**Purpose:** Understand how Calibre's backend works - servers, databases, and the request/response cycle

**Prerequisites:** Basic JavaScript/React knowledge, willingness to learn Python

**Time to Read:** 45-60 minutes

---

## Table of Contents

1. [Backend Fundamentals](#backend-fundamentals)
2. [Calibre's HTTP Server](#calibre-http-server)
3. [Routing System](#routing-system)
4. [Request/Response Cycle](#request-response-cycle)
5. [Database Layer](#database-layer)
6. [Architecture Patterns](#architecture-patterns)
7. [Comparison to Modern Frameworks](#comparison)

---

<a name="backend-fundamentals"></a>
## 1. Backend Fundamentals

### 🧠 **Mental Model: What IS a Backend?**

If you've been doing React, you've been working in the **browser**:
- User clicks button
- React updates UI
- JavaScript calls API
- Display response

But what happens on the **other side** of that API call?

```
┌─────────────┐                           ┌─────────────┐
│   Browser   │                           │   Server    │
│   (React)   │                           │  (Python)   │
├─────────────┤                           ├─────────────┤
│             │   fetch('/api/books')     │             │
│  Click! ────┼──────────────────────────>│  Receive    │
│             │                           │  Route      │
│             │                           │  Query DB   │
│             │                           │  Format     │
│  Display <──┼───────────────────────────┤  Respond    │
│             │   JSON: [{...}, {...}]    │             │
└─────────────┘                           └─────────────┘
```

**The backend's job:**
1. **Listen** for HTTP requests (GET, POST, PUT, DELETE)
2. **Route** requests to the right handler function
3. **Process** the request (query database, transform data, business logic)
4. **Respond** with data (usually JSON)

🌉 **Bridge from React:**
Think of Express.js or Next.js API routes - that's backend! Calibre implements the same concept, just without Express.

---

### The Web Stack (Every Backend Needs This)

```
┌──────────────────────────────────────────────────┐
│              Application Layer                    │
│  (Your code: routes, business logic)             │
│                                                   │
│  calibre.srv.routes → Your route handlers        │
└──────────────┬───────────────────────────────────┘
               │
┌──────────────┴───────────────────────────────────┐
│              HTTP Server Layer                    │
│  (Parse requests, manage connections)            │
│                                                   │
│  calibre.srv.loop → Custom async server          │
└──────────────┬───────────────────────────────────┘
               │
┌──────────────┴───────────────────────────────────┐
│              TCP/Socket Layer                     │
│  (Network communication)                         │
│                                                   │
│  Python sockets → OS-level networking            │
└──────────────────────────────────────────────────┘
```

**In Express/Next.js:** Most of this is hidden. You just write routes.
**In Calibre:** They built the HTTP server layer themselves for maximum control.

---

<a name="calibre-http-server"></a>
## 2. Calibre's HTTP Server

### Why Build Your Own Server?

**Why not use Express (Node.js) or Flask/FastAPI (Python)?**

Calibre predates most modern frameworks and needed:
- ✅ **Zero external dependencies** for the server
- ✅ **Async I/O** for handling many concurrent connections
- ✅ **Embedded use** - run inside the desktop app
- ✅ **Small footprint** - no framework overhead
- ✅ **Complete control** over every aspect

🎯 **Result:** A custom async HTTP server built from scratch in pure Python.

### The Server Architecture

**File:** [src/calibre/srv/loop.py](../../src/calibre/srv/loop.py)

```python
# Simplified version of calibre's server loop
class Server:
    def __init__(self, handler, opts):
        self.handler = handler  # Your route handler
        self.socket = self.create_listen_socket()

    def serve_forever(self):
        while True:
            # 1. Wait for connections (like a restaurant host waiting for customers)
            readable, writable, _ = select.select([self.socket], [], [])

            # 2. Accept new connections
            for s in readable:
                client_socket, address = s.accept()
                self.handle_connection(client_socket)

    def handle_connection(self, client):
        # 3. Read HTTP request
        request_data = self.read_request(client)

        # 4. Route to handler
        response = self.handler.dispatch(request_data)

        # 5. Send HTTP response
        self.write_response(client, response)
        client.close()
```

🌉 **Bridge to Node.js:**
```javascript
// This is what Express does internally
const http = require('http');
const server = http.createServer((req, res) => {
  // Express middleware & routing happens here
  res.end('Hello World');
});
server.listen(3000);
```

Calibre's server is doing the same thing, just more manually.

---

### How Calibre Handles Async I/O

**The Challenge:** Handle 100s of concurrent connections without threads.

**React Developer Translation:**
- In React: You use `async/await` for API calls
- In Node.js: Event loop handles async I/O
- In Calibre: Python's `select()` + non-blocking sockets

```python
# Pseudo-code for Calibre's async pattern
connections = {}

while True:
    # Who's ready to read/write?
    readable, writable, errored = select.select(
        list(connections.keys()),
        list(connections.keys()),
        list(connections.keys()),
        timeout=1.0
    )

    # Read from sockets that have data
    for sock in readable:
        data = sock.recv(8192)
        connections[sock].buffer += data

        # Complete request?
        if request_complete(connections[sock]):
            process_request(connections[sock])

    # Write to sockets ready for writing
    for sock in writable:
        if connections[sock].has_response():
            sock.send(connections[sock].response_data)
```

🎯 **Remember This:** It's like a restaurant:
- **select()** = Host checking "which tables need service?"
- **readable** = "Table 3 is ready to order"
- **writable** = "Table 5's food is ready to deliver"

---

<a name="routing-system"></a>
## 3. Routing System

### How Routes Work (React Router vs Calibre)

```javascript
// React Router (Frontend)
<Routes>
  <Route path="/books" element={<BookList />} />
  <Route path="/books/:id" element={<BookDetail />} />
  <Route path="/books/:id/edit" element={<BookEdit />} />
</Routes>
```

```python
# Calibre Routes (Backend)
# File: src/calibre/srv/routes.py

@endpoint('/api/books', methods=('GET',))
def get_books(ctx, request_data):
    """Get all books"""
    books = ctx.db.all_book_ids()
    return json(ctx, request_data, endpoint, {'books': books})

@endpoint('/api/books/{book_id}', types={'book_id': int})
def get_book_detail(ctx, request_data, book_id):
    """Get single book by ID"""
    metadata = ctx.db.get_metadata(book_id)
    return json(ctx, request_data, endpoint, metadata)

@endpoint('/api/books/{book_id}/edit', methods=('POST',), needs_db_write=True)
def update_book(ctx, request_data, book_id):
    """Update book metadata"""
    data = json.loads(request_data.read())
    ctx.db.set_metadata(book_id, data)
    return json(ctx, request_data, endpoint, {'success': True})
```

### The @endpoint Decorator

**File:** [src/calibre/srv/routes.py:57-108](../../src/calibre/srv/routes.py#L57-L108)

This is like `@app.get()` in FastAPI or route handlers in Next.js:

```python
def endpoint(
    route,                    # URL pattern: '/api/books/{id}'
    methods=('GET', 'HEAD'),  # HTTP methods allowed
    types=None,               # Type validation: {'id': int}
    auth_required=True,       # Require authentication?
    cache_control=False,      # HTTP caching headers
    needs_db_write=False      # Need write lock on database?
):
    # Decorator magic that registers the route
    def annotate(f):
        f.route = route
        f.methods = methods
        # ... more setup
        return f
    return annotate
```

🌉 **Bridge to Next.js:**
```typescript
// Next.js API Route
// app/api/books/[id]/route.ts
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const book = await db.book.findUnique({ where: { id: params.id } });
  return Response.json(book);
}
```

Same concept, different syntax!

---

### Route Matching Algorithm

**File:** [src/calibre/srv/routes.py:174-196](../../src/calibre/srv/routes.py#L174-L196)

```python
class Route:
    def matches(self, path):
        """Check if this route matches the request path"""
        args_map = {}

        # Match each path component
        for component, (name, matcher) in zip(path, self.matchers):
            if matcher is True:  # Variable component like {id}
                args_map[name] = component
            elif not matcher(component):  # Fixed component like 'api'
                return False  # No match

        # Type checking
        for name, type_checker in self.type_checkers.items():
            args_map[name] = type_checker(args_map[name])  # int('123') → 123

        return (args_map[name] for name in self.names)
```

**Example:**
```
Route pattern: /api/books/{book_id}/formats/{fmt}
Request path:  /api/books/42/formats/epub

Match process:
  'api' == 'api' ✓
  'books' == 'books' ✓
  {book_id} = '42' → Convert to int(42) ✓
  'formats' == 'formats' ✓
  {fmt} = 'epub' ✓

Result: book_id=42, fmt='epub'
```

---

<a name="request-response-cycle"></a>
## 4. Request/Response Cycle

### Complete Lifecycle of a Request

Let's trace a request from browser to database and back:

```
USER CLICKS → REACT → NETWORK → CALIBRE SERVER → DATABASE → BACK TO USER
```

#### Example: Get Book Details

**1. User Action (Browser/React)**
```javascript
// User clicks on a book
function BookDetail({ bookId }) {
  const [book, setBook] = useState(null);

  useEffect(() => {
    fetch(`http://localhost:8080/api/books/${bookId}`)
      .then(res => res.json())
      .then(setBook);
  }, [bookId]);

  return <div>{book?.title}</div>;
}
```

**2. HTTP Request Goes Over Network**
```
GET /api/books/42 HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0...
Accept: application/json
Cookie: session=abc123...
```

**3. Server Receives Request (Python)**

**File:** [src/calibre/srv/loop.py](../../src/calibre/srv/loop.py)
```python
# 3a. Socket receives data
client_socket, address = server_socket.accept()
raw_data = client_socket.recv(8192)  # Read HTTP request

# 3b. Parse HTTP request
request_data = parse_http_request(raw_data)
# → method='GET', path='/api/books/42', headers={...}
```

**4. Router Finds Handler**

**File:** [src/calibre/srv/routes.py:269-284](../../src/calibre/srv/routes.py#L269-L284)
```python
# 4a. Find matching route
endpoint, args = router.find_route(['api', 'books', '42'])
# → endpoint = get_book_detail, args = (42,)

# 4b. Check authentication
if endpoint.auth_required:
    authenticate(request_data)  # Verify session cookie

# 4c. Call the endpoint function
response = endpoint(ctx, request_data, *args)
```

**5. Endpoint Queries Database**

**File:** [src/calibre/srv/books.py](../../src/calibre/srv/books.py) (example endpoint)
```python
@endpoint('/api/books/{book_id}', types={'book_id': int})
def get_book_detail(ctx, request_data, book_id):
    # 5a. Query database cache
    metadata = ctx.db.get_metadata(book_id)
    # → Returns: {id: 42, title: "1984", author: "George Orwell", ...}

    # 5b. Format response
    return json(ctx, request_data, endpoint, metadata)
```

**6. Database Layer Executes**

**File:** [src/calibre/srv/cache.py](../../src/calibre/srv/cache.py)
```python
def get_metadata(self, book_id):
    # 6a. Check in-memory cache first
    if book_id in self.cache:
        return self.cache[book_id]

    # 6b. Query SQLite
    with self.conn:
        cursor = self.conn.execute(
            "SELECT * FROM books WHERE id = ?",
            (book_id,)
        )
        row = cursor.fetchone()

    # 6c. Transform to metadata object
    metadata = self.row_to_metadata(row)

    # 6d. Cache it
    self.cache[book_id] = metadata
    return metadata
```

**7. Response Goes Back**

```python
# 7a. Convert to JSON
response_body = json.dumps(metadata)

# 7b. Build HTTP response
http_response = f"""HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: {len(response_body)}

{response_body}"""

# 7c. Send over socket
client_socket.send(http_response.encode('utf-8'))
client_socket.close()
```

**8. React Receives and Renders**
```javascript
// Back in React
fetch(...).then(data => {
  setBook(data);  // → Triggers re-render
});
```

---

### 💡 **Aha Moment:** It's All Just Functions!

At its core, a backend is just:
```python
def handle_request(method, path, headers, body):
    # 1. Figure out what they want (routing)
    handler = find_handler(method, path)

    # 2. Do it (business logic + database)
    result = handler(path, body)

    # 3. Send it back (response)
    return to_json(result)
```

Everything else (routing, middleware, ORM) is sugar on top of this pattern!

---

<a name="database-layer"></a>
## 5. Database Layer

### The Cache Pattern (In-Memory ORM)

**File:** [src/calibre/db/cache.py:138-150](../../src/calibre/db/cache.py#L138-L150)

```python
class Cache:
    '''
    An in-memory cache of the metadata.db file.
    This class serves as a threadsafe API for accessing the database.

    SQLITE is simply used as a way to read and write from metadata.db.
    All table reading/sorting/searching/caching logic is re-implemented.
    '''
```

🧠 **Mental Model:** Prisma vs Calibre's Cache

```typescript
// Prisma (Modern ORM)
const book = await prisma.book.findUnique({ where: { id: 42 } });
// → Generates SQL, executes, returns typed object
```

```python
# Calibre (Custom Cache + ORM)
book = cache.get_metadata(42)
# → Checks cache → Queries SQLite → Returns dict
```

**Why a cache?**
- **Speed:** Read from memory (nanoseconds) vs disk (milliseconds)
- **Consistency:** All code uses same cached data
- **Thread safety:** Lock management for concurrent access

---

### Database Schema Example

**File:** `metadata.db` (SQLite database)

```sql
-- Books table
CREATE TABLE books (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    sort TEXT,
    timestamp TEXT DEFAULT "2000-01-01T00:00:00",
    pubdate TEXT DEFAULT "2000-01-01T00:00:00",
    series_index REAL DEFAULT 1.0,
    author_sort TEXT,
    isbn TEXT DEFAULT "",
    lccn TEXT DEFAULT "",
    path TEXT DEFAULT "",
    flags INTEGER DEFAULT 1,
    uuid TEXT,
    has_cover BOOL DEFAULT 0,
    last_modified TEXT DEFAULT "2000-01-01T00:00:00"
);

-- Authors table (many-to-many relationship)
CREATE TABLE authors (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    sort TEXT,
    link TEXT DEFAULT ""
);

-- Junction table
CREATE TABLE books_authors_link (
    id INTEGER PRIMARY KEY,
    book INTEGER NOT NULL,
    author INTEGER NOT NULL,
    UNIQUE(book, author)
);
```

🌉 **Bridge to Prisma Schema:**
```prisma
model Book {
  id            Int      @id @default(autoincrement())
  title         String
  authors       Author[]  @relation("BookAuthors")
  timestamp     DateTime @default(now())
  // ...
}

model Author {
  id    Int    @id @default(autoincrement())
  name  String
  books Book[] @relation("BookAuthors")
}
```

Same structure, different syntax!

---

<a name="architecture-patterns"></a>
## 6. Architecture Patterns

### Pattern 1: Layered Architecture

```
┌─────────────────────────────────────────┐
│  Presentation Layer (HTTP Endpoints)    │  ← Routes, JSON serialization
├─────────────────────────────────────────┤
│  Business Logic Layer                   │  ← Controllers, services
├─────────────────────────────────────────┤
│  Data Access Layer (Cache/ORM)          │  ← Database queries
├─────────────────────────────────────────┤
│  Storage Layer (SQLite)                 │  ← Actual database file
└─────────────────────────────────────────┘
```

**Example in Calibre:**
```python
# Presentation (calibre/srv/books.py)
@endpoint('/api/books/{book_id}')
def get_book(ctx, rd, book_id):
    book = ctx.db.get_metadata(book_id)  # ← Call data layer
    return json(ctx, rd, endpoint, book)

# Data Access (calibre/db/cache.py)
def get_metadata(self, book_id):
    if book_id in self.cache:
        return self.cache[book_id]
    row = self.backend.get_book(book_id)  # ← Call storage
    return self.row_to_metadata(row)

# Storage (calibre/db/backend.py)
def get_book(self, book_id):
    return self.conn.execute(
        "SELECT * FROM books WHERE id=?", (book_id,)
    ).fetchone()
```

---

### Pattern 2: Context Object (Dependency Injection)

```python
# Instead of global variables, pass context
class Context:
    def __init__(self, db, user, session):
        self.db = db          # Database instance
        self.user = user      # Current user
        self.session = session # Session data

@endpoint('/api/books/{id}')
def get_book(ctx, request_data, id):
    # ctx.db is injected!
    return ctx.db.get_metadata(id)
```

🌉 **Bridge to Next.js:**
```typescript
// Next.js uses request context too
export async function GET(request: Request) {
  const session = await getServerSession();  // Context
  const db = await getDatabase();            // Context
  // Use them
}
```

---

### Pattern 3: Decorators for Configuration

```python
# Decorators = metadata + registration
@endpoint('/api/books', methods=('GET', 'POST'), auth_required=True)
def handle_books(ctx, rd):
    pass
```

🌉 **Bridge to TypeScript:**
```typescript
// Similar to NestJS decorators
@Get('/books')
@UseGuards(AuthGuard)
async getBooks() { }
```

---

<a name="comparison"></a>
## 7. Comparison to Modern Frameworks

### Calibre vs Express.js

| **Feature** | **Calibre** | **Express.js** |
|------------|------------|---------------|
| **Language** | Python | JavaScript |
| **HTTP Server** | Custom (select-based) | Node.js built-in |
| **Routing** | Custom decorators | Middleware chain |
| **Database** | Custom Cache + SQLite | Sequelize/TypeORM + SQL |
| **Async** | select() + callbacks | async/await + event loop |
| **Size** | ~5000 lines | ~1000 lines (minimal) |

### Calibre vs Next.js API Routes

| **Feature** | **Calibre** | **Next.js** |
|------------|------------|------------|
| **Route Files** | One big routes.py | app/api/**/*.ts |
| **Database** | In-memory cache | Prisma/Drizzle |
| **Type Safety** | Python type hints | TypeScript |
| **Hot Reload** | Manual restart | Built-in HMR |

### Calibre vs FastAPI (Modern Python)

| **Feature** | **Calibre** | **FastAPI** |
|------------|------------|------------|
| **Type Checking** | Manual | Automatic (Pydantic) |
| **Auto Docs** | None | Swagger UI built-in |
| **Async** | select() | async/await (ASGI) |
| **Validation** | Manual | Automatic |

---

## 🎯 **Key Takeaways**

1. **Backends are just request handlers**
   - Receive HTTP → Process → Respond
   - Everything else is optimization

2. **Calibre built its own server**
   - More control, fewer dependencies
   - Same concepts as Express/Fastify, different implementation

3. **The database is cached in memory**
   - Fast reads (no disk I/O)
   - Thread-safe access
   - Write-through to SQLite

4. **Routing is just pattern matching**
   - `/api/books/{id}` → Extract `id` → Call function
   - Types are validated at route level

5. **Python ≈ JavaScript**
   - Different syntax, same concepts
   - Decorators = Higher-order functions
   - Dicts = Objects, Lists = Arrays

---

## 📖 **Further Reading**

- **[DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md)** - Deep dive into SQLite and ORM
- **[DATA_FLOW_GUIDE.md](./DATA_FLOW_GUIDE.md)** - Follow a request through the entire system
- **[CODE_TOURS.md](./CODE_TOURS.md)** - Guided walkthroughs of actual code

---

## ✅ **Self-Check Questions**

1. What are the 3 main jobs of a backend?
2. Why did Calibre build its own HTTP server instead of using Flask?
3. How does the routing system match `/api/books/42` to a handler?
4. What's the difference between the Cache and the SQLite database?
5. How is a Python `@endpoint` decorator similar to Next.js route handlers?

**Answers:**
1. Receive requests, process data, respond with results
2. Zero dependencies, complete control, embedded in desktop app
3. Pattern matching: split path, match components, extract variables, type check
4. Cache = in-memory (fast), SQLite = on-disk (persistent)
5. Both map URL patterns to handler functions with metadata

---

**Next:** [DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md) - Learn how data is stored and queried
