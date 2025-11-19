# Testing Guide for Calibre

Comprehensive guide to testing in Calibre, explaining pytest, fixtures, mocking, and testing strategies with comparisons to JavaScript/React testing frameworks.

## Table of Contents

1. [Overview](#overview)
2. [Testing Philosophy](#testing-philosophy)
3. [pytest Fundamentals](#pytest-fundamentals)
4. [Unit Testing](#unit-testing)
5. [Integration Testing](#integration-testing)
6. [GUI Testing](#gui-testing)
7. [API Testing](#api-testing)
8. [Database Testing](#database-testing)
9. [Fixtures and Mocking](#fixtures-and-mocking)
10. [Test Coverage](#test-coverage)
11. [Continuous Integration](#continuous-integration)
12. [Best Practices](#best-practices)

---

## Overview

### Testing Stack

**Calibre uses:**
- **pytest** - Primary testing framework (like Jest for JavaScript)
- **unittest.mock** - Mocking library (like jest.mock)
- **pytest-qt** - PyQt6 GUI testing
- **coverage.py** - Code coverage (like Istanbul/nyc)

**🔗 Code References**:
- [src/calibre/test/](../../src/calibre/test/) - Test directory
- [setup.py](../../setup.py) - Test configuration

### Testing Pyramid

```
        E2E Tests (10%)
       /              \
    Integration (20%)
   /                    \
  Unit Tests (70%)
```

**70% Unit Tests** - Fast, isolated, test individual functions
**20% Integration** - Test module interactions
**10% E2E** - Full application workflows

**💭 Same as React testing!**

---

## Testing Philosophy

### Calibre's Testing Approach

**What Gets Tested:**
- ✅ Core business logic (metadata, conversion, library management)
- ✅ Database operations (queries, transactions, migrations)
- ✅ API endpoints (request/response, validation, errors)
- ✅ Critical algorithms (search, sorting, parsing)

**What's Less Tested:**
- ⚠️ GUI (some manual testing)
- ⚠️ Device drivers (requires hardware)
- ⚠️ Format converters (extensive but not 100%)

**Why:**
- Core features change frequently → need tests
- GUI is stable → manual testing sufficient
- Hardware testing is expensive → selective coverage

### Test-Driven Development (Optional)

```python
# 1. Write failing test
def test_rating_validation():
    book = BookMetadata('Title', ['Author'])
    with pytest.raises(ValueError):
        book.set_rating(6.0)  # ❌ Fails (not implemented yet)

# 2. Implement feature
class BookMetadata:
    def set_rating(self, rating):
        if not 0 <= rating <= 5:
            raise ValueError("Rating must be 0-5")
        self.rating = rating

# 3. Test passes ✅
```

---

## pytest Fundamentals

### Installation and Setup

```bash
# Install pytest
pip install pytest pytest-cov pytest-qt

# Run all tests
pytest

# Run specific test file
pytest src/calibre/test/test_metadata.py

# Run specific test
pytest src/calibre/test/test_metadata.py::test_rating_validation

# Run with coverage
pytest --cov=calibre --cov-report=html

# Run in parallel (faster)
pytest -n auto
```

### Basic Test Structure

```python
# test_book_metadata.py
import pytest
from calibre.ebooks.metadata.book.base import BookMetadata

def test_book_creation():
    """Test creating a book with title and authors"""
    # Arrange
    title = "The Great Gatsby"
    authors = ["F. Scott Fitzgerald"]

    # Act
    book = BookMetadata(title, authors)

    # Assert
    assert book.title == "The Great Gatsby"
    assert book.authors == ["F. Scott Fitzgerald"]
    assert book.rating is None  # Default value

def test_rating_validation():
    """Test that invalid ratings raise ValueError"""
    book = BookMetadata("Title", ["Author"])

    # Valid ratings
    book.set_rating(0)    # ✅ OK
    book.set_rating(5)    # ✅ OK
    book.set_rating(3.5)  # ✅ OK

    # Invalid ratings
    with pytest.raises(ValueError, match="0-5"):
        book.set_rating(-1)  # ❌ Too low

    with pytest.raises(ValueError, match="0-5"):
        book.set_rating(6)   # ❌ Too high
```

**💭 Jest Comparison**:
```javascript
// test_book_metadata.test.js
import { BookMetadata } from './book_metadata'

describe('BookMetadata', () => {
  test('creates book with title and authors', () => {
    // Arrange
    const title = "The Great Gatsby"
    const authors = ["F. Scott Fitzgerald"]

    // Act
    const book = new BookMetadata(title, authors)

    // Assert
    expect(book.title).toBe("The Great Gatsby")
    expect(book.authors).toEqual(["F. Scott Fitzgerald"])
    expect(book.rating).toBeNull()
  })

  test('validates rating range', () => {
    const book = new BookMetadata("Title", ["Author"])

    // Valid
    expect(() => book.setRating(0)).not.toThrow()
    expect(() => book.setRating(5)).not.toThrow()

    // Invalid
    expect(() => book.setRating(-1)).toThrow("0-5")
    expect(() => book.setRating(6)).toThrow("0-5")
  })
})
```

Very similar syntax!

### Test Discovery

pytest automatically finds tests:

```
src/calibre/
├─ test/              # Test directory
│  ├─ __init__.py
│  ├─ test_metadata.py    # Tests for metadata module
│  ├─ test_database.py    # Tests for database
│  └─ test_conversion.py  # Tests for conversion
```

**Naming conventions:**
- Test files: `test_*.py` or `*_test.py`
- Test functions: `test_*`
- Test classes: `Test*`

---

## Unit Testing

### Testing Pure Functions

```python
# src/calibre/utils/date.py
from datetime import datetime

def parse_date(date_string):
    """Parse various date formats"""
    formats = [
        '%Y-%m-%d',
        '%d/%m/%Y',
        '%m/%d/%Y',
    ]

    for fmt in formats:
        try:
            return datetime.strptime(date_string, fmt)
        except ValueError:
            continue

    raise ValueError(f"Could not parse date: {date_string}")

# test_date.py
import pytest
from datetime import datetime
from calibre.utils.date import parse_date

def test_parse_date_iso_format():
    """Test parsing ISO date format"""
    result = parse_date('2025-11-19')
    assert result == datetime(2025, 11, 19)

def test_parse_date_european_format():
    """Test parsing European format"""
    result = parse_date('19/11/2025')
    assert result == datetime(2025, 11, 19)

def test_parse_date_american_format():
    """Test parsing American format"""
    result = parse_date('11/19/2025')
    assert result == datetime(2025, 11, 19)

def test_parse_date_invalid():
    """Test that invalid dates raise ValueError"""
    with pytest.raises(ValueError, match="Could not parse"):
        parse_date('not a date')

# Parametrized tests (test multiple inputs)
@pytest.mark.parametrize('date_string,expected', [
    ('2025-11-19', datetime(2025, 11, 19)),
    ('19/11/2025', datetime(2025, 11, 19)),
    ('11/19/2025', datetime(2025, 11, 19)),
])
def test_parse_date_formats(date_string, expected):
    """Test parsing various date formats"""
    assert parse_date(date_string) == expected
```

**💭 Jest Comparison**:
```javascript
describe('parseDate', () => {
  test.each([
    ['2025-11-19', new Date(2025, 10, 19)],
    ['19/11/2025', new Date(2025, 10, 19)],
    ['11/19/2025', new Date(2025, 10, 19)],
  ])('parses %s correctly', (dateString, expected) => {
    expect(parseDate(dateString)).toEqual(expected)
  })
})
```

### Testing Classes

```python
# test_metadata.py
import pytest
from calibre.ebooks.metadata.book.base import BookMetadata

class TestBookMetadata:
    """Test suite for BookMetadata class"""

    def test_initialization(self):
        """Test creating a book"""
        book = BookMetadata('Title', ['Author'])
        assert book.title == 'Title'
        assert book.authors == ['Author']

    def test_add_author(self):
        """Test adding authors"""
        book = BookMetadata('Title', ['Author 1'])
        book.add_author('Author 2')
        assert len(book.authors) == 2
        assert 'Author 2' in book.authors

    def test_add_duplicate_author(self):
        """Test that duplicate authors are ignored"""
        book = BookMetadata('Title', ['Author 1'])
        book.add_author('Author 1')  # Duplicate
        assert len(book.authors) == 1

    def test_rating_property(self):
        """Test rating getter/setter"""
        book = BookMetadata('Title', ['Author'])

        # Default is None
        assert book.rating is None

        # Set valid rating
        book.set_rating(4.5)
        assert book.rating == 4.5

        # Invalid rating
        with pytest.raises(ValueError):
            book.set_rating(10)
```

---

## Integration Testing

### Testing Database Operations

```python
# test_database_integration.py
import pytest
import tempfile
import os
from calibre.db.cache import Cache

@pytest.fixture
def test_db():
    """Create temporary test database"""
    # Setup: Create temp database
    fd, db_path = tempfile.mkstemp(suffix='.db')
    os.close(fd)

    db = Cache(db_path)
    db.initialize()  # Create tables

    yield db  # Test runs here

    # Teardown: Clean up
    db.close()
    os.unlink(db_path)

def test_add_and_retrieve_book(test_db):
    """Test adding and retrieving a book"""
    # Add book
    book_id = test_db.add_book({
        'title': 'Test Book',
        'authors': ['Test Author'],
        'rating': 4.5
    })

    assert book_id > 0

    # Retrieve book
    metadata = test_db.get_metadata(book_id)
    assert metadata.title == 'Test Book'
    assert metadata.rating == 4.5

    # Check authors
    authors = test_db.authors(book_id)
    assert authors == ['Test Author']

def test_update_book_metadata(test_db):
    """Test updating book metadata"""
    # Add book
    book_id = test_db.add_book({'title': 'Original Title'})

    # Update
    test_db.set_metadata(book_id, {'title': 'New Title', 'rating': 5})

    # Verify
    metadata = test_db.get_metadata(book_id)
    assert metadata.title == 'New Title'
    assert metadata.rating == 5

def test_delete_book(test_db):
    """Test deleting a book"""
    # Add book
    book_id = test_db.add_book({'title': 'To Delete'})
    assert book_id in test_db.all_book_ids()

    # Delete
    test_db.delete_book(book_id)

    # Verify deleted
    assert book_id not in test_db.all_book_ids()
```

**💭 React/Prisma Testing**:
```javascript
describe('Database Integration', () => {
  let db

  beforeEach(async () => {
    db = await createTestDatabase()
  })

  afterEach(async () => {
    await db.cleanup()
  })

  test('adds and retrieves book', async () => {
    const bookId = await db.book.create({
      data: { title: 'Test Book', rating: 4.5 }
    })

    const book = await db.book.findUnique({ where: { id: bookId } })
    expect(book.title).toBe('Test Book')
    expect(book.rating).toBe(4.5)
  })
})
```

---

## GUI Testing

### Testing PyQt6 Widgets

**🔗 Code Reference**: [pytest-qt documentation](https://pytest-qt.readthedocs.io/)

```python
# test_widgets.py
import pytest
from PyQt6.QtCore import Qt
from PyQt6.QtWidgets import QPushButton, QLineEdit
from calibre.gui2.widgets import StarRatingWidget

@pytest.fixture
def qtbot(qtbot):
    """pytest-qt fixture for testing Qt widgets"""
    return qtbot

def test_button_click(qtbot):
    """Test button click signal"""
    button = QPushButton('Click Me')

    # Track clicks
    clicks = []
    button.clicked.connect(lambda: clicks.append(1))

    # Simulate click
    qtbot.mouseClick(button, Qt.MouseButton.LeftButton)

    # Verify
    assert len(clicks) == 1

def test_line_edit_input(qtbot):
    """Test text input"""
    line_edit = QLineEdit()

    # Type text
    qtbot.keyClicks(line_edit, 'Hello World')

    # Verify
    assert line_edit.text() == 'Hello World'

def test_star_rating_widget(qtbot):
    """Test custom star rating widget"""
    widget = StarRatingWidget(initial_rating=3.0)

    # Verify initial state
    assert widget.rating == 3.0

    # Track rating changes
    new_ratings = []
    widget.ratingChanged.connect(lambda r: new_ratings.append(r))

    # Click 5th star
    widget.set_rating(5.0)

    # Verify
    assert widget.rating == 5.0
    assert new_ratings == [5.0]

def test_widget_visibility(qtbot):
    """Test widget show/hide"""
    widget = QLineEdit()

    # Initially hidden
    assert not widget.isVisible()

    # Show widget
    widget.show()
    qtbot.waitExposed(widget)  # Wait for widget to appear

    # Verify visible
    assert widget.isVisible()
```

**💭 React Testing Library Comparison**:
```javascript
import { render, screen, fireEvent } from '@testing-library/react'

test('button click', () => {
  const handleClick = jest.fn()
  render(<button onClick={handleClick}>Click Me</button>)

  fireEvent.click(screen.getByText('Click Me'))

  expect(handleClick).toHaveBeenCalledTimes(1)
})

test('text input', () => {
  render(<input />)
  const input = screen.getByRole('textbox')

  fireEvent.change(input, { target: { value: 'Hello World' } })

  expect(input.value).toBe('Hello World')
})
```

---

## API Testing

### Testing HTTP Endpoints

```python
# test_api.py
import pytest
import json
from calibre.srv.routes import Router
from calibre.srv.errors import HTTPNotFound

@pytest.fixture
def test_router(test_db):
    """Create test router with test database"""
    router = Router(test_db)
    return router

def test_get_book_endpoint(test_router, test_db):
    """Test GET /api/book/{id}"""
    # Add test book
    book_id = test_db.add_book({'title': 'Test Book', 'rating': 4.5})

    # Make request
    response = test_router.handle_request('GET', f'/api/book/{book_id}')

    # Verify response
    assert response.status_code == 200
    data = json.loads(response.body)
    assert data['title'] == 'Test Book'
    assert data['rating'] == 4.5

def test_get_nonexistent_book(test_router):
    """Test GET /api/book/{id} with invalid ID"""
    with pytest.raises(HTTPNotFound):
        test_router.handle_request('GET', '/api/book/99999')

def test_update_rating_endpoint(test_router, test_db):
    """Test POST /api/book/{id}/rating"""
    # Add book
    book_id = test_db.add_book({'title': 'Test Book'})

    # Update rating
    response = test_router.handle_request(
        'POST',
        f'/api/book/{book_id}/rating',
        body=json.dumps({'rating': 5.0})
    )

    # Verify response
    assert response.status_code == 200

    # Verify database updated
    metadata = test_db.get_metadata(book_id)
    assert metadata.rating == 5.0

def test_rating_validation(test_router, test_db):
    """Test that invalid ratings are rejected"""
    book_id = test_db.add_book({'title': 'Test Book'})

    # Invalid rating (too high)
    response = test_router.handle_request(
        'POST',
        f'/api/book/{book_id}/rating',
        body=json.dumps({'rating': 10.0})
    )

    assert response.status_code == 400
    assert 'error' in json.loads(response.body)
```

**💭 Express/Supertest Comparison**:
```javascript
import request from 'supertest'
import app from './app'

describe('API Endpoints', () => {
  test('GET /api/book/:id', async () => {
    const book = await db.book.create({ data: { title: 'Test Book' } })

    const response = await request(app)
      .get(`/api/book/${book.id}`)
      .expect(200)

    expect(response.body.title).toBe('Test Book')
  })

  test('POST /api/book/:id/rating', async () => {
    const book = await db.book.create({ data: { title: 'Test Book' } })

    await request(app)
      .post(`/api/book/${book.id}/rating`)
      .send({ rating: 5.0 })
      .expect(200)

    const updated = await db.book.findUnique({ where: { id: book.id } })
    expect(updated.rating).toBe(5.0)
  })
})
```

---

## Database Testing

### Testing Transactions

```python
def test_transaction_rollback(test_db):
    """Test that failed transactions rollback"""
    # Add initial book
    book_id = test_db.add_book({'title': 'Original'})

    try:
        # Start transaction
        test_db.begin_transaction()

        # Make changes
        test_db.set_metadata(book_id, {'title': 'Changed'})

        # Simulate error
        raise Exception("Something went wrong")

        test_db.commit()  # Never reached
    except Exception:
        test_db.rollback()

    # Verify rollback worked
    metadata = test_db.get_metadata(book_id)
    assert metadata.title == 'Original'  # Unchanged

def test_transaction_commit(test_db):
    """Test that successful transactions commit"""
    book_id = test_db.add_book({'title': 'Original'})

    # Transaction
    test_db.begin_transaction()
    test_db.set_metadata(book_id, {'title': 'Changed'})
    test_db.commit()

    # Verify committed
    metadata = test_db.get_metadata(book_id)
    assert metadata.title == 'Changed'
```

### Testing Queries

```python
def test_search_books_by_title(test_db):
    """Test full-text search"""
    # Add books
    test_db.add_book({'title': 'Quantum Physics'})
    test_db.add_book({'title': 'Classical Mechanics'})
    test_db.add_book({'title': 'Quantum Mechanics'})

    # Search
    results = test_db.search('quantum')

    # Verify
    assert len(results) == 2
    titles = [test_db.get_metadata(id).title for id in results]
    assert 'Quantum Physics' in titles
    assert 'Quantum Mechanics' in titles

def test_filter_by_rating(test_db):
    """Test filtering books by rating"""
    # Add books with different ratings
    test_db.add_book({'title': 'Great Book', 'rating': 5.0})
    test_db.add_book({'title': 'Good Book', 'rating': 4.0})
    test_db.add_book({'title': 'OK Book', 'rating': 3.0})

    # Filter high-rated books
    results = test_db.books_with_rating_gte(4.0)

    # Verify
    assert len(results) == 2
```

---

## Fixtures and Mocking

### pytest Fixtures

**Fixtures provide test data and setup:**

```python
# conftest.py (shared fixtures)
import pytest
import tempfile
import os

@pytest.fixture
def temp_dir():
    """Create temporary directory"""
    dirpath = tempfile.mkdtemp()
    yield dirpath
    # Cleanup
    import shutil
    shutil.rmtree(dirpath)

@pytest.fixture
def sample_book():
    """Sample book metadata"""
    return {
        'title': 'The Great Gatsby',
        'authors': ['F. Scott Fitzgerald'],
        'rating': 4.5,
        'pubdate': '1925-04-10'
    }

@pytest.fixture
def test_db_with_books(test_db, sample_book):
    """Database pre-populated with books"""
    # Add sample books
    for i in range(10):
        test_db.add_book({
            **sample_book,
            'title': f'Book {i}'
        })

    return test_db

# Usage
def test_with_fixtures(test_db_with_books, sample_book):
    """Test using multiple fixtures"""
    all_books = test_db_with_books.all_book_ids()
    assert len(all_books) == 10

    # Can also use sample_book fixture
    assert sample_book['rating'] == 4.5
```

**Fixture Scopes:**

```python
@pytest.fixture(scope='function')  # Default: new for each test
def per_test_fixture():
    return setup()

@pytest.fixture(scope='module')  # Shared across module
def per_module_fixture():
    return expensive_setup()

@pytest.fixture(scope='session')  # Shared across all tests
def per_session_fixture():
    return very_expensive_setup()
```

### Mocking with unittest.mock

```python
from unittest.mock import Mock, patch, MagicMock

def test_with_mock():
    """Test using mock objects"""
    # Create mock database
    mock_db = Mock()
    mock_db.get_metadata.return_value = {
        'title': 'Mocked Book',
        'rating': 5.0
    }

    # Use mock
    result = mock_db.get_metadata(42)

    # Verify
    assert result['title'] == 'Mocked Book'
    mock_db.get_metadata.assert_called_once_with(42)

def test_with_patch():
    """Test using patch decorator"""
    with patch('calibre.db.cache.Cache') as MockCache:
        # Configure mock
        mock_instance = MockCache.return_value
        mock_instance.all_book_ids.return_value = [1, 2, 3]

        # Code under test uses mocked Cache
        cache = Cache('/fake/path')
        books = cache.all_book_ids()

        # Verify
        assert books == [1, 2, 3]

@patch('calibre.utils.date.datetime')
def test_with_patch_decorator(mock_datetime):
    """Test using patch as decorator"""
    # Mock current time
    mock_datetime.now.return_value = datetime(2025, 11, 19, 10, 30)

    # Code under test
    from calibre.utils.date import get_current_time
    current = get_current_time()

    # Verify
    assert current == datetime(2025, 11, 19, 10, 30)
```

**💭 Jest Mock Comparison**:
```javascript
// Jest mocking
test('with mock', () => {
  const mockDb = {
    getMetadata: jest.fn().mockReturnValue({
      title: 'Mocked Book',
      rating: 5.0
    })
  }

  const result = mockDb.getMetadata(42)

  expect(result.title).toBe('Mocked Book')
  expect(mockDb.getMetadata).toHaveBeenCalledWith(42)
})

// Module mocking
jest.mock('./database')
import { Cache } from './database'

test('with module mock', () => {
  Cache.mockImplementation(() => ({
    allBookIds: () => [1, 2, 3]
  }))

  const cache = new Cache('/fake/path')
  expect(cache.allBookIds()).toEqual([1, 2, 3])
})
```

---

## Test Coverage

### Running Coverage Reports

```bash
# Run tests with coverage
pytest --cov=calibre --cov-report=html --cov-report=term

# Output:
# Name                     Stmts   Miss  Cover
# --------------------------------------------
# calibre/__init__.py         10      0   100%
# calibre/db/cache.py        250     15    94%
# calibre/srv/routes.py      180     25    86%
# --------------------------------------------
# TOTAL                     1500    125    92%

# Open HTML report
# open htmlcov/index.html
```

### Coverage Configuration

```ini
# .coveragerc or pyproject.toml
[coverage:run]
source = calibre
omit =
    */tests/*
    */test_*.py
    */migrations/*
    */setup.py

[coverage:report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:
    if TYPE_CHECKING:
```

### Coverage Goals

**Calibre's Target:**
- **Overall:** 80%+ coverage
- **Core modules:** 90%+ coverage
- **New code:** 100% coverage required

**What to Cover:**
- ✅ Business logic (100%)
- ✅ Database operations (95%+)
- ✅ API endpoints (90%+)
- ⚠️ GUI (50%+, manual testing)
- ⚠️ Error handling (branch coverage)

---

## Continuous Integration

### Running Tests in CI

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          pip install -e .
          pip install pytest pytest-cov pytest-qt

      - name: Run tests
        run: |
          pytest --cov=calibre --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
```

---

## Best Practices

### 1. Test Naming

```python
# ✅ GOOD: Descriptive names
def test_rating_validation_rejects_negative_values():
    pass

def test_search_returns_books_matching_query():
    pass

# ❌ BAD: Vague names
def test_1():
    pass

def test_rating():
    pass
```

### 2. Arrange-Act-Assert Pattern

```python
def test_add_book():
    # Arrange: Set up test data
    book_data = {'title': 'Test Book', 'rating': 4.5}

    # Act: Perform action
    book_id = db.add_book(book_data)

    # Assert: Verify outcome
    assert book_id > 0
    assert db.get_metadata(book_id).title == 'Test Book'
```

### 3. Test One Thing

```python
# ✅ GOOD: Tests one behavior
def test_rating_accepts_zero():
    book.set_rating(0)
    assert book.rating == 0

def test_rating_accepts_five():
    book.set_rating(5)
    assert book.rating == 5

# ❌ BAD: Tests multiple behaviors
def test_rating():
    book.set_rating(0)
    assert book.rating == 0
    book.set_rating(5)
    assert book.rating == 5
    book.set_rating(3.5)
    assert book.rating == 3.5
```

### 4. Use Fixtures for Setup

```python
# ✅ GOOD: Fixture handles setup
@pytest.fixture
def book_with_authors():
    return BookMetadata('Title', ['Author 1', 'Author 2'])

def test_author_count(book_with_authors):
    assert len(book_with_authors.authors) == 2

# ❌ BAD: Setup in every test
def test_author_count():
    book = BookMetadata('Title', ['Author 1', 'Author 2'])
    assert len(book.authors) == 2
```

### 5. Don't Test Implementation Details

```python
# ✅ GOOD: Test behavior
def test_search_finds_matching_books():
    results = db.search('python')
    titles = [db.get_metadata(id).title for id in results]
    assert 'Python Programming' in titles

# ❌ BAD: Test implementation
def test_search_uses_fts5():
    # Don't test internal SQL queries
    assert 'FTS5' in db._get_search_query()
```

### 6. Fast Tests

```python
# ✅ GOOD: Use in-memory database
@pytest.fixture
def fast_db():
    return Cache(':memory:')  # SQLite in-memory

# ❌ BAD: Use real files
@pytest.fixture
def slow_db():
    return Cache('/tmp/test.db')  # Slow file I/O
```

### 7. Isolated Tests

```python
# ✅ GOOD: Each test is independent
def test_add_book(test_db):
    book_id = test_db.add_book({'title': 'Book 1'})
    assert book_id == 1

def test_add_another_book(test_db):
    # Fresh database (doesn't see previous test's book)
    book_id = test_db.add_book({'title': 'Book 2'})
    assert book_id == 1

# ❌ BAD: Tests depend on each other
def test_add_book():
    global book_id
    book_id = db.add_book({'title': 'Book 1'})

def test_update_book():
    # Depends on previous test
    db.update_book(book_id, {'rating': 5})
```

---

## Summary

### Testing Checklist

**Before Committing:**
- [ ] All tests pass locally
- [ ] New code has tests (100% coverage)
- [ ] Tests are fast (< 1 minute total)
- [ ] Tests are isolated (no dependencies)
- [ ] Descriptive test names
- [ ] No skipped tests without reason

**Test Types:**
- [ ] Unit tests for business logic
- [ ] Integration tests for database
- [ ] API tests for endpoints
- [ ] GUI tests for critical workflows (optional)

**Quality Metrics:**
- [ ] Overall coverage ≥ 80%
- [ ] Core modules ≥ 90%
- [ ] New code = 100%

### pytest vs Jest Quick Reference

| Feature | pytest | Jest |
|---------|--------|------|
| **Test file** | `test_*.py` | `*.test.js` |
| **Test function** | `def test_*()` | `test('name', () => {})` |
| **Assertions** | `assert x == y` | `expect(x).toBe(y)` |
| **Fixtures** | `@pytest.fixture` | `beforeEach(() => {})` |
| **Mocking** | `unittest.mock` | `jest.mock()` |
| **Parametrize** | `@pytest.mark.parametrize` | `test.each([...])` |
| **Coverage** | `pytest --cov` | `jest --coverage` |

### Next Steps

1. Read existing tests in [src/calibre/test/](../../src/calibre/test/)
2. Try writing tests for exercises in [EXERCISES.md](EXERCISES.md)
3. Run test suite: `pytest`
4. Check coverage: `pytest --cov=calibre --cov-report=html`
5. See [DEBUGGING_GUIDE.md](DEBUGGING_GUIDE.md) for debugging failing tests

---

*Write tests. Sleep better. Ship confidently.* ✅
