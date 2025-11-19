# Frontend Architecture: PyQt6 Desktop GUI

Complete guide to Calibre's desktop interface, explaining how the PyQt6 GUI is structured and how it compares to modern React applications.

## Table of Contents

1. [Overview](#overview)
2. [Application Structure](#application-structure)
3. [Main Window Architecture](#main-window-architecture)
4. [Widgets and Components](#widgets-and-components)
5. [Layouts and Styling](#layouts-and-styling)
6. [Event System (Signals/Slots)](#event-system-signalsslots)
7. [State Management](#state-management)
8. [Model-View Architecture](#model-view-architecture)
9. [Dialogs and Modals](#dialogs-and-modals)
10. [Menus and Toolbars](#menus-and-toolbars)
11. [Custom Widgets](#custom-widgets)
12. [Threading and Async](#threading-and-async)
13. [Performance Optimization](#performance-optimization)
14. [Comparison to React](#comparison-to-react)

---

## Overview

### Desktop vs Web Architecture

```
Calibre Desktop (PyQt6)                React Web App
========================                ==============
QApplication                            ReactDOM.render()
    ↓                                       ↓
MainWindow (QMainWindow)                <App />
    ↓                                       ↓
Widgets (QWidget, QLabel, etc.)         Components (<Button>, <Input>)
    ↓                                       ↓
Layouts (QVBoxLayout, etc.)             CSS/Flexbox/Grid
    ↓                                       ↓
Signals/Slots (events)                  Props/Callbacks (events)
```

### Key Differences

| Aspect | PyQt6 Desktop | React Web |
|--------|---------------|-----------|
| **Rendering** | Native OS widgets | Browser DOM |
| **Styling** | QSS (Qt Style Sheets) | CSS/CSS-in-JS |
| **State** | Object properties | useState/Redux |
| **Events** | Signals → Slots | Props → Callbacks |
| **Updates** | Manual refresh | Virtual DOM diffing |
| **Layout** | Layout managers | Flexbox/Grid |
| **Build** | Python imports | Webpack/Vite |

---

## Application Structure

### Entry Point

[src/calibre/gui2/main.py](../../src/calibre/gui2/main.py):
```python
from PyQt6.QtWidgets import QApplication
from PyQt6.QtCore import Qt
import sys

def main():
    """
    Application entry point

    Similar to ReactDOM.render() in React apps
    """
    # Create application instance (singleton)
    app = QApplication(sys.argv)

    # Set application metadata
    app.setApplicationName('Calibre')
    app.setApplicationVersion('7.0.0')
    app.setOrganizationName('Kovid Goyal')

    # Enable high DPI scaling
    app.setAttribute(Qt.ApplicationAttribute.AA_EnableHighDpiScaling)
    app.setAttribute(Qt.ApplicationAttribute.AA_UseHighDpiPixmaps)

    # Create and show main window
    main_window = MainWindow()
    main_window.show()

    # Start event loop (blocking)
    sys.exit(app.exec())

if __name__ == '__main__':
    main()
```

**💭 React Comparison**:
```javascript
// React entry point:
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'

const root = ReactDOM.createRoot(document.getElementById('root'))
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
)
```

Key differences:
- PyQt6: `app.exec()` starts event loop (blocking)
- React: Event loop already running (browser)

---

## Main Window Architecture

### MainWindow Structure

[src/calibre/gui2/main.py](../../src/calibre/gui2/main.py#L100):
```python
from PyQt6.QtWidgets import (
    QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
    QSplitter, QMenuBar, QToolBar, QStatusBar
)
from PyQt6.QtCore import Qt, pyqtSignal

class MainWindow(QMainWindow):
    """
    Main application window

    Structure:
    ┌─────────────────────────────────────┐
    │ Menu Bar                            │
    ├─────────────────────────────────────┤
    │ Tool Bar                            │
    ├──────────┬─────────────┬────────────┤
    │          │             │            │
    │ Tag      │ Book List   │ Book       │
    │ Browser  │ (Table)     │ Details    │
    │          │             │            │
    ├──────────┴─────────────┴────────────┤
    │ Status Bar                          │
    └─────────────────────────────────────┘
    """

    # Define signals
    book_selected = pyqtSignal(int)  # Emits book_id
    books_changed = pyqtSignal()

    def __init__(self, db_path=None):
        super().__init__()

        # Load database
        self.db = load_database(db_path)

        # Window setup
        self.setWindowTitle('Calibre - E-book Management')
        self.setGeometry(100, 100, 1400, 900)

        # Create UI
        self.setup_menubar()
        self.setup_toolbar()
        self.setup_central_widget()
        self.setup_statusbar()

        # Connect signals
        self.setup_connections()

    def setup_central_widget(self):
        """Create the main content area"""
        # Create splitter (resizable panels)
        splitter = QSplitter(Qt.Orientation.Horizontal)

        # Left panel: Tag browser
        self.tag_browser = TagBrowserWidget(self.db)
        splitter.addWidget(self.tag_browser)

        # Middle panel: Book list
        self.book_list = BookListWidget(self.db)
        splitter.addWidget(self.book_list)

        # Right panel: Book details
        self.book_details = BookDetailsWidget()
        splitter.addWidget(self.book_details)

        # Set initial sizes (30%, 40%, 30%)
        splitter.setSizes([300, 500, 300])

        # Set as central widget
        self.setCentralWidget(splitter)

    def setup_menubar(self):
        """Create menu bar"""
        menubar = self.menuBar()

        # File menu
        file_menu = menubar.addMenu('&File')
        file_menu.addAction('&Add Books', self.add_books, 'Ctrl+A')
        file_menu.addAction('&Preferences', self.show_preferences, 'Ctrl+P')
        file_menu.addSeparator()
        file_menu.addAction('&Quit', self.close, 'Ctrl+Q')

        # Edit menu
        edit_menu = menubar.addMenu('&Edit')
        edit_menu.addAction('Edit &Metadata', self.edit_metadata, 'Ctrl+E')

        # View menu
        view_menu = menubar.addMenu('&View')
        view_menu.addAction('&Refresh', self.refresh_view, 'F5')

    def setup_toolbar(self):
        """Create toolbar with common actions"""
        toolbar = self.addToolBar('Main Toolbar')
        toolbar.setMovable(False)

        # Add actions with icons
        toolbar.addAction(get_icon('add.png'), 'Add Books', self.add_books)
        toolbar.addAction(get_icon('edit.png'), 'Edit Metadata', self.edit_metadata)
        toolbar.addAction(get_icon('delete.png'), 'Remove Books', self.remove_books)
        toolbar.addSeparator()
        toolbar.addAction(get_icon('convert.png'), 'Convert Books', self.convert_books)

    def setup_statusbar(self):
        """Create status bar"""
        self.statusBar().showMessage('Ready')

        # Add permanent widgets to status bar
        self.book_count_label = QLabel()
        self.statusBar().addPermanentWidget(self.book_count_label)
        self.update_book_count()

    def setup_connections(self):
        """Connect signals to slots"""
        # When book selected in list → Show details
        self.book_list.book_selected.connect(self.on_book_selected)

        # When tag clicked → Filter book list
        self.tag_browser.tag_clicked.connect(self.book_list.filter_by_tag)

        # When books change → Refresh count
        self.books_changed.connect(self.update_book_count)

    # Event handlers (slots)
    def on_book_selected(self, book_id):
        """Handle book selection"""
        metadata = self.db.get_metadata(book_id)
        self.book_details.display_book(metadata)
        self.book_selected.emit(book_id)

    def update_book_count(self):
        """Update book count in status bar"""
        count = len(self.db.all_book_ids())
        self.book_count_label.setText(f'{count} books')

    # Action handlers
    def add_books(self):
        """Show file dialog to add books"""
        from PyQt6.QtWidgets import QFileDialog

        files, _ = QFileDialog.getOpenFileNames(
            self,
            'Select Books',
            '',
            'E-books (*.epub *.pdf *.mobi *.azw3);;All Files (*.*)'
        )

        if files:
            self.import_books(files)

    def import_books(self, file_paths):
        """Import books from file paths"""
        # Show progress dialog
        progress = ProgressDialog('Importing books...', len(file_paths), self)

        for i, path in enumerate(file_paths):
            self.db.add_book(path)
            progress.setValue(i + 1)

        progress.close()
        self.books_changed.emit()
        self.statusBar().showMessage(f'Added {len(file_paths)} books')
```

**🔗 Code References**:
- [src/calibre/gui2/main.py](../../src/calibre/gui2/main.py) - Main window implementation

**💭 React Comparison**:
```javascript
// React equivalent:
function MainWindow() {
  const [selectedBookId, setSelectedBookId] = useState(null)
  const [books, setBooks] = useState([])

  const handleBookSelected = (bookId) => {
    setSelectedBookId(bookId)
  }

  const handleAddBooks = async (files) => {
    const newBooks = await importBooks(files)
    setBooks([...books, ...newBooks])
  }

  return (
    <div className="main-window">
      <MenuBar onAddBooks={handleAddBooks} />
      <ToolBar onAddBooks={handleAddBooks} />
      <Splitter>
        <TagBrowser onTagClick={handleTagClick} />
        <BookList
          books={books}
          onBookSelected={handleBookSelected}
        />
        <BookDetails bookId={selectedBookId} />
      </Splitter>
      <StatusBar bookCount={books.length} />
    </div>
  )
}
```

---

## Widgets and Components

### Common Widgets

```python
from PyQt6.QtWidgets import (
    QLabel,        # Text label (like <label> or <span>)
    QPushButton,   # Button (like <button>)
    QLineEdit,     # Text input (like <input type="text">)
    QTextEdit,     # Multi-line text (like <textarea>)
    QSpinBox,      # Number input (like <input type="number">)
    QComboBox,     # Dropdown (like <select>)
    QCheckBox,     # Checkbox (like <input type="checkbox">)
    QRadioButton,  # Radio button (like <input type="radio">)
    QSlider,       # Slider (like <input type="range">)
    QProgressBar,  # Progress bar (like <progress>)
    QTableView,    # Table (like <table> but with model)
    QListView,     # List (like <ul> but with model)
    QTreeView,     # Tree (like nested <ul> but with model)
)
```

### Widget Examples

#### 1. Simple Form

```python
class BookEditForm(QWidget):
    """Form to edit book metadata"""

    def __init__(self, book_id, db):
        super().__init__()
        self.book_id = book_id
        self.db = db

        # Load current data
        self.metadata = db.get_metadata(book_id)

        self.setup_ui()

    def setup_ui(self):
        """Build the form"""
        layout = QFormLayout()

        # Title input
        self.title_input = QLineEdit(self.metadata.title)
        layout.addRow('Title:', self.title_input)

        # Authors input
        self.authors_input = QLineEdit(', '.join(self.db.authors(self.book_id)))
        layout.addRow('Authors:', self.authors_input)

        # Rating input
        self.rating_input = QDoubleSpinBox()
        self.rating_input.setRange(0, 5)
        self.rating_input.setSingleStep(0.5)
        self.rating_input.setValue(self.metadata.rating or 0)
        layout.addRow('Rating:', self.rating_input)

        # Publisher input
        self.publisher_input = QLineEdit(self.metadata.publisher or '')
        layout.addRow('Publisher:', self.publisher_input)

        # Comments (multi-line)
        self.comments_input = QTextEdit(self.metadata.comments or '')
        layout.addRow('Comments:', self.comments_input)

        # Buttons
        button_layout = QHBoxLayout()
        save_btn = QPushButton('Save')
        save_btn.clicked.connect(self.save_changes)
        cancel_btn = QPushButton('Cancel')
        cancel_btn.clicked.connect(self.close)
        button_layout.addWidget(cancel_btn)
        button_layout.addWidget(save_btn)

        layout.addRow(button_layout)

        self.setLayout(layout)

    def save_changes(self):
        """Save form data to database"""
        # Get values from inputs
        title = self.title_input.text()
        authors = [a.strip() for a in self.authors_input.text().split(',')]
        rating = self.rating_input.value()
        publisher = self.publisher_input.text()
        comments = self.comments_input.toPlainText()

        # Update database
        self.db.set_metadata(self.book_id, {
            'title': title,
            'rating': rating,
            'publisher': publisher,
            'comments': comments
        })
        self.db.set_authors(self.book_id, authors)

        # Close form
        self.close()
```

**💭 React Comparison**:
```javascript
function BookEditForm({ bookId, onSave, onCancel }) {
  const [title, setTitle] = useState('')
  const [authors, setAuthors] = useState('')
  const [rating, setRating] = useState(0)
  const [publisher, setPublisher] = useState('')
  const [comments, setComments] = useState('')

  useEffect(() => {
    // Load book data
    fetchBook(bookId).then(book => {
      setTitle(book.title)
      setAuthors(book.authors.join(', '))
      setRating(book.rating)
      setPublisher(book.publisher)
      setComments(book.comments)
    })
  }, [bookId])

  const handleSave = () => {
    onSave({
      title,
      authors: authors.split(',').map(a => a.trim()),
      rating,
      publisher,
      comments
    })
  }

  return (
    <form>
      <label>Title: <input value={title} onChange={e => setTitle(e.target.value)} /></label>
      <label>Authors: <input value={authors} onChange={e => setAuthors(e.target.value)} /></label>
      <label>Rating: <input type="number" value={rating} onChange={e => setRating(e.target.value)} /></label>
      <label>Publisher: <input value={publisher} onChange={e => setPublisher(e.target.value)} /></label>
      <label>Comments: <textarea value={comments} onChange={e => setComments(e.target.value)} /></label>
      <button onClick={onCancel}>Cancel</button>
      <button onClick={handleSave}>Save</button>
    </form>
  )
}
```

Very similar patterns!

#### 2. Custom Star Rating Widget

```python
from PyQt6.QtCore import pyqtSignal, Qt
from PyQt6.QtWidgets import QWidget, QHBoxLayout, QLabel
from PyQt6.QtGui import QCursor

class StarRatingWidget(QWidget):
    """
    Custom widget for star rating

    Displays: ★★★⯪☆ (3.5 stars)
    """

    # Signal emitted when rating changes
    ratingChanged = pyqtSignal(float)

    def __init__(self, initial_rating=0, parent=None):
        super().__init__(parent)
        self.rating = initial_rating
        self.hovering_over = -1  # Track mouse hover
        self.star_labels = []

        self.setup_ui()
        self.update_display()

    def setup_ui(self):
        """Create 5 star labels"""
        layout = QHBoxLayout()
        layout.setContentsMargins(0, 0, 0, 0)
        layout.setSpacing(2)

        for i in range(5):
            star = ClickableLabel('☆', self)
            star.setCursor(QCursor(Qt.CursorShape.PointingHandCursor))
            star.clicked.connect(lambda idx=i: self.set_rating(idx + 1))
            star.setStyleSheet('font-size: 20px;')

            # Mouse hover effects
            star.setMouseTracking(True)
            star.mouseMoved.connect(lambda idx=i: self.on_mouse_over(idx))

            self.star_labels.append(star)
            layout.addWidget(star)

        layout.addStretch()
        self.setLayout(layout)

    def update_display(self):
        """Update star characters based on rating"""
        full_stars = int(self.rating)
        has_half = (self.rating % 1) >= 0.5

        for i, label in enumerate(self.star_labels):
            if i < full_stars:
                label.setText('★')  # Full star
                label.setStyleSheet('font-size: 20px; color: gold;')
            elif i == full_stars and has_half:
                label.setText('⯪')  # Half star
                label.setStyleSheet('font-size: 20px; color: gold;')
            else:
                label.setText('☆')  # Empty star
                label.setStyleSheet('font-size: 20px; color: gray;')

    def set_rating(self, new_rating):
        """Set the rating (user clicked a star)"""
        if new_rating != self.rating:
            self.rating = new_rating
            self.update_display()
            self.ratingChanged.emit(new_rating)

    def on_mouse_over(self, star_index):
        """Preview rating on hover"""
        self.hovering_over = star_index
        # Show preview (temporary)
        for i, label in enumerate(self.star_labels):
            if i <= star_index:
                label.setText('★')
                label.setStyleSheet('font-size: 20px; color: orange;')
            else:
                label.setText('☆')
                label.setStyleSheet('font-size: 20px; color: gray;')

    def leaveEvent(self, event):
        """Mouse left widget - restore actual rating"""
        self.hovering_over = -1
        self.update_display()


class ClickableLabel(QLabel):
    """QLabel that emits signal on click"""
    clicked = pyqtSignal()
    mouseMoved = pyqtSignal()

    def mousePressEvent(self, event):
        self.clicked.emit()
        super().mousePressEvent(event)

    def mouseMoveEvent(self, event):
        self.mouseMoved.emit()
        super().mouseMoveEvent(event)
```

**Usage**:
```python
# In book details widget:
rating_widget = StarRatingWidget(initial_rating=4.5)
rating_widget.ratingChanged.connect(lambda r: self.update_rating(book_id, r))
layout.addWidget(rating_widget)
```

**💭 React Comparison**:
```javascript
function StarRatingWidget({ initialRating, onRatingChange }) {
  const [rating, setRating] = useState(initialRating)
  const [hovering, setHovering] = useState(-1)

  const handleClick = (index) => {
    const newRating = index + 1
    setRating(newRating)
    onRatingChange(newRating)
  }

  const displayRating = hovering >= 0 ? hovering + 1 : rating

  return (
    <div onMouseLeave={() => setHovering(-1)}>
      {[...Array(5)].map((_, i) => {
        const fullStars = Math.floor(displayRating)
        const hasHalf = (displayRating % 1) >= 0.5

        let char
        if (i < fullStars) char = '★'
        else if (i === fullStars && hasHalf) char = '⯪'
        else char = '☆'

        return (
          <span
            key={i}
            onClick={() => handleClick(i)}
            onMouseEnter={() => setHovering(i)}
            style={{
              fontSize: '20px',
              color: i <= hovering || i < fullStars ? 'gold' : 'gray',
              cursor: 'pointer'
            }}
          >
            {char}
          </span>
        )
      })}
    </div>
  )
}
```

---

## Layouts and Styling

### Layout Managers

```python
# 1. QVBoxLayout - Vertical stack
layout = QVBoxLayout()
layout.addWidget(QLabel('Header'))
layout.addWidget(QLabel('Content'))
layout.addWidget(QPushButton('Footer'))

# 2. QHBoxLayout - Horizontal row
layout = QHBoxLayout()
layout.addWidget(QPushButton('Cancel'))
layout.addStretch()  # Push buttons apart
layout.addWidget(QPushButton('OK'))

# 3. QGridLayout - Grid
layout = QGridLayout()
layout.addWidget(QLabel('Name:'), 0, 0)  # Row 0, Col 0
layout.addWidget(QLineEdit(), 0, 1)      # Row 0, Col 1
layout.addWidget(QLabel('Email:'), 1, 0)  # Row 1, Col 0
layout.addWidget(QLineEdit(), 1, 1)      # Row 1, Col 1

# 4. QFormLayout - Label-field pairs
layout = QFormLayout()
layout.addRow('Name:', QLineEdit())
layout.addRow('Email:', QLineEdit())

# 5. QSplitter - Resizable panels
splitter = QSplitter(Qt.Orientation.Horizontal)
splitter.addWidget(left_panel)
splitter.addWidget(right_panel)
splitter.setSizes([300, 700])  # Initial widths
```

### Styling with QSS (Qt Style Sheets)

QSS is like CSS for Qt:

```python
# Apply stylesheet to widget
widget.setStyleSheet('''
    QWidget {
        background-color: #2b2b2b;
        color: #ffffff;
        font-family: "Segoe UI", Arial, sans-serif;
        font-size: 14px;
    }

    QPushButton {
        background-color: #0078d4;
        color: white;
        border: none;
        border-radius: 4px;
        padding: 8px 16px;
        min-width: 80px;
    }

    QPushButton:hover {
        background-color: #1084d8;
    }

    QPushButton:pressed {
        background-color: #006cbd;
    }

    QPushButton:disabled {
        background-color: #cccccc;
        color: #666666;
    }

    QLineEdit {
        background-color: #3c3c3c;
        color: white;
        border: 1px solid #555555;
        border-radius: 3px;
        padding: 4px 8px;
    }

    QLineEdit:focus {
        border: 1px solid #0078d4;
    }

    QTableView {
        background-color: #2b2b2b;
        alternate-background-color: #333333;
        selection-background-color: #0078d4;
        gridline-color: #555555;
    }

    QHeaderView::section {
        background-color: #3c3c3c;
        color: white;
        padding: 5px;
        border: none;
        border-bottom: 2px solid #555555;
    }

    QScrollBar:vertical {
        background-color: #2b2b2b;
        width: 12px;
        margin: 0px;
    }

    QScrollBar::handle:vertical {
        background-color: #555555;
        border-radius: 6px;
        min-height: 20px;
    }

    QScrollBar::handle:vertical:hover {
        background-color: #666666;
    }

    QToolBar {
        background-color: #3c3c3c;
        border: none;
        padding: 4px;
    }

    QStatusBar {
        background-color: #3c3c3c;
        color: #aaaaaa;
    }
''')
```

**QSS Selectors**:
```css
/* Type selector */
QPushButton { color: blue; }

/* ID selector (objectName) */
#saveButton { background: green; }

/* Class selector */
.danger-button { background: red; }

/* Pseudo-states */
QPushButton:hover { background: lightblue; }
QPushButton:pressed { background: darkblue; }
QPushButton:disabled { opacity: 0.5; }
QPushButton:checked { background: yellow; }

/* Descendant selector */
QDialog QPushButton { font-size: 12px; }

/* Sub-controls (for complex widgets) */
QComboBox::drop-down { border: none; }
QScrollBar::handle:vertical { border-radius: 3px; }
```

**💭 CSS Comparison**:
```css
/* CSS equivalent: */
button { color: blue; }
#saveButton { background: green; }
.danger-button { background: red; }
button:hover { background: lightblue; }
button:active { background: darkblue; }
button:disabled { opacity: 0.5; }
```

Very similar syntax!

---

## Event System (Signals/Slots)

See [PATTERNS_AND_CONVENTIONS.md](PATTERNS_AND_CONVENTIONS.md#observer-pattern-signalsslots) for detailed examples.

### Built-in Signals

```python
# Button signals
button = QPushButton('Click Me')
button.clicked.connect(lambda: print('Clicked!'))
button.pressed.connect(lambda: print('Pressed'))
button.released.connect(lambda: print('Released'))

# Text input signals
line_edit = QLineEdit()
line_edit.textChanged.connect(lambda text: print(f'Changed: {text}'))
line_edit.editingFinished.connect(lambda: print('Done editing'))
line_edit.returnPressed.connect(lambda: print('Enter pressed'))

# Selection signals
combo = QComboBox()
combo.currentIndexChanged.connect(lambda idx: print(f'Selected: {idx}'))
combo.currentTextChanged.connect(lambda text: print(f'Text: {text}'))

# Table/List signals
table = QTableView()
table.clicked.connect(lambda index: print(f'Clicked: {index.row()}, {index.column()}'))
table.doubleClicked.connect(lambda index: print(f'Double-clicked'))

# Slider signals
slider = QSlider()
slider.valueChanged.connect(lambda val: print(f'Value: {val}'))
slider.sliderMoved.connect(lambda val: print(f'Slider moved: {val}'))
slider.sliderPressed.connect(lambda: print('Pressed'))
slider.sliderReleased.connect(lambda: print('Released'))
```

### Custom Signals

```python
class BookManager(QObject):
    """Manages books and emits signals on changes"""

    # Define custom signals
    bookAdded = pyqtSignal(int, dict)  # book_id, metadata
    bookRemoved = pyqtSignal(int)  # book_id
    bookUpdated = pyqtSignal(int, dict)  # book_id, changes
    searchCompleted = pyqtSignal(list)  # List of book_ids

    def __init__(self, db):
        super().__init__()
        self.db = db

    def add_book(self, file_path):
        """Add a book and emit signal"""
        book_id, metadata = self.db.add_book(file_path)
        self.bookAdded.emit(book_id, metadata)
        return book_id

    def remove_book(self, book_id):
        """Remove a book and emit signal"""
        self.db.remove_book(book_id)
        self.bookRemoved.emit(book_id)

    def update_book(self, book_id, changes):
        """Update a book and emit signal"""
        self.db.set_metadata(book_id, changes)
        self.bookUpdated.emit(book_id, changes)

    def search(self, query):
        """Search books and emit results"""
        results = self.db.search(query)
        self.searchCompleted.emit(results)


# Usage:
manager = BookManager(db)

# Connect signals
manager.bookAdded.connect(lambda book_id, meta: print(f'Added: {meta["title"]}'))
manager.bookUpdated.connect(lambda book_id, changes: refresh_ui(book_id))
manager.searchCompleted.connect(lambda results: display_results(results))

# Trigger actions
manager.add_book('/path/to/book.epub')  # Emits bookAdded
manager.update_book(42, {'rating': 5})  # Emits bookUpdated
```

---

## State Management

### Local Widget State

```python
class BookListWidget(QWidget):
    """
    Widget managing its own state

    Like React component with useState
    """

    def __init__(self):
        super().__init__()
        # State
        self.books = []
        self.selected_book_id = None
        self.sort_column = 0
        self.sort_order = Qt.SortOrder.AscendingOrder
        self.filter_text = ''

        self.setup_ui()

    def set_books(self, books):
        """Update state and refresh UI"""
        self.books = books
        self.refresh_display()

    def set_filter(self, text):
        """Update filter and refresh"""
        self.filter_text = text
        self.refresh_display()

    def set_sort(self, column, order):
        """Update sort and refresh"""
        self.sort_column = column
        self.sort_order = order
        self.refresh_display()

    def refresh_display(self):
        """Re-render based on current state"""
        # Filter
        filtered = [b for b in self.books if self.filter_text.lower() in b['title'].lower()]

        # Sort
        filtered.sort(
            key=lambda b: b.get(['title', 'author', 'rating'][self.sort_column], ''),
            reverse=(self.sort_order == Qt.SortOrder.DescendingOrder)
        )

        # Update table
        self.table.setRowCount(len(filtered))
        for i, book in enumerate(filtered):
            self.table.setItem(i, 0, QTableWidgetItem(book['title']))
            self.table.setItem(i, 1, QTableWidgetItem(book['author']))
            self.table.setItem(i, 2, QTableWidgetItem(str(book['rating'])))
```

**💭 React Comparison**:
```javascript
function BookListWidget() {
  const [books, setBooks] = useState([])
  const [selectedBookId, setSelectedBookId] = useState(null)
  const [sortColumn, setSortColumn] = useState(0)
  const [sortOrder, setSortOrder] = useState('asc')
  const [filterText, setFilterText] = useState('')

  const filteredAndSorted = useMemo(() => {
    let result = books.filter(b =>
      b.title.toLowerCase().includes(filterText.toLowerCase())
    )

    result.sort((a, b) => {
      const key = ['title', 'author', 'rating'][sortColumn]
      return sortOrder === 'asc'
        ? a[key] < b[key] ? -1 : 1
        : a[key] > b[key] ? -1 : 1
    })

    return result
  }, [books, filterText, sortColumn, sortOrder])

  return (
    <table>
      {filteredAndSorted.map(book => (
        <tr key={book.id}>
          <td>{book.title}</td>
          <td>{book.author}</td>
          <td>{book.rating}</td>
        </tr>
      ))}
    </table>
  )
}
```

### Global State (Singleton Pattern)

```python
class AppState(QObject):
    """
    Global application state

    Like Redux store or React Context
    """

    # Signals for state changes
    booksChanged = pyqtSignal(list)
    selectedBookChanged = pyqtSignal(int)
    filterChanged = pyqtSignal(str)

    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._initialized = False
        return cls._instance

    def __init__(self):
        if self._initialized:
            return

        super().__init__()
        self.books = []
        self.selected_book_id = None
        self.filter_text = ''
        self._initialized = True

    def set_books(self, books):
        self.books = books
        self.booksChanged.emit(books)

    def set_selected_book(self, book_id):
        self.selected_book_id = book_id
        self.selectedBookChanged.emit(book_id)

    def set_filter(self, text):
        self.filter_text = text
        self.filterChanged.emit(text)


# Usage in any widget:
state = AppState()
state.booksChanged.connect(self.on_books_changed)
state.set_books(new_books)  # All connected widgets notified
```

**💭 Redux Comparison**:
```javascript
// Redux store:
const store = configureStore({
  reducer: {
    books: booksReducer,
    selectedBookId: selectedBookReducer,
    filter: filterReducer
  }
})

// Dispatch actions:
store.dispatch(setBooks(newBooks))
store.dispatch(setSelectedBook(bookId))

// Subscribe to changes:
store.subscribe(() => {
  const state = store.getState()
  updateUI(state)
})
```

---

## Model-View Architecture

See [PATTERNS_AND_CONVENTIONS.md](PATTERNS_AND_CONVENTIONS.md#model-view-pattern) for comprehensive examples.

### Table Model Example

```python
from PyQt6.QtCore import QAbstractTableModel, Qt

class BookTableModel(QAbstractTableModel):
    """
    Model for book table

    Separates data from presentation
    """

    def __init__(self, db):
        super().__init__()
        self.db = db
        self.books = []
        self.columns = ['title', 'authors', 'rating', 'pubdate']
        self.load_books()

    def load_books(self):
        """Load books from database"""
        self.beginResetModel()
        self.books = [
            self.db.get_metadata(book_id)
            for book_id in self.db.all_book_ids()
        ]
        self.endResetModel()

    # Required methods:
    def rowCount(self, parent=None):
        return len(self.books)

    def columnCount(self, parent=None):
        return len(self.columns)

    def data(self, index, role=Qt.ItemDataRole.DisplayRole):
        """Get data for cell"""
        if not index.isValid():
            return None

        book = self.books[index.row()]
        col = self.columns[index.column()]

        if role == Qt.ItemDataRole.DisplayRole:
            if col == 'title':
                return book.title
            elif col == 'authors':
                return ', '.join(self.db.authors(book.id))
            elif col == 'rating':
                return book.rating or 0
            elif col == 'pubdate':
                return book.pubdate.strftime('%Y-%m-%d') if book.pubdate else ''

        elif role == Qt.ItemDataRole.TextAlignmentRole:
            if col == 'rating':
                return Qt.AlignmentFlag.AlignCenter

        return None

    def headerData(self, section, orientation, role=Qt.ItemDataRole.DisplayRole):
        """Get column headers"""
        if role == Qt.ItemDataRole.DisplayRole and orientation == Qt.Orientation.Horizontal:
            return self.columns[section].title()
        return None

    # Optional: Make editable
    def flags(self, index):
        """Set cell flags"""
        return Qt.ItemFlag.ItemIsEnabled | Qt.ItemFlag.ItemIsSelectable | Qt.ItemFlag.ItemIsEditable

    def setData(self, index, value, role=Qt.ItemDataRole.EditRole):
        """Handle cell edit"""
        if role == Qt.ItemDataRole.EditRole:
            book = self.books[index.row()]
            col = self.columns[index.column()]

            if col == 'rating':
                self.db.set_metadata(book.id, {'rating': float(value)})
                book.rating = float(value)
                self.dataChanged.emit(index, index)
                return True

        return False

# Usage:
model = BookTableModel(db)
table_view = QTableView()
table_view.setModel(model)

# Model handles all data, view just displays it
```

---

## Dialogs and Modals

```python
from PyQt6.QtWidgets import QDialog, QDialogButtonBox

class PreferencesDialog(QDialog):
    """
    Modal dialog for preferences

    Like React modal but native
    """

    def __init__(self, current_settings, parent=None):
        super().__init__(parent)
        self.settings = current_settings.copy()

        self.setWindowTitle('Preferences')
        self.setModal(True)  # Block parent window
        self.setup_ui()

    def setup_ui(self):
        layout = QVBoxLayout()

        # Settings
        self.theme_combo = QComboBox()
        self.theme_combo.addItems(['Light', 'Dark', 'Auto'])
        self.theme_combo.setCurrentText(self.settings.get('theme', 'Auto'))
        layout.addRow('Theme:', self.theme_combo)

        # OK/Cancel buttons
        buttons = QDialogButtonBox(
            QDialogButtonBox.StandardButton.Ok |
            QDialogButtonBox.StandardButton.Cancel
        )
        buttons.accepted.connect(self.accept)  # Closes with result=1
        buttons.rejected.connect(self.reject)  # Closes with result=0
        layout.addWidget(buttons)

        self.setLayout(layout)

    def accept(self):
        """OK clicked - save settings"""
        self.settings['theme'] = self.theme_combo.currentText()
        super().accept()

    def get_settings(self):
        """Get updated settings"""
        return self.settings


# Usage:
dialog = PreferencesDialog(current_settings, parent=self)
if dialog.exec() == QDialog.DialogCode.Accepted:
    new_settings = dialog.get_settings()
    apply_settings(new_settings)
else:
    print('Cancelled')
```

**💭 React Modal Comparison**:
```javascript
function PreferencesDialog({ isOpen, settings, onSave, onCancel }) {
  const [theme, setTheme] = useState(settings.theme)

  const handleSave = () => {
    onSave({ ...settings, theme })
  }

  if (!isOpen) return null

  return (
    <Modal>
      <h2>Preferences</h2>
      <label>
        Theme:
        <select value={theme} onChange={e => setTheme(e.target.value)}>
          <option>Light</option>
          <option>Dark</option>
          <option>Auto</option>
        </select>
      </label>
      <button onClick={onCancel}>Cancel</button>
      <button onClick={handleSave}>OK</button>
    </Modal>
  )
}
```

---

## Threading and Async

PyQt6 apps must keep the UI thread responsive. Long operations should run in background threads.

```python
from PyQt6.QtCore import QThread, pyqtSignal

class BookImportThread(QThread):
    """
    Background thread for importing books

    Like Web Worker or async function in React
    """

    # Signals (thread-safe communication)
    progressChanged = pyqtSignal(int, int)  # current, total
    bookImported = pyqtSignal(int, str)  # book_id, title
    finished = pyqtSignal(list)  # List of imported book_ids
    error = pyqtSignal(str)  # Error message

    def __init__(self, db, file_paths):
        super().__init__()
        self.db = db
        self.file_paths = file_paths
        self.imported_ids = []

    def run(self):
        """
        Runs in background thread

        Do NOT access UI from here!
        Use signals to communicate with UI thread.
        """
        try:
            total = len(self.file_paths)

            for i, path in enumerate(self.file_paths):
                # Import book
                book_id, metadata = self.db.add_book(path)
                self.imported_ids.append(book_id)

                # Emit progress (safe - signals are thread-safe)
                self.progressChanged.emit(i + 1, total)
                self.bookImported.emit(book_id, metadata['title'])

                # Check if should stop
                if self.isInterruptionRequested():
                    break

            # Done
            self.finished.emit(self.imported_ids)

        except Exception as e:
            self.error.emit(str(e))


# Usage in UI:
class MainWindow(QMainWindow):
    def import_books(self, file_paths):
        """Start import in background"""
        # Create progress dialog
        self.progress_dialog = ProgressDialog('Importing books...', len(file_paths), self)
        self.progress_dialog.canceled.connect(self.cancel_import)

        # Create and start thread
        self.import_thread = BookImportThread(self.db, file_paths)

        # Connect signals
        self.import_thread.progressChanged.connect(self.on_import_progress)
        self.import_thread.bookImported.connect(self.on_book_imported)
        self.import_thread.finished.connect(self.on_import_finished)
        self.import_thread.error.connect(self.on_import_error)

        # Start (non-blocking!)
        self.import_thread.start()

    def on_import_progress(self, current, total):
        """Update progress dialog (runs in UI thread)"""
        self.progress_dialog.setValue(current)
        self.progress_dialog.setLabelText(f'Importing book {current} of {total}...')

    def on_book_imported(self, book_id, title):
        """Book imported successfully"""
        print(f'Imported: {title}')

    def on_import_finished(self, book_ids):
        """All books imported"""
        self.progress_dialog.close()
        self.statusBar().showMessage(f'Imported {len(book_ids)} books')
        self.refresh_book_list()

    def on_import_error(self, error_msg):
        """Import failed"""
        self.progress_dialog.close()
        QMessageBox.critical(self, 'Import Error', f'Failed to import books: {error_msg}')

    def cancel_import(self):
        """User cancelled import"""
        self.import_thread.requestInterruption()
```

**💭 React Async Comparison**:
```javascript
// React with async/await:
async function importBooks(filePaths, onProgress, onBookImported) {
  const importedIds = []

  for (let i = 0; i < filePaths.length; i++) {
    const path = filePaths[i]

    try {
      const { bookId, metadata } = await db.addBook(path)
      importedIds.push(bookId)

      onProgress(i + 1, filePaths.length)
      onBookImported(bookId, metadata.title)

    } catch (error) {
      throw new Error(`Failed to import ${path}: ${error.message}`)
    }
  }

  return importedIds
}

// Usage in component:
function MainWindow() {
  const [importing, setImporting] = useState(false)
  const [progress, setProgress] = useState(0)

  const handleImport = async (filePaths) => {
    setImporting(true)

    try {
      const bookIds = await importBooks(
        filePaths,
        (current, total) => setProgress((current / total) * 100),
        (bookId, title) => console.log(`Imported: ${title}`)
      )

      alert(`Imported ${bookIds.length} books`)

    } catch (error) {
      alert(`Import failed: ${error.message}`)

    } finally {
      setImporting(false)
    }
  }

  return (
    <div>
      {importing && <ProgressBar value={progress} />}
      <button onClick={() => handleImport(files)}>Import</button>
    </div>
  )
}
```

---

## Performance Optimization

### 1. Lazy Loading

```python
# Load data only when visible
class BookListWidget(QWidget):
    def showEvent(self, event):
        """Called when widget becomes visible"""
        if not self.loaded:
            self.load_books()
            self.loaded = True
        super().showEvent(event)
```

### 2. Virtual Scrolling (Built-in)

```python
# QTableView automatically does virtual scrolling
# Only visible rows are rendered
table = QTableView()
model = BookTableModel(100000)  # 100k books
table.setModel(model)  # Still fast!
```

### 3. Batch Updates

```python
# Disable updates during bulk changes
model.layoutAboutToBeChanged.emit()
for i in range(1000):
    model.add_book(book)
model.layoutChanged.emit()  # Single refresh
```

### 4. Caching

```python
class BookMetadataCache:
    def __init__(self):
        self.cache = {}

    def get(self, book_id):
        if book_id not in self.cache:
            self.cache[book_id] = db.get_metadata(book_id)
        return self.cache[book_id]
```

---

## Comparison to React

| Feature | PyQt6 | React |
|---------|-------|-------|
| **Components** | Widgets (classes) | Components (functions) |
| **Props** | Constructor params | Props |
| **State** | Instance variables | useState |
| **Events** | Signals → Slots | Props → Callbacks |
| **Lifecycle** | showEvent, closeEvent | useEffect |
| **Styling** | QSS | CSS/CSS-in-JS |
| **Layout** | Layout managers | Flexbox/Grid |
| **Rendering** | Manual (update()) | Automatic (Virtual DOM) |
| **Performance** | Native C++ | JavaScript + DOM |
| **Dev Tools** | Qt Designer | React DevTools |

---

## Summary

### Key Concepts for React Developers

1. **Widgets = Components**: Both are reusable UI building blocks
2. **Signals/Slots = Props/Callbacks**: Both handle events
3. **Layouts = Flexbox**: Both position elements
4. **QSS = CSS**: Both style elements
5. **Model-View = State Management**: Both separate data from UI
6. **Threads = Web Workers**: Both run code in background

### Learning Path

1. **Start with**: Basic widgets (QLabel, QPushButton, QLineEdit)
2. **Then learn**: Layouts (QVBoxLayout, QHBoxLayout)
3. **Then understand**: Signals and slots
4. **Then master**: Model-View architecture
5. **Finally**: Threading and performance

### Next Steps

- Try the exercises in [EXERCISES.md](EXERCISES.md#exercise-14-explore-the-pyqt6-gui-structure)
- Read the source: [src/calibre/gui2/](../../src/calibre/gui2/)
- Build your first custom widget!

---

*See [TECH_STACK_GUIDE.md](TECH_STACK_GUIDE.md#pyqt6) for more on PyQt6 technology and [PATTERNS_AND_CONVENTIONS.md](PATTERNS_AND_CONVENTIONS.md#gui-patterns) for GUI patterns.*
