# Getting Started: Development Environment Setup

**Purpose:** Set up Calibre for local development

**Time:** 30-60 minutes

**For:** React developers new to Python/Calibre

---

## Prerequisites

### Required Skills
- ✅ Comfortable with terminal/command line
- ✅ Basic Git knowledge
- ✅ JavaScript/TypeScript experience

### Required Software
- **Python 3.10+** - [Download](https://www.python.org/downloads/)
- **Git** - [Download](https://git-scm.com/downloads)
- **Qt6** - Installed via pip (see below)
- **Code Editor** - VS Code recommended

---

## Quick Start (5 minutes)

```bash
# 1. Clone the repository
git clone https://github.com/kovidgoyal/calibre.git
cd calibre

# 2. Check Python version
python3 --version  # Should be 3.10 or higher

# 3. Install dependencies
pip3 install -r requirements.txt  # Note: No requirements.txt exists!
# Calibre uses pyproject.toml instead

# 4. Build Calibre (this compiles C extensions and generates resources)
python3 setup.py build

# 5. Run Calibre
python3 setup.py develop  # Install in development mode
calibre  # Launch the GUI
```

---

## Detailed Setup

### Step 1: Install Python

**macOS:**
```bash
brew install python@3.11
```

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install python3.11 python3.11-dev
```

**Windows:**
- Download from [python.org](https://www.python.org/downloads/)
- Check "Add Python to PATH" during installation

**Verify:**
```bash
python3 --version
# Should show: Python 3.11.x or 3.10.x
```

---

### Step 2: Install Build Dependencies

**macOS:**
```bash
# Install Qt6
brew install qt6 pyqt6

# Install other dependencies
brew install libpng libjpeg libmtp libusb
```

**Ubuntu/Debian:**
```bash
sudo apt install \
    python3-pyqt6 \
    python3-pyqt6.qtwebengine \
    python3-sip \
    qt6-base-dev \
    libsqlite3-dev \
    libpng-dev \
    libjpeg-dev
```

**Windows:**
- Dependencies will be installed via pip

---

### Step 3: Clone & Build

```bash
# Clone repository
git clone https://github.com/kovidgoyal/calibre.git
cd calibre

# Install Python dependencies
pip3 install -e .  # Editable install

# Build (compiles C extensions, generates resources)
python3 setup.py build

# This will:
# - Compile C/C++ extensions
# - Generate Qt .ui files to .py
# - Bundle resources
# - Create executables
```

**Expected output:**
```
Building calibre...
Compiling C extensions...
Generating forms...
Building resources...
✓ Build complete!
```

---

### Step 4: Run Calibre

```bash
# Run the desktop GUI
calibre

# Or run the content server (web interface)
calibre-server /path/to/library

# Or use the debug runner
python3 src/calibre/debug.py
```

---

## Development Workflow

### Project Structure

```
calibre/
├── src/
│   └── calibre/           # Main source code
│       ├── __init__.py
│       ├── gui2/          # Desktop GUI (PyQt6)
│       ├── srv/           # Content server (web)
│       ├── db/            # Database layer
│       ├── ebooks/        # E-book processing
│       └── ...
├── resources/             # Images, CSS, JS
├── recipes/               # News download recipes
├── setup/                 # Build scripts
├── manual/                # Documentation
├── pyproject.toml         # Dependencies
└── setup.py               # Build configuration
```

---

### Making Changes

**1. Create a branch:**
```bash
git checkout -b feature/my-awesome-feature
```

**2. Make changes:**
- Edit Python files in `src/calibre/`
- Changes are live (no rebuild needed for Python)

**3. Rebuild if needed:**
```bash
# Only if you modify:
# - C/C++ extensions
# - Qt .ui files
# - Resources
python3 setup.py build
```

**4. Test your changes:**
```bash
# Run Calibre to test
calibre

# Or run specific module
python3 src/calibre/debug.py -c "from calibre.srv import test; test()"
```

**5. Run tests:**
```bash
python3 setup.py test

# Or run specific test
python3 -m pytest src/calibre/db/tests/test_cache.py
```

---

### VS Code Setup

**Recommended Extensions:**
- Python (Microsoft)
- Pylance (Microsoft)
- Python Debugger

**Settings (`.vscode/settings.json`):**
```json
{
  "python.defaultInterpreterPath": "/usr/bin/python3",
  "python.linting.enabled": true,
  "python.linting.pylintEnabled": false,
  "python.linting.flake8Enabled": true,
  "python.formatting.provider": "black",
  "editor.formatOnSave": true,
  "[python]": {
    "editor.defaultFormatter": "ms-python.black-formatter",
    "editor.tabSize": 4
  }
}
```

**Launch configuration (`.vscode/launch.json`):**
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Calibre GUI",
      "type": "python",
      "request": "launch",
      "program": "${workspaceFolder}/src/calibre/debug.py",
      "args": ["--gui"],
      "console": "integratedTerminal"
    },
    {
      "name": "Calibre Server",
      "type": "python",
      "request": "launch",
      "program": "${workspaceFolder}/src/calibre/debug.py",
      "args": ["--serve-books", "/path/to/test/library"],
      "console": "integratedTerminal"
    }
  ]
}
```

---

## Common Tasks

### Run Content Server Locally

```bash
# Create test library
mkdir -p ~/test-calibre-library

# Run server
calibre-server ~/test-calibre-library

# Or with debug mode
python3 src/calibre/debug.py \
    --serve-books ~/test-calibre-library \
    --port 8080
```

**Access:** http://localhost:8080

---

### Add Sample Books

```bash
# Generate test data
python3 setup.py test_data

# Or manually:
calibre-debug -e /path/to/book.epub
```

---

### Explore the Database

```bash
# Open library database
sqlite3 ~/test-calibre-library/metadata.db

# Sample queries
sqlite> .tables
sqlite> SELECT * FROM books LIMIT 5;
sqlite> SELECT name FROM authors;
```

---

### Debug a Request

**1. Set breakpoint in code:**
```python
# File: src/calibre/srv/books.py
@endpoint('/api/books/{book_id}')
def get_book(ctx, rd, book_id):
    import pdb; pdb.set_trace()  # ← Breakpoint
    metadata = ctx.db.get_metadata(book_id)
    return json(ctx, rd, endpoint, metadata)
```

**2. Run with debugger:**
```bash
python3 -m pdb src/calibre/debug.py --serve-books ~/library
```

**3. Make request:**
```bash
curl http://localhost:8080/api/books/1
```

**4. Debug in terminal:**
```
(Pdb) print(book_id)
1
(Pdb) print(ctx.db)
<Cache object at 0x...>
(Pdb) next  # Step to next line
(Pdb) continue  # Resume execution
```

---

## Troubleshooting

### Python Version Issues

**Error:** `calibre requires Python >= 3.10`

**Fix:**
```bash
# Check version
python3 --version

# On Ubuntu, install newer Python
sudo apt install python3.11

# Use specific version
python3.11 -m pip install -e .
```

---

### Missing Qt/PyQt6

**Error:** `No module named 'PyQt6'`

**Fix:**
```bash
# Install PyQt6
pip3 install PyQt6 PyQt6-WebEngine
```

---

### Build Failures

**Error:** `error: command 'gcc' failed`

**Fix:**
```bash
# macOS
xcode-select --install

# Ubuntu
sudo apt install build-essential python3-dev

# Windows
# Install Visual Studio Build Tools
```

---

### Permission Errors

**Error:** `Permission denied` when running calibre

**Fix:**
```bash
# Don't use sudo with pip!
# Instead, use a virtual environment:

python3 -m venv calibre-env
source calibre-env/bin/activate  # Linux/macOS
# calibre-env\Scripts\activate  # Windows

pip install -e .
```

---

## Next Steps

1. **Explore the code:** [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md)
2. **Understand the architecture:** [ARCHITECTURE_OVERVIEW.md](./ARCHITECTURE_OVERVIEW.md)
3. **Trace a request:** [DATA_FLOW_GUIDE.md](./DATA_FLOW_GUIDE.md)
4. **Make your first change:** [FIRST_CONTRIBUTIONS.md](./FIRST_CONTRIBUTIONS.md)

---

## Getting Help

- **Documentation:** [manual.calibre-ebook.com](https://manual.calibre-ebook.com/develop.html)
- **Forum:** [MobileRead Calibre Dev Forum](https://www.mobileread.com/forums/forumdisplay.php?f=166)
- **Bug Tracker:** [Launchpad](https://bugs.launchpad.net/calibre)
- **GitHub:** [Issues](https://github.com/kovidgoyal/calibre/issues) (code only, not bugs)

---

**Ready to code!** 🚀
