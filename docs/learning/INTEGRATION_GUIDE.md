# Calibre Integration Guide

> **For React developers**: This guide covers integrating third-party services and APIs, similar to integrating Stripe, Auth0, or Google Analytics in React apps. Calibre's plugin system is like React's component ecosystem, and its integrations are like connecting to external APIs.

---

## Table of Contents

1. [Overview](#overview)
2. [Metadata Source Integrations](#metadata-source-integrations)
3. [Cloud Storage Integrations](#cloud-storage-integrations)
4. [Email Integration](#email-integration)
5. [Device Integrations](#device-integrations)
6. [News Download Integration](#news-download-integration)
7. [Plugin Development](#plugin-development)
8. [External API Integration](#external-api-integration)
9. [Database Integrations](#database-integrations)
10. [Authentication Integrations](#authentication-integrations)
11. [Best Practices](#best-practices)

---

## Overview

Calibre integrates with numerous third-party services:

### Integration Types

| Type | Examples | Use Case |
|------|----------|----------|
| **Metadata Sources** | Google Books, Amazon, Goodreads | Download book metadata |
| **Cloud Storage** | Dropbox, Google Drive, OneDrive | Sync library |
| **Email** | Gmail, SMTP | Send books to devices |
| **Devices** | Kindle, Kobo, iPad | Transfer books |
| **News** | RSS feeds, websites | Download news |
| **Plugins** | Custom functionality | Extend Calibre |

**React equivalent**: Like integrating with Firebase, Supabase, or third-party APIs

---

## Metadata Source Integrations

### Built-in Sources

**Location**: [src/calibre/ebooks/metadata/sources/](../../src/calibre/ebooks/metadata/sources/)

Calibre includes metadata sources for:
- Google Books
- Amazon
- Open Library
- Goodreads (unofficial API)
- ISBNdb
- WorldCat

### Creating Custom Metadata Source

```python
from calibre.ebooks.metadata.sources.base import Source
from calibre.ebooks.metadata.book.base import Metadata

class MyMetadataSource(Source):
    """Custom metadata source plugin"""

    name = 'My Source'
    description = 'Downloads metadata from my source'
    supported_platforms = ['windows', 'osx', 'linux']
    author = 'Your Name'
    version = (1, 0, 0)

    capabilities = frozenset(['identify', 'cover'])
    touched_fields = frozenset([
        'title', 'authors', 'identifier:mysource',
        'rating', 'publisher', 'pubdate', 'comments'
    ])

    supports_gzip_transfer_encoding = True

    def identify(self, log, result_queue, abort, title=None, authors=None,
                identifiers={}, timeout=30):
        """
        Identify book from metadata

        Args:
            log: Log object for debugging
            result_queue: Queue to put results in
            abort: Event to check if search was aborted
            title: Book title
            authors: List of author names
            identifiers: Dict of identifiers (isbn, amazon, etc.)
            timeout: Timeout in seconds
        """
        # Check if aborted
        if abort.is_set():
            return

        # Query your API
        results = self.query_api(title, authors, identifiers)

        # Process each result
        for result in results:
            if abort.is_set():
                break

            # Create metadata object
            metadata = Metadata(result['title'])

            # Add authors
            if 'authors' in result:
                metadata.authors = result['authors']

            # Add identifier
            if 'id' in result:
                metadata.set_identifier('mysource', result['id'])

            # Add other fields
            if 'isbn' in result:
                metadata.isbn = result['isbn']

            if 'publisher' in result:
                metadata.publisher = result['publisher']

            if 'pubdate' in result:
                from datetime import datetime
                metadata.pubdate = datetime.strptime(result['pubdate'], '%Y-%m-%d')

            if 'rating' in result:
                metadata.rating = float(result['rating'])

            if 'description' in result:
                metadata.comments = result['description']

            # Add to result queue
            result_queue.put(metadata)

    def download_cover(self, log, result_queue, abort, title=None, authors=None,
                      identifiers={}, timeout=30, get_best_cover=False):
        """
        Download cover image

        Args:
            log: Log object
            result_queue: Queue to put (url, data) tuples
            abort: Abort event
            title: Book title
            authors: Author list
            identifiers: Identifiers dict
            timeout: Timeout in seconds
            get_best_cover: If True, only return highest quality
        """
        if abort.is_set():
            return

        # Get cover URL
        cover_url = self.get_cover_url(identifiers)

        if cover_url:
            # Download cover
            br = self.browser
            try:
                log.info(f'Downloading cover from: {cover_url}')
                response = br.open_novisit(cover_url, timeout=timeout)
                data = response.read()

                # Add to queue
                result_queue.put((cover_url, data))

            except Exception as e:
                log.exception(f'Failed to download cover: {e}')

    def query_api(self, title, authors, identifiers):
        """Query your API and return results"""
        import requests

        # Build query
        params = {'title': title}
        if authors:
            params['author'] = authors[0]

        # Make request
        response = requests.get(
            'https://api.example.com/search',
            params=params,
            timeout=30
        )

        # Parse response
        return response.json()['results']

    def get_cover_url(self, identifiers):
        """Get cover URL from identifiers"""
        if 'mysource' in identifiers:
            book_id = identifiers['mysource']
            return f'https://api.example.com/covers/{book_id}.jpg'
        return None
```

### Using Metadata Source

```python
from calibre.ebooks.metadata.sources.identify import identify

# Search for book
results = identify(
    title='1984',
    authors=['George Orwell'],
    identifiers={'isbn': '9780451524935'},
    sources=['google', 'amazon', 'mysource'],
    timeout=30
)

# results is list of Metadata objects
for metadata in results:
    print(f"{metadata.title} by {', '.join(metadata.authors)}")
    if metadata.isbn:
        print(f"ISBN: {metadata.isbn}")
    if metadata.rating:
        print(f"Rating: {metadata.rating}")
```

### API Best Practices

```python
class GoodMetadataSource(Source):
    """Best practices example"""

    def identify(self, log, result_queue, abort, **kwargs):
        # 1. Respect rate limits
        import time
        time.sleep(1)  # 1 request per second

        # 2. Use caching
        cache_key = f"{kwargs.get('title')}:{kwargs.get('authors')}"
        if cache_key in self.cache:
            log.info('Using cached results')
            result_queue.put(self.cache[cache_key])
            return

        # 3. Handle errors gracefully
        try:
            results = self.query_api(**kwargs)
        except requests.ConnectionError:
            log.error('Connection failed, API may be down')
            return
        except requests.Timeout:
            log.error('Request timeout')
            return
        except Exception as e:
            log.exception(f'Unexpected error: {e}')
            return

        # 4. Validate results
        for result in results:
            if not result.get('title'):
                log.warning('Skipping result without title')
                continue

            metadata = self.create_metadata(result)

            # 5. Check abort frequently
            if abort.is_set():
                return

            result_queue.put(metadata)

        # 6. Cache results
        if results:
            self.cache[cache_key] = results[0]
```

**Example**: [src/calibre/ebooks/metadata/sources/google.py](../../src/calibre/ebooks/metadata/sources/google.py)

---

## Cloud Storage Integrations

### Dropbox Integration

```python
import dropbox
from dropbox.exceptions import ApiError

class DropboxSync:
    """Sync Calibre library to Dropbox"""

    def __init__(self, access_token):
        self.dbx = dropbox.Dropbox(access_token)

    def upload_book(self, local_path, dropbox_path):
        """Upload book to Dropbox"""
        with open(local_path, 'rb') as f:
            try:
                # Upload file
                self.dbx.files_upload(
                    f.read(),
                    dropbox_path,
                    mode=dropbox.files.WriteMode.overwrite
                )
                print(f'Uploaded: {dropbox_path}')

            except ApiError as e:
                print(f'Error uploading: {e}')

    def download_book(self, dropbox_path, local_path):
        """Download book from Dropbox"""
        try:
            # Download file
            metadata, response = self.dbx.files_download(dropbox_path)

            with open(local_path, 'wb') as f:
                f.write(response.content)

            print(f'Downloaded: {local_path}')

        except ApiError as e:
            print(f'Error downloading: {e}')

    def sync_library(self, library_path, dropbox_folder):
        """Sync entire library to Dropbox"""
        import os

        for root, dirs, files in os.walk(library_path):
            for filename in files:
                if filename.endswith(('.epub', '.mobi', '.pdf')):
                    local_path = os.path.join(root, filename)

                    # Construct Dropbox path
                    rel_path = os.path.relpath(local_path, library_path)
                    dropbox_path = os.path.join(dropbox_folder, rel_path)

                    # Upload
                    self.upload_book(local_path, dropbox_path)

# Usage
sync = DropboxSync('YOUR_ACCESS_TOKEN')
sync.sync_library('/path/to/calibre/library', '/Calibre')
```

### Google Drive Integration

```python
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from google.oauth2.credentials import Credentials

class GoogleDriveSync:
    """Sync Calibre library to Google Drive"""

    def __init__(self, credentials):
        self.service = build('drive', 'v3', credentials=credentials)

    def create_folder(self, name, parent_id=None):
        """Create folder in Google Drive"""
        file_metadata = {
            'name': name,
            'mimeType': 'application/vnd.google-apps.folder'
        }

        if parent_id:
            file_metadata['parents'] = [parent_id]

        folder = self.service.files().create(
            body=file_metadata,
            fields='id'
        ).execute()

        return folder['id']

    def upload_file(self, local_path, parent_id=None):
        """Upload file to Google Drive"""
        import os

        filename = os.path.basename(local_path)

        file_metadata = {'name': filename}
        if parent_id:
            file_metadata['parents'] = [parent_id]

        media = MediaFileUpload(local_path, resumable=True)

        file = self.service.files().create(
            body=file_metadata,
            media_body=media,
            fields='id'
        ).execute()

        print(f'Uploaded: {filename} (ID: {file["id"]})')
        return file['id']

    def list_files(self, parent_id=None):
        """List files in folder"""
        query = f"'{parent_id}' in parents" if parent_id else None

        results = self.service.files().list(
            q=query,
            fields='files(id, name, mimeType)'
        ).execute()

        return results.get('files', [])

# Usage with OAuth2
# See: https://developers.google.com/drive/api/v3/quickstart/python
```

---

## Email Integration

### SMTP Email

**Send books via email** (e.g., to Kindle):

```python
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.base import MIMEBase
from email.mime.text import MIMEText
from email import encoders

class EmailSender:
    """Send books via email"""

    def __init__(self, smtp_host, smtp_port, username, password):
        self.smtp_host = smtp_host
        self.smtp_port = smtp_port
        self.username = username
        self.password = password

    def send_book(self, book_path, to_email, from_email=None):
        """Send book as email attachment"""
        import os

        if from_email is None:
            from_email = self.username

        # Create message
        msg = MIMEMultipart()
        msg['From'] = from_email
        msg['To'] = to_email
        msg['Subject'] = 'Book from Calibre'

        # Add body
        body = 'Your book is attached.'
        msg.attach(MIMEText(body, 'plain'))

        # Attach book file
        filename = os.path.basename(book_path)

        with open(book_path, 'rb') as f:
            attachment = MIMEBase('application', 'octet-stream')
            attachment.set_payload(f.read())

        encoders.encode_base64(attachment)
        attachment.add_header(
            'Content-Disposition',
            f'attachment; filename= {filename}'
        )
        msg.attach(attachment)

        # Send email
        try:
            server = smtplib.SMTP(self.smtp_host, self.smtp_port)
            server.starttls()
            server.login(self.username, self.password)
            server.send_message(msg)
            server.quit()

            print(f'Sent {filename} to {to_email}')

        except Exception as e:
            print(f'Error sending email: {e}')

# Usage
sender = EmailSender(
    smtp_host='smtp.gmail.com',
    smtp_port=587,
    username='your_email@gmail.com',
    password='your_app_password'
)

sender.send_book(
    book_path='/library/book.mobi',
    to_email='your_kindle@kindle.com'
)
```

### Gmail API Integration

```python
from googleapiclient.discovery import build
from email.mime.text import MIMEText
import base64

class GmailSender:
    """Send books via Gmail API"""

    def __init__(self, credentials):
        self.service = build('gmail', 'v1', credentials=credentials)

    def send_book(self, book_path, to_email):
        """Send book via Gmail"""
        import os
        from email.mime.multipart import MIMEMultipart
        from email.mime.base import MIMEBase
        from email import encoders

        # Create message
        msg = MIMEMultipart()
        msg['to'] = to_email
        msg['subject'] = 'Book from Calibre'

        # Attach book
        with open(book_path, 'rb') as f:
            attachment = MIMEBase('application', 'octet-stream')
            attachment.set_payload(f.read())

        encoders.encode_base64(attachment)
        filename = os.path.basename(book_path)
        attachment.add_header('Content-Disposition', f'attachment; filename={filename}')
        msg.attach(attachment)

        # Send
        raw = base64.urlsafe_b64encode(msg.as_bytes()).decode()
        message = {'raw': raw}

        self.service.users().messages().send(
            userId='me',
            body=message
        ).execute()

        print(f'Sent {filename} to {to_email}')
```

**Example in Calibre**: [src/calibre/devices/kindle/driver.py](../../src/calibre/devices/kindle/driver.py)

---

## Device Integrations

### USB Device Detection

```python
import usb.core
import usb.util

class DeviceDetector:
    """Detect USB ebook readers"""

    # Known device vendor/product IDs
    DEVICES = {
        'kindle': {'vendor': 0x1949, 'product': 0x0004},
        'kobo': {'vendor': 0x2237, 'product': 0x4161},
        'nook': {'vendor': 0x2080, 'product': 0x0002},
    }

    def find_device(self, device_type):
        """Find specific device type"""
        device_info = self.DEVICES.get(device_type)
        if not device_info:
            return None

        device = usb.core.find(
            idVendor=device_info['vendor'],
            idProduct=device_info['product']
        )

        return device

    def list_devices(self):
        """List all connected ebook readers"""
        found = []

        for name, info in self.DEVICES.items():
            device = usb.core.find(
                idVendor=info['vendor'],
                idProduct=info['product']
            )

            if device:
                found.append({
                    'name': name,
                    'vendor': info['vendor'],
                    'product': info['product'],
                    'device': device
                })

        return found

# Usage
detector = DeviceDetector()

# Find Kindle
kindle = detector.find_device('kindle')
if kindle:
    print('Kindle connected!')

# List all devices
devices = detector.list_devices()
for device in devices:
    print(f"Found: {device['name']}")
```

### Kindle Integration

```python
import os
import shutil

class KindleDevice:
    """Manage Kindle device"""

    def __init__(self, mount_point):
        """
        Args:
            mount_point: Path where Kindle is mounted
                        (e.g., /media/Kindle or E:\ on Windows)
        """
        self.mount_point = mount_point
        self.documents_path = os.path.join(mount_point, 'documents')

    def is_connected(self):
        """Check if Kindle is connected"""
        return os.path.exists(self.documents_path)

    def send_book(self, book_path):
        """Copy book to Kindle"""
        if not self.is_connected():
            raise IOError('Kindle not connected')

        filename = os.path.basename(book_path)
        dest_path = os.path.join(self.documents_path, filename)

        shutil.copy2(book_path, dest_path)
        print(f'Sent {filename} to Kindle')

    def list_books(self):
        """List books on Kindle"""
        if not self.is_connected():
            return []

        books = []
        for filename in os.listdir(self.documents_path):
            if filename.endswith(('.mobi', '.azw', '.azw3')):
                books.append(filename)

        return books

    def get_device_info(self):
        """Get Kindle device information"""
        info_file = os.path.join(self.mount_point, 'system', 'version.txt')

        if os.path.exists(info_file):
            with open(info_file, 'r') as f:
                return f.read()

        return None

# Usage
kindle = KindleDevice('/media/Kindle')

if kindle.is_connected():
    kindle.send_book('/library/book.mobi')
    books = kindle.list_books()
    print(f'Books on Kindle: {books}')
```

**Example in Calibre**: [src/calibre/devices/](../../src/calibre/devices/)

---

## News Download Integration

### RSS Feed Integration

```python
import feedparser
from datetime import datetime

class NewsFeed:
    """Download news from RSS feed"""

    def __init__(self, feed_url):
        self.feed_url = feed_url

    def fetch_articles(self, max_articles=10):
        """Fetch latest articles from feed"""
        feed = feedparser.parse(self.feed_url)

        articles = []
        for entry in feed.entries[:max_articles]:
            article = {
                'title': entry.get('title', 'Untitled'),
                'url': entry.get('link'),
                'author': entry.get('author', 'Unknown'),
                'published': self.parse_date(entry.get('published')),
                'summary': entry.get('summary', ''),
                'content': entry.get('content', [{}])[0].get('value', '')
            }
            articles.append(article)

        return articles

    def parse_date(self, date_string):
        """Parse date from RSS feed"""
        if not date_string:
            return datetime.now()

        from dateutil import parser
        return parser.parse(date_string)

    def create_ebook(self, articles, output_path):
        """Create ebook from articles"""
        from ebooklib import epub

        book = epub.EpubBook()

        # Set metadata
        book.set_identifier('news_feed_001')
        book.set_title('News Digest')
        book.set_language('en')

        # Add articles as chapters
        for i, article in enumerate(articles):
            chapter = epub.EpubHtml(
                title=article['title'],
                file_name=f'chapter_{i}.xhtml',
                lang='en'
            )

            chapter.content = f'''
            <h1>{article['title']}</h1>
            <p><em>By {article['author']} on {article['published']}</em></p>
            <p>{article['content']}</p>
            '''

            book.add_item(chapter)

        # Define Table of Contents
        book.toc = tuple(book.get_items_of_type(epub.EpubHtml))

        # Add navigation files
        book.add_item(epub.EpubNcx())
        book.add_item(epub.EpubNav())

        # Define spine
        book.spine = ['nav'] + list(book.get_items_of_type(epub.EpubHtml))

        # Write to file
        epub.write_epub(output_path, book)

        print(f'Created ebook: {output_path}')

# Usage
feed = NewsFeed('https://example.com/rss')
articles = feed.fetch_articles(max_articles=20)
feed.create_ebook(articles, 'news_digest.epub')
```

### Web Scraping Integration

```python
import requests
from bs4 import BeautifulSoup

class WebArticle:
    """Scrape article from website"""

    def __init__(self, url):
        self.url = url

    def fetch_content(self):
        """Fetch and parse article content"""
        response = requests.get(self.url)
        response.raise_for_status()

        soup = BeautifulSoup(response.content, 'html.parser')

        # Extract title
        title_tag = soup.find('h1')
        title = title_tag.text.strip() if title_tag else 'Untitled'

        # Extract author
        author_tag = soup.find('meta', {'name': 'author'})
        author = author_tag['content'] if author_tag else 'Unknown'

        # Extract content (customize for each site)
        content_div = soup.find('div', class_='article-content')
        content = content_div.get_text(strip=True) if content_div else ''

        return {
            'title': title,
            'author': author,
            'content': content,
            'url': self.url
        }

# Usage
article = WebArticle('https://example.com/article')
content = article.fetch_content()
print(content['title'])
```

**Example in Calibre**: [recipes/](../../recipes/) - 1000+ news source recipes

---

## Plugin Development

### Plugin Types

Calibre supports various plugin types:

1. **Interface Action** - Add toolbar button
2. **Metadata Source** - Download metadata
3. **Metadata Download** - Download covers
4. **Format Conversion** - Add input/output formats
5. **Device Driver** - Support new devices
6. **Preferences** - Add settings panel
7. **Catalog** - Generate book catalogs

### Complete Plugin Example

**File structure**:
```
my_plugin/
├── __init__.py          # Plugin entry point
├── plugin-import-name-my_plugin.txt
├── ui.py                # User interface
├── config.py            # Configuration
└── icon.png             # Plugin icon
```

**`__init__.py`**:
```python
from calibre.customize import InterfaceActionBase

class MyPluginAction(InterfaceActionBase):
    """My custom plugin"""

    name = 'My Plugin'
    description = 'Adds custom functionality to Calibre'
    supported_platforms = ['windows', 'osx', 'linux']
    author = 'Your Name'
    version = (1, 0, 0)
    minimum_calibre_version = (5, 0, 0)

    actual_plugin = 'calibre_plugins.my_plugin.ui:MyPluginUI'

    def is_customizable(self):
        return True

    def config_widget(self):
        from calibre_plugins.my_plugin.config import ConfigWidget
        return ConfigWidget()

    def save_settings(self, config_widget):
        config_widget.save_settings()
```

**`ui.py`**:
```python
from PyQt6.QtWidgets import QDialog, QVBoxLayout, QPushButton, QLabel
from PyQt6.QtGui import QIcon
from calibre.gui2.actions import InterfaceAction

class MyPluginUI(InterfaceAction):
    """Plugin user interface"""

    name = 'My Plugin'
    action_spec = ('My Plugin', None, 'My plugin tooltip', 'Ctrl+Shift+M')
    action_type = 'current'

    def genesis(self):
        """Initialize plugin"""
        # Set icon
        icon = QIcon('path/to/icon.png')
        self.qaction.setIcon(icon)

        # Connect action
        self.qaction.triggered.connect(self.show_dialog)

    def show_dialog(self):
        """Show plugin dialog"""
        # Get selected books
        rows = self.gui.library_view.selectionModel().selectedRows()
        if not rows:
            from calibre.gui2 import error_dialog
            error_dialog(
                self.gui,
                'No Selection',
                'Please select at least one book',
                show=True
            )
            return

        book_ids = [self.gui.library_view.model().id(row) for row in rows]

        # Show dialog
        dialog = MyDialog(self.gui, book_ids)
        dialog.exec()

class MyDialog(QDialog):
    """Plugin dialog"""

    def __init__(self, parent, book_ids):
        super().__init__(parent)

        self.book_ids = book_ids
        self.db = parent.current_db

        self.setWindowTitle('My Plugin')

        layout = QVBoxLayout(self)

        # Add label
        label = QLabel(f'Selected {len(book_ids)} books')
        layout.addWidget(label)

        # Add button
        button = QPushButton('Process Books')
        button.clicked.connect(self.process_books)
        layout.addWidget(button)

    def process_books(self):
        """Process selected books"""
        for book_id in self.book_ids:
            metadata = self.db.get_metadata(book_id)
            print(f'Processing: {metadata.title}')

            # Do something with book
            # ...

        self.accept()
```

**`config.py`**:
```python
from PyQt6.QtWidgets import QWidget, QVBoxLayout, QLabel, QLineEdit

class ConfigWidget(QWidget):
    """Plugin configuration widget"""

    def __init__(self):
        super().__init__()

        layout = QVBoxLayout(self)

        # Add settings
        layout.addWidget(QLabel('API Key:'))
        self.api_key_edit = QLineEdit()
        layout.addWidget(self.api_key_edit)

        # Load settings
        self.load_settings()

    def load_settings(self):
        """Load plugin settings"""
        from calibre.utils.config import JSONConfig
        prefs = JSONConfig('plugins/my_plugin')

        self.api_key_edit.setText(prefs.get('api_key', ''))

    def save_settings(self):
        """Save plugin settings"""
        from calibre.utils.config import JSONConfig
        prefs = JSONConfig('plugins/my_plugin')

        prefs['api_key'] = self.api_key_edit.text()
```

### Installing Plugin

```bash
# 1. Create ZIP file
zip -r my_plugin.zip my_plugin/

# 2. Install via GUI
# Calibre → Preferences → Plugins → Load plugin from file

# Or via command line
calibre-customize -a my_plugin.zip
```

**Example plugins**: [src/calibre/customize/builtins.py](../../src/calibre/customize/builtins.py)

---

## External API Integration

### REST API Client

```python
import requests
from typing import Optional, Dict, List

class CalibreAPIClient:
    """Client for Calibre Content Server API"""

    def __init__(self, base_url: str, username: Optional[str] = None,
                 password: Optional[str] = None):
        self.base_url = base_url.rstrip('/')
        self.session = requests.Session()

        if username and password:
            self.login(username, password)

    def login(self, username: str, password: str):
        """Authenticate with server"""
        response = self.session.post(
            f'{self.base_url}/login',
            json={'username': username, 'password': password}
        )
        response.raise_for_status()

    def get_books(self, limit: int = 50, offset: int = 0) -> List[Dict]:
        """Get list of books"""
        response = self.session.get(
            f'{self.base_url}/api/books',
            params={'limit': limit, 'offset': offset}
        )
        response.raise_for_status()
        return response.json()['books']

    def get_book(self, book_id: int) -> Dict:
        """Get single book metadata"""
        response = self.session.get(f'{self.base_url}/api/books/{book_id}')
        response.raise_for_status()
        return response.json()

    def search_books(self, query: str) -> List[Dict]:
        """Search books"""
        response = self.session.get(
            f'{self.base_url}/api/search',
            params={'q': query}
        )
        response.raise_for_status()
        return response.json()['results']

    def download_book(self, book_id: int, format: str, output_path: str):
        """Download book file"""
        response = self.session.get(
            f'{self.base_url}/api/books/{book_id}/{format}',
            stream=True
        )
        response.raise_for_status()

        with open(output_path, 'wb') as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)

# Usage
client = CalibreAPIClient('http://localhost:8080', 'admin', 'password')

# List books
books = client.get_books(limit=10)
for book in books:
    print(f"{book['title']} by {', '.join(book['authors'])}")

# Search
results = client.search_books('author:orwell')
print(f'Found {len(results)} books')

# Download
client.download_book(123, 'epub', 'book.epub')
```

---

## Database Integrations

### PostgreSQL Integration

**Migrate Calibre library to PostgreSQL**:

```python
import apsw
import psycopg2
from psycopg2.extras import execute_values

class PostgreSQLMigration:
    """Migrate Calibre SQLite to PostgreSQL"""

    def __init__(self, sqlite_path, pg_connection_string):
        self.sqlite_conn = apsw.Connection(sqlite_path)
        self.pg_conn = psycopg2.connect(pg_connection_string)

    def migrate_table(self, table_name):
        """Migrate single table"""
        # Get data from SQLite
        cursor = self.sqlite_conn.cursor()
        rows = cursor.execute(f'SELECT * FROM {table_name}').fetchall()

        # Get column names
        column_info = cursor.execute(f'PRAGMA table_info({table_name})').fetchall()
        columns = [col[1] for col in column_info]

        # Insert into PostgreSQL
        pg_cursor = self.pg_conn.cursor()

        insert_sql = f'''
        INSERT INTO {table_name} ({', '.join(columns)})
        VALUES %s
        '''

        execute_values(pg_cursor, insert_sql, rows)
        self.pg_conn.commit()

        print(f'Migrated {len(rows)} rows from {table_name}')

    def migrate_all(self):
        """Migrate all tables"""
        # Get table names
        cursor = self.sqlite_conn.cursor()
        tables = cursor.execute('''
            SELECT name FROM sqlite_master
            WHERE type='table' AND name NOT LIKE 'sqlite_%'
        ''').fetchall()

        for (table_name,) in tables:
            self.migrate_table(table_name)

# Usage
migration = PostgreSQLMigration(
    '/path/to/metadata.db',
    'postgresql://user:password@localhost/calibre'
)
migration.migrate_all()
```

---

## Authentication Integrations

### OAuth2 Integration

```python
from requests_oauthlib import OAuth2Session

class OAuth2Integration:
    """OAuth2 authentication for services"""

    def __init__(self, client_id, client_secret, authorization_url,
                 token_url, redirect_uri):
        self.client_id = client_id
        self.client_secret = client_secret
        self.authorization_url = authorization_url
        self.token_url = token_url
        self.redirect_uri = redirect_uri

    def get_authorization_url(self):
        """Get URL for user to authorize"""
        oauth = OAuth2Session(self.client_id, redirect_uri=self.redirect_uri)
        authorization_url, state = oauth.authorization_url(self.authorization_url)
        return authorization_url, state

    def fetch_token(self, authorization_response):
        """Exchange authorization code for access token"""
        oauth = OAuth2Session(self.client_id, redirect_uri=self.redirect_uri)

        token = oauth.fetch_token(
            self.token_url,
            authorization_response=authorization_response,
            client_secret=self.client_secret
        )

        return token

    def make_request(self, url, token):
        """Make authenticated request"""
        oauth = OAuth2Session(self.client_id, token=token)
        return oauth.get(url)

# Usage (Google Drive example)
oauth = OAuth2Integration(
    client_id='YOUR_CLIENT_ID',
    client_secret='YOUR_CLIENT_SECRET',
    authorization_url='https://accounts.google.com/o/oauth2/v2/auth',
    token_url='https://oauth2.googleapis.com/token',
    redirect_uri='http://localhost:8080/callback'
)

# Get authorization URL
auth_url, state = oauth.get_authorization_url()
print(f'Visit: {auth_url}')

# After user authorizes, exchange code for token
# authorization_response = 'http://localhost:8080/callback?code=...'
# token = oauth.fetch_token(authorization_response)

# Make requests
# response = oauth.make_request('https://www.googleapis.com/drive/v3/files', token)
```

---

## Best Practices

### 1. Error Handling

```python
# ✅ GOOD: Comprehensive error handling
def integrate_service(api_key):
    try:
        response = requests.get(
            'https://api.example.com/data',
            headers={'Authorization': f'Bearer {api_key}'},
            timeout=30
        )
        response.raise_for_status()
        return response.json()

    except requests.ConnectionError:
        logger.error('Connection failed - check network')
        raise IntegrationError('Cannot connect to service')

    except requests.Timeout:
        logger.error('Request timeout')
        raise IntegrationError('Service timeout')

    except requests.HTTPError as e:
        if e.response.status_code == 401:
            raise IntegrationError('Invalid API key')
        elif e.response.status_code == 429:
            raise IntegrationError('Rate limit exceeded')
        else:
            raise IntegrationError(f'HTTP error: {e}')

    except Exception as e:
        logger.exception('Unexpected error')
        raise IntegrationError(f'Integration failed: {e}')
```

### 2. Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=1):
    """Decorator to rate limit function calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = min_interval - elapsed

            if wait_time > 0:
                time.sleep(wait_time)

            result = func(*args, **kwargs)
            last_called[0] = time.time()

            return result
        return wrapper
    return decorator

# Usage
@rate_limit(calls_per_second=2)
def api_call():
    return requests.get('https://api.example.com/data')
```

### 3. Caching

```python
from functools import lru_cache
from datetime import datetime, timedelta

class CachedAPI:
    """API client with caching"""

    def __init__(self):
        self.cache = {}
        self.cache_duration = timedelta(hours=1)

    def get_data(self, key):
        """Get data with caching"""
        # Check cache
        if key in self.cache:
            data, timestamp = self.cache[key]
            if datetime.now() - timestamp < self.cache_duration:
                return data

        # Fetch from API
        data = self.fetch_from_api(key)

        # Cache result
        self.cache[key] = (data, datetime.now())

        return data

    def fetch_from_api(self, key):
        """Fetch from API"""
        response = requests.get(f'https://api.example.com/data/{key}')
        return response.json()
```

### 4. Retry Logic

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def api_call_with_retry():
    """API call with automatic retry"""
    response = requests.get('https://api.example.com/data', timeout=30)
    response.raise_for_status()
    return response.json()

# Retries up to 3 times with exponential backoff:
# 1st retry: wait 2 seconds
# 2nd retry: wait 4 seconds
# 3rd retry: wait 8 seconds
```

---

## Quick Reference

### Common Integration Patterns

```python
# HTTP API client
import requests
response = requests.get('https://api.example.com/data')
data = response.json()

# OAuth2
from requests_oauthlib import OAuth2Session
oauth = OAuth2Session(client_id)
token = oauth.fetch_token(token_url, code=auth_code)

# Database
import psycopg2
conn = psycopg2.connect('postgresql://user:pass@localhost/db')
cursor = conn.cursor()
cursor.execute('SELECT * FROM table')

# Email
import smtplib
server = smtplib.SMTP('smtp.gmail.com', 587)
server.starttls()
server.login(username, password)
server.send_message(msg)
```

---

## Next Steps

Now that you understand integrations:

1. **Practice**: Create a custom metadata source plugin
2. **Read**: [PLUGIN_DEVELOPMENT.md](./PLUGIN_DEVELOPMENT.md) - Deep dive
3. **Explore**: [src/calibre/customize/](../../src/calibre/customize/) - Built-in plugins
4. **Contribute**: Add new metadata sources or device drivers

---

## Additional Resources

### Official Documentation
- [Plugin Tutorial](https://manual.calibre-ebook.com/plugins.html)
- [API Documentation](https://manual.calibre-ebook.com/develop.html)
- [Device Driver Tutorial](https://manual.calibre-ebook.com/plugins.html#device-drivers)

### Examples
- [src/calibre/ebooks/metadata/sources/](../../src/calibre/ebooks/metadata/sources/) - Metadata sources
- [src/calibre/devices/](../../src/calibre/devices/) - Device drivers
- [recipes/](../../recipes/) - News sources

### Tools
- [requests](https://requests.readthedocs.io/) - HTTP client
- [feedparser](https://feedparser.readthedocs.io/) - RSS parser
- [BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/) - Web scraping
- [ebooklib](https://github.com/aerkalov/ebooklib) - EPUB creation

---

**Remember**: Good integrations are reliable, handle errors gracefully, respect rate limits, and provide clear feedback to users!
