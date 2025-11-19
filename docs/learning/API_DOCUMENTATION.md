# Calibre API Documentation

> **For React developers**: This guide covers both REST APIs (similar to Express/Next.js API routes) and internal Python APIs (similar to library APIs like Axios, React Query utilities). Calibre's HTTP server exposes a REST API for external access, while internal Python APIs provide programmatic control.

---

## Table of Contents

1. [Overview](#overview)
2. [Content Server REST API](#content-server-rest-api)
3. [Database API](#database-api)
4. [Metadata API](#metadata-api)
5. [Format Conversion API](#format-conversion-api)
6. [Plugin API](#plugin-api)
7. [GUI API](#gui-api)
8. [Utility APIs](#utility-apis)
9. [Creating Custom Endpoints](#creating-custom-endpoints)
10. [API Best Practices](#api-best-practices)

---

## Overview

Calibre has two types of APIs:

### 1. REST API (Content Server)

**For external clients** (web browsers, mobile apps, other services):
- HTTP/JSON based
- Defined in [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py)
- Accessed via `http://localhost:8080/api/...`
- Similar to Express.js or Next.js API routes

### 2. Internal Python API

**For plugins, scripts, and internal use**:
- Direct Python function calls
- Type-safe, no HTTP overhead
- Used by Calibre GUI and CLI tools
- Similar to importing library functions

---

## Content Server REST API

### Starting the Server

```bash
# Start content server
calibre-server --port 8080 --with-library /path/to/library

# Or in Python
from calibre.srv.standalone import create_server

server = create_server({
    'port': 8080,
    'library_dir': '/path/to/library'
})
server.start()
```

### Authentication

Most endpoints require authentication:

```python
import requests

# Login
response = requests.post('http://localhost:8080/login', json={
    'username': 'admin',
    'password': 'password'
})

# Get session cookie
session_id = response.cookies['session_id']

# Use session in subsequent requests
headers = {'Cookie': f'session_id={session_id}'}
response = requests.get('http://localhost:8080/api/books', headers=headers)
```

---

## REST API Endpoints

### Books Endpoints

#### GET /api/books

**List all books**

```bash
curl http://localhost:8080/api/books
```

**Response**:
```json
{
  "books": [
    {
      "id": 1,
      "title": "1984",
      "authors": ["George Orwell"],
      "rating": 4.5,
      "tags": ["Fiction", "Dystopian"],
      "formats": ["EPUB", "MOBI"],
      "pubdate": "1949-06-08"
    }
  ],
  "total": 1234
}
```

**Query parameters**:
- `limit`: Max results (default: 50)
- `offset`: Pagination offset (default: 0)
- `sort`: Sort field (`title`, `author`, `rating`, `pubdate`)
- `order`: Sort order (`asc`, `desc`)

**Example**:
```bash
curl "http://localhost:8080/api/books?limit=10&offset=20&sort=rating&order=desc"
```

#### GET /api/books/{book_id}

**Get single book**

```bash
curl http://localhost:8080/api/books/123
```

**Response**:
```json
{
  "id": 123,
  "title": "1984",
  "authors": ["George Orwell"],
  "author_sort": "Orwell, George",
  "rating": 4.5,
  "tags": ["Fiction", "Dystopian"],
  "series": "Classic Literature",
  "series_index": 1.0,
  "publisher": "Penguin Books",
  "pubdate": "1949-06-08",
  "isbn": "9780451524935",
  "formats": ["EPUB", "MOBI", "PDF"],
  "has_cover": true,
  "timestamp": "2024-01-15T10:30:00Z",
  "last_modified": "2024-01-20T14:45:00Z",
  "comments": "A dystopian novel...",
  "languages": ["eng"]
}
```

**Implementation**: [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py#L234)

#### POST /api/books

**Add new book**

```bash
curl -X POST http://localhost:8080/api/books \
  -H "Content-Type: application/json" \
  -d '{
    "title": "New Book",
    "authors": ["Author Name"],
    "tags": ["Tag1", "Tag2"]
  }'
```

**Response**:
```json
{
  "book_id": 1235,
  "status": "success"
}
```

#### PUT /api/books/{book_id}

**Update book metadata**

```bash
curl -X PUT http://localhost:8080/api/books/123 \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Updated Title",
    "rating": 5.0
  }'
```

**Response**:
```json
{
  "status": "success",
  "updated_fields": ["title", "rating"]
}
```

#### DELETE /api/books/{book_id}

**Delete book**

```bash
curl -X DELETE http://localhost:8080/api/books/123
```

**Response**:
```json
{
  "status": "success",
  "deleted_book_id": 123
}
```

### Search Endpoints

#### GET /api/search

**Search books**

```bash
curl "http://localhost:8080/api/search?q=title:1984+author:orwell"
```

**Query syntax** (similar to Calibre GUI):
- `title:1984` - Search title
- `author:orwell` - Search author
- `tag:fiction` - Search tags
- `rating:>=4` - Rating filter
- `series:true` - Books in series
- `formats:epub` - Has EPUB format

**Response**:
```json
{
  "results": [
    {"id": 123, "title": "1984", "authors": ["George Orwell"]}
  ],
  "total": 1,
  "query": "title:1984 author:orwell"
}
```

**Implementation**: [src/calibre/db/search.py](../../src/calibre/db/search.py)

### Format Endpoints

#### GET /api/books/{book_id}/formats

**List available formats**

```bash
curl http://localhost:8080/api/books/123/formats
```

**Response**:
```json
{
  "formats": [
    {
      "format": "EPUB",
      "size": 1234567,
      "path": "/library/Author/Book (123)/Book.epub"
    },
    {
      "format": "MOBI",
      "size": 2345678,
      "path": "/library/Author/Book (123)/Book.mobi"
    }
  ]
}
```

#### GET /api/books/{book_id}/{format}

**Download book in specific format**

```bash
curl http://localhost:8080/api/books/123/epub --output book.epub
```

**Response**: Binary file download

#### POST /api/books/{book_id}/formats

**Upload new format**

```bash
curl -X POST http://localhost:8080/api/books/123/formats \
  -F "format=epub" \
  -F "file=@book.epub"
```

**Response**:
```json
{
  "status": "success",
  "format": "EPUB",
  "size": 1234567
}
```

### Cover Endpoints

#### GET /api/books/{book_id}/cover

**Get book cover image**

```bash
curl http://localhost:8080/api/books/123/cover --output cover.jpg
```

**Query parameters**:
- `width`: Max width (default: original)
- `height`: Max height (default: original)

**Example**:
```bash
curl "http://localhost:8080/api/books/123/cover?width=300&height=400"
```

#### POST /api/books/{book_id}/cover

**Upload cover image**

```bash
curl -X POST http://localhost:8080/api/books/123/cover \
  -F "cover=@cover.jpg"
```

### Metadata Source Endpoints

#### GET /api/metadata/sources

**List metadata sources**

```bash
curl http://localhost:8080/api/metadata/sources
```

**Response**:
```json
{
  "sources": [
    {"id": "google", "name": "Google Books"},
    {"id": "amazon", "name": "Amazon"},
    {"id": "goodreads", "name": "Goodreads"}
  ]
}
```

#### POST /api/metadata/identify

**Identify book from metadata sources**

```bash
curl -X POST http://localhost:8080/api/metadata/identify \
  -H "Content-Type: application/json" \
  -d '{
    "title": "1984",
    "authors": ["George Orwell"],
    "isbn": "9780451524935"
  }'
```

**Response**:
```json
{
  "results": [
    {
      "source": "google",
      "title": "1984",
      "authors": ["George Orwell"],
      "isbn": "9780451524935",
      "publisher": "Penguin Books",
      "pubdate": "1949-06-08",
      "cover_url": "https://..."
    }
  ]
}
```

**Implementation**: [src/calibre/ebooks/metadata/sources/](../../src/calibre/ebooks/metadata/sources/)

### Conversion Endpoints

#### POST /api/convert

**Convert book format**

```bash
curl -X POST http://localhost:8080/api/convert \
  -H "Content-Type: application/json" \
  -d '{
    "book_id": 123,
    "input_format": "mobi",
    "output_format": "epub",
    "options": {
      "output_profile": "tablet",
      "remove_paragraph_spacing": true
    }
  }'
```

**Response**:
```json
{
  "job_id": "abc123",
  "status": "queued"
}
```

#### GET /api/convert/{job_id}

**Check conversion status**

```bash
curl http://localhost:8080/api/convert/abc123
```

**Response**:
```json
{
  "job_id": "abc123",
  "status": "completed",
  "progress": 100,
  "output_format": "epub",
  "download_url": "/api/books/123/epub"
}
```

---

## Database API

### Cache API

**Main database interface**: [src/calibre/db/cache.py](../../src/calibre/db/cache.py#L138)

```python
from calibre.db.cache import Cache

# Open database
cache = Cache('/path/to/library/metadata.db')

# Get all book IDs
all_ids = cache.all_book_ids()  # [1, 2, 3, ...]

# Get metadata for single book
book = cache.get_metadata(123)
print(book.title)       # "1984"
print(book.authors)     # ["George Orwell"]
print(book.rating)      # 4.5
print(book.tags)        # ["Fiction", "Dystopian"]

# Get metadata for multiple books
books = cache.all_book_ids()
metadata_list = [cache.get_metadata(book_id) for book_id in books[:10]]

# Search
from calibre.db.search import Search
search = Search(cache)
results = search.search('author:orwell')  # [123, 456, ...]

# Get field value
title = cache.field_for('title', 123)    # "1984"
authors = cache.field_for('authors', 123)  # ["George Orwell"]

# Get multiple field values
titles = cache.all_field_for('title', [123, 456, 789])
# {123: "1984", 456: "Animal Farm", 789: "Homage to Catalonia"}

# Close database
cache.close()
```

### Metadata Operations

```python
from calibre.ebooks.metadata.book.base import Metadata

# Create metadata object
metadata = Metadata('1984', ['George Orwell'])
metadata.rating = 4.5
metadata.tags = ['Fiction', 'Dystopian']
metadata.publisher = 'Penguin Books'
metadata.pubdate = datetime(1949, 6, 8)
metadata.isbn = '9780451524935'
metadata.comments = 'A dystopian novel...'

# Add book to database
book_id = cache.create_book_entry(metadata)

# Update book metadata
cache.set_metadata(book_id, metadata)

# Update specific field
cache.set_field('title', book_id, 'Nineteen Eighty-Four')
cache.set_field('rating', book_id, 5.0)
cache.set_field('tags', book_id, ['Fiction', 'Dystopian', 'Classic'])

# Remove book
cache.remove_books([book_id])
```

### Format Operations

```python
# Check if book has format
has_epub = cache.has_format(book_id, 'EPUB')

# Get format path
epub_path = cache.format_abspath(book_id, 'EPUB')
# /library/George Orwell/1984 (123)/1984.epub

# Get all formats for book
formats = cache.formats(book_id)  # ['EPUB', 'MOBI', 'PDF']

# Add format
with open('book.epub', 'rb') as f:
    cache.add_format(book_id, 'EPUB', f, replace=True)

# Remove format
cache.remove_format(book_id, 'EPUB')
```

### Cover Operations

```python
# Check if book has cover
has_cover = cache.has_cover(book_id)

# Get cover image data
cover_data = cache.cover(book_id)  # bytes

# Set cover
with open('cover.jpg', 'rb') as f:
    cover_data = f.read()
cache.set_cover(book_id, cover_data)

# Remove cover
cache.remove_cover(book_id)
```

### Search API

**Advanced search**: [src/calibre/db/search.py](../../src/calibre/db/search.py)

```python
from calibre.db.search import Search

search = Search(cache)

# Simple search
results = search.search('orwell')  # Search all fields
# [123, 456, 789]

# Field-specific search
results = search.search('title:1984')
results = search.search('author:orwell')
results = search.search('tag:fiction')

# Boolean operators
results = search.search('author:orwell AND tag:fiction')
results = search.search('author:orwell OR author:hemingway')
results = search.search('tag:fiction NOT tag:romance')

# Comparison operators
results = search.search('rating:>=4')
results = search.search('rating:>4.5')
results = search.search('pubdate:<2000')

# Regular expressions
results = search.search('title:"~^[A-Z]"')  # Starts with capital

# Series search
results = search.search('series:true')     # Has series
results = search.search('series:false')    # No series
results = search.search('series:="Lord of the Rings"')

# Format search
results = search.search('formats:epub')    # Has EPUB
results = search.search('formats:=epub')   # ONLY has EPUB

# Date ranges
results = search.search('pubdate:>2020-01-01')
results = search.search('timestamp:30daysago')

# Complex queries
results = search.search('''
    (author:orwell OR author:huxley)
    AND tag:dystopian
    AND rating:>=4
    AND formats:epub
''')
```

### Virtual Library API

```python
# Get all virtual libraries
vlibs = cache.pref('virtual_libraries', {})

# Create virtual library
vlibs['SciFi'] = 'tag:science-fiction'
cache.set_pref('virtual_libraries', vlibs)

# Use virtual library
cache.set_virtual_library('SciFi')
scifi_books = cache.all_book_ids()  # Only SciFi books

# Reset to all books
cache.set_virtual_library(None)
```

---

## Metadata API

### Metadata Class

**Core metadata class**: [src/calibre/ebooks/metadata/book/base.py](../../src/calibre/ebooks/metadata/book/base.py)

```python
from calibre.ebooks.metadata.book.base import Metadata
from datetime import datetime

# Create metadata
metadata = Metadata('Book Title')

# Basic fields
metadata.title = '1984'
metadata.authors = ['George Orwell']
metadata.author_sort = 'Orwell, George'
metadata.title_sort = '1984'

# Ratings and tags
metadata.rating = 4.5  # 0-5 scale
metadata.tags = ['Fiction', 'Dystopian']
metadata.series = 'Classic Literature'
metadata.series_index = 1.0

# Publishing info
metadata.publisher = 'Penguin Books'
metadata.pubdate = datetime(1949, 6, 8)
metadata.timestamp = datetime.now()
metadata.isbn = '9780451524935'
metadata.languages = ['eng']

# Description
metadata.comments = '''
A dystopian novel set in Airstrip One, formerly Great Britain...
'''

# Custom columns (user-defined fields)
metadata.set('#custom_field', 'value')
value = metadata.get('#custom_field')

# Print metadata
print(metadata)
# Title: 1984
# Author(s): George Orwell
# Rating: 4.5
# Tags: Fiction, Dystopian
# ...
```

### Metadata Sources

**Download metadata from online sources**: [src/calibre/ebooks/metadata/sources/](../../src/calibre/ebooks/metadata/sources/)

```python
from calibre.ebooks.metadata.sources.identify import identify

# Identify book
identifiers = {
    'isbn': '9780451524935',
    # or 'amazon': 'B003JTHWJQ',
    # or 'google': 'kotPYEqJu-AC'
}

results = identify(
    title='1984',
    authors=['George Orwell'],
    identifiers=identifiers,
    sources=['google', 'amazon', 'goodreads']
)

# results is list of Metadata objects
for metadata in results:
    print(f"{metadata.title} by {', '.join(metadata.authors)}")
    print(f"Publisher: {metadata.publisher}")
    print(f"ISBN: {metadata.isbn}")
    if metadata.cover_url:
        print(f"Cover: {metadata.cover_url}")
    print()
```

### Cover Download

```python
from calibre.ebooks.metadata.sources.covers import download_cover

# Download cover
cover_data = download_cover(
    title='1984',
    authors=['George Orwell'],
    isbn='9780451524935'
)

# Save cover
if cover_data:
    with open('cover.jpg', 'wb') as f:
        f.write(cover_data)
```

### Metadata from eBook Files

```python
from calibre.ebooks.metadata.meta import get_metadata

# Extract metadata from EPUB
with open('book.epub', 'rb') as f:
    metadata = get_metadata(f, 'epub')

print(metadata.title)
print(metadata.authors)

# Extract from other formats
with open('book.mobi', 'rb') as f:
    metadata = get_metadata(f, 'mobi')

with open('book.pdf', 'rb') as f:
    metadata = get_metadata(f, 'pdf')
```

### Write Metadata to eBook Files

```python
from calibre.ebooks.metadata.meta import set_metadata

# Update metadata in EPUB
metadata = Metadata('New Title', ['New Author'])
metadata.rating = 5.0

with open('book.epub', 'r+b') as f:
    set_metadata(f, metadata, 'epub')

# Now book.epub has updated metadata
```

---

## Format Conversion API

### Plumber (Conversion Engine)

**Main conversion class**: [src/calibre/ebooks/conversion/plumber.py](../../src/calibre/ebooks/conversion/plumber.py)

```python
from calibre.ebooks.conversion.plumber import Plumber
from calibre.utils.logging import default_log

# Create log
log = default_log

# Convert MOBI to EPUB
plumber = Plumber(
    input='book.mobi',
    output='book.epub',
    log=log
)

# Set conversion options
plumber.opts.output_profile = 'tablet'
plumber.opts.remove_paragraph_spacing = True
plumber.opts.insert_blank_line = True
plumber.opts.smarten_punctuation = True

# Run conversion
plumber.run()

print(f"Conversion complete: {plumber.output}")
```

### Conversion Options

```python
from calibre.customize.conversion import OptionRecommendation

# Common options
opts = {
    # Input options
    'input_encoding': 'utf-8',

    # Output options
    'output_profile': 'tablet',  # or 'kindle', 'ipad', 'sony', etc.
    'output_encoding': 'utf-8',

    # Look & Feel
    'remove_paragraph_spacing': True,
    'insert_blank_line': True,
    'linearize_tables': False,
    'base_font_size': 12,
    'font_size_mapping': '6, 9, 12, 14, 16, 20, 24',
    'line_height': 1.2,
    'embed_font_family': 'Liberation Serif',

    # Page Setup
    'margin_top': 72.0,      # 1 inch = 72 points
    'margin_bottom': 72.0,
    'margin_left': 72.0,
    'margin_right': 72.0,
    'input_profile': 'default',

    # Heuristic Processing
    'enable_heuristics': True,
    'markup_chapter_headings': True,
    'italicize_common_cases': True,
    'fix_indents': True,
    'remove_fake_margins': True,

    # Search & Replace
    'search_replace': [
        ('old text', 'new text'),
        (r'regex pattern', r'replacement'),
    ],

    # Structure Detection
    'chapter': '//h:h1',  # XPath expression
    'chapter_mark': 'pagebreak',  # or 'rule', 'both', 'none'
    'page_breaks_before': '/',
    'remove_first_image': False,

    # Table of Contents
    'toc_threshold': 6,
    'max_toc_links': 50,
    'duplicate_links_in_toc': False,
    'level1_toc': None,  # XPath
    'level2_toc': None,
    'level3_toc': None,

    # Metadata
    'title': 'Book Title',
    'authors': ['Author Name'],
    'publisher': 'Publisher',
    'tags': ['Tag1', 'Tag2'],
    'rating': 4.5,

    # Debug
    'verbose': 2,
    'debug_pipeline': None,  # Directory to save debug info
}

plumber = Plumber('input.mobi', 'output.epub', log, **opts)
plumber.run()
```

### Format-Specific Conversions

```python
# MOBI to EPUB
from calibre.ebooks.conversion.plumber import Plumber

plumber = Plumber('book.mobi', 'book.epub', log)
plumber.opts.output_profile = 'tablet'
plumber.run()

# PDF to EPUB (OCR if needed)
plumber = Plumber('book.pdf', 'book.epub', log)
plumber.opts.enable_heuristics = True
plumber.run()

# TXT to EPUB
plumber = Plumber('book.txt', 'book.epub', log)
plumber.opts.formatting_type = 'markdown'  # or 'plain', 'textile'
plumber.opts.paragraph_type = 'auto'
plumber.run()

# HTML to EPUB
plumber = Plumber('book.html', 'book.epub', log)
plumber.opts.input_encoding = 'utf-8'
plumber.run()
```

### Batch Conversion

```python
import os
from concurrent.futures import ThreadPoolExecutor

def convert_file(input_file, output_file):
    """Convert single file"""
    plumber = Plumber(input_file, output_file, log)
    plumber.opts.output_profile = 'tablet'
    plumber.run()
    return output_file

# Convert all MOBI files to EPUB
mobi_files = [f for f in os.listdir('.') if f.endswith('.mobi')]

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = []
    for mobi_file in mobi_files:
        epub_file = mobi_file.replace('.mobi', '.epub')
        future = executor.submit(convert_file, mobi_file, epub_file)
        futures.append(future)

    for future in futures:
        result = future.result()
        print(f"Converted: {result}")
```

---

## Plugin API

### Plugin Structure

**Plugin system**: [src/calibre/customize/](../../src/calibre/customize/)

```python
from calibre.customize import InterfaceActionBase

class MyPlugin(InterfaceActionBase):
    """Example plugin"""

    # Plugin metadata
    name = 'My Plugin'
    description = 'Does something useful'
    supported_platforms = ['windows', 'osx', 'linux']
    author = 'Your Name'
    version = (1, 0, 0)
    minimum_calibre_version = (5, 0, 0)

    # Plugin type
    actual_plugin = 'calibre_plugins.my_plugin.ui:MyPluginAction'

    def is_customizable(self):
        """Can be configured"""
        return True

    def config_widget(self):
        """Return config widget"""
        from calibre_plugins.my_plugin.config import ConfigWidget
        return ConfigWidget()

    def save_settings(self, config_widget):
        """Save settings"""
        config_widget.save_settings()
```

### Plugin Types

```python
# 1. Interface Action Plugin (GUI button)
from calibre.customize import InterfaceActionBase

class MyAction(InterfaceActionBase):
    name = 'My Action'

    def genesis(self):
        """Initialize plugin"""
        self.qaction.triggered.connect(self.show_dialog)

    def show_dialog(self):
        """Show plugin dialog"""
        from .dialog import MyDialog
        dialog = MyDialog(self.gui)
        dialog.exec()

# 2. Metadata Source Plugin
from calibre.ebooks.metadata.sources.base import Source

class MyMetadataSource(Source):
    name = 'My Source'
    supported_platforms = ['windows', 'osx', 'linux']

    def identify(self, log, result_queue, abort, title=None, authors=None, **kwargs):
        """Identify book"""
        # Query your metadata source
        results = query_source(title, authors)

        # Add results to queue
        for result in results:
            metadata = Metadata(result['title'], result['authors'])
            metadata.isbn = result['isbn']
            result_queue.put(metadata)

# 3. Conversion Plugin
from calibre.customize.conversion import InputFormatPlugin

class MyInputPlugin(InputFormatPlugin):
    name = 'My Format Input'
    file_types = {'myformat'}

    def convert(self, stream, options, file_ext, log, accelerators):
        """Convert to OEB"""
        from calibre.ebooks.oeb.base import OEB

        # Parse your format
        content = parse_my_format(stream)

        # Return OEB document
        return create_oeb(content)
```

### Accessing Calibre Database from Plugin

```python
from calibre.customize import InterfaceActionBase

class MyPlugin(InterfaceActionBase):
    def process_books(self):
        """Process selected books"""

        # Get database
        db = self.gui.current_db

        # Get selected book IDs
        rows = self.gui.library_view.selectionModel().selectedRows()
        book_ids = [self.gui.library_view.model().id(row) for row in rows]

        # Process each book
        for book_id in book_ids:
            # Get metadata
            metadata = db.get_metadata(book_id)
            print(f"Processing: {metadata.title}")

            # Update metadata
            metadata.tags.append('processed')
            db.set_metadata(book_id, metadata)

        # Refresh view
        self.gui.library_view.model().refresh_ids(book_ids)
```

---

## GUI API

### Main GUI Access

**Main window**: [src/calibre/gui2/main.py](../../src/calibre/gui2/main.py)

```python
from calibre.gui2.ui import Main

# Get main window instance
gui = Main()

# Access database
db = gui.current_db

# Access library view
library_view = gui.library_view

# Get selected books
rows = library_view.selectionModel().selectedRows()
book_ids = [library_view.model().id(row) for row in rows]

# Refresh view
library_view.model().refresh()
library_view.model().refresh_ids([123, 456])

# Show message
gui.status_bar.show_message('Operation complete', 3000)  # 3 seconds

# Show error
from calibre.gui2 import error_dialog
error_dialog(gui, 'Error', 'Something went wrong', show=True)

# Show info
from calibre.gui2 import info_dialog
info_dialog(gui, 'Success', 'Operation completed successfully', show=True)

# Ask question
from calibre.gui2 import question_dialog
if question_dialog(gui, 'Confirm', 'Are you sure?'):
    # User clicked yes
    pass
```

### Library View

```python
# Get library view
view = gui.library_view

# Get model
model = view.model()

# Get all book IDs in view
all_ids = model.id_list

# Get book ID from row
book_id = model.id(row)

# Get row from book ID
row = model.row(book_id)

# Select books
from PyQt6.QtCore import QItemSelectionModel
selection_model = view.selectionModel()

for row in rows_to_select:
    index = model.index(row, 0)
    selection_model.select(index, QItemSelectionModel.Select | QItemSelectionModel.Rows)

# Scroll to book
row = model.row(book_id)
view.scrollTo(model.index(row, 0))
```

### Dialogs

```python
from PyQt6.QtWidgets import QDialog, QVBoxLayout, QPushButton, QLabel
from calibre.gui2 import Application

class MyDialog(QDialog):
    def __init__(self, parent=None):
        super().__init__(parent)

        self.setWindowTitle('My Dialog')

        layout = QVBoxLayout(self)

        # Add widgets
        layout.addWidget(QLabel('Hello, world!'))

        button = QPushButton('Click me')
        button.clicked.connect(self.on_click)
        layout.addWidget(button)

    def on_click(self):
        print('Button clicked!')
        self.accept()  # Close dialog with success

# Show dialog
dialog = MyDialog(gui)
if dialog.exec():  # Returns True if accepted
    print('Dialog accepted')
```

---

## Utility APIs

### File Utilities

```python
from calibre.ptempfile import PersistentTemporaryFile, PersistentTemporaryDirectory

# Create temporary file
with PersistentTemporaryFile(suffix='.epub') as f:
    f.write(b'content')
    temp_path = f.name

print(f"Temp file: {temp_path}")
# File exists after context manager exits

# Create temporary directory
with PersistentTemporaryDirectory() as tdir:
    # Create files in tdir
    file_path = os.path.join(tdir, 'file.txt')
    with open(file_path, 'w') as f:
        f.write('content')

# Directory still exists after context
```

### HTTP Client

```python
from calibre.utils.browser import Browser

# Create browser (user agent, cookies, etc.)
br = Browser()

# GET request
response = br.open('https://example.com')
html = response.read()

# POST request
response = br.open('https://example.com/api', data={'key': 'value'})

# Custom headers
br.addheaders = [('User-Agent', 'My App')]
response = br.open('https://example.com')
```

### Image Processing

```python
from calibre.utils.img import image_from_data, scale_image, image_to_data

# Load image from bytes
img = image_from_data(cover_data)

# Get dimensions
width = img.width()
height = img.height()

# Resize image
scaled = scale_image(img, width=300, height=400, as_png=False)

# Convert to bytes
jpg_data = image_to_data(scaled, fmt='JPEG')
png_data = image_to_data(scaled, fmt='PNG')
```

### Logging

```python
from calibre.utils.logging import default_log, Log

# Use default log
log = default_log
log.info('Information message')
log.error('Error message')
log.debug('Debug message')

# Create custom log
log = Log(level=Log.DEBUG)
log.info('Message')

# Log to file
log = Log(level=Log.DEBUG)
log.outputs = [open('calibre.log', 'w')]
log.info('This goes to file')
```

---

## Creating Custom Endpoints

### Adding New REST Endpoint

**In** [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py):

```python
from calibre.srv.routes import endpoint, json

@endpoint('/api/custom/hello', methods=['GET'])
def custom_hello(ctx, rd):
    """Custom endpoint example"""
    return json({'message': 'Hello, world!'})

@endpoint('/api/custom/greet/{name}', types={'name': str}, methods=['GET'])
def custom_greet(ctx, rd, name):
    """Custom endpoint with path parameter"""
    return json({'message': f'Hello, {name}!'})

@endpoint('/api/custom/books/search', methods=['POST'])
def custom_search(ctx, rd):
    """Custom endpoint with POST data"""

    # Get JSON body
    data = json.loads(rd.read())

    query = data.get('query', '')
    limit = data.get('limit', 10)

    # Search database
    from calibre.db.search import Search
    search = Search(ctx.db)
    results = search.search(query)

    # Get metadata for results
    books = []
    for book_id in results[:limit]:
        metadata = ctx.db.get_metadata(book_id)
        books.append({
            'id': book_id,
            'title': metadata.title,
            'authors': metadata.authors
        })

    return json({'books': books, 'total': len(results)})

@endpoint('/api/custom/stats', methods=['GET'], auth_required=True)
def custom_stats(ctx, rd):
    """Protected endpoint (requires authentication)"""

    # Only authenticated users can access
    total_books = len(ctx.db.all_book_ids())

    # Get books per author
    authors = ctx.db.all_field_names('authors')
    author_counts = {}
    for author in authors:
        results = ctx.db.search(f'author:="{author}"')
        author_counts[author] = len(results)

    return json({
        'total_books': total_books,
        'total_authors': len(authors),
        'books_per_author': author_counts
    })
```

### Custom Validation

```python
from calibre.srv.errors import HTTPBadRequest

@endpoint('/api/custom/rate-book/{book_id}', types={'book_id': int}, methods=['POST'])
def rate_book(ctx, rd, book_id):
    """Rate a book"""

    # Parse JSON
    data = json.loads(rd.read())
    rating = data.get('rating')

    # Validate rating
    if not isinstance(rating, (int, float)):
        raise HTTPBadRequest('rating must be a number')

    if not 0 <= rating <= 5:
        raise HTTPBadRequest('rating must be between 0 and 5')

    # Check book exists
    if not ctx.db.has_id(book_id):
        raise HTTPNotFound(f'Book {book_id} not found')

    # Update rating
    ctx.db.set_field('rating', book_id, rating)

    return json({'status': 'success', 'book_id': book_id, 'rating': rating})
```

---

## API Best Practices

### 1. Error Handling

```python
from calibre.srv.errors import HTTPNotFound, HTTPBadRequest, HTTPServerError

@endpoint('/api/books/{book_id}', types={'book_id': int})
def get_book(ctx, rd, book_id):
    try:
        # Validate book exists
        if not ctx.db.has_id(book_id):
            raise HTTPNotFound(f'Book {book_id} not found')

        # Get metadata
        metadata = ctx.db.get_metadata(book_id)

        return json({
            'id': book_id,
            'title': metadata.title,
            'authors': metadata.authors
        })

    except HTTPNotFound:
        raise  # Re-raise HTTP exceptions

    except Exception as e:
        # Log error
        ctx.log.error(f'Error getting book {book_id}', exc_info=True)

        # Return generic error
        raise HTTPServerError('Internal server error')
```

### 2. Input Validation

```python
@endpoint('/api/books/search', methods=['POST'])
def search_books(ctx, rd):
    # Parse JSON
    try:
        data = json.loads(rd.read())
    except json.JSONDecodeError:
        raise HTTPBadRequest('Invalid JSON')

    # Validate fields
    query = data.get('query', '')
    if not isinstance(query, str):
        raise HTTPBadRequest('query must be string')

    if len(query) > 1000:
        raise HTTPBadRequest('query too long (max 1000 chars)')

    limit = data.get('limit', 50)
    if not isinstance(limit, int) or limit < 1 or limit > 1000:
        raise HTTPBadRequest('limit must be 1-1000')

    # Perform search
    results = search(query, limit)
    return json({'results': results})
```

### 3. Pagination

```python
@endpoint('/api/books', methods=['GET'])
def list_books(ctx, rd):
    # Get pagination parameters
    try:
        limit = int(rd.query.get('limit', ['50'])[0])
        offset = int(rd.query.get('offset', ['0'])[0])
    except ValueError:
        raise HTTPBadRequest('limit and offset must be integers')

    # Validate
    if limit < 1 or limit > 1000:
        raise HTTPBadRequest('limit must be 1-1000')

    if offset < 0:
        raise HTTPBadRequest('offset must be >= 0')

    # Get books
    all_ids = ctx.db.all_book_ids()
    total = len(all_ids)

    book_ids = all_ids[offset:offset + limit]

    books = []
    for book_id in book_ids:
        metadata = ctx.db.get_metadata(book_id)
        books.append({
            'id': book_id,
            'title': metadata.title,
            'authors': metadata.authors
        })

    return json({
        'books': books,
        'total': total,
        'limit': limit,
        'offset': offset,
        'has_more': offset + limit < total
    })
```

### 4. Rate Limiting

```python
from collections import defaultdict
from datetime import datetime, timedelta

# Simple rate limiter
class RateLimiter:
    def __init__(self):
        self.requests = defaultdict(list)

    def is_allowed(self, ip, max_requests=100, window_seconds=60):
        now = datetime.now()
        window_start = now - timedelta(seconds=window_seconds)

        # Remove old requests
        self.requests[ip] = [
            ts for ts in self.requests[ip]
            if ts > window_start
        ]

        # Check limit
        if len(self.requests[ip]) >= max_requests:
            return False

        # Record request
        self.requests[ip].append(now)
        return True

rate_limiter = RateLimiter()

@endpoint('/api/expensive-operation', methods=['POST'])
def expensive_operation(ctx, rd):
    # Check rate limit
    client_ip = rd.remote_addr

    if not rate_limiter.is_allowed(client_ip, max_requests=10, window_seconds=60):
        raise HTTPTooManyRequests('Rate limit exceeded')

    # Perform operation
    result = do_expensive_operation()
    return json({'result': result})
```

---

## Quick Reference

### Common API Patterns

```python
# Database operations
db = cache  # or ctx.db in endpoints
book_ids = db.all_book_ids()
metadata = db.get_metadata(book_id)
db.set_field('title', book_id, 'New Title')

# Search
from calibre.db.search import Search
search = Search(db)
results = search.search('author:orwell')

# Metadata
from calibre.ebooks.metadata.book.base import Metadata
metadata = Metadata('Title', ['Author'])
metadata.rating = 4.5

# Conversion
from calibre.ebooks.conversion.plumber import Plumber
plumber = Plumber('input.mobi', 'output.epub', log)
plumber.run()

# REST endpoint
@endpoint('/api/path', methods=['GET'])
def handler(ctx, rd):
    return json({'result': 'value'})
```

---

## Next Steps

Now that you understand the APIs:

1. **Practice**: Create a custom plugin or endpoint
2. **Read**: [PLUGIN_DEVELOPMENT_GUIDE.md](./PLUGIN_DEVELOPMENT_GUIDE.md) - Deep dive into plugins
3. **Read**: [DEVELOPMENT_WORKFLOW.md](./DEVELOPMENT_WORKFLOW.md) - Professional workflow
4. **Explore**: [src/calibre/customize/](../../src/calibre/customize/) - Built-in plugins

---

## Additional Resources

### Official Documentation
- [Calibre API Documentation](https://manual.calibre-ebook.com/develop.html)
- [Plugin Tutorial](https://manual.calibre-ebook.com/plugins.html)
- [Conversion API](https://manual.calibre-ebook.com/conversion.html)

### Code Examples
- [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py) - REST endpoints
- [src/calibre/db/cache.py](../../src/calibre/db/cache.py) - Database API
- [src/calibre/customize/builtins.py](../../src/calibre/customize/builtins.py) - Built-in plugins

---

**Remember**: APIs are interfaces - they should be stable, well-documented, and backward-compatible. When designing APIs, think about the users who will consume them!
