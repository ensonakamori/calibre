# 🎓 Welcome to Calibre: A Complete Onboarding Guide for React Developers

**Created:** November 19, 2025
**For:** Mid-level React.js developers transitioning to full-stack development
**Project:** Calibre - The e-book management application

---

## 👋 Hello, React Developer!

Welcome! If you're here, you're probably a React developer who's comfortable with:
- ✅ JavaScript/TypeScript and modern ES6+
- ✅ Component-based architecture (props, state, hooks)
- ✅ Frontend build tools (webpack, Vite, etc.)
- ✅ API consumption (fetch, axios, React Query)

But you may be **NEW** to:
- 🆕 Backend development and server architecture
- 🆕 Database design and SQL
- 🆕 Python programming
- 🆕 Desktop application development

**This guide is specifically designed for YOU!**

We'll bridge your React knowledge to help you understand Calibre's architecture, backend technologies, and the possibility of modernizing it with React/Next.js.

---

## 📋 Table of Contents

### **Part 1: Understanding Calibre**
1. [What is Calibre?](#what-is-calibre) - The big picture
2. [Technology Stack Overview](#technology-stack) - What you're working with
3. [Architecture Overview](#architecture) - How everything connects

### **Part 2: Backend Fundamentals (For React Developers)**
4. [Backend 101: Servers, Databases, and APIs](#backend-101) - Core concepts explained
5. [Python for React Developers](#python-basics) - Language comparison
6. [Database Architecture Deep Dive](#database-architecture) - SQLite and ORM patterns
7. [HTTP Server Implementation](#http-server) - Request/response cycle

### **Part 3: Calibre's Architecture**
8. [Desktop Application (Qt/PyQt6)](#desktop-app) - GUI layer
9. [Content Server (Web Interface)](#content-server) - Web layer
10. [Database Layer](#database-layer) - Data persistence
11. [Data Flow & Architecture Diagrams](#data-flow) - Visual understanding

### **Part 4: Modern Web Development Analysis**
12. [Rewrite Analysis: React/Next.js Feasibility](#rewrite-analysis) - Can we modernize this?
13. [Architecture Comparison](#architecture-comparison) - Old vs New patterns
14. [Migration Strategy](#migration-strategy) - How would we do it?

### **Part 5: Getting Started**
15. [Development Environment Setup](#setup) - Get coding!
16. [First Contributions](#first-contributions) - Quick wins
17. [Resources & Next Steps](#resources) - Keep learning

---

<a name="what-is-calibre"></a>
## 1. What is Calibre?

### 🧠 **Mental Model for React Developers**

Think of Calibre as a **full-stack e-book management system** with:
- **Desktop GUI** (like VS Code, but for e-books) ← Built with Qt/PyQt6
- **Built-in Web Server** (like Next.js dev server, but production-ready) ← Built with custom Python HTTP server
- **Database** (like PostgreSQL, but embedded) ← Built with SQLite
- **REST API** (like Express/Fastify routes) ← Built custom in Python

🌉 **Bridge from React:**
```
React App = Desktop GUI (PyQt6)
Next.js API Routes = Python Content Server (calibre.srv)
React Router = Qt Navigation/Views
Redux/Zustand = Qt Signals/Slots + Database Cache
```

### What Does It Do?

Calibre is like **iTunes for e-books**:
- 📚 Manage your e-book library (EPUB, MOBI, PDF, etc.)
- 🔄 Convert between formats
- 📱 Sync to e-readers (Kindle, Kobo, etc.)
- 🌐 Serve books over the web
- ✏️ Edit e-book metadata and content
- 📰 Download news and convert to e-books

### Key Stats
- **~600,000 lines of Python code**
- **Active since 2006** (19 years!)
- **Cross-platform:** Linux, Windows, macOS
- **Millions of users worldwide**

---

<a name="technology-stack"></a>
## 2. Technology Stack Overview

### For React Developers: A Translation Table

| **React World** | **Calibre World** | **Purpose** |
|----------------|------------------|------------|
| React | PyQt6 (Qt bindings) | UI framework |
| Next.js | Custom Python HTTP server | Web framework |
| Node.js | Python 3.10+ | Runtime |
| npm/yarn | pip | Package manager |
| TypeScript | Python (with type hints) | Language |
| PostgreSQL/MySQL | SQLite | Database |
| Prisma/TypeORM | Custom ORM (calibre.db.cache) | Database abstraction |
| Express routes | calibre.srv.routes | HTTP routing |
| Webpack/Vite | setuptools | Build system |
| Jest/Vitest | pytest | Testing |
| CSS Modules | Qt Stylesheets (QSS) | Styling |

### Core Technologies

#### **Python 3.10+** 🐍
- **Role:** Main programming language (replaces JavaScript/TypeScript)
- **Why:** Cross-platform, excellent libraries for file manipulation, readable
- **Learning curve:** EASY for JS developers! Python is often called "executable pseudocode"

#### **PyQt6** 🖼️
- **Role:** Desktop GUI framework (replaces React for desktop)
- **What it is:** Python bindings for Qt (C++ GUI framework)
- **Think of it as:** "React, but for native desktop apps"
- **Key concept:** Widgets (like React components) + Signals/Slots (like event handlers)

#### **SQLite** 🗄️
- **Role:** Database (file-based, no server needed)
- **What it is:** A complete SQL database in a single file (`metadata.db`)
- **Think of it as:** "PostgreSQL, but embedded in your app as a file"
- **Why SQLite?** Portable, fast, zero configuration

#### **Custom HTTP Server** 🌐
- **Role:** Web server for remote access (calibre.srv)
- **What it is:** Custom-built async HTTP server in pure Python
- **Think of it as:** "Express.js, but built from scratch in Python"
- **Serves:** REST API + SPA for web interface

#### **APSW (Another Python SQLite Wrapper)** 🔗
- **Role:** SQLite interface (replaces SQL libraries like pg/mysql2)
- **What it is:** Low-level SQLite bindings for Python
- **Think of it as:** "Like using raw SQL instead of Prisma"

---

<a name="architecture"></a>
## 3. Architecture Overview

### 🎯 **The Big Picture**

Calibre has **TWO main interfaces** to the same underlying data:

```
┌─────────────────────────────────────────────────────────────┐
│                     CALIBRE APPLICATION                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐              ┌──────────────────┐     │
│  │  Desktop GUI     │              │  Content Server  │     │
│  │  (PyQt6)         │              │  (Web Interface) │     │
│  │                  │              │                  │     │
│  │  • Book list     │              │  • REST API      │     │
│  │  • Editor        │              │  • SPA           │     │
│  │  • Conversion    │              │  • OPDS feed     │     │
│  └────────┬─────────┘              └────────┬─────────┘     │
│           │                                 │               │
│           └────────────┬────────────────────┘               │
│                        ▼                                    │
│              ┌──────────────────┐                           │
│              │  Database Layer  │                           │
│              │  (calibre.db)    │                           │
│              │                  │                           │
│              │  • Cache         │                           │
│              │  • ORM           │                           │
│              │  • Search (FTS)  │                           │
│              └────────┬─────────┘                           │
│                       ▼                                     │
│              ┌──────────────────┐                           │
│              │  SQLite Database │                           │
│              │  (metadata.db)   │                           │
│              │                  │                           │
│              │  • Books table   │                           │
│              │  • Authors       │                           │
│              │  • Tags, Series  │                           │
│              └──────────────────┘                           │
│                                                              │
│              ┌──────────────────┐                           │
│              │  Filesystem      │                           │
│              │  (Book files)    │                           │
│              │                  │                           │
│              │  /Author/Book/   │                           │
│              │    book.epub     │                           │
│              │    cover.jpg     │                           │
│              └──────────────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

### 🌉 **Bridge to React/Next.js Architecture**

If Calibre were a modern Next.js app:

```
Desktop GUI (PyQt6)     →  Would be: Electron/Tauri app with React
Content Server (srv)    →  Would be: Next.js API routes + App Router
Database Layer (db)     →  Would be: Prisma ORM + PostgreSQL
SQLite (metadata.db)    →  Would be: PostgreSQL database
Filesystem              →  Would be: S3/Cloud storage or local filesystem
```

**Key Difference:** Calibre is **desktop-first** with web as optional. A modern rewrite would likely be **web-first** with desktop as optional (Electron).

---

<a name="backend-101"></a>
## 4. Backend 101: Servers, Databases, and APIs

### 🧠 **Mental Model: Frontend vs Backend**

As a React developer, you've been working on the **client side**:

```javascript
// Frontend (React) - what you know
function BookList() {
  const [books, setBooks] = useState([]);

  useEffect(() => {
    fetch('/api/books')  // ← You send the request
      .then(res => res.json())
      .then(setBooks);
  }, []);

  return <div>{books.map(book => <BookCard {...book} />)}</div>;
}
```

Now you need to understand the **server side** - what happens when you `fetch('/api/books')`:

```python
# Backend (Python/Calibre) - what you're learning
@endpoint('/api/books')  # ← This receives your request
def get_books(ctx, request_data):
    # 1. Authentication - who are you?
    user = authenticate(request_data)

    # 2. Database query - get the data
    books = ctx.db.all_book_ids()  # ← Queries SQLite
    book_data = [ctx.db.get_metadata(id) for id in books]

    # 3. Transform - format for response
    response = [serialize_book(book) for book in book_data]

    # 4. Send response - return JSON
    return json(ctx, request_data, endpoint, response)
```

### 🎯 **Remember This:** The Backend's Three Jobs

1. **RECEIVE** requests (HTTP, routing)
2. **PROCESS** data (business logic, database queries)
3. **RESPOND** with results (JSON, HTML, files)

Think of it like a restaurant:
- **Frontend (React):** The customer who orders
- **Backend (Python):** The kitchen that prepares
- **Database (SQLite):** The pantry with ingredients
- **HTTP:** The waiter carrying messages

---

<a name="python-basics"></a>
## 5. Python for React Developers

### Side-by-Side Comparison

#### Variables & Types

```javascript
// JavaScript/TypeScript
const name = "Calibre";           // string
let count = 42;                   // number
const tags = ["fiction", "scifi"]; // array
const book = { title: "1984" };   // object
```

```python
# Python
name = "Calibre"           # str
count = 42                 # int
tags = ["fiction", "scifi"] # list
book = {"title": "1984"}   # dict (dictionary)
```

#### Functions

```javascript
// JavaScript
function getBookTitle(book) {
  return book.title || "Unknown";
}

const getAuthor = (book) => book.author;
```

```python
# Python
def get_book_title(book):
    return book.get('title') or "Unknown"

# Python doesn't have arrow functions, but lambdas exist
get_author = lambda book: book.get('author')
```

#### Classes & Components

```javascript
// React Component
class BookCard extends React.Component {
  constructor(props) {
    super(props);
    this.state = { isExpanded: false };
  }

  toggleExpand = () => {
    this.setState({ isExpanded: !this.state.isExpanded });
  };

  render() {
    return <div onClick={this.toggleExpand}>...</div>;
  }
}
```

```python
# Python Class
class BookWidget(QWidget):  # Inherits from QWidget
    def __init__(self, book_data):
        super().__init__()
        self.is_expanded = False  # instance variable

    def toggle_expand(self):
        self.is_expanded = not self.is_expanded
        self.update()  # re-render

    # Qt signal/slot connection (like event handlers)
    def paintEvent(self, event):
        # Draw the widget
        pass
```

#### Async/Promises

```javascript
// JavaScript
async function fetchBook(id) {
  const response = await fetch(`/api/books/${id}`);
  return await response.json();
}

fetchBook(123).then(book => console.log(book));
```

```python
# Python (async/await exists!)
import asyncio

async def fetch_book(id):
    response = await http_client.get(f"/api/books/{id}")
    return await response.json()

# Run async function
asyncio.run(fetch_book(123))
```

#### Array/List Operations

```javascript
// JavaScript
const titles = books.map(b => b.title);
const fiction = books.filter(b => b.genre === "fiction");
const total = numbers.reduce((sum, n) => sum + n, 0);
```

```python
# Python - List comprehensions!
titles = [b.title for b in books]
fiction = [b for b in books if b.genre == "fiction"]
total = sum(numbers)  # built-in sum()
```

### 💡 **Aha Moment:** Python is Like JavaScript's Simpler Cousin

Key similarities:
- ✅ Dynamically typed (no compile step)
- ✅ First-class functions
- ✅ Dictionaries/Objects work similarly
- ✅ Has async/await
- ✅ Duck typing ("if it quacks like a duck...")

Key differences:
- 🔄 Indentation matters (no braces `{}`)
- 🔄 `self` instead of `this`
- 🔄 Different naming convention: `snake_case` not `camelCase`
- 🔄 No semicolons needed
- 🔄 List comprehensions instead of map/filter chains

---

## 📚 **Quick Start Guide**

### Fastest Path to Understanding Calibre:

**Week 1: Backend Fundamentals**
1. Read [Backend 101: Servers, Databases, and APIs](#backend-101)
2. Read [Python for React Developers](#python-basics)
3. Follow setup guide: [Development Environment Setup](#setup)

**Week 2: Architecture Deep Dive**
4. Study [Database Architecture Deep Dive](#database-architecture)
5. Explore [HTTP Server Implementation](#http-server)
6. Read actual code: [src/calibre/srv/routes.py](../../src/calibre/srv/routes.py)

**Week 3: Modernization Analysis**
7. Review [Rewrite Analysis: React/Next.js Feasibility](#rewrite-analysis)
8. Study [Architecture Comparison](#architecture-comparison)
9. Contribute ideas to [Migration Strategy](#migration-strategy)

---

## 🎯 **Learning Objectives**

By the end of this guide, you will:
- ✅ Understand how backend servers work (HTTP, routing, middleware)
- ✅ Know how databases store and query data (SQL, ORM patterns)
- ✅ Read and understand Python code
- ✅ Comprehend Calibre's architecture (desktop + web)
- ✅ Evaluate the feasibility of a React/Next.js rewrite
- ✅ Make meaningful contributions to the codebase

---

## 📖 **Document Index**

All learning materials are in `docs/learning/`:

### Core Documentation
- **[GETTING_STARTED.md](./GETTING_STARTED.md)** - Set up your dev environment
- **[ARCHITECTURE_OVERVIEW.md](./ARCHITECTURE_OVERVIEW.md)** - Complete system architecture
- **[PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md)** - File organization guide
- **[TECH_STACK_GUIDE.md](./TECH_STACK_GUIDE.md)** - Technology deep dives

### Backend Learning
- **[BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md)** - Server & database patterns
- **[DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md)** - SQLite, ORM, queries
- **[DATA_FLOW_GUIDE.md](./DATA_FLOW_GUIDE.md)** - Request → Response lifecycle

### Rewrite Analysis
- **[REWRITE_ANALYSIS.md](./REWRITE_ANALYSIS.md)** - React/Next.js feasibility study
- **[MIGRATION_STRATEGY.md](./MIGRATION_STRATEGY.md)** - How to modernize Calibre

### Practical Guides
- **[HOW_TO_GUIDE.md](./HOW_TO_GUIDE.md)** - Common tasks cookbook
- **[CODE_TOURS.md](./CODE_TOURS.md)** - Guided code walkthroughs
- **[FIRST_CONTRIBUTIONS.md](./FIRST_CONTRIBUTIONS.md)** - Your first PR

---

## 🚀 **Ready to Dive In?**

Start here:
1. **[Backend 101 Section](#backend-101)** - Understand core concepts
2. **[GETTING_STARTED.md](./GETTING_STARTED.md)** - Set up locally
3. **[CODE_TOURS.md](./CODE_TOURS.md)** - Follow a request through the system

---

## 💭 **Common Questions**

**Q: Do I need to learn Python before I can contribute?**
A: Not really! Python is very readable. Start by reading code and you'll pick it up.

**Q: Can this really be rewritten in React/Next.js?**
A: The web interface? Absolutely. The desktop app? That's more complex. See [REWRITE_ANALYSIS.md](./REWRITE_ANALYSIS.md).

**Q: Why not just use Electron?**
A: Great question! See the architecture comparison in [REWRITE_ANALYSIS.md](./REWRITE_ANALYSIS.md#why-not-electron).

**Q: How is the web server implemented without Express/Fastify?**
A: Calibre uses a custom async HTTP server. It's like Express but built from scratch for maximum control. See [HTTP Server Implementation](#http-server).

---

## 🤝 **Contributing**

Found something unclear? See a mistake? Want to add more React ↔ Python comparisons?

1. Edit the relevant `.md` file in `docs/learning/`
2. Submit a PR with your improvements
3. Help the next React developer who joins!

---

**Next:** [Backend 101: Servers, Databases, and APIs](#backend-101) or [Get Your Environment Set Up](./GETTING_STARTED.md)
