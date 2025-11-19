# First Contributions: Your Path to Success

**Purpose:** Guide you from zero to your first merged Pull Request

**For:** React developers ready to contribute to Calibre

**Time to First PR:** 1-2 weeks (including learning time)

---

## Table of Contents

1. [Quick Wins: Start Here](#quick-wins)
2. [Good First Issues](#good-first-issues)
3. [Step-by-Step: Your First PR](#first-pr)
4. [Code Review Process](#code-review)
5. [Common Mistakes to Avoid](#mistakes)

---

<a name="quick-wins"></a>
## 1. Quick Wins: Start Here

### Level 1: Documentation (No Code!)

**Time: 1-2 hours | Difficulty: ⭐☆☆☆☆**

**Tasks:**
- Fix typos in docstrings
- Add missing docstrings
- Improve code comments
- Update README files

**Example:**

**File:** [src/calibre/db/cache.py](../../src/calibre/db/cache.py)

```python
# Before (missing docstring)
def all_book_ids(self):
    return list(self._metadata_cache.keys())

# After (add docstring)
def all_book_ids(self):
    '''
    Get IDs of all books in the library.

    Returns:
        list[int]: List of book IDs in no particular order

    Example:
        >>> db.all_book_ids()
        [1, 2, 3, 42, 87, ...]
    '''
    return list(self._metadata_cache.keys())
```

**How to Find:**
```bash
# Find functions without docstrings
grep -r "def [a-z_]*(" src/calibre/db/ | grep -v "'''" | head -20
```

**Submit:**
1. Make changes
2. Test: `python src/calibre/debug.py`
3. Commit: `git commit -m "docs: add docstrings to cache.py methods"`
4. Create PR

---

### Level 2: Add Type Hints

**Time: 2-3 hours | Difficulty: ⭐⭐☆☆☆**

**Goal:** Add Python type hints to improve IDE autocomplete

**Example:**

**File:** [src/calibre/db/cache.py](../../src/calibre/db/cache.py)

```python
# Before (no types)
def get_metadata(self, book_id):
    return self._metadata_cache.get(book_id)

# After (with types)
from typing import Optional, Dict, Any

def get_metadata(self, book_id: int) -> Optional[Dict[str, Any]]:
    '''
    Get book metadata by ID.

    Args:
        book_id: The database ID of the book

    Returns:
        Dictionary of metadata if book exists, None otherwise
    '''
    return self._metadata_cache.get(book_id)
```

**Benefits:**
- Better IDE autocomplete
- Catch type errors early
- Easier for new developers

**How to Find:**
```bash
# Find files with few type hints
find src/calibre/db -name "*.py" -exec grep -L "from typing import" {} \;
```

---

### Level 3: Add Unit Tests

**Time: 3-5 hours | Difficulty: ⭐⭐⭐☆☆**

**Goal:** Test an existing function

**Example:**

**File:** `src/calibre/db/tests/test_search.py`

```python
import unittest
from calibre.db.tests.base import BaseTest

class SearchTest(BaseTest):
    '''Test search functionality'''

    def test_search_by_title(self):
        '''Test searching books by title'''

        # Setup: Create test database
        db = self.init_db()

        # Add test books
        book1 = db.create_book_entry({
            'title': '1984',
            'authors': ['George Orwell']
        })
        book2 = db.create_book_entry({
            'title': 'Animal Farm',
            'authors': ['George Orwell']
        })
        book3 = db.create_book_entry({
            'title': 'Brave New World',
            'authors': ['Aldous Huxley']
        })

        # Test: Search by title
        results = db.search('title:1984')

        # Verify: Only book1 matches
        self.assertEqual(len(results), 1)
        self.assertIn(book1, results)
        self.assertNotIn(book2, results)
        self.assertNotIn(book3, results)

    def test_search_by_author(self):
        '''Test searching books by author'''

        db = self.init_db()

        # Add books...
        book1 = db.create_book_entry({
            'title': '1984',
            'authors': ['George Orwell']
        })
        book2 = db.create_book_entry({
            'title': 'Animal Farm',
            'authors': ['George Orwell']
        })

        # Search
        results = db.search('author:orwell')

        # Verify both books found
        self.assertEqual(len(results), 2)
        self.assertIn(book1, results)
        self.assertIn(book2, results)

if __name__ == '__main__':
    unittest.main()
```

**Run Tests:**
```bash
# Run your new test
python -m pytest src/calibre/db/tests/test_search.py -v

# Expected output:
# test_search_by_title ... ok
# test_search_by_author ... ok
```

---

<a name="good-first-issues"></a>
## 2. Good First Issues

### Beginner-Friendly Tasks

#### 1. Add API Endpoint for Book Count

**Difficulty: ⭐⭐☆☆☆ | Time: 2-3 hours**

**Goal:** Add `GET /api/stats/count` endpoint

**File:** [src/calibre/srv/ajax.py](../../src/calibre/srv/ajax.py)

```python
@endpoint('/api/stats/count', auth_required=False)
def api_stats_count(ctx, rd):
    '''
    Get library statistics.

    Returns:
    {
      "total_books": 1234,
      "total_authors": 456,
      "total_tags": 78
    }
    '''
    db = ctx.db

    return {
        'total_books': len(db.all_book_ids()),
        'total_authors': len(db.all_author_ids()),
        'total_tags': len(db.all_tag_ids())
    }
```

**Test:**
```bash
curl http://localhost:8080/api/stats/count
# {"total_books": 100, "total_authors": 45, "total_tags": 32}
```

**Why Good for Beginners:**
- Simple endpoint (no complex logic)
- Uses existing DB methods
- Easy to test
- Useful feature!

---

#### 2. Add Search Filter for Recently Added

**Difficulty: ⭐⭐⭐☆☆ | Time: 4-5 hours**

**Goal:** Add `recent:` search filter

**Example:** `recent:7d` = books added in last 7 days

**File:** [src/calibre/db/search.py](../../src/calibre/db/search.py)

```python
def parse_recent_filter(self, value):
    '''
    Parse recent: filter.

    Examples:
    - recent:7d  → Last 7 days
    - recent:1w  → Last 1 week
    - recent:1m  → Last 1 month
    '''
    import re
    from datetime import datetime, timedelta

    # Parse value (e.g., "7d", "1w", "1m")
    match = re.match(r'(\d+)([dwm])', value)
    if not match:
        raise ValueError(f'Invalid recent filter: {value}')

    amount = int(match.group(1))
    unit = match.group(2)

    # Calculate cutoff date
    now = datetime.now()
    if unit == 'd':  # days
        cutoff = now - timedelta(days=amount)
    elif unit == 'w':  # weeks
        cutoff = now - timedelta(weeks=amount)
    elif unit == 'm':  # months (approximate)
        cutoff = now - timedelta(days=amount * 30)

    # Find books added after cutoff
    results = set()
    for book_id in self.db.all_book_ids():
        timestamp = self.db.field_for('timestamp', book_id)
        if timestamp >= cutoff:
            results.add(book_id)

    return results
```

**Test:**
```python
# Search for books added in last week
recent_books = db.search('recent:7d')

# Combine with other filters
scifi_recent = db.search('tags:scifi AND recent:1m')
```

**Why Good:**
- Real feature users want
- Teaches search system
- Involves date handling
- More challenging but still manageable

---

#### 3. Improve Error Messages

**Difficulty: ⭐⭐☆☆☆ | Time: 2-3 hours**

**Goal:** Make error messages more helpful

**Before:**
```python
# File: src/calibre/db/cache.py
def get_metadata(self, book_id):
    if book_id not in self._metadata_cache:
        raise ValueError('Book not found')  # ← Not helpful!
    return self._metadata_cache[book_id]
```

**After:**
```python
def get_metadata(self, book_id):
    if book_id not in self._metadata_cache:
        # ✅ Much better error message!
        total_books = len(self._metadata_cache)
        raise ValueError(
            f'Book with ID {book_id} not found. '
            f'Library has {total_books} books with IDs {min(self._metadata_cache)}-{max(self._metadata_cache)}. '
            f'Did you mean to use search() instead?'
        )
    return self._metadata_cache[book_id]
```

**Why Good:**
- Helps users and developers
- Easy to implement
- Improves codebase quality
- Low risk of breaking things

---

<a name="first-pr"></a>
## 3. Step-by-Step: Your First PR

### Complete Walkthrough: Add "Last Read" Field

Let's walk through adding a "last read date" feature from start to finish.

#### Step 1: Set Up Dev Environment

```bash
# Clone repository
git clone https://github.com/kovidgoyal/calibre.git
cd calibre

# Create feature branch
git checkout -b feature/add-last-read-date

# Install dependencies
pip install -e .

# Build
python setup.py build
```

#### Step 2: Make Changes

**A. Add Database Field**

**File:** [src/calibre/db/fields.py](../../src/calibre/db/fields.py)

```python
# Add to standard fields
'last_read': {
    'datatype': 'datetime',
    'label': 'last_read',
    'name': 'Last Read',
    'display': {},
    'is_custom': False
}
```

**B. Add Database Column (Migration)**

**File:** [src/calibre/db/legacy.py](../../src/calibre/db/legacy.py)

```python
def upgrade_schema(conn):
    '''Add last_read column to books table'''

    # Check if column exists
    cursor = conn.execute("PRAGMA table_info(books)")
    columns = [row[1] for row in cursor]

    if 'last_read' not in columns:
        # Add column
        conn.execute('''
            ALTER TABLE books
            ADD COLUMN last_read TEXT DEFAULT NULL
        ''')
        conn.commit()
```

**C. Add API Method**

**File:** [src/calibre/db/cache.py](../../src/calibre/db/cache.py)

```python
@write_api
def set_last_read(self, book_id, date=None):
    '''
    Set the last read date for a book.

    Args:
        book_id: Book ID
        date: datetime object, or None for now
    '''
    from datetime import datetime

    if date is None:
        date = datetime.now()

    # Update cache
    self._metadata_cache[book_id]['last_read'] = date

    # Update database
    with self.write_lock:
        self.backend.set_field('last_read', {book_id: date})

@read_api
def get_last_read(self, book_id):
    '''Get last read date for a book'''
    return self._metadata_cache[book_id].get('last_read')
```

**D. Add Web Endpoint**

**File:** [src/calibre/srv/ajax.py](../../src/calibre/srv/ajax.py)

```python
@endpoint('/api/book/{book_id}/last-read', methods=('GET', 'POST'))
def api_book_last_read(ctx, rd, book_id):
    '''Get or set last read date'''

    db = ctx.db

    if rd.method == 'GET':
        # Get last read date
        last_read = db.get_last_read(book_id)
        return {
            'book_id': book_id,
            'last_read': last_read.isoformat() if last_read else None
        }

    elif rd.method == 'POST':
        # Mark as read now
        from datetime import datetime
        db.set_last_read(book_id, datetime.now())

        return {
            'success': True,
            'last_read': datetime.now().isoformat()
        }
```

#### Step 3: Add Tests

**File:** `src/calibre/db/tests/test_last_read.py`

```python
import unittest
from datetime import datetime, timedelta
from calibre.db.tests.base import BaseTest

class LastReadTest(BaseTest):
    '''Test last read functionality'''

    def test_set_last_read(self):
        '''Test setting last read date'''

        db = self.init_db()

        # Add book
        book_id = db.create_book_entry({'title': 'Test Book'})

        # Set last read
        now = datetime.now()
        db.set_last_read(book_id, now)

        # Verify
        last_read = db.get_last_read(book_id)
        self.assertIsNotNone(last_read)
        self.assertEqual(last_read.date(), now.date())

    def test_last_read_defaults_to_none(self):
        '''Test that last_read is None for new books'''

        db = self.init_db()
        book_id = db.create_book_entry({'title': 'New Book'})

        last_read = db.get_last_read(book_id)
        self.assertIsNone(last_read)

if __name__ == '__main__':
    unittest.main()
```

#### Step 4: Run Tests

```bash
# Run your new tests
python -m pytest src/calibre/db/tests/test_last_read.py -v

# Run all tests to make sure nothing broke
python setup.py test
```

#### Step 5: Test Manually

```bash
# Start server
calibre-server ~/test-library

# Test GET endpoint
curl http://localhost:8080/api/book/1/last-read
# {"book_id": 1, "last_read": null}

# Test POST endpoint (mark as read)
curl -X POST http://localhost:8080/api/book/1/last-read
# {"success": true, "last_read": "2025-11-19T10:30:00"}

# Verify it was saved
curl http://localhost:8080/api/book/1/last-read
# {"book_id": 1, "last_read": "2025-11-19T10:30:00"}
```

#### Step 6: Commit Changes

```bash
# Stage your changes
git add src/calibre/db/fields.py
git add src/calibre/db/cache.py
git add src/calibre/srv/ajax.py
git add src/calibre/db/tests/test_last_read.py

# Commit with good message
git commit -m "feat: add last read date tracking

- Add last_read field to database schema
- Add set_last_read() and get_last_read() methods to Cache
- Add /api/book/{id}/last-read endpoint (GET/POST)
- Add comprehensive unit tests

Allows users to track when they last read a book.
Useful for reading statistics and recommendations."
```

#### Step 7: Push and Create PR

```bash
# Push to your fork
git push -u origin feature/add-last-read-date

# Go to GitHub and create Pull Request
# Title: "Add last read date tracking"
# Description: See template below
```

**PR Description Template:**

```markdown
## Summary

Adds ability to track when a book was last read.

## Changes

- Added `last_read` field to database schema
- Added `set_last_read()` and `get_last_read()` API methods
- Added `/api/book/{id}/last-read` REST endpoint
- Added comprehensive unit tests

## Testing

- ✅ All existing tests pass
- ✅ New unit tests added and passing
- ✅ Manual testing via curl
- ✅ Tested with 10,000 book library

## Screenshots

(If UI changes, add screenshots)

## Checklist

- [x] Code follows project style guide
- [x] Tests added and passing
- [x] Documentation updated (docstrings)
- [x] No breaking changes
- [x] Tested manually

## Related Issues

Closes #12345 (if applicable)
```

---

<a name="code-review"></a>
## 4. Code Review Process

### What to Expect

1. **Automated Checks** (immediate)
   - Code style (Ruff)
   - Tests pass
   - No Python errors

2. **Maintainer Review** (1-7 days)
   - Code quality
   - Architecture fit
   - Tests adequate

3. **Requested Changes** (maybe)
   - "Please add more tests"
   - "Can you handle this edge case?"
   - "Please update docstrings"

4. **Approval & Merge** (after changes)
   - "LGTM!" (Looks Good To Me)
   - Merged into main branch
   - Included in next release

### Responding to Feedback

**Good Response:**
```markdown
Thanks for the feedback! I've made the following changes:

1. Added edge case handling for None values
2. Expanded tests to cover timezones
3. Updated docstrings as suggested

Ready for another review!
```

**How to Make Changes:**

```bash
# Make requested changes
vim src/calibre/db/cache.py

# Amend previous commit
git add src/calibre/db/cache.py
git commit --amend --no-edit

# Force push (updates PR)
git push -f origin feature/add-last-read-date
```

---

<a name="mistakes"></a>
## 5. Common Mistakes to Avoid

### ❌ Don't: Change Unrelated Code

```python
# Your feature: Add last read date
def set_last_read(self, book_id, date):
    ...

# ❌ DON'T: Refactor unrelated code in same PR
def get_metadata(self, book_id):
    # Refactored this function (unrelated to your feature)
    ...
```

**Why:** Makes review harder, mixes concerns

**Do Instead:** Separate PRs for separate concerns

---

### ❌ Don't: Skip Tests

**Without tests:**
```python
# Added new feature
def calculate_reading_time(self, book_id):
    pages = self.get_page_count(book_id)
    return pages * 2  # 2 minutes per page
```

**Why:** How do we know it works? What if pages is None?

**Do Instead:**
```python
def test_calculate_reading_time(self):
    db = self.init_db()
    book_id = db.create_book_entry({'title': 'Test', 'pages': 300})

    reading_time = db.calculate_reading_time(book_id)

    self.assertEqual(reading_time, 600)  # 300 * 2
```

---

### ❌ Don't: Forget Type Hints

```python
# ❌ No types
def get_books(query):
    return search(query)

# ✅ With types
from typing import List, Dict, Any

def get_books(query: str) -> List[Dict[str, Any]]:
    '''
    Search books by query.

    Args:
        query: Search query string

    Returns:
        List of book metadata dicts
    '''
    return search(query)
```

---

### ❌ Don't: Have Huge Commits

```bash
# ❌ One commit with 50 files changed
git commit -m "Add features"

# ✅ Multiple focused commits
git commit -m "feat: add last_read field to schema"
git commit -m "feat: add Cache methods for last_read"
git commit -m "feat: add API endpoint for last_read"
git commit -m "test: add tests for last_read feature"
```

---

## 🎯 Success Checklist

Before submitting your PR:

- [ ] Code works locally
- [ ] All tests pass (`python setup.py test`)
- [ ] Added new tests for your changes
- [ ] Added/updated docstrings
- [ ] Added type hints
- [ ] Followed code style (run `ruff check`)
- [ ] No commented-out code
- [ ] No debugging print statements
- [ ] Commit messages are clear
- [ ] PR description explains WHY

---

## 📖 Next Steps

**After Your First PR:**

1. ✅ Celebrate! 🎉 You contributed to open source!
2. 📚 Read [CODE_TOURS.md](./CODE_TOURS.md) to understand more
3. 🔧 Try a harder issue (⭐⭐⭐☆☆)
4. 💬 Join the community forum
5. 🎯 Help others with their first PR!

**Recommended Path:**

1. Week 1: Documentation fixes (build confidence)
2. Week 2: Add type hints (learn codebase)
3. Week 3: Add unit tests (understand testing)
4. Week 4: Small feature (everything together!)

---

## 🤝 Getting Help

**Stuck? Ask!**

- **Forum:** [MobileRead Calibre](https://www.mobileread.com/forums/forumdisplay.php?f=166)
- **GitHub:** Comment on issue/PR
- **Documentation:** [docs/learning/](.)

**Good Questions:**
- "I'm trying to add X, should I modify Y or Z?"
- "How do I test this edge case?"
- "Which file should this go in?"

**Remember:** Everyone was a beginner once! Maintainers are friendly and want to help.

---

**You've got this! Start with a quick win and build from there.** 🚀
