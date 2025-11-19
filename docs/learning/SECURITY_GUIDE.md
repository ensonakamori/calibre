# Calibre Security Guide

> **For React developers**: This guide covers backend security concepts that are different from frontend security. While React apps worry about XSS and CSRF, Python/Calibre apps must also handle SQL injection, path traversal, arbitrary code execution, and server-side vulnerabilities.

---

## Table of Contents

1. [Overview](#overview)
2. [Security Principles](#security-principles)
3. [Input Validation](#input-validation)
4. [SQL Injection Prevention](#sql-injection-prevention)
5. [Path Traversal Prevention](#path-traversal-prevention)
6. [Command Injection Prevention](#command-injection-prevention)
7. [File Upload Security](#file-upload-security)
8. [Authentication & Authorization](#authentication--authorization)
9. [Cryptography](#cryptography)
10. [HTTP Server Security](#http-server-security)
11. [Dependency Security](#dependency-security)
12. [Code Review Checklist](#code-review-checklist)
13. [Common Vulnerabilities](#common-vulnerabilities)
14. [Security Testing](#security-testing)

---

## Overview

Security in Calibre is critical because:
1. **Handles user files** - eBooks, covers, metadata
2. **Runs HTTP server** - Exposed to network attacks
3. **Executes conversions** - Processes untrusted file formats
4. **Database access** - SQLite queries from user input
5. **Plugin system** - Third-party code execution

### OWASP Top 10 (2021) Relevance to Calibre

| Vulnerability | Relevant? | Where in Calibre |
|---------------|-----------|------------------|
| **Broken Access Control** | ✅ Yes | HTTP server, file access |
| **Cryptographic Failures** | ✅ Yes | Password storage, encrypted books |
| **Injection** | ✅ Yes | SQL, command, path injection |
| **Insecure Design** | ⚠️ Partial | Architecture decisions |
| **Security Misconfiguration** | ✅ Yes | Server settings |
| **Vulnerable Components** | ✅ Yes | Dependencies (lxml, Pillow, etc.) |
| **Authentication Failures** | ✅ Yes | Content server login |
| **Data Integrity Failures** | ⚠️ Partial | File verification |
| **Logging Failures** | ⚠️ Partial | Security event logging |
| **SSRF** | ⚠️ Partial | Metadata downloads |

---

## Security Principles

### 1. Defense in Depth

**Multiple layers of security** - if one fails, others protect:

```python
# ❌ BAD: Single point of failure
def delete_book(book_id):
    db.delete(book_id)  # No validation!

# ✅ GOOD: Multiple security layers
def delete_book(book_id):
    # Layer 1: Input validation
    if not isinstance(book_id, int):
        raise ValueError("book_id must be integer")

    # Layer 2: Authorization check
    if not current_user.can_delete_books():
        raise PermissionError("User cannot delete books")

    # Layer 3: Existence check
    if not db.has_id(book_id):
        raise ValueError(f"Book {book_id} doesn't exist")

    # Layer 4: Audit logging
    logger.warning(f"User {current_user.name} deleting book {book_id}")

    # Layer 5: Transaction (can rollback)
    with db.transaction():
        db.delete(book_id)
```

### 2. Principle of Least Privilege

**Grant minimum necessary permissions**:

```python
# ❌ BAD: Running as root/admin
if os.getuid() == 0:
    # Running as root - dangerous!
    pass

# ✅ GOOD: Drop privileges after startup
import pwd

def drop_privileges(user='calibre'):
    """Drop root privileges"""
    if os.getuid() == 0:
        # Get user info
        pwnam = pwd.getpwnam(user)

        # Drop privileges
        os.setgid(pwnam.pw_gid)
        os.setuid(pwnam.pw_uid)

        # Verify
        if os.getuid() == 0:
            raise RuntimeError("Failed to drop privileges")
```

### 3. Fail Securely

**Errors should not expose information or grant access**:

```python
# ❌ BAD: Revealing error messages
def login(username, password):
    user = db.get_user(username)
    if not user:
        raise ValueError("User doesn't exist")  # ⚠️ Username enumeration!

    if not user.check_password(password):
        raise ValueError("Wrong password")  # ⚠️ Reveals username is valid!

# ✅ GOOD: Generic error messages
def login(username, password):
    user = db.get_user(username)

    # Generic error - doesn't reveal if username exists
    if not user or not user.check_password(password):
        logger.warning(f"Failed login attempt for: {username}")
        raise ValueError("Invalid username or password")

    return user
```

### 4. Don't Trust User Input

**NEVER trust data from users, even "safe" users**:

```python
# ❌ BAD: Trusting user input
def set_book_title(book_id, title):
    # What if title is malicious?
    db.execute(f"UPDATE books SET title = '{title}' WHERE id = {book_id}")

# ✅ GOOD: Validate and sanitize
def set_book_title(book_id, title):
    # Validate book_id type
    if not isinstance(book_id, int):
        raise TypeError("book_id must be integer")

    # Validate title
    if not isinstance(title, str):
        raise TypeError("title must be string")

    if len(title) > 500:
        raise ValueError("title too long (max 500 chars)")

    # Sanitize: remove control characters
    title = ''.join(c for c in title if c.isprintable() or c.isspace())

    # Use parameterized query
    db.execute("UPDATE books SET title = ? WHERE id = ?", (title, book_id))
```

---

## Input Validation

### 1. Type Validation

**React equivalent**: TypeScript runtime validation (like Zod)

```python
from typing import Union

def validate_book_id(book_id: Union[int, str]) -> int:
    """Validate and convert book_id to int"""

    # Accept int or string
    if isinstance(book_id, str):
        if not book_id.isdigit():
            raise ValueError(f"Invalid book_id: {book_id!r}")
        book_id = int(book_id)

    if not isinstance(book_id, int):
        raise TypeError(f"book_id must be int, got {type(book_id).__name__}")

    if book_id < 1:
        raise ValueError(f"book_id must be positive, got {book_id}")

    return book_id

# Usage
book_id = validate_book_id(request.args.get('book_id'))
```

**Example in Calibre**: [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py#L57) - `types` parameter

### 2. String Validation

```python
import re

def validate_isbn(isbn: str) -> str:
    """Validate ISBN-10 or ISBN-13"""

    # Remove hyphens and spaces
    isbn = isbn.replace('-', '').replace(' ', '')

    # Check format
    if not re.match(r'^\d{10}(\d{3})?$', isbn):
        raise ValueError(f"Invalid ISBN format: {isbn!r}")

    # Validate length
    if len(isbn) not in (10, 13):
        raise ValueError(f"ISBN must be 10 or 13 digits, got {len(isbn)}")

    return isbn

def validate_email(email: str) -> str:
    """Validate email address"""

    # Basic format check (not comprehensive)
    if not re.match(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$', email):
        raise ValueError(f"Invalid email: {email!r}")

    if len(email) > 254:  # RFC 5321
        raise ValueError("Email too long")

    return email.lower()
```

### 3. Range Validation

```python
def validate_rating(rating: float) -> float:
    """Validate book rating (0-5)"""

    if not isinstance(rating, (int, float)):
        raise TypeError(f"rating must be number, got {type(rating).__name__}")

    if not 0 <= rating <= 5:
        raise ValueError(f"rating must be 0-5, got {rating}")

    return float(rating)

def validate_page_number(page: int, total_pages: int) -> int:
    """Validate page number"""

    if not isinstance(page, int):
        raise TypeError("page must be integer")

    if not 1 <= page <= total_pages:
        raise ValueError(f"page must be 1-{total_pages}, got {page}")

    return page
```

### 4. Whitelist Validation

**Safer than blacklist**:

```python
# ❌ BAD: Blacklist (easy to bypass)
def validate_format(format):
    # What if user sends ".ExE" or ".eXe"?
    if format in ['.exe', '.sh', '.bat']:
        raise ValueError("Dangerous format")
    return format

# ✅ GOOD: Whitelist (only allow known-safe)
ALLOWED_FORMATS = {'.epub', '.mobi', '.azw3', '.pdf', '.txt'}

def validate_format(format: str) -> str:
    """Validate ebook format"""

    # Normalize to lowercase
    format = format.lower().strip()

    # Ensure starts with dot
    if not format.startswith('.'):
        format = '.' + format

    # Check against whitelist
    if format not in ALLOWED_FORMATS:
        raise ValueError(f"Unsupported format: {format!r}")

    return format
```

**Example in Calibre**: [src/calibre/ebooks/](../../src/calibre/ebooks/) - format handlers

---

## SQL Injection Prevention

### What is SQL Injection?

**React equivalent**: Similar to inserting `<script>` tags in React (XSS), but for databases

```python
# ❌ CRITICAL VULNERABILITY: SQL Injection
def get_books_by_author(author_name):
    # If author_name = "'; DROP TABLE books; --"
    # SQL becomes: SELECT * FROM books WHERE author = ''; DROP TABLE books; --'
    sql = f"SELECT * FROM books WHERE author = '{author_name}'"
    return db.execute(sql).fetchall()

# Attack:
get_books_by_author("'; DROP TABLE books; --")
# Result: ALL BOOKS DELETED! 💥
```

### Solution: Parameterized Queries

```python
# ✅ SAFE: Parameterized query
def get_books_by_author(author_name):
    # Database driver escapes author_name automatically
    sql = "SELECT * FROM books WHERE author = ?"
    return db.execute(sql, (author_name,)).fetchall()

# Attack attempt:
get_books_by_author("'; DROP TABLE books; --")
# Result: Searches for author named "'; DROP TABLE books; --" (harmless)
```

### APSW (Calibre's DB Library) Examples

**See**: [src/calibre/db/tables.py](../../src/calibre/db/tables.py#L89)

```python
import apsw

# ✅ SAFE: Named parameters
def update_book(book_id, title, author):
    cursor = db.conn.cursor()
    cursor.execute("""
        UPDATE books
        SET title = :title, author = :author
        WHERE id = :book_id
    """, {'title': title, 'author': author, 'book_id': book_id})

# ✅ SAFE: Positional parameters
def delete_book(book_id):
    cursor = db.conn.cursor()
    cursor.execute("DELETE FROM books WHERE id = ?", (book_id,))

# ✅ SAFE: Multiple parameters
def search_books(title, author, min_rating):
    cursor = db.conn.cursor()
    cursor.execute("""
        SELECT * FROM books
        WHERE title LIKE ? AND author LIKE ? AND rating >= ?
    """, (f'%{title}%', f'%{author}%', min_rating))
    return cursor.fetchall()
```

### Dynamic Queries (Advanced)

**Sometimes you need dynamic SQL** - do it safely:

```python
# ❌ DANGEROUS: Dynamic column name
def sort_books(sort_column):
    # If sort_column = "id; DROP TABLE books; --"
    sql = f"SELECT * FROM books ORDER BY {sort_column}"
    return db.execute(sql).fetchall()

# ✅ SAFE: Whitelist column names
ALLOWED_SORT_COLUMNS = {'id', 'title', 'author', 'rating', 'pubdate'}

def sort_books(sort_column):
    # Validate against whitelist
    if sort_column not in ALLOWED_SORT_COLUMNS:
        raise ValueError(f"Invalid sort column: {sort_column!r}")

    # Safe to use in SQL now
    sql = f"SELECT * FROM books ORDER BY {sort_column}"
    return db.execute(sql).fetchall()
```

### ORM Safety

**If using SQLAlchemy or other ORMs**:

```python
from sqlalchemy import select

# ✅ SAFE: ORM automatically parameterizes
def get_books_by_author(author_name):
    stmt = select(Book).where(Book.author == author_name)
    return session.execute(stmt).all()

# Still need to validate input though!
def get_books_by_rating(rating):
    # Validate first
    if not isinstance(rating, (int, float)) or not 0 <= rating <= 5:
        raise ValueError("Invalid rating")

    stmt = select(Book).where(Book.rating >= rating)
    return session.execute(stmt).all()
```

---

## Path Traversal Prevention

### What is Path Traversal?

**Attacker accesses files outside allowed directory**:

```python
# ❌ CRITICAL VULNERABILITY: Path Traversal
def get_book_file(filename):
    # If filename = "../../etc/passwd"
    # Opens /etc/passwd instead of book!
    path = os.path.join('/library', filename)
    return open(path, 'rb').read()

# Attack:
get_book_file("../../../etc/passwd")
# Result: Reads system password file! 💥
```

### Solution: Path Validation

```python
import os

# ✅ SAFE: Validate path stays in allowed directory
def get_book_file(filename):
    library_path = '/var/calibre/library'

    # Join paths
    file_path = os.path.join(library_path, filename)

    # Resolve to absolute path (eliminates .. and symlinks)
    file_path = os.path.realpath(file_path)

    # Check if still within library_path
    if not file_path.startswith(os.path.realpath(library_path)):
        raise ValueError(f"Path traversal attempt: {filename!r}")

    # Check if file exists
    if not os.path.isfile(file_path):
        raise FileNotFoundError(f"File not found: {filename!r}")

    return open(file_path, 'rb').read()

# Attack attempt:
get_book_file("../../../etc/passwd")
# Result: ValueError raised, file not accessed ✅
```

### Safer Alternative: Use IDs

```python
# ✅ SAFER: Don't use filenames from users at all
def get_book_file(book_id, format):
    # Validate inputs
    book_id = validate_book_id(book_id)
    format = validate_format(format)

    # Construct path from database, not user input
    book = db.get_metadata(book_id)
    if format not in book.formats:
        raise ValueError(f"Book {book_id} not available in {format}")

    # Get path from database
    file_path = book.format_abspath(format)

    return open(file_path, 'rb').read()
```

**Example in Calibre**: [src/calibre/db/cache.py](../../src/calibre/db/cache.py#L234) - `format_abspath()`

### Directory Listing Protection

```python
# ❌ DANGEROUS: Listing user-specified directory
def list_directory(path):
    return os.listdir(path)  # Can list any directory!

# ✅ SAFE: Only list subdirectories of allowed path
def list_directory(subpath):
    base_path = '/var/calibre/library'

    # Validate subpath
    if '..' in subpath or subpath.startswith('/'):
        raise ValueError("Invalid path")

    full_path = os.path.join(base_path, subpath)
    full_path = os.path.realpath(full_path)

    if not full_path.startswith(os.path.realpath(base_path)):
        raise ValueError("Path traversal attempt")

    return os.listdir(full_path)
```

---

## Command Injection Prevention

### What is Command Injection?

**Attacker executes arbitrary commands**:

```python
# ❌ CRITICAL VULNERABILITY: Command Injection
def convert_book(input_file, output_file):
    # If input_file = "book.epub; rm -rf /"
    # Executes: convert book.epub output.pdf; rm -rf /
    os.system(f"ebook-convert {input_file} {output_file}")

# Attack:
convert_book("book.epub; rm -rf /", "output.pdf")
# Result: Deletes entire filesystem! 💥
```

### Solution: Use subprocess with List Arguments

```python
import subprocess

# ✅ SAFE: Use list arguments (no shell)
def convert_book(input_file, output_file):
    # Validate file paths first
    input_file = validate_book_path(input_file)
    output_file = validate_output_path(output_file)

    # Use list arguments - prevents injection
    subprocess.run([
        'ebook-convert',
        input_file,
        output_file
    ], check=True, shell=False)  # shell=False is critical!

# Attack attempt:
convert_book("book.epub; rm -rf /", "output.pdf")
# Result: Command fails because file doesn't exist (safe) ✅
```

### More Examples

```python
# ❌ DANGEROUS: Shell=True with user input
def compress_book(filename):
    os.system(f"gzip {filename}")  # Shell injection!

# ✅ SAFE: No shell
def compress_book(filename):
    filename = validate_book_path(filename)
    subprocess.run(['gzip', filename], check=True)

# ❌ DANGEROUS: Using shell features
def backup_library(backup_name):
    # User could inject commands via backup_name
    os.system(f"tar czf {backup_name}.tar.gz /library/*")

# ✅ SAFE: List arguments
def backup_library(backup_name):
    # Validate backup_name
    if not re.match(r'^[a-zA-Z0-9_-]+$', backup_name):
        raise ValueError("Invalid backup name")

    subprocess.run([
        'tar', 'czf', f'{backup_name}.tar.gz',
        '/var/calibre/library'
    ], check=True)
```

### When You MUST Use Shell

**Sometimes shell features are needed** (pipes, redirection):

```python
import shlex

# ⚠️ USE WITH EXTREME CAUTION
def convert_with_shell(input_file, output_file):
    # Validate inputs first
    input_file = validate_book_path(input_file)
    output_file = validate_output_path(output_file)

    # Use shlex.quote() to escape shell metacharacters
    cmd = f"ebook-convert {shlex.quote(input_file)} {shlex.quote(output_file)}"

    subprocess.run(cmd, shell=True, check=True)

# Attack attempt:
convert_with_shell("book.epub; rm -rf /", "out.pdf")
# Result: Looks for file named "book.epub; rm -rf /" (safe) ✅
```

**Example in Calibre**: [src/calibre/ebooks/conversion/cli.py](../../src/calibre/ebooks/conversion/cli.py)

---

## File Upload Security

### 1. File Type Validation

**Never trust file extensions**:

```python
import magic  # python-magic library

# ❌ DANGEROUS: Trust file extension
def save_upload(filename, data):
    ext = os.path.splitext(filename)[1]
    if ext == '.epub':
        # User could rename .exe to .epub!
        save_book(data)

# ✅ SAFE: Check actual file content (magic bytes)
def save_upload(filename, data):
    # Check magic bytes
    mime_type = magic.from_buffer(data, mime=True)

    # Whitelist of allowed MIME types
    ALLOWED_TYPES = {
        'application/epub+zip',
        'application/pdf',
        'application/x-mobipocket-ebook',
        'text/plain'
    }

    if mime_type not in ALLOWED_TYPES:
        raise ValueError(f"Invalid file type: {mime_type}")

    # Also validate extension
    ext = os.path.splitext(filename)[1].lower()
    if ext not in {'.epub', '.pdf', '.mobi', '.txt'}:
        raise ValueError(f"Invalid extension: {ext}")

    save_book(data)
```

### 2. Filename Sanitization

```python
import re
import unicodedata

def sanitize_filename(filename):
    """Sanitize filename for safe storage"""

    # Normalize Unicode (NFKC form)
    filename = unicodedata.normalize('NFKC', filename)

    # Remove path separators
    filename = filename.replace('/', '_').replace('\\', '_')

    # Remove null bytes
    filename = filename.replace('\x00', '')

    # Remove control characters
    filename = ''.join(c for c in filename if c.isprintable())

    # Remove leading/trailing dots and spaces
    filename = filename.strip('. ')

    # Limit length
    name, ext = os.path.splitext(filename)
    if len(name) > 200:
        name = name[:200]
    filename = name + ext

    # Ensure not empty
    if not filename:
        raise ValueError("Filename is empty after sanitization")

    # Check for reserved names (Windows)
    RESERVED_NAMES = {
        'CON', 'PRN', 'AUX', 'NUL',
        'COM1', 'COM2', 'COM3', 'COM4',
        'LPT1', 'LPT2', 'LPT3'
    }
    if name.upper() in RESERVED_NAMES:
        filename = f'_{filename}'

    return filename
```

### 3. File Size Limits

```python
MAX_BOOK_SIZE = 500 * 1024 * 1024  # 500 MB

def upload_book(file_data):
    """Upload book with size validation"""

    # Check size before processing
    if len(file_data) > MAX_BOOK_SIZE:
        raise ValueError(f"File too large: {len(file_data)} bytes (max {MAX_BOOK_SIZE})")

    # Validate file type
    mime_type = magic.from_buffer(file_data, mime=True)
    if mime_type not in ALLOWED_TYPES:
        raise ValueError(f"Invalid file type: {mime_type}")

    # Save file
    return save_book(file_data)
```

### 4. Malicious Content Scanning

```python
# For EPUB files, check for malicious content
def scan_epub(epub_path):
    """Scan EPUB for malicious content"""
    import zipfile

    with zipfile.ZipFile(epub_path, 'r') as zf:
        # Check for suspicious files
        for name in zf.namelist():
            # Check for path traversal in zip
            if name.startswith('/') or '..' in name:
                raise ValueError(f"Suspicious path in EPUB: {name}")

            # Check for executable files
            if name.endswith(('.exe', '.sh', '.bat', '.cmd')):
                raise ValueError(f"Executable file in EPUB: {name}")

            # Check file size (zip bomb protection)
            info = zf.getinfo(name)
            if info.file_size > 100 * 1024 * 1024:  # 100 MB uncompressed
                raise ValueError(f"Suspicious file size in EPUB: {name}")

        # Extract and scan HTML files for XSS
        for name in zf.namelist():
            if name.endswith(('.html', '.xhtml', '.htm')):
                content = zf.read(name).decode('utf-8', errors='ignore')

                # Check for JavaScript (not allowed in EPUBs)
                if '<script' in content.lower():
                    raise ValueError(f"JavaScript found in {name}")
```

**Example in Calibre**: [src/calibre/ebooks/epub/](../../src/calibre/ebooks/epub/)

---

## Authentication & Authorization

### 1. Password Storage

**NEVER store plaintext passwords**:

```python
import hashlib
import os

# ❌ CRITICAL VULNERABILITY: Plaintext password
class User:
    def __init__(self, username, password):
        self.username = username
        self.password = password  # ⚠️ Plaintext!

    def check_password(self, password):
        return self.password == password  # ⚠️ Timing attack!

# ✅ SAFE: Hashed with salt
import bcrypt

class User:
    def __init__(self, username, password_hash=None):
        self.username = username
        self.password_hash = password_hash

    @classmethod
    def create(cls, username, password):
        """Create user with hashed password"""
        # Hash password with bcrypt (includes salt)
        password_hash = bcrypt.hashpw(
            password.encode('utf-8'),
            bcrypt.gensalt(rounds=12)
        )
        return cls(username, password_hash)

    def check_password(self, password):
        """Check password (timing-safe)"""
        if not self.password_hash:
            return False

        return bcrypt.checkpw(
            password.encode('utf-8'),
            self.password_hash
        )
```

### 2. Session Management

```python
import secrets
from datetime import datetime, timedelta

class SessionManager:
    def __init__(self):
        self.sessions = {}  # session_id -> {user, expires}

    def create_session(self, user):
        """Create new session"""
        # Generate cryptographically secure random session ID
        session_id = secrets.token_urlsafe(32)

        # Set expiration (1 hour)
        expires = datetime.now() + timedelta(hours=1)

        self.sessions[session_id] = {
            'user': user,
            'expires': expires
        }

        return session_id

    def get_user(self, session_id):
        """Get user from session"""
        session = self.sessions.get(session_id)

        if not session:
            return None

        # Check expiration
        if datetime.now() > session['expires']:
            del self.sessions[session_id]
            return None

        # Refresh expiration (sliding window)
        session['expires'] = datetime.now() + timedelta(hours=1)

        return session['user']

    def destroy_session(self, session_id):
        """Logout"""
        if session_id in self.sessions:
            del self.sessions[session_id]
```

### 3. Authorization Checks

```python
from enum import Enum

class Permission(Enum):
    READ_BOOKS = 'read_books'
    EDIT_METADATA = 'edit_metadata'
    DELETE_BOOKS = 'delete_books'
    MANAGE_USERS = 'manage_users'

class Role:
    def __init__(self, name, permissions):
        self.name = name
        self.permissions = set(permissions)

    def has_permission(self, permission):
        return permission in self.permissions

# Define roles
ROLES = {
    'guest': Role('guest', [Permission.READ_BOOKS]),
    'user': Role('user', [Permission.READ_BOOKS, Permission.EDIT_METADATA]),
    'admin': Role('admin', [
        Permission.READ_BOOKS,
        Permission.EDIT_METADATA,
        Permission.DELETE_BOOKS,
        Permission.MANAGE_USERS
    ])
}

class User:
    def __init__(self, username, role_name='guest'):
        self.username = username
        self.role = ROLES[role_name]

    def can(self, permission):
        """Check if user has permission"""
        return self.role.has_permission(permission)

# Usage
def delete_book(user, book_id):
    # Check permission
    if not user.can(Permission.DELETE_BOOKS):
        raise PermissionError(f"User {user.username} cannot delete books")

    db.delete(book_id)
```

### 4. Rate Limiting

**Prevent brute force attacks**:

```python
from collections import defaultdict
from datetime import datetime, timedelta

class RateLimiter:
    def __init__(self, max_attempts=5, window_seconds=60):
        self.max_attempts = max_attempts
        self.window = timedelta(seconds=window_seconds)
        self.attempts = defaultdict(list)  # username -> [timestamps]

    def is_allowed(self, username):
        """Check if user can make request"""
        now = datetime.now()

        # Remove old attempts outside window
        self.attempts[username] = [
            ts for ts in self.attempts[username]
            if now - ts < self.window
        ]

        # Check if under limit
        if len(self.attempts[username]) >= self.max_attempts:
            return False

        # Record attempt
        self.attempts[username].append(now)
        return True

# Usage
rate_limiter = RateLimiter(max_attempts=5, window_seconds=60)

def login(username, password):
    # Check rate limit
    if not rate_limiter.is_allowed(username):
        logger.warning(f"Rate limit exceeded for {username}")
        raise ValueError("Too many login attempts. Try again later.")

    # Proceed with login
    user = authenticate(username, password)
    return user
```

**Example in Calibre**: [src/calibre/srv/auth.py](../../src/calibre/srv/auth.py)

---

## Cryptography

### 1. Random Numbers

**Use cryptographically secure random**:

```python
import secrets
import random

# ❌ INSECURE: Predictable random
session_id = ''.join(random.choices('0123456789', k=16))  # Predictable!

# ✅ SECURE: Cryptographically secure random
session_id = secrets.token_urlsafe(32)  # Unpredictable
api_key = secrets.token_hex(32)
reset_token = secrets.token_urlsafe(64)
```

### 2. Encryption

**Use modern, vetted libraries**:

```python
from cryptography.fernet import Fernet

# ❌ DANGEROUS: Custom encryption
def encrypt_data(data, password):
    # Don't roll your own crypto!
    return ''.join(chr(ord(c) ^ ord(password[i % len(password)])) for i, c in enumerate(data))

# ✅ SAFE: Use vetted library (Fernet)
def encrypt_data(data, key):
    """Encrypt data with Fernet (AES-128-CBC + HMAC)"""
    f = Fernet(key)
    return f.encrypt(data.encode('utf-8'))

def decrypt_data(encrypted_data, key):
    """Decrypt data"""
    f = Fernet(key)
    return f.decrypt(encrypted_data).decode('utf-8')

# Generate key
key = Fernet.generate_key()

# Usage
encrypted = encrypt_data("secret message", key)
decrypted = decrypt_data(encrypted, key)
```

### 3. Hashing

```python
import hashlib

def hash_file(file_path):
    """Hash file with SHA-256"""
    hasher = hashlib.sha256()

    with open(file_path, 'rb') as f:
        # Read in chunks (memory efficient)
        for chunk in iter(lambda: f.read(65536), b''):
            hasher.update(chunk)

    return hasher.hexdigest()

def verify_file(file_path, expected_hash):
    """Verify file integrity"""
    actual_hash = hash_file(file_path)

    # Timing-safe comparison
    return secrets.compare_digest(actual_hash, expected_hash)
```

---

## HTTP Server Security

### 1. Security Headers

```python
# In src/calibre/srv/loop.py
def add_security_headers(response):
    """Add security headers to HTTP response"""

    # Prevent XSS attacks
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '1; mode=block'

    # Content Security Policy (restrict resources)
    response.headers['Content-Security-Policy'] = (
        "default-src 'self'; "
        "script-src 'self'; "
        "style-src 'self' 'unsafe-inline'; "
        "img-src 'self' data:; "
        "font-src 'self'; "
        "connect-src 'self'; "
        "frame-ancestors 'none'"
    )

    # HTTPS only (if using HTTPS)
    if request.is_secure:
        response.headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains'

    # Referrer policy
    response.headers['Referrer-Policy'] = 'strict-origin-when-cross-origin'

    # Permissions policy
    response.headers['Permissions-Policy'] = 'geolocation=(), microphone=(), camera=()'

    return response
```

**See**: [src/calibre/srv/loop.py](../../src/calibre/srv/loop.py)

### 2. CORS (Cross-Origin Resource Sharing)

```python
def handle_cors(request, response):
    """Handle CORS requests"""

    # Whitelist of allowed origins
    ALLOWED_ORIGINS = {
        'https://calibre.example.com',
        'https://app.example.com'
    }

    origin = request.headers.get('Origin')

    # Check if origin is allowed
    if origin in ALLOWED_ORIGINS:
        response.headers['Access-Control-Allow-Origin'] = origin
        response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE'
        response.headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
        response.headers['Access-Control-Max-Age'] = '3600'

    return response
```

### 3. CSRF Protection

**React equivalent**: CSRF tokens in forms

```python
import secrets

class CSRFProtection:
    def __init__(self):
        self.tokens = {}  # session_id -> token

    def generate_token(self, session_id):
        """Generate CSRF token for session"""
        token = secrets.token_urlsafe(32)
        self.tokens[session_id] = token
        return token

    def validate_token(self, session_id, token):
        """Validate CSRF token"""
        expected = self.tokens.get(session_id)

        if not expected:
            return False

        # Timing-safe comparison
        return secrets.compare_digest(expected, token)

# Usage in endpoint
@endpoint('/api/delete-book', methods=['POST'])
def delete_book(ctx, rd):
    # Get session
    session_id = rd.cookies.get('session_id')

    # Get CSRF token from request
    token = rd.json.get('csrf_token')

    # Validate token
    if not csrf.validate_token(session_id, token):
        raise ValueError("Invalid CSRF token")

    # Proceed with deletion
    book_id = rd.json['book_id']
    ctx.db.delete(book_id)
```

### 4. Request Size Limits

```python
MAX_REQUEST_SIZE = 10 * 1024 * 1024  # 10 MB

def handle_request(request):
    """Handle HTTP request with size limit"""

    # Check Content-Length header
    content_length = request.headers.get('Content-Length')

    if content_length:
        content_length = int(content_length)
        if content_length > MAX_REQUEST_SIZE:
            return Response(status=413, body='Request too large')

    # Read body with limit
    body = request.read(MAX_REQUEST_SIZE + 1)

    if len(body) > MAX_REQUEST_SIZE:
        return Response(status=413, body='Request too large')

    # Process request
    return process_request(request, body)
```

---

## Dependency Security

### 1. Keep Dependencies Updated

```bash
# Check for outdated packages
pip list --outdated

# Check for known vulnerabilities
pip install safety
safety check

# Or use pip-audit (newer)
pip install pip-audit
pip-audit
```

### 2. Pin Dependency Versions

**In requirements.txt**:

```txt
# ❌ BAD: Unpinned versions (can break)
lxml
Pillow
PyQt6

# ✅ GOOD: Pinned versions
lxml==5.1.0
Pillow==10.3.0
PyQt6==6.6.1

# ⚠️ ALSO ACCEPTABLE: Allow patch updates only
lxml~=5.1.0  # Allows 5.1.x, not 5.2.0
```

### 3. Verify Package Integrity

```bash
# Use hash checking
pip install --require-hashes -r requirements.txt

# requirements.txt with hashes:
# lxml==5.1.0 \
#     --hash=sha256:abc123...
```

### 4. Scan for Vulnerabilities

```python
# Use bandit for security linting
# pip install bandit

# Run scan
# bandit -r src/calibre/

# Example issues bandit finds:
# - Use of exec()
# - Hardcoded passwords
# - SQL injection risks
# - Use of shell=True
# - Weak cryptography
```

---

## Code Review Checklist

### Security Review Checklist

**Use this when reviewing code**:

#### Input Validation
- [ ] All user inputs validated (type, range, format)?
- [ ] Whitelist validation used instead of blacklist?
- [ ] String inputs have length limits?
- [ ] Numeric inputs have range checks?

#### SQL Security
- [ ] All SQL queries use parameterized queries?
- [ ] No string formatting/concatenation in SQL?
- [ ] Dynamic column/table names whitelisted?

#### File Security
- [ ] File paths validated (no path traversal)?
- [ ] File types validated by content, not extension?
- [ ] File sizes limited?
- [ ] Filenames sanitized?
- [ ] Files stored outside web root?

#### Command Execution
- [ ] No use of os.system(), eval(), exec()?
- [ ] subprocess.run() uses list arguments, not strings?
- [ ] shell=False in subprocess calls?
- [ ] If shell=True required, inputs escaped with shlex.quote()?

#### Authentication/Authorization
- [ ] Passwords hashed with bcrypt/argon2?
- [ ] Session IDs cryptographically random?
- [ ] Authorization checks before sensitive operations?
- [ ] Rate limiting on authentication endpoints?

#### Cryptography
- [ ] Using secrets module for random numbers?
- [ ] Using vetted libraries (cryptography, not custom)?
- [ ] No hardcoded secrets/keys in code?

#### Error Handling
- [ ] Error messages don't leak sensitive information?
- [ ] Stack traces not shown to users in production?
- [ ] Failures are logged for security monitoring?

#### HTTP Server
- [ ] Security headers added to responses?
- [ ] CORS configured correctly?
- [ ] CSRF protection on state-changing endpoints?
- [ ] Request size limits enforced?

---

## Common Vulnerabilities

### 1. Directory Listing

```python
# ❌ VULNERABLE: Exposing directory structure
@endpoint('/files/{path}')
def serve_file(ctx, rd, path):
    if os.path.isdir(path):
        # Lists all files in directory!
        return {'files': os.listdir(path)}
    else:
        return send_file(path)

# ✅ FIXED: Don't allow directory listing
@endpoint('/files/{path}')
def serve_file(ctx, rd, path):
    # Validate path
    path = validate_path(path)

    if not os.path.isfile(path):
        raise FileNotFoundError()

    return send_file(path)
```

### 2. Information Disclosure

```python
# ❌ VULNERABLE: Leaking information in errors
@endpoint('/api/books/{book_id}')
def get_book(ctx, rd, book_id):
    try:
        book = ctx.db.get_metadata(book_id)
        return {'book': book}
    except Exception as e:
        # Reveals database structure, file paths!
        return {'error': str(e), 'traceback': traceback.format_exc()}

# ✅ FIXED: Generic error messages
@endpoint('/api/books/{book_id}')
def get_book(ctx, rd, book_id):
    try:
        book = ctx.db.get_metadata(book_id)
        return {'book': book}
    except Exception as e:
        # Log full error for debugging
        logger.error(f"Error getting book {book_id}", exc_info=True)

        # Return generic message to user
        return {'error': 'Book not found'}, 404
```

### 3. XML External Entity (XXE)

```python
import lxml.etree as ET

# ❌ VULNERABLE: XXE attack
def parse_opf(opf_content):
    # Can read local files or make network requests!
    tree = ET.fromstring(opf_content)
    return tree

# ✅ FIXED: Disable external entities
def parse_opf(opf_content):
    parser = ET.XMLParser(
        resolve_entities=False,  # Disable XXE
        no_network=True,         # Disable network access
        remove_blank_text=True
    )
    tree = ET.fromstring(opf_content, parser=parser)
    return tree
```

### 4. Insecure Deserialization

```python
import pickle

# ❌ CRITICAL VULNERABILITY: Arbitrary code execution
def load_data(data):
    # pickle can execute arbitrary code!
    return pickle.loads(data)

# ✅ SAFE: Use JSON instead
import json

def load_data(data):
    # JSON is data-only, can't execute code
    return json.loads(data)

# If you MUST use pickle, validate source
def load_trusted_pickle(file_path):
    # Only load from trusted sources
    if not is_trusted_path(file_path):
        raise ValueError("Untrusted pickle file")

    with open(file_path, 'rb') as f:
        return pickle.load(f)
```

---

## Security Testing

### 1. Automated Security Testing

```python
# tests/security/test_sql_injection.py
import pytest

def test_sql_injection_author_search():
    """Test that SQL injection is prevented"""

    # SQL injection payload
    malicious_input = "'; DROP TABLE books; --"

    # Should not execute SQL
    results = db.search_authors(malicious_input)

    # Should treat as literal string search
    assert len(results) == 0  # No author with that name

    # Verify books table still exists
    assert db.table_exists('books')

def test_sql_injection_parameterized():
    """Test that parameterized queries prevent injection"""

    # Try various injection payloads
    payloads = [
        "' OR '1'='1",
        "'; DROP TABLE books; --",
        "1' UNION SELECT * FROM users--",
        "admin'--",
    ]

    for payload in payloads:
        # Should not break or expose data
        with pytest.raises(ValueError):
            db.get_metadata(payload)
```

### 2. Path Traversal Testing

```python
# tests/security/test_path_traversal.py
def test_path_traversal_prevention():
    """Test that path traversal is prevented"""

    payloads = [
        "../../../etc/passwd",
        "..\\..\\..\\windows\\system32\\config\\sam",
        "....//....//....//etc/passwd",
        "%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd",  # URL encoded
    ]

    for payload in payloads:
        with pytest.raises((ValueError, FileNotFoundError)):
            get_book_file(payload)
```

### 3. Authentication Testing

```python
# tests/security/test_authentication.py
def test_password_hashing():
    """Test that passwords are hashed"""

    user = User.create("alice", "password123")

    # Password should not be stored in plaintext
    assert user.password_hash != b"password123"
    assert len(user.password_hash) > 20  # Hashes are long

    # Should be able to verify correct password
    assert user.check_password("password123")

    # Should reject wrong password
    assert not user.check_password("wrongpassword")

def test_timing_attack_resistance():
    """Test that password check is timing-safe"""
    import time

    user = User.create("alice", "password123")

    # Measure time for wrong password
    times_wrong = []
    for _ in range(100):
        start = time.perf_counter()
        user.check_password("wrong")
        times_wrong.append(time.perf_counter() - start)

    # Measure time for partially correct password
    times_partial = []
    for _ in range(100):
        start = time.perf_counter()
        user.check_password("password")  # Close but wrong
        times_partial.append(time.perf_counter() - start)

    # Timing should be similar (bcrypt is timing-safe)
    avg_wrong = sum(times_wrong) / len(times_wrong)
    avg_partial = sum(times_partial) / len(times_partial)

    # Difference should be small (<10%)
    assert abs(avg_wrong - avg_partial) / avg_wrong < 0.1
```

### 4. Fuzzing

```python
# tests/security/test_fuzzing.py
import hypothesis
from hypothesis import strategies as st

@hypothesis.given(st.text())
def test_filename_sanitization_fuzzing(filename):
    """Fuzz test filename sanitization"""

    # Should never raise exception
    try:
        sanitized = sanitize_filename(filename)

        # Should never contain path separators
        assert '/' not in sanitized
        assert '\\' not in sanitized

        # Should never contain null bytes
        assert '\x00' not in sanitized

    except ValueError:
        # Empty filename is acceptable error
        pass
```

---

## Quick Reference

### Security Command Cheatsheet

```bash
# Check for vulnerabilities
pip install safety
safety check

pip install pip-audit
pip-audit

# Security linting
pip install bandit
bandit -r src/

# Check for secrets in code
pip install detect-secrets
detect-secrets scan

# Update dependencies
pip list --outdated
pip install --upgrade package-name
```

### Common Secure Patterns

```python
# SQL: Always use parameterized queries
cursor.execute("SELECT * FROM books WHERE id = ?", (book_id,))

# Files: Validate paths
path = os.path.realpath(os.path.join(base_path, user_path))
if not path.startswith(os.path.realpath(base_path)):
    raise ValueError("Path traversal")

# Commands: Use list arguments
subprocess.run(['command', arg1, arg2], shell=False)

# Random: Use secrets module
token = secrets.token_urlsafe(32)

# Passwords: Use bcrypt
bcrypt.hashpw(password.encode(), bcrypt.gensalt())
```

---

## Next Steps

Now that you understand security:

1. **Practice**: Review existing Calibre code for vulnerabilities
2. **Read**: [TESTING_GUIDE.md](./TESTING_GUIDE.md) - Write security tests
3. **Read**: [CODE_REVIEW_GUIDE.md](./CODE_REVIEW_GUIDE.md) - Review for security
4. **Read**: [DEVELOPMENT_WORKFLOW.md](./DEVELOPMENT_WORKFLOW.md) - Secure workflow

---

## Additional Resources

### Official Documentation
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Cheat Sheets](https://cheatsheetseries.owasp.org/)
- [Python Security Best Practices](https://python.readthedocs.io/en/stable/library/security_warnings.html)
- [CWE Top 25](https://cwe.mitre.org/top25/)

### Tools
- [Bandit](https://github.com/PyCQA/bandit) - Python security linter
- [Safety](https://github.com/pyupio/safety) - Dependency vulnerability scanner
- [pip-audit](https://github.com/pypa/pip-audit) - Audit Python packages
- [detect-secrets](https://github.com/Yelp/detect-secrets) - Find secrets in code

### Books
- "The Web Application Hacker's Handbook" - Stuttard & Pinto
- "Black Hat Python" - Justin Seitz
- "Serious Cryptography" - Jean-Philippe Aumasson

---

**Remember**: Security is not a feature you add at the end. It must be built in from the start. Every line of code that touches user input is a potential vulnerability.
