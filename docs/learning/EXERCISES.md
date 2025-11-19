# Hands-On Exercises for React Developers

Welcome to the practical exercises section! These exercises are designed to help you learn Calibre's architecture through hands-on coding. Each exercise builds on concepts from previous ones and includes solutions with detailed explanations.

## Table of Contents

1. [How to Use This Guide](#how-to-use-this-guide)
2. [Exercise Categories](#exercise-categories)
3. [Beginner Exercises (⭐)](#beginner-exercises-)
4. [Intermediate Exercises (⭐⭐⭐)](#intermediate-exercises-)
5. [Advanced Exercises (⭐⭐⭐⭐⭐)](#advanced-exercises-)
6. [Project-Based Exercises](#project-based-exercises)
7. [Solutions and Explanations](#solutions-and-explanations)

---

## How to Use This Guide

### Learning Path

```
⭐ Beginner
├─ Read existing code
├─ Understand patterns
└─ Make small changes

⭐⭐⭐ Intermediate
├─ Implement new features
├─ Work with database
└─ Add API endpoints

⭐⭐⭐⭐⭐ Advanced
├─ Architectural changes
├─ Performance optimization
└─ Complex integrations
```

### For React Developers

Each exercise includes:
- 🎯 **Learning Objective**: What you'll learn
- 🔗 **Code References**: Direct links to relevant files
- 💭 **React Analogy**: How this relates to React concepts
- ✅ **Acceptance Criteria**: When you're done
- 💡 **Hints**: Guidance without spoiling the solution
- 📝 **Solution**: Complete working code with explanation

### Setup

Before starting exercises:

```bash
# 1. Ensure development environment is set up
cd /path/to/calibre
python setup.py develop

# 2. Run Calibre to ensure it works
calibre

# 3. Run tests to ensure baseline
python setup.py test
```

---

## Exercise Categories

### By Skill Level
- **Beginner (⭐)**: Reading code, understanding patterns, small modifications
- **Intermediate (⭐⭐⭐)**: New features, database work, API endpoints
- **Advanced (⭐⭐⭐⭐⭐)**: Architecture, optimization, complex systems

### By Topic
- **HTTP & Routing**: Understanding the web server
- **Database**: Working with SQLite and the cache
- **Desktop GUI**: PyQt6 widgets and forms
- **Business Logic**: Core calibre functionality
- **Integration**: Connecting components

---

## Beginner Exercises (⭐)

### Exercise 1.1: Add a Simple GET Endpoint

**Difficulty**: ⭐ (30 minutes)

**🎯 Learning Objective**: Understand the `@endpoint` decorator and basic routing

**💭 React Analogy**:
```javascript
// In Next.js, you'd create pages/api/hello.ts:
export default function handler(req, res) {
  res.json({ message: 'Hello World' })
}

// In Calibre, you use @endpoint decorator in routes.py
```

**🔗 Code References**:
- [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py#L57) - @endpoint decorator
- [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py#L269) - Router.find_route

**Task**: Add a new endpoint `/api/hello` that returns a JSON response with a greeting message.

**Requirements**:
1. Add endpoint to [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py)
2. Return JSON: `{"message": "Hello from Calibre!", "version": "1.0"}`
3. Test it by visiting `http://localhost:8080/api/hello`

**✅ Acceptance Criteria**:
- [ ] Endpoint responds with 200 OK
- [ ] JSON is properly formatted
- [ ] No errors in console

**💡 Hints**:
<details>
<summary>Hint 1: Where to add the endpoint</summary>

Look at existing endpoints in `src/calibre/srv/routes.py`. Find the pattern where `@endpoint` is used. Your new endpoint should follow the same pattern.
</details>

<details>
<summary>Hint 2: Decorator syntax</summary>

```python
@endpoint('/api/your-path')
def ajax_your_function(ctx, rd):
    # ctx = context (has database, settings, etc.)
    # rd = request data
    return {'key': 'value'}  # Automatically converted to JSON
```
</details>

<details>
<summary>Hint 3: Where exactly to place it</summary>

Add your function near other `ajax_*` functions in the file, around line 400-500. The function name should start with `ajax_`.
</details>

---

### Exercise 1.2: Add Query Parameters to Endpoint

**Difficulty**: ⭐ (45 minutes)

**🎯 Learning Objective**: Handle query parameters and type validation

**💭 React Analogy**:
```javascript
// Next.js API route with query params:
export default function handler(req, res) {
  const { name, age } = req.query
  res.json({ name, age: parseInt(age) })
}

// Calibre automatically validates and converts types
```

**🔗 Code References**:
- [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py#L57) - @endpoint with types parameter
- [src/calibre/srv/http_request.py](../../src/calibre/srv/http_request.py) - Query parsing

**Task**: Extend your `/api/hello` endpoint to accept a `name` parameter and an optional `count` parameter (default: 1).

**Requirements**:
1. Accept `name` as required string parameter
2. Accept `count` as optional integer parameter (default: 1)
3. Return greeting repeated `count` times: `{"greetings": ["Hello, John!", "Hello, John!"]}`
4. Validate that `count` is between 1 and 10

**Example Requests**:
```
GET /api/hello?name=Alice
→ {"greetings": ["Hello, Alice!"]}

GET /api/hello?name=Bob&count=3
→ {"greetings": ["Hello, Bob!", "Hello, Bob!", "Hello, Bob!"]}

GET /api/hello?name=Charlie&count=20
→ {"error": "count must be between 1 and 10"}
```

**✅ Acceptance Criteria**:
- [ ] Required parameter validation works
- [ ] Optional parameter has default value
- [ ] Type conversion works (count is int)
- [ ] Validation errors return proper error messages
- [ ] All test cases pass

**💡 Hints**:
<details>
<summary>Hint 1: Type specification</summary>

```python
@endpoint('/api/hello', types={'name': str, 'count': int})
def ajax_hello(ctx, rd, name, count=1):
    # Parameters are automatically validated and converted
    pass
```

The `types` dict tells Calibre what type each parameter should be. Calibre will automatically convert strings to ints, validate presence, etc.
</details>

<details>
<summary>Hint 2: Validation</summary>

```python
if count < 1 or count > 10:
    raise ValueError("count must be between 1 and 10")
```

Raising exceptions will automatically return proper error responses to the client.
</details>

<details>
<summary>Hint 3: Building the response</summary>

```python
greeting = f"Hello, {name}!"
greetings = [greeting] * count
return {"greetings": greetings}
```

Python's list multiplication makes this easy!
</details>

---

### Exercise 1.3: Read Data from Database

**Difficulty**: ⭐⭐ (1 hour)

**🎯 Learning Objective**: Use the database cache to read book metadata

**💭 React Analogy**:
```javascript
// In React with Prisma:
const book = await prisma.book.findUnique({
  where: { id: bookId }
})

// In Calibre:
metadata = ctx.db.get_metadata(book_id)
```

**🔗 Code References**:
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py#L138) - Cache class
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py) - get_metadata method
- [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py) - Example endpoints using ctx.db

**Task**: Create an endpoint `/api/book/{book_id}` that returns book information.

**Requirements**:
1. Accept `book_id` as URL path parameter (integer)
2. Look up book in database using `ctx.db`
3. Return book title, authors, publisher, publish date
4. Handle case where book doesn't exist (return 404)

**Example Response**:
```json
{
  "id": 1,
  "title": "The Great Gatsby",
  "authors": ["F. Scott Fitzgerald"],
  "publisher": "Scribner",
  "pubdate": "1925-04-10",
  "formats": ["EPUB", "PDF"]
}
```

**✅ Acceptance Criteria**:
- [ ] Endpoint accepts book_id in path
- [ ] Returns book metadata for valid IDs
- [ ] Returns 404 for non-existent books
- [ ] JSON response is properly formatted
- [ ] Authors are returned as array

**💡 Hints**:
<details>
<summary>Hint 1: Path parameters</summary>

```python
@endpoint('/api/book/{book_id}', types={'book_id': int})
def ajax_get_book(ctx, rd, book_id):
    # book_id is automatically extracted from URL and converted to int
    pass
```

The `{book_id}` in the path and the `book_id` parameter name must match.
</details>

<details>
<summary>Hint 2: Accessing the database</summary>

```python
# ctx.db is the database cache
metadata = ctx.db.get_metadata(book_id)

# Check if book exists
if book_id not in ctx.db.all_book_ids():
    raise HTTPNotFound("Book not found")
```

See [src/calibre/db/cache.py](../../src/calibre/db/cache.py) for all available methods.
</details>

<details>
<summary>Hint 3: Getting book details</summary>

```python
# Get various metadata fields
title = metadata.title
authors = ctx.db.authors(book_id)  # Returns list of author names
publisher = metadata.publisher
pubdate = metadata.pubdate.isoformat() if metadata.pubdate else None
formats = ctx.db.formats(book_id)  # Returns list like ['EPUB', 'PDF']

return {
    'id': book_id,
    'title': title,
    'authors': authors,
    'publisher': publisher,
    'pubdate': pubdate,
    'formats': formats
}
```
</details>

---

### Exercise 1.4: Explore the PyQt6 GUI Structure

**Difficulty**: ⭐ (45 minutes)

**🎯 Learning Objective**: Understand how PyQt6 widgets work and relate to React components

**💭 React Analogy**:
```javascript
// React component hierarchy:
<MainWindow>
  <Header />
  <BookList items={books} />
  <Footer />
</MainWindow>

// PyQt6 widget hierarchy (similar structure):
MainWindow
  ├─ HeaderWidget
  ├─ BookListWidget (with model/view)
  └─ FooterWidget
```

**🔗 Code References**:
- [src/calibre/gui2/main.py](../../src/calibre/gui2/main.py) - Main window
- [src/calibre/gui2/library/views.py](../../src/calibre/gui2/library/views.py) - Book list view
- [src/calibre/gui2/widgets.py](../../src/calibre/gui2/widgets.py) - Custom widgets

**Task**: Read and understand the main window structure, then document the widget hierarchy.

**Requirements**:
1. Find the `Main` class in [src/calibre/gui2/main.py](../../src/calibre/gui2/main.py)
2. Identify the major child widgets (panels, toolbars, status bar)
3. Create a diagram showing the widget hierarchy
4. Write a comparison to React component tree

**Deliverable**: Create a file `widget-hierarchy.md` with:
```markdown
# Main Window Widget Hierarchy

## PyQt6 Structure
Main (QMainWindow)
├─ central_widget (QWidget)
│  ├─ book_list (BookListView)
│  ├─ book_details (BookDetailsPanel)
│  └─ tag_browser (TagBrowserWidget)
├─ menu_bar (QMenuBar)
└─ tool_bar (QToolBar)

## React Equivalent
<Main>
  <CentralWidget>
    <BookList />
    <BookDetails />
    <TagBrowser />
  </CentralWidget>
  <MenuBar />
  <ToolBar />
</Main>

## Key Differences
1. PyQt6: Imperative widget creation
2. React: Declarative JSX
3. PyQt6: Signals/slots for events
4. React: Props/callbacks for events
```

**✅ Acceptance Criteria**:
- [ ] Correctly identified main widget classes
- [ ] Hierarchy diagram is accurate
- [ ] React comparison is meaningful
- [ ] Documented at least 5 major widgets

**💡 Hints**:
<details>
<summary>Hint 1: Finding widgets</summary>

Look for lines like:
```python
self.book_list = BookListView(self)
self.addWidget(self.book_list)
```

These create and add child widgets to the parent.
</details>

<details>
<summary>Hint 2: Understanding signals/slots</summary>

```python
# PyQt6: Connect signal to slot
self.book_list.selectionChanged.connect(self.on_book_selected)

# React equivalent:
<BookList onSelectionChange={handleBookSelected} />
```
</details>

---

## Intermediate Exercises (⭐⭐⭐)

### Exercise 2.1: Implement POST Endpoint with Database Write

**Difficulty**: ⭐⭐⭐ (2 hours)

**🎯 Learning Objective**: Handle POST requests and write to database

**💭 React Analogy**:
```javascript
// Next.js API route:
export default async function handler(req, res) {
  if (req.method === 'POST') {
    const data = req.body
    await db.book.create({ data })
    res.json({ success: true })
  }
}
```

**🔗 Code References**:
- [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py) - POST endpoints
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py) - Write methods
- [src/calibre/db/write.py](../../src/calibre/db/write.py) - Database write operations

**Task**: Create an endpoint `/api/book/{book_id}/rating` that allows updating a book's rating.

**Requirements**:
1. Accept POST request with JSON body: `{"rating": 4.5}`
2. Validate rating is between 0 and 5
3. Update book rating in database
4. Return success response with updated rating
5. Handle errors (book not found, invalid rating)

**Example Request**:
```bash
curl -X POST http://localhost:8080/api/book/1/rating \
  -H "Content-Type: application/json" \
  -d '{"rating": 4.5}'
```

**Example Response**:
```json
{
  "success": true,
  "book_id": 1,
  "new_rating": 4.5,
  "message": "Rating updated successfully"
}
```

**✅ Acceptance Criteria**:
- [ ] POST endpoint accepts JSON body
- [ ] Rating validation works
- [ ] Database is updated correctly
- [ ] Error handling for invalid book_id
- [ ] Error handling for invalid rating values
- [ ] Response includes confirmation

**💡 Hints**:
<details>
<summary>Hint 1: POST endpoint structure</summary>

```python
@endpoint('/api/book/{book_id}/rating', types={'book_id': int}, methods=['POST'])
def ajax_update_rating(ctx, rd, book_id):
    # rd.request_body_json contains the parsed JSON
    data = rd.request_body_json
    rating = data.get('rating')

    # Validate and update...
```

The `methods=['POST']` restricts this endpoint to POST requests only.
</details>

<details>
<summary>Hint 2: Database write</summary>

```python
# First, check if book exists
if book_id not in ctx.db.all_book_ids():
    raise HTTPNotFound(f"Book {book_id} not found")

# Validate rating
if not isinstance(rating, (int, float)) or rating < 0 or rating > 5:
    raise ValueError("Rating must be between 0 and 5")

# Update the database
ctx.db.set_metadata(book_id, {'rating': rating})
```

See [src/calibre/db/cache.py](../../src/calibre/db/cache.py) for the `set_metadata` method.
</details>

<details>
<summary>Hint 3: Complete solution structure</summary>

```python
@endpoint('/api/book/{book_id}/rating', types={'book_id': int}, methods=['POST'])
def ajax_update_rating(ctx, rd, book_id):
    # Validate book exists
    if book_id not in ctx.db.all_book_ids():
        raise HTTPNotFound(f"Book {book_id} not found")

    # Get rating from request
    data = rd.request_body_json
    rating = data.get('rating')

    # Validate rating
    if rating is None:
        raise ValueError("Missing 'rating' in request body")
    if not isinstance(rating, (int, float)):
        raise ValueError("Rating must be a number")
    if rating < 0 or rating > 5:
        raise ValueError("Rating must be between 0 and 5")

    # Update database
    ctx.db.set_metadata(book_id, {'rating': rating})

    return {
        'success': True,
        'book_id': book_id,
        'new_rating': rating,
        'message': 'Rating updated successfully'
    }
```
</details>

---

### Exercise 2.2: Create a Database Query with Joins

**Difficulty**: ⭐⭐⭐ (2.5 hours)

**🎯 Learning Objective**: Write SQL queries with joins and understand the database schema

**💭 React Analogy**:
```javascript
// Prisma with relations:
const books = await prisma.book.findMany({
  include: {
    authors: true,
    tags: true
  },
  where: {
    authors: {
      some: { name: { contains: 'Stephen' } }
    }
  }
})

// In Calibre, you write raw SQL with joins
```

**🔗 Code References**:
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py) - Database access
- [src/calibre/db/tables.py](../../src/calibre/db/tables.py) - Table definitions
- [src/calibre/library/sqlite.py](../../src/calibre/library/sqlite.py) - SQLite helpers

**Task**: Create an endpoint `/api/books/by-author` that finds all books by an author (partial name match).

**Requirements**:
1. Accept `author` query parameter (string)
2. Search for authors matching the query (case-insensitive partial match)
3. Return all books by matching authors
4. Include book title, rating, and all authors for each book
5. Sort by book title

**Example Request**:
```
GET /api/books/by-author?author=stephen
```

**Example Response**:
```json
{
  "author_query": "stephen",
  "matching_authors": ["Stephen King", "Stephen Hawking"],
  "books": [
    {
      "id": 15,
      "title": "A Brief History of Time",
      "authors": ["Stephen Hawking"],
      "rating": 4.5
    },
    {
      "id": 42,
      "title": "The Shining",
      "authors": ["Stephen King"],
      "rating": 4.8
    }
  ],
  "total": 2
}
```

**✅ Acceptance Criteria**:
- [ ] Partial name matching works
- [ ] Case-insensitive search
- [ ] Books include all their authors (not just matched one)
- [ ] Results are sorted by title
- [ ] Empty results handled gracefully
- [ ] SQL join is efficient (no N+1 queries)

**💡 Hints**:
<details>
<summary>Hint 1: Database schema</summary>

Calibre's database has these relevant tables:
- `books` - book metadata (id, title, etc.)
- `authors` - author names (id, name)
- `books_authors_link` - many-to-many relationship (book, author)

You'll need to join all three tables.
</details>

<details>
<summary>Hint 2: SQL query structure</summary>

```sql
SELECT DISTINCT books.id, books.title, books.rating
FROM books
JOIN books_authors_link ON books.id = books_authors_link.book
JOIN authors ON books_authors_link.author = authors.id
WHERE authors.name LIKE ?
ORDER BY books.title
```

The `?` is a parameter placeholder (use `'%stephen%'` for partial match).
</details>

<details>
<summary>Hint 3: Executing the query</summary>

```python
# Get database connection
conn = ctx.db.conn

# Execute query
query = '''
    SELECT DISTINCT books.id
    FROM books
    JOIN books_authors_link bal ON books.id = bal.book
    JOIN authors ON bal.author = authors.id
    WHERE authors.name LIKE ? COLLATE NOCASE
    ORDER BY books.title
'''

book_ids = [row[0] for row in conn.execute(query, (f'%{author}%',))]

# Now get full metadata for each book
books = []
for book_id in book_ids:
    metadata = ctx.db.get_metadata(book_id)
    books.append({
        'id': book_id,
        'title': metadata.title,
        'authors': ctx.db.authors(book_id),
        'rating': metadata.rating or 0
    })
```

`COLLATE NOCASE` makes the search case-insensitive.
</details>

---

### Exercise 2.3: Add a Custom Widget to GUI

**Difficulty**: ⭐⭐⭐⭐ (3 hours)

**🎯 Learning Objective**: Create a custom PyQt6 widget with signals and slots

**💭 React Analogy**:
```javascript
// React custom component with state and callbacks:
function BookRatingWidget({ bookId, initialRating, onRatingChange }) {
  const [rating, setRating] = useState(initialRating)

  const handleClick = (newRating) => {
    setRating(newRating)
    onRatingChange(bookId, newRating)
  }

  return <div>...</div>
}

// PyQt6: Similar but with classes, signals, and slots
```

**🔗 Code References**:
- [src/calibre/gui2/widgets.py](../../src/calibre/gui2/widgets.py) - Custom widgets
- [src/calibre/gui2/library/views.py](../../src/calibre/gui2/library/views.py) - View widgets
- [PyQt6 documentation](https://www.riverbankcomputing.com/static/Docs/PyQt6/)

**Task**: Create a star rating widget that displays and allows editing a book's rating.

**Requirements**:
1. Create `StarRatingWidget` class inheriting from `QWidget`
2. Display 5 stars (★ for filled, ☆ for empty)
3. Show current rating visually (3.5 → ★★★⯪☆)
4. Allow clicking to set rating
5. Emit signal when rating changes
6. Add to book details panel

**Visual Design**:
```
Current rating: 3.5
Display: ★★★⯪☆
Click on 4th star → ★★★★☆ → emit ratingChanged(4)
```

**✅ Acceptance Criteria**:
- [ ] Widget displays current rating correctly
- [ ] Clicking updates rating
- [ ] Signal is emitted on change
- [ ] Half-stars displayed for .5 ratings
- [ ] Hover effect shows preview
- [ ] Widget integrates into main window

**💡 Hints**:
<details>
<summary>Hint 1: Basic widget structure</summary>

```python
from PyQt6.QtCore import pyqtSignal, Qt
from PyQt6.QtWidgets import QWidget, QHBoxLayout, QLabel

class StarRatingWidget(QWidget):
    # Define signal (like React callback)
    ratingChanged = pyqtSignal(float)

    def __init__(self, initial_rating=0, parent=None):
        super().__init__(parent)
        self.rating = initial_rating
        self.setup_ui()

    def setup_ui(self):
        layout = QHBoxLayout()
        # Add star labels...
        self.setLayout(layout)
```

Signals are like event emitters in React.
</details>

<details>
<summary>Hint 2: Creating clickable stars</summary>

```python
class StarLabel(QLabel):
    clicked = pyqtSignal()

    def __init__(self, text, parent=None):
        super().__init__(text, parent)
        self.setCursor(Qt.CursorShape.PointingHandCursor)

    def mousePressEvent(self, event):
        self.clicked.emit()
        super().mousePressEvent(event)

# In StarRatingWidget:
for i in range(5):
    star = StarLabel('☆', self)
    star.clicked.connect(lambda idx=i: self.set_rating(idx + 1))
    layout.addWidget(star)
```
</details>

<details>
<summary>Hint 3: Updating display</summary>

```python
def update_display(self):
    """Update star characters based on current rating"""
    full_stars = int(self.rating)
    has_half = (self.rating % 1) >= 0.5

    for i, star_label in enumerate(self.star_labels):
        if i < full_stars:
            star_label.setText('★')
        elif i == full_stars and has_half:
            star_label.setText('⯪')  # Half star
        else:
            star_label.setText('☆')

def set_rating(self, new_rating):
    """Called when user clicks a star"""
    if new_rating != self.rating:
        self.rating = new_rating
        self.update_display()
        self.ratingChanged.emit(new_rating)
```
</details>

<details>
<summary>Hint 4: Integration into main window</summary>

In [src/calibre/gui2/library/views.py](../../src/calibre/gui2/library/views.py) or book details panel:

```python
from calibre.gui2.widgets import StarRatingWidget

class BookDetailsPanel(QWidget):
    def __init__(self, parent=None):
        super().__init__(parent)
        # ... other widgets ...

        # Add rating widget
        self.rating_widget = StarRatingWidget()
        self.rating_widget.ratingChanged.connect(self.on_rating_changed)
        layout.addWidget(self.rating_widget)

    def on_rating_changed(self, new_rating):
        """Handle rating change signal"""
        book_id = self.current_book_id
        # Update database
        self.db.set_metadata(book_id, {'rating': new_rating})
```
</details>

---

### Exercise 2.4: Implement Search with Full-Text Search

**Difficulty**: ⭐⭐⭐⭐ (3 hours)

**🎯 Learning Objective**: Use SQLite FTS5 for full-text search

**💭 React Analogy**:
```javascript
// Modern full-text search with ElasticSearch:
const results = await elasticsearch.search({
  index: 'books',
  body: {
    query: {
      multi_match: {
        query: 'science fiction',
        fields: ['title', 'authors', 'comments']
      }
    }
  }
})

// SQLite FTS5: Similar power, embedded
```

**🔗 Code References**:
- [src/calibre/db/search.py](../../src/calibre/db/search.py) - Search implementation
- [src/calibre/db/fts.py](../../src/calibre/db/fts.py) - Full-text search
- [SQLite FTS5 docs](https://www.sqlite.org/fts5.html)

**Task**: Create an endpoint `/api/books/search` that searches books using full-text search.

**Requirements**:
1. Accept `query` parameter (search terms)
2. Search in: title, authors, comments, tags
3. Support multi-word queries ("science fiction")
4. Support phrase queries ("\"science fiction\"")
5. Return ranked results (relevance score)
6. Highlight matching terms in results

**Example Request**:
```
GET /api/books/search?query=quantum physics
```

**Example Response**:
```json
{
  "query": "quantum physics",
  "results": [
    {
      "id": 25,
      "title": "Quantum Physics for Beginners",
      "authors": ["Brian Cox"],
      "score": 0.95,
      "highlights": {
        "title": "<mark>Quantum</mark> <mark>Physics</mark> for Beginners",
        "comments": "Introduction to <mark>quantum</mark> mechanics..."
      }
    }
  ],
  "total": 1,
  "time_ms": 12
}
```

**✅ Acceptance Criteria**:
- [ ] Full-text search works across multiple fields
- [ ] Results are ranked by relevance
- [ ] Phrase search works ("quoted queries")
- [ ] Results include match highlights
- [ ] Performance is acceptable (< 100ms for typical query)
- [ ] Empty query returns error

**💡 Hints**:
<details>
<summary>Hint 1: Calibre's search system</summary>

Calibre already has a powerful search system. Look at [src/calibre/db/search.py](../../src/calibre/db/search.py):

```python
from calibre.db.search import Search

# Create search instance
search = Search(ctx.db)

# Perform search
book_ids = search.search(query_string)
```

You can leverage this existing system!
</details>

<details>
<summary>Hint 2: Using the search API</summary>

```python
@endpoint('/api/books/search', types={'query': str})
def ajax_search_books(ctx, rd, query):
    import time
    start = time.time()

    if not query or not query.strip():
        raise ValueError("Search query cannot be empty")

    # Use Calibre's built-in search
    from calibre.db.search import Search
    search = Search(ctx.db)

    try:
        book_ids = search.search(query)
    except Exception as e:
        raise ValueError(f"Invalid search query: {e}")

    # Get metadata for results
    results = []
    for book_id in book_ids[:50]:  # Limit to 50 results
        metadata = ctx.db.get_metadata(book_id)
        results.append({
            'id': book_id,
            'title': metadata.title,
            'authors': ctx.db.authors(book_id)
        })

    time_ms = int((time.time() - start) * 1000)

    return {
        'query': query,
        'results': results,
        'total': len(book_ids),
        'time_ms': time_ms
    }
```
</details>

<details>
<summary>Hint 3: Advanced search with FTS5</summary>

For custom FTS5 queries with highlights:

```python
# Create FTS5 query
conn = ctx.db.conn
fts_query = f'''
    SELECT
        books.id,
        books.title,
        snippet(fts_books, 1, '<mark>', '</mark>', '...', 50) as title_highlight,
        rank as score
    FROM fts_books
    JOIN books ON fts_books.id = books.id
    WHERE fts_books MATCH ?
    ORDER BY rank
    LIMIT 50
'''

results = []
for row in conn.execute(fts_query, (query,)):
    book_id = row[0]
    results.append({
        'id': book_id,
        'title': row[1],
        'authors': ctx.db.authors(book_id),
        'score': abs(row[3]),  # FTS5 rank is negative
        'highlights': {
            'title': row[2]
        }
    })
```

The `snippet()` function generates highlighted excerpts.
</details>

---

## Advanced Exercises (⭐⭐⭐⭐⭐)

### Exercise 3.1: Implement WebSocket for Real-Time Updates

**Difficulty**: ⭐⭐⭐⭐⭐ (4-6 hours)

**🎯 Learning Objective**: Add WebSocket support for real-time notifications

**💭 React Analogy**:
```javascript
// React with Socket.IO:
const socket = io('http://localhost:3000')
socket.on('bookUpdated', (data) => {
  setBooks(prev => updateBook(prev, data))
})

// Calibre: Need to implement WebSocket in custom server
```

**🔗 Code References**:
- [src/calibre/srv/loop.py](../../src/calibre/srv/loop.py#L392) - Server loop
- [src/calibre/srv/http_request.py](../../src/calibre/srv/http_request.py) - Request handling
- [Python websockets library](https://websockets.readthedocs.io/)

**Context**: Currently, Calibre's web interface requires manual refresh to see updates. Adding WebSocket support would enable real-time updates when books are added, ratings change, etc.

**Task**: Implement WebSocket endpoint that broadcasts book updates in real-time.

**Requirements**:
1. Add WebSocket endpoint `/ws`
2. Accept WebSocket connections
3. Broadcast messages when books are updated
4. Support multiple concurrent clients
5. Handle client disconnections gracefully
6. Send initial state on connection

**Message Protocol**:
```javascript
// Client → Server (subscribe to book updates)
{
  "type": "subscribe",
  "resource": "books"
}

// Server → Client (book updated)
{
  "type": "bookUpdated",
  "bookId": 42,
  "changes": {
    "rating": 4.5
  },
  "timestamp": "2025-11-19T10:30:00Z"
}

// Server → Client (heartbeat)
{
  "type": "ping",
  "timestamp": "2025-11-19T10:30:00Z"
}
```

**✅ Acceptance Criteria**:
- [ ] WebSocket endpoint accepts connections
- [ ] Multiple clients can connect simultaneously
- [ ] Updates broadcast to all connected clients
- [ ] Clients can subscribe to specific event types
- [ ] Disconnections handled without crashing server
- [ ] Heartbeat keeps connections alive
- [ ] Integration with existing database write methods

**💡 Hints**:
<details>
<summary>Hint 1: WebSocket in custom server</summary>

Calibre's server uses `select()`, so you'll need to integrate WebSocket handling into the existing event loop. Consider using the `websockets` library with threading:

```python
import asyncio
import websockets
import threading
import json

class WebSocketManager:
    def __init__(self):
        self.clients = set()

    async def handler(self, websocket, path):
        # Register client
        self.clients.add(websocket)
        try:
            async for message in websocket:
                data = json.loads(message)
                await self.handle_message(websocket, data)
        finally:
            self.clients.remove(websocket)

    async def broadcast(self, message):
        if self.clients:
            await asyncio.gather(
                *[client.send(json.dumps(message)) for client in self.clients],
                return_exceptions=True
            )

    def start(self, host='0.0.0.0', port=8081):
        async def serve():
            async with websockets.serve(self.handler, host, port):
                await asyncio.Future()  # Run forever

        def run():
            asyncio.run(serve())

        thread = threading.Thread(target=run, daemon=True)
        thread.start()
```
</details>

<details>
<summary>Hint 2: Integrating with database updates</summary>

Hook into database write methods to broadcast updates:

```python
# In src/calibre/db/cache.py or create a wrapper

original_set_metadata = Cache.set_metadata

def set_metadata_with_broadcast(self, book_id, metadata):
    # Call original method
    result = original_set_metadata(self, book_id, metadata)

    # Broadcast update
    if hasattr(self, 'websocket_manager'):
        asyncio.run_coroutine_threadsafe(
            self.websocket_manager.broadcast({
                'type': 'bookUpdated',
                'bookId': book_id,
                'changes': metadata,
                'timestamp': datetime.utcnow().isoformat() + 'Z'
            }),
            self.websocket_manager.loop
        )

    return result

Cache.set_metadata = set_metadata_with_broadcast
```
</details>

<details>
<summary>Hint 3: Client-side JavaScript</summary>

Example client code for testing:

```javascript
const ws = new WebSocket('ws://localhost:8081/ws')

ws.onopen = () => {
  console.log('Connected to Calibre WebSocket')
  ws.send(JSON.stringify({
    type: 'subscribe',
    resource: 'books'
  }))
}

ws.onmessage = (event) => {
  const data = JSON.parse(event.data)
  console.log('Received:', data)

  if (data.type === 'bookUpdated') {
    // Update UI
    updateBookInList(data.bookId, data.changes)
  }
}

ws.onerror = (error) => {
  console.error('WebSocket error:', error)
}

ws.onclose = () => {
  console.log('Disconnected from WebSocket')
  // Implement reconnection logic
  setTimeout(() => connectWebSocket(), 5000)
}
```
</details>

---

### Exercise 3.2: Optimize Database Query Performance

**Difficulty**: ⭐⭐⭐⭐⭐ (4-6 hours)

**🎯 Learning Objective**: Profile and optimize slow database queries

**💭 React Analogy**:
```javascript
// Unoptimized: N+1 query problem
const books = await db.book.findMany()
for (const book of books) {
  book.authors = await db.author.findMany({ where: { bookId: book.id } })
}

// Optimized: Single query with join
const books = await db.book.findMany({
  include: { authors: true }
})
```

**🔗 Code References**:
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py) - Cache implementation
- [src/calibre/db/search.py](../../src/calibre/db/search.py) - Search queries
- [SQLite EXPLAIN QUERY PLAN](https://www.sqlite.org/eqp.html)

**Context**: When dealing with large libraries (10,000+ books), some operations become slow. This exercise teaches you to identify and fix performance bottlenecks.

**Task**: Profile the `/api/books` endpoint (returns all books) and optimize it for large libraries.

**Requirements**:
1. Create `/api/books` endpoint that returns all books with authors and tags
2. Benchmark current performance with 10,000 books
3. Identify slow queries using SQLite EXPLAIN QUERY PLAN
4. Optimize to handle 10,000+ books in < 200ms
5. Document optimization techniques used
6. Add caching strategy

**Performance Targets**:
- 1,000 books: < 50ms
- 10,000 books: < 200ms
- 100,000 books: < 1000ms

**✅ Acceptance Criteria**:
- [ ] Initial benchmark documented
- [ ] Identified specific slow queries
- [ ] Implemented at least 3 optimizations
- [ ] Achieved performance targets
- [ ] No N+1 query problems
- [ ] Proper indexing verified
- [ ] Caching strategy implemented
- [ ] Documentation includes EXPLAIN QUERY PLAN outputs

**💡 Hints**:
<details>
<summary>Hint 1: Benchmarking setup</summary>

```python
import time
import cProfile
import pstats

@endpoint('/api/books')
def ajax_get_all_books(ctx, rd):
    start = time.time()

    # Your implementation here
    results = get_all_books_with_metadata(ctx.db)

    elapsed = time.time() - start

    return {
        'books': results,
        'count': len(results),
        'query_time_ms': int(elapsed * 1000)
    }

# Profile with:
# cProfile.run('ajax_get_all_books(ctx, rd)', 'profile_stats')
# stats = pstats.Stats('profile_stats')
# stats.sort_stats('cumulative').print_stats(20)
```
</details>

<details>
<summary>Hint 2: Finding slow queries</summary>

Use SQLite's EXPLAIN QUERY PLAN:

```python
conn = ctx.db.conn

# Your query
query = '''
    SELECT books.id, books.title
    FROM books
    JOIN books_authors_link ON books.id = books_authors_link.book
    WHERE books.title LIKE '%python%'
'''

# Analyze it
explain = conn.execute(f'EXPLAIN QUERY PLAN {query}').fetchall()
for row in explain:
    print(row)
```

Look for:
- `SCAN TABLE` (bad - full table scan)
- `SEARCH TABLE USING INDEX` (good - using index)
- `USING COVERING INDEX` (best - all data in index)
</details>

<details>
<summary>Hint 3: Common optimizations</summary>

**Optimization 1: Batch fetching instead of N+1**

```python
# BAD: N+1 queries
def get_books_with_authors_slow(db):
    books = []
    for book_id in db.all_book_ids():
        metadata = db.get_metadata(book_id)
        authors = db.authors(book_id)  # Separate query for each book!
        books.append({'id': book_id, 'title': metadata.title, 'authors': authors})
    return books

# GOOD: Single query with join
def get_books_with_authors_fast(db):
    query = '''
        SELECT
            books.id,
            books.title,
            GROUP_CONCAT(authors.name, '|') as authors
        FROM books
        LEFT JOIN books_authors_link bal ON books.id = bal.book
        LEFT JOIN authors ON bal.author = authors.id
        GROUP BY books.id
        ORDER BY books.title
    '''

    books = []
    for row in db.conn.execute(query):
        books.append({
            'id': row[0],
            'title': row[1],
            'authors': row[2].split('|') if row[2] else []
        })
    return books
```

**Optimization 2: Add indexes**

```python
# Check if index exists
def ensure_indexes(conn):
    # Create index on commonly filtered columns
    conn.execute('CREATE INDEX IF NOT EXISTS idx_books_title ON books(title)')
    conn.execute('CREATE INDEX IF NOT EXISTS idx_authors_name ON authors(name)')
```

**Optimization 3: Implement caching**

```python
from functools import lru_cache
import hashlib

class CachedBookList:
    def __init__(self, db):
        self.db = db
        self.cache = {}
        self.cache_timestamp = None

    def get_all_books(self, force_refresh=False):
        # Check if cache is still valid
        current_timestamp = self.db.last_modified()

        if (not force_refresh and
            self.cache_timestamp == current_timestamp and
            'all_books' in self.cache):
            return self.cache['all_books']

        # Refresh cache
        books = self._fetch_all_books()
        self.cache['all_books'] = books
        self.cache_timestamp = current_timestamp

        return books
```
</details>

<details>
<summary>Hint 4: Complete optimized solution</summary>

```python
@endpoint('/api/books')
def ajax_get_all_books(ctx, rd):
    import time
    start = time.time()

    # Use single query with all joins
    query = '''
        SELECT
            books.id,
            books.title,
            books.rating,
            books.pubdate,
            GROUP_CONCAT(DISTINCT authors.name, '|') as authors,
            GROUP_CONCAT(DISTINCT tags.name, '|') as tags
        FROM books
        LEFT JOIN books_authors_link bal ON books.id = bal.book
        LEFT JOIN authors ON bal.author = authors.id
        LEFT JOIN books_tags_link btl ON books.id = btl.book
        LEFT JOIN tags ON btl.tag = tags.id
        GROUP BY books.id
        ORDER BY books.title
    '''

    books = []
    for row in ctx.db.conn.execute(query):
        books.append({
            'id': row[0],
            'title': row[1],
            'rating': row[2] or 0,
            'pubdate': row[3],
            'authors': row[4].split('|') if row[4] else [],
            'tags': row[5].split('|') if row[5] else []
        })

    elapsed = time.time() - start

    return {
        'books': books,
        'count': len(books),
        'query_time_ms': int(elapsed * 1000)
    }
```

This reduces from O(N) queries to O(1) query!
</details>

---

### Exercise 3.3: Implement Undo/Redo for Metadata Changes

**Difficulty**: ⭐⭐⭐⭐⭐ (6-8 hours)

**🎯 Learning Objective**: Implement command pattern for undo/redo functionality

**💭 React Analogy**:
```javascript
// Redux with time-travel debugging:
const reducer = (state, action) => {
  switch (action.type) {
    case 'UPDATE_BOOK':
      return { ...state, books: updateBook(state.books, action.payload) }
    case 'UNDO':
      return previousState  // Redux DevTools handles this
  }
}

// Calibre: Implement command pattern manually
```

**🔗 Code References**:
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py) - Database modifications
- [src/calibre/db/write.py](../../src/calibre/db/write.py) - Write operations
- [Command Pattern](https://refactoring.guru/design-patterns/command)

**Context**: Professional applications need undo/redo. This exercise teaches the Command pattern and state management.

**Task**: Implement undo/redo system for book metadata changes (rating, title, authors, etc.)

**Requirements**:
1. Create `CommandHistory` class to track changes
2. Implement `Command` base class with `execute()` and `undo()` methods
3. Create commands for: update rating, update title, update authors
4. Add `/api/undo` and `/api/redo` endpoints
5. Limit history to last 50 actions
6. Persist history to survive restarts (optional)
7. Support keyboard shortcuts (Ctrl+Z, Ctrl+Shift+Z)

**Commands to Implement**:
- `UpdateRatingCommand`
- `UpdateTitleCommand`
- `UpdateAuthorsCommand`
- `AddBookCommand`
- `DeleteBookCommand`

**Example Usage**:
```python
# User updates rating
POST /api/book/1/rating {"rating": 4.5}
→ Creates UpdateRatingCommand(book_id=1, old_rating=3.0, new_rating=4.5)

# User undoes
POST /api/undo
→ Executes UpdateRatingCommand.undo() → rating back to 3.0

# User redoes
POST /api/redo
→ Executes UpdateRatingCommand.execute() → rating back to 4.5
```

**✅ Acceptance Criteria**:
- [ ] All commands implement execute() and undo()
- [ ] Undo/redo endpoints work correctly
- [ ] History maintains max 50 items
- [ ] Redo stack cleared when new command executed
- [ ] Commands are properly tested
- [ ] Thread-safe implementation
- [ ] Optional: History persisted to database

**💡 Hints**:
<details>
<summary>Hint 1: Command pattern structure</summary>

```python
from abc import ABC, abstractmethod
from typing import Any, Dict

class Command(ABC):
    """Base class for all commands"""

    @abstractmethod
    def execute(self) -> Any:
        """Execute the command"""
        pass

    @abstractmethod
    def undo(self) -> Any:
        """Undo the command"""
        pass

    @abstractmethod
    def description(self) -> str:
        """Human-readable description"""
        pass

class UpdateRatingCommand(Command):
    def __init__(self, db, book_id: int, new_rating: float):
        self.db = db
        self.book_id = book_id
        self.new_rating = new_rating
        self.old_rating = None

    def execute(self):
        # Save old value for undo
        metadata = self.db.get_metadata(self.book_id)
        self.old_rating = metadata.rating

        # Apply change
        self.db.set_metadata(self.book_id, {'rating': self.new_rating})

        return {'success': True, 'new_rating': self.new_rating}

    def undo(self):
        # Restore old value
        self.db.set_metadata(self.book_id, {'rating': self.old_rating})
        return {'success': True, 'rating': self.old_rating}

    def description(self):
        return f"Update rating for book {self.book_id} to {self.new_rating}"
```
</details>

<details>
<summary>Hint 2: Command history manager</summary>

```python
class CommandHistory:
    """Manages undo/redo history"""

    def __init__(self, max_history=50):
        self.max_history = max_history
        self.undo_stack = []  # Commands that can be undone
        self.redo_stack = []  # Commands that can be redone

    def execute_command(self, command: Command):
        """Execute a command and add to history"""
        result = command.execute()

        # Add to undo stack
        self.undo_stack.append(command)
        if len(self.undo_stack) > self.max_history:
            self.undo_stack.pop(0)

        # Clear redo stack (new action invalidates redo)
        self.redo_stack.clear()

        return result

    def undo(self):
        """Undo the last command"""
        if not self.undo_stack:
            raise ValueError("Nothing to undo")

        command = self.undo_stack.pop()
        result = command.undo()
        self.redo_stack.append(command)

        return result

    def redo(self):
        """Redo the last undone command"""
        if not self.redo_stack:
            raise ValueError("Nothing to redo")

        command = self.redo_stack.pop()
        result = command.execute()
        self.undo_stack.append(command)

        return result

    def can_undo(self):
        return len(self.undo_stack) > 0

    def can_redo(self):
        return len(self.redo_stack) > 0

    def get_history(self):
        """Get list of commands in history"""
        return [cmd.description() for cmd in self.undo_stack]
```
</details>

<details>
<summary>Hint 3: Integration with API endpoints</summary>

```python
# Global command history (in production, use proper dependency injection)
command_history = CommandHistory()

@endpoint('/api/book/{book_id}/rating', types={'book_id': int}, methods=['POST'])
def ajax_update_rating(ctx, rd, book_id):
    data = rd.request_body_json
    new_rating = data.get('rating')

    # Validate...

    # Create and execute command
    command = UpdateRatingCommand(ctx.db, book_id, new_rating)
    result = command_history.execute_command(command)

    return result

@endpoint('/api/undo', methods=['POST'])
def ajax_undo(ctx, rd):
    try:
        result = command_history.undo()
        return {
            'success': True,
            'action': 'undo',
            'result': result,
            'can_undo': command_history.can_undo(),
            'can_redo': command_history.can_redo()
        }
    except ValueError as e:
        raise HTTPBadRequest(str(e))

@endpoint('/api/redo', methods=['POST'])
def ajax_redo(ctx, rd):
    try:
        result = command_history.redo()
        return {
            'success': True,
            'action': 'redo',
            'result': result,
            'can_undo': command_history.can_undo(),
            'can_redo': command_history.can_redo()
        }
    except ValueError as e:
        raise HTTPBadRequest(str(e))

@endpoint('/api/history')
def ajax_get_history(ctx, rd):
    return {
        'history': command_history.get_history(),
        'can_undo': command_history.can_undo(),
        'can_redo': command_history.can_redo()
    }
```
</details>

<details>
<summary>Hint 4: More command examples</summary>

```python
class UpdateTitleCommand(Command):
    def __init__(self, db, book_id: int, new_title: str):
        self.db = db
        self.book_id = book_id
        self.new_title = new_title
        self.old_title = None

    def execute(self):
        metadata = self.db.get_metadata(self.book_id)
        self.old_title = metadata.title
        self.db.set_metadata(self.book_id, {'title': self.new_title})
        return {'success': True, 'title': self.new_title}

    def undo(self):
        self.db.set_metadata(self.book_id, {'title': self.old_title})
        return {'success': True, 'title': self.old_title}

    def description(self):
        return f'Update title to "{self.new_title}"'

class UpdateAuthorsCommand(Command):
    def __init__(self, db, book_id: int, new_authors: list):
        self.db = db
        self.book_id = book_id
        self.new_authors = new_authors
        self.old_authors = None

    def execute(self):
        self.old_authors = self.db.authors(self.book_id)
        self.db.set_authors(self.book_id, self.new_authors)
        return {'success': True, 'authors': self.new_authors}

    def undo(self):
        self.db.set_authors(self.book_id, self.old_authors)
        return {'success': True, 'authors': self.old_authors}

    def description(self):
        return f'Update authors to {", ".join(self.new_authors)}'

# Compound command (multiple changes as one action)
class CompoundCommand(Command):
    def __init__(self, commands: list):
        self.commands = commands

    def execute(self):
        results = []
        for cmd in self.commands:
            results.append(cmd.execute())
        return results

    def undo(self):
        # Undo in reverse order
        results = []
        for cmd in reversed(self.commands):
            results.append(cmd.undo())
        return results

    def description(self):
        return f"Compound: {'; '.join(cmd.description() for cmd in self.commands)}"
```
</details>

---

## Project-Based Exercises

### Project 1: Build a Reading List Feature

**Difficulty**: ⭐⭐⭐⭐ (8-12 hours)

**🎯 Learning Objective**: Full-stack feature from database schema to UI

**Description**: Add a "reading list" feature where users can create multiple reading lists, add books to them, and reorder books within lists.

**Features**:
1. Create/update/delete reading lists
2. Add/remove books from lists
3. Reorder books within a list (drag-and-drop)
4. Mark books as "completed" in a list
5. Share lists (export as JSON/HTML)

**Database Schema**:
```sql
CREATE TABLE reading_lists (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE reading_list_items (
    id INTEGER PRIMARY KEY,
    list_id INTEGER REFERENCES reading_lists(id),
    book_id INTEGER REFERENCES books(id),
    position INTEGER NOT NULL,
    completed BOOLEAN DEFAULT 0,
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(list_id, book_id)
);
```

**API Endpoints to Implement**:
- `GET /api/reading-lists` - List all lists
- `POST /api/reading-lists` - Create list
- `GET /api/reading-lists/{id}` - Get list with books
- `PUT /api/reading-lists/{id}` - Update list
- `DELETE /api/reading-lists/{id}` - Delete list
- `POST /api/reading-lists/{id}/books` - Add book to list
- `DELETE /api/reading-lists/{id}/books/{book_id}` - Remove book
- `PUT /api/reading-lists/{id}/reorder` - Reorder books

**UI Components** (PyQt6):
- Reading lists panel in main window
- List creation dialog
- Drag-and-drop book reordering
- Context menu for adding books to lists

**Bonus**:
- Progress bar showing % completed
- Estimated reading time based on page count
- Export list to Markdown/HTML
- Sync lists across devices (using simple file export/import)

---

### Project 2: Advanced Book Recommendations

**Difficulty**: ⭐⭐⭐⭐⭐ (12-16 hours)

**🎯 Learning Objective**: Machine learning integration and advanced algorithms

**Description**: Implement a recommendation system that suggests books based on reading history, ratings, and book metadata.

**Algorithm Approaches**:
1. **Collaborative Filtering**: "Users who rated X highly also liked Y"
2. **Content-Based**: Similar authors, tags, genres
3. **Hybrid**: Combine both approaches

**Features**:
1. "Books you might like" based on ratings
2. "Similar books" for each book
3. "Because you read X" personalized recommendations
4. Trending books in your library
5. Hidden gems (good books you haven't read)

**Implementation Steps**:

**Step 1**: Data collection
```python
# Collect rating patterns
def get_user_ratings(db):
    # In Calibre, "user" is the library owner
    query = '''
        SELECT id, rating
        FROM books
        WHERE rating IS NOT NULL
        ORDER BY rating DESC
    '''
    return db.conn.execute(query).fetchall()
```

**Step 2**: Feature extraction
```python
def extract_book_features(db, book_id):
    """Extract features for similarity calculation"""
    metadata = db.get_metadata(book_id)

    return {
        'authors': db.authors(book_id),
        'tags': db.tags(book_id),
        'publisher': metadata.publisher,
        'pubdate': metadata.pubdate.year if metadata.pubdate else None,
        'series': metadata.series,
        'rating': metadata.rating or 0
    }
```

**Step 3**: Similarity calculation
```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

def calculate_similarity(db):
    """Calculate book-to-book similarity matrix"""
    books = []
    book_ids = []

    for book_id in db.all_book_ids():
        features = extract_book_features(db, book_id)
        # Combine features into text
        text = ' '.join([
            ' '.join(features['authors']),
            ' '.join(features['tags']),
            features['publisher'] or '',
            features['series'] or ''
        ])
        books.append(text)
        book_ids.append(book_id)

    # Create TF-IDF matrix
    vectorizer = TfidfVectorizer()
    tfidf_matrix = vectorizer.fit_transform(books)

    # Calculate cosine similarity
    similarity_matrix = cosine_similarity(tfidf_matrix)

    return book_ids, similarity_matrix
```

**Step 4**: Recommendation generation
```python
def get_recommendations(db, book_id, similarity_matrix, book_ids, n=10):
    """Get N most similar books"""
    idx = book_ids.index(book_id)
    similarity_scores = list(enumerate(similarity_matrix[idx]))

    # Sort by similarity (descending)
    similarity_scores = sorted(similarity_scores, key=lambda x: x[1], reverse=True)

    # Get top N (excluding self)
    top_books = similarity_scores[1:n+1]

    recommendations = []
    for i, score in top_books:
        rec_book_id = book_ids[i]
        metadata = db.get_metadata(rec_book_id)
        recommendations.append({
            'id': rec_book_id,
            'title': metadata.title,
            'authors': db.authors(rec_book_id),
            'similarity_score': float(score),
            'rating': metadata.rating or 0
        })

    return recommendations
```

**API Endpoints**:
- `GET /api/recommendations/for-you` - Personalized recommendations
- `GET /api/recommendations/similar/{book_id}` - Similar books
- `GET /api/recommendations/trending` - Trending in library
- `POST /api/recommendations/train` - Retrain recommendation model

**Bonus**:
- Cache recommendations (update daily)
- A/B testing different algorithms
- Explanation of why book was recommended
- User feedback ("Not interested", "Already read")

---

## Solutions and Explanations

### Solution 1.1: Simple GET Endpoint

```python
# In src/calibre/srv/routes.py

@endpoint('/api/hello')
def ajax_hello(ctx, rd):
    """
    Simple hello world endpoint

    Returns a greeting message with version info.
    """
    return {
        'message': 'Hello from Calibre!',
        'version': '1.0'
    }
```

**Explanation**:

1. **@endpoint decorator**: Registers this function as an HTTP endpoint
   - Path is `/api/hello`
   - Automatically handles HTTP request/response
   - Converts return dict to JSON

2. **Function signature**: `(ctx, rd)`
   - `ctx`: Context object with database, settings, etc.
   - `rd`: Request data (headers, body, query params)

3. **Return value**: Plain Python dict
   - Automatically serialized to JSON
   - `Content-Type: application/json` header added
   - Status code 200 OK (default)

**Testing**:
```bash
curl http://localhost:8080/api/hello
# → {"message": "Hello from Calibre!", "version": "1.0"}
```

**React Comparison**:
```javascript
// This is equivalent to Next.js API route:
// pages/api/hello.ts
export default function handler(req, res) {
  res.status(200).json({
    message: 'Hello from Calibre!',
    version: '1.0'
  })
}
```

---

### Solution 1.2: Query Parameters

```python
@endpoint('/api/hello', types={'name': str, 'count': int})
def ajax_hello(ctx, rd, name, count=1):
    """
    Personalized greeting endpoint with repetition

    Args:
        name: Person's name (required)
        count: Number of greetings (optional, 1-10, default: 1)

    Returns:
        JSON with list of greeting messages

    Raises:
        ValueError: If count is out of range
    """
    # Validate count range
    if count < 1 or count > 10:
        raise ValueError("count must be between 1 and 10")

    # Generate greetings
    greeting = f"Hello, {name}!"
    greetings = [greeting] * count

    return {
        'greetings': greetings
    }
```

**Explanation**:

1. **Type specification**: `types={'name': str, 'count': int}`
   - Calibre automatically extracts query params
   - Validates type (converts string to int)
   - Raises error if type conversion fails

2. **Default parameter**: `count=1`
   - Makes parameter optional
   - If not provided, uses default value

3. **Validation**: `if count < 1 or count > 10`
   - Custom business logic validation
   - `raise ValueError` returns 400 Bad Request with error message

4. **List multiplication**: `[greeting] * count`
   - Python feature: `['a'] * 3 → ['a', 'a', 'a']`

**Testing**:
```bash
# Required parameter
curl 'http://localhost:8080/api/hello?name=Alice'
# → {"greetings": ["Hello, Alice!"]}

# With count
curl 'http://localhost:8080/api/hello?name=Bob&count=3'
# → {"greetings": ["Hello, Bob!", "Hello, Bob!", "Hello, Bob!"]}

# Error case
curl 'http://localhost:8080/api/hello?name=Charlie&count=20'
# → {"error": "count must be between 1 and 10"}
```

**React Comparison**:
```javascript
// Next.js with validation:
export default function handler(req, res) {
  const { name, count = 1 } = req.query

  // Type conversion (query params are strings)
  const countNum = parseInt(count)

  // Validation
  if (!name) {
    return res.status(400).json({ error: 'name is required' })
  }
  if (countNum < 1 || countNum > 10) {
    return res.status(400).json({ error: 'count must be between 1 and 10' })
  }

  const greeting = `Hello, ${name}!`
  const greetings = Array(countNum).fill(greeting)

  res.json({ greetings })
}
```

Calibre's `@endpoint` with `types` parameter handles all the validation automatically!

---

### Solution 1.3: Database Read

```python
from calibre.srv.errors import HTTPNotFound

@endpoint('/api/book/{book_id}', types={'book_id': int})
def ajax_get_book(ctx, rd, book_id):
    """
    Get book metadata by ID

    Args:
        book_id: Book ID in the library

    Returns:
        JSON with book title, authors, publisher, dates, formats

    Raises:
        HTTPNotFound: If book doesn't exist
    """
    # Check if book exists
    if book_id not in ctx.db.all_book_ids():
        raise HTTPNotFound(f"Book {book_id} not found")

    # Get metadata
    metadata = ctx.db.get_metadata(book_id)

    # Get related data
    authors = ctx.db.authors(book_id)
    formats = ctx.db.formats(book_id)

    # Format publish date
    pubdate = None
    if metadata.pubdate:
        pubdate = metadata.pubdate.isoformat()

    return {
        'id': book_id,
        'title': metadata.title,
        'authors': authors,
        'publisher': metadata.publisher,
        'pubdate': pubdate,
        'formats': formats,
        'rating': metadata.rating or 0,
        'tags': ctx.db.tags(book_id),
        'comments': metadata.comments or ''
    }
```

**Explanation**:

1. **Path parameter**: `{book_id}` in path, `book_id` in function signature
   - Automatically extracted from URL
   - Converted to int via `types={'book_id': int}`

2. **Existence check**: `if book_id not in ctx.db.all_book_ids()`
   - Returns set of all valid book IDs
   - Fast O(1) lookup

3. **Metadata object**: `metadata = ctx.db.get_metadata(book_id)`
   - Returns `Metadata` object (like a dataclass)
   - Has attributes: title, publisher, pubdate, rating, etc.

4. **Related data**: `authors()`, `formats()`, `tags()`
   - Separate methods because they involve joins
   - Return lists

5. **Date serialization**: `metadata.pubdate.isoformat()`
   - Python datetime → ISO 8601 string ("2025-11-19")
   - JSON-compatible

**Database Queries Behind the Scenes**:

When you call `ctx.db.authors(book_id)`, Calibre runs:
```sql
SELECT authors.name
FROM authors
JOIN books_authors_link ON authors.id = books_authors_link.author
WHERE books_authors_link.book = ?
ORDER BY books_authors_link.id
```

This is cached in memory, so subsequent calls are fast!

**Testing**:
```bash
curl http://localhost:8080/api/book/1
# → {
#   "id": 1,
#   "title": "The Great Gatsby",
#   "authors": ["F. Scott Fitzgerald"],
#   ...
# }

curl http://localhost:8080/api/book/99999
# → {"error": "Book 99999 not found"}
```

**React Comparison**:
```javascript
// Next.js with Prisma:
export default async function handler(req, res) {
  const { book_id } = req.query

  const book = await prisma.book.findUnique({
    where: { id: parseInt(book_id) },
    include: {
      authors: true,
      formats: true,
      tags: true
    }
  })

  if (!book) {
    return res.status(404).json({ error: `Book ${book_id} not found` })
  }

  res.json({
    id: book.id,
    title: book.title,
    authors: book.authors.map(a => a.name),
    publisher: book.publisher,
    pubdate: book.pubdate?.toISOString(),
    formats: book.formats.map(f => f.format),
    rating: book.rating || 0,
    tags: book.tags.map(t => t.name),
    comments: book.comments || ''
  })
}
```

Very similar! Calibre's `ctx.db` is like Prisma client.

---

### Additional Exercise Solutions

The remaining solutions follow similar patterns. Key concepts to master:

**For Database Exercises**:
- Always check existence before queries
- Use JOINs instead of N+1 queries
- Leverage SQLite's built-in features (FTS5, JSON functions)
- Cache expensive queries

**For GUI Exercises**:
- Widgets = Components
- Signals/slots = Props/callbacks
- Layouts = Flexbox/Grid
- Qt Designer = Visual component builder

**For Advanced Exercises**:
- Use design patterns (Command, Observer, Strategy)
- Profile before optimizing
- Batch operations when possible
- Test edge cases (empty data, large datasets)

---

## Learning Resources

### Internal Documentation
- [README.md](README.md) - Start here
- [TECH_STACK_RESEARCH.md](TECH_STACK_RESEARCH.md) - Technology overview
- [PROJECT_STRUCTURE.md](PROJECT_STRUCTURE.md) - Codebase navigation
- [CODE_TOURS.md](CODE_TOURS.md) - Guided code walkthroughs
- [HOW_TO_GUIDE.md](HOW_TO_GUIDE.md) - Common tasks cookbook

### External Resources
- [Python Official Tutorial](https://docs.python.org/3/tutorial/)
- [PyQt6 Documentation](https://www.riverbankcomputing.com/static/Docs/PyQt6/)
- [SQLite Documentation](https://www.sqlite.org/docs.html)
- [Design Patterns](https://refactoring.guru/design-patterns)

### Practice Datasets

For testing with realistic data:

```bash
# Create test library with 1000 books
python scripts/create_test_library.py --count 1000

# Import sample data
python scripts/import_gutenberg.py --limit 100
```

---

## Next Steps

After completing these exercises:

1. **Read Real Code**: Browse Calibre's source and understand how features work
2. **Fix a Bug**: Find an issue on GitHub and fix it
3. **Add a Feature**: Implement something you wish Calibre had
4. **Contribute**: Submit your first pull request!

See [FIRST_CONTRIBUTIONS.md](FIRST_CONTRIBUTIONS.md) for guidance on making your first contribution.

---

## Getting Help

**Stuck on an exercise?**
1. Re-read the hints (they're progressive)
2. Check the solution (learn from it)
3. Read the referenced source files
4. Ask in [Calibre forums](https://www.mobileread.com/forums/forumdisplay.php?f=166)

**Found a mistake?**
- Open an issue or PR to fix the documentation
- Help future learners!

**Want more exercises?**
- Propose new ones in the discussions
- Share your solutions with the community

---

*Remember: The goal isn't to complete every exercise, but to understand the patterns and principles. Pick exercises that interest you and go deep!*
