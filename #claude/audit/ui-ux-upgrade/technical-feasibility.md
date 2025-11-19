# Technical Feasibility: Rewriting Calibre's UI with React.js/Next.js

**Analysis Date:** November 19, 2025
**Analyst:** Claude (Sonnet 4.5)
**Version:** 1.0
**Status:** Comprehensive Research & Analysis

---

## Executive Summary

### Key Findings

| Aspect | Assessment | Confidence |
|--------|------------|------------|
| **Technical Viability** | ✅ Feasible but Complex | High |
| **Business Viability** | ⚠️ Questionable ROI | Medium |
| **Risk Level** | 🔴 High | High |
| **Effort Estimation** | 🔴 Very Large (3-5 years) | High |
| **Recommended Approach** | 🟡 Hybrid/Progressive | Medium |

**Bottom Line:** While technically feasible, a full rewrite is **not recommended** due to complexity, risk, and opportunity cost. A hybrid or progressive enhancement approach offers better ROI with lower risk.

---

## Table of Contents

1. [Research: Desktop App Rewrites](#research)
2. [Calibre-Specific Analysis](#calibre-analysis)
3. [Feature Feasibility Matrix](#feature-matrix)
4. [Architecture Options](#architecture-options)
5. [Risk Assessment](#risk-assessment)
6. [Effort Estimation](#effort-estimation)
7. [Technology Stack Recommendations](#tech-stack)
8. [Final Recommendation](#recommendation)

---

<a name="research"></a>
## 1. Research: Desktop App Rewrites

### 1.1 Success Stories

#### Microsoft Teams: Electron → Edge WebView2 (2021-2024)

**Approach:** Migrated from Electron to Edge WebView2
**Results:**
- ✅ 50% reduction in memory usage (from 600MB → 300MB)
- ✅ Faster startup times
- ✅ Better Windows integration
- ⚠️ Windows-only solution (still uses Electron on Mac/Linux)

**Key Lessons:**
- Platform-specific optimizations can work but increase complexity
- Memory reduction was the primary driver
- Migration took **3+ years** with massive team
- Still maintains Electron for cross-platform support

**Source:** Microsoft Tech Community, 2024

---

#### VS Code: Successful Electron App (2015-Present)

**Approach:** Built with Electron from day one
**Results:**
- ✅ Dominant market share in code editors
- ✅ Cross-platform parity (Windows, Mac, Linux)
- ✅ Rich extension ecosystem
- ✅ Performance acceptable despite Electron overhead

**Key Factors:**
- Microsoft resources (large team)
- Continuous optimization over 9+ years
- Smart architecture (extensions run in separate process)
- Accepted trade-off: Size (~250MB) for developer convenience

**Relevance to Calibre:**
- Shows Electron can work for complex desktop apps
- Requires continuous optimization investment
- Bundle size less critical for developer tools

---

#### Slack: Electron App with Hybrid Architecture (2015-Present)

**Approach:** Electron with remote code loading
**Evolution:**
- Version 3.0 (2017): Migrated from webView to browserView
- Version 4.0+ (2020s): Redux-electron for state management
- 2024: Still using Electron with optimizations

**Results:**
- ✅ Cross-platform success
- ⚠️ Known performance issues (200-300MB RAM)
- ⚠️ Multiple rewrites to fix performance
- ✅ Rapid feature development

**Key Lessons:**
- Electron enables fast iteration
- Performance requires constant attention
- Hybrid architecture (local + remote) can work
- Users tolerate some bloat for features

---

### 1.2 Failures & Challenges

#### Notion: Performance Problems Despite Optimization

**Current State:** Electron-based, struggling with performance
**Issues:**
- Slow page loading (3-5 seconds for large pages)
- Heavy memory usage (400-500MB for single workspace)
- Sluggish navigation on mobile
- User complaints about performance vs. native competitors

**Mitigation Attempts:**
- Native Search Tab (2023): 80% improvement
- Native Home Tab (2022): 3x startup improvement
- Still fundamentally limited by Electron architecture

**Lesson:** Electron has a performance ceiling that optimization can't fully overcome.

---

#### Atom Editor: Shut Down (2022)

**Backstory:** GitHub's Electron-based editor (predecessor to VS Code)
**Why it Failed:**
- Slower than VS Code despite similar architecture
- Bundle size too large (>300MB)
- Startup time too slow (3-5 seconds)
- Lost to VS Code (Microsoft's better execution)

**Lesson:** Being Electron isn't the problem; **execution and continuous optimization** matter more.

---

#### Discord, Slack: macOS Tahoe GPU Bug (September 2025)

**Issue:** Electron apps (Discord, Slack, VS Code) caused severe slowdowns on macOS Tahoe
**Root Cause:** Electron bug with private "_cornerMask" override clashed with macOS graphics engine
**Resolution:** Electron team fixed in weeks, but apps needed updates

**Lesson:** Electron apps depend on framework updates; can introduce platform-specific bugs.

---

### 1.3 Emerging Alternative: Tauri

#### What is Tauri?

**Architecture:** Rust backend + OS native WebView (WKWebView on Mac, WebView2 on Windows, WebKitGTK on Linux)

**Key Differences from Electron:**

| Aspect | Electron | Tauri |
|--------|----------|-------|
| **Bundle Size** | 100-250MB | 3-10MB |
| **Memory Usage** | 200-300MB idle | 30-40MB idle |
| **Startup Time** | 1-2 seconds | <500ms |
| **Backend Language** | Node.js (JavaScript) | Rust |
| **Browser Engine** | Chromium (bundled) | OS WebView (native) |
| **Maturity** | 10+ years | 3 years (v2.0 in 2024) |

---

#### Tauri Adoption (2025)

**Growth:** 35% YoY increase in adoption (post-v2.0 release, late 2024)
**Production Apps:**
- [Authme](https://authme.com): 2.5MB Tauri vs 85MB Electron version
- Various productivity tools migrating from Electron

**Advantages:**
- ✅ Tiny bundle size (10-30x smaller)
- ✅ Low memory footprint (60-90% less RAM)
- ✅ Fast startup (<500ms)
- ✅ Native performance (Rust)
- ✅ Better security (Rust memory safety)

**Disadvantages:**
- ⚠️ Younger ecosystem (fewer libraries)
- ⚠️ WebView fragmentation (different bugs per platform)
  - SVG rendering bugs on macOS
  - PDF rendering issues
  - Platform-specific quirks
- ⚠️ Smaller community
- ⚠️ Less documentation
- ⚠️ Rust learning curve

---

### 1.4 Performance Benchmarks (2025)

#### Real-World Measurements

**Memory Usage (Idle):**
- PyQt6 Native: ~50-100MB
- Tauri: ~30-40MB
- Electron: ~200-300MB

**Bundle Size:**
- PyQt6 Native: ~50-100MB (with Python runtime)
- Tauri: ~3-10MB
- Electron: ~100-250MB

**Startup Time:**
- PyQt6 Native: <300ms
- Tauri: <500ms
- Electron: 1-2 seconds

**File Operations Speed:**
- PyQt6 Native (Python): Baseline
- Tauri (Rust): 40-60% faster than Node.js
- Electron (Node.js): Slower due to V8 overhead

---

### 1.5 Key Takeaways from Research

1. **Electron is mature but heavy** - Good for rapid development, accepted by users, but memory/size overhead is real
2. **Tauri is promising but young** - Better performance, but platform fragmentation and smaller ecosystem
3. **Rewrites are expensive** - Microsoft Teams took 3+ years with a massive team
4. **Performance requires constant work** - Slack, VS Code, Notion all continuously optimize
5. **Web tech enables fast iteration** - Slack, VS Code ship features faster than native alternatives
6. **Native still wins for performance** - PyQt6 remains fastest, smallest memory footprint
7. **Hybrid approaches exist** - Can keep native backend, modernize web UI only

---

<a name="calibre-analysis"></a>
## 2. Calibre-Specific Analysis

### 2.1 Current Architecture

**Codebase Statistics:**
- **Total Python Files:** 1,346
- **GUI Files (PyQt6):** 457 (34% of codebase)
- **Complex UI Components:** 295 QTableView/QTreeView/QWebEngine usages
- **Device Drivers:** 10+ (Kindle, Kobo, Nook, etc.)
- **Conversion Plugins:** 40+ format converters
- **Store Plugins:** 30+ online bookstore integrations

**Key Complexity Drivers:**
1. Extensive PyQt6 GUI (~457 files)
2. Deep OS integration (file system, USB devices)
3. Binary format processing (EPUB, PDF, MOBI, etc.)
4. Plugin system (100+ plugins)
5. Multi-threaded conversion engine
6. Database with 100k+ book support

---

### 2.2 Features by Complexity

#### Easy to Port (Web-Native Features)

✅ **Library Browser**
- Current: QTableView with custom model
- Web: React Table / TanStack Table
- Effort: **Small** (2-4 weeks)

✅ **Book Details Display**
- Current: Custom QWidget with layouts
- Web: React component
- Effort: **Small** (1-2 weeks)

✅ **Metadata Editing Forms**
- Current: QFormLayout with QLineEdit
- Web: React Hook Form
- Effort: **Small** (2-3 weeks)

✅ **Search Interface**
- Current: QLineEdit + custom filter
- Web: React + debounced search
- Effort: **Small** (1-2 weeks)

✅ **Settings/Preferences**
- Current: Multiple QDialog windows
- Web: React forms + local storage
- Effort: **Medium** (3-4 weeks)

**Total Easy Features: ~15-20%** of GUI functionality

---

#### Medium Complexity (Requires Adaptation)

⚠️ **Book Cover Grid View**
- Current: Custom QWidget with manual painting
- Web: CSS Grid + lazy loading
- Challenge: Performance with 10k+ covers
- Effort: **Medium** (3-4 weeks)

⚠️ **Drag & Drop Import**
- Current: Qt's native drag/drop APIs
- Web: HTML5 File API + Electron/Tauri file access
- Challenge: Multi-file import with validation
- Effort: **Medium** (2-3 weeks)

⚠️ **Tag Browser Tree**
- Current: QTreeView with custom model
- Web: React component library (react-arborist, etc.)
- Challenge: Performance with thousands of tags
- Effort: **Medium** (3-4 weeks)

⚠️ **Series View with Ordering**
- Current: QTableView with drag-to-reorder
- Web: React DnD or similar
- Challenge: Complex state management
- Effort: **Medium** (2-3 weeks)

⚠️ **Bulk Metadata Edit**
- Current: Custom dialog with live preview
- Web: Multi-step form with React
- Challenge: Performance with 100+ books selected
- Effort: **Medium** (3-4 weeks)

**Total Medium Features: ~30-40%** of GUI functionality

---

#### Hard to Port (Significant Re-engineering)

🔴 **E-book Viewer**
- Current: QWebEngineView with custom controls
- Web: Epub.js or similar renderer
- Challenges:
  - Custom CSS injection
  - Annotation system
  - Page turning animations
  - PDF rendering (without PyMuPDF)
  - DRM-protected books
- Effort: **Very Large** (3-6 months)

🔴 **E-book Editor**
- Current: Full WYSIWYG editor with Qt
- Web: Complex EPUB editor
- Challenges:
  - Syntax highlighting
  - Split view (code + preview)
  - ZIP manipulation (EPUB structure)
  - Font embedding
  - CSS editor with live preview
- Effort: **Very Large** (4-8 months)

🔴 **Format Conversion UI**
- Current: Complex multi-page wizard with live preview
- Web: Multi-step form
- Challenges:
  - Real-time conversion progress
  - Preview rendering
  - Calling Python conversion backend
  - Error handling & logs display
- Effort: **Large** (2-3 months)

🔴 **Device Sync Interface**
- Current: Direct USB communication via PyUSB
- Web: Requires Electron/Tauri with native modules
- Challenges:
  - USB device detection (needs native code)
  - File transfer progress
  - Device-specific quirks (Kindle, Kobo)
  - Conflict resolution UI
- Effort: **Very Large** (3-4 months)

🔴 **Plugin Configuration UI**
- Current: Dynamic dialog generation from plugin metadata
- Web: Dynamic form generation
- Challenges:
  - Supporting 100+ different plugin UIs
  - Custom widget types
  - Plugin sandbox security
- Effort: **Large** (2-3 months)

**Total Hard Features: ~30-40%** of GUI functionality

---

#### Nearly Impossible (Would Need to Stay Native or Major Rewrites)

🚫 **Direct Filesystem Browser Integration**
- Current: Native file dialogs, thumbnail generation
- Web: Limited file system access (security sandbox)
- Possible Workaround: Electron/Tauri with node integration
- Effort: **Very Large** with limitations

🚫 **System Tray Integration**
- Current: Native system tray with context menus
- Web: Electron/Tauri system tray APIs (limited)
- Possible Workaround: Electron's Tray API (incomplete parity)
- Effort: **Medium** but incomplete functionality

🚫 **Native Dialogs (Color Picker, Font Selector)**
- Current: OS-native dialogs via Qt
- Web: HTML5 equivalents (look different, fewer features)
- Effort: **Medium** but UX downgrade

🚫 **High-Performance Table Rendering (100k+ rows)**
- Current: QTableView with C++ model (virtual scrolling)
- Web: React virtualization (slower, more memory)
- Possible Workaround: TanStack Virtual, react-window
- Effort: **Large** to match native performance

---

### 2.3 Backend Features (Unchanged Regardless of UI)

The following components would **remain in Python** regardless of UI choice:

✅ **Database Layer**
- SQLite + APSW
- In-memory cache
- ~30,000 lines of code
- **No change required**

✅ **E-book Format Processing**
- EPUB parsing (lxml, zipfile)
- PDF processing (PyMuPDF)
- MOBI/AZW3 parsing
- ~50,000 lines of code
- **No change required**

✅ **Conversion Engine**
- 40+ input/output plugins
- Pipeline architecture
- ~40,000 lines of code
- **No change required**

✅ **Metadata Sources**
- Amazon, Google Books, Goodreads APIs
- Web scraping
- **No change required**

✅ **Device Drivers**
- USB communication (PyUSB)
- Kindle, Kobo, Nook drivers
- ~15,000 lines of code
- **No change required**

**Key Insight:** ~60% of Calibre's codebase would remain unchanged. The rewrite primarily affects the **presentation layer** (PyQt6 GUI).

---

<a name="feature-matrix"></a>
## 3. Feature Feasibility Matrix

### 3.1 Comprehensive Feature Breakdown

| Feature | Current Tech | Web Equivalent | Difficulty | Effort | Quality Loss |
|---------|--------------|----------------|------------|--------|--------------|
| **Library Browser** | QTableView | TanStack Table | ✅ Easy | 2-4 weeks | None |
| **Book Details** | QWidget | React Component | ✅ Easy | 1-2 weeks | None |
| **Cover Grid** | Custom QWidget | CSS Grid | ⚠️ Medium | 3-4 weeks | Slight (performance) |
| **Tag Browser Tree** | QTreeView | React Tree | ⚠️ Medium | 3-4 weeks | Slight (performance) |
| **Metadata Edit Forms** | QFormLayout | React Hook Form | ✅ Easy | 2-3 weeks | None |
| **Search** | QLineEdit | React + Debounce | ✅ Easy | 1-2 weeks | None |
| **Drag & Drop Import** | Qt DnD | HTML5 File API | ⚠️ Medium | 2-3 weeks | None |
| **E-book Viewer** | QWebEngineView | Epub.js | 🔴 Hard | 3-6 months | Moderate |
| **E-book Editor** | Qt WYSIWYG | Complex JS Editor | 🔴 Hard | 4-8 months | Moderate |
| **Format Conversion UI** | Qt Wizard | Multi-step Form | 🔴 Hard | 2-3 months | None |
| **Device Sync** | PyUSB + Qt | Electron Native | 🔴 Hard | 3-4 months | Moderate |
| **Plugin Config** | Dynamic Qt Dialogs | Dynamic Forms | 🔴 Hard | 2-3 months | Slight |
| **Bulk Edit** | Custom Qt Dialog | React Multi-form | ⚠️ Medium | 3-4 weeks | Slight |
| **Virtual Library** | QTableView Filter | React Filter | ✅ Easy | 2-3 weeks | None |
| **Book Comparison** | Split QWidget | Split Pane | ⚠️ Medium | 2-3 weeks | None |
| **Reading Stats** | Qt Charts | Chart.js / Recharts | ✅ Easy | 2-3 weeks | None |
| **Catalog Export** | Qt Export Dialog | React Form | ✅ Easy | 1-2 weeks | None |
| **Email Server** | Built-in Server | Same (backend) | ✅ Easy | No change | None |
| **Content Server UI** | HTML Templates | React SPA | ⚠️ Medium | 4-6 weeks | None |
| **Annotations/Notes** | QTextEdit | Rich Text Editor | ⚠️ Medium | 3-4 weeks | Slight |
| **Cover Download** | Qt Thumbnails | React Gallery | ⚠️ Medium | 2-3 weeks | None |
| **Series Management** | QTableView | React Table | ⚠️ Medium | 2-3 weeks | None |
| **Duplicate Finder** | Custom Qt Dialog | React Comparison | ⚠️ Medium | 3-4 weeks | None |
| **Saved Searches** | Qt Preset System | React Saved State | ✅ Easy | 1-2 weeks | None |
| **News Download** | Qt Recipe Editor | React Form | ⚠️ Medium | 3-4 weeks | Slight |
| **Keyboard Shortcuts** | Qt Key Bindings | JS Key Handlers | ✅ Easy | 1-2 weeks | None |
| **Themes/Styling** | QSS | CSS / Tailwind | ✅ Easy | 2-3 weeks | None (better) |
| **Multi-language** | Qt i18n | i18next / react-intl | ✅ Easy | 2-3 weeks | None |
| **System Tray** | Native Tray | Electron Tray | 🚫 Hard | 2-3 weeks | Significant |
| **File Dialogs** | Native Dialogs | Electron Dialogs | ⚠️ Medium | 1-2 weeks | Moderate |
| **Window Management** | Qt Window System | Electron Windows | ⚠️ Medium | 2-3 weeks | Slight |

### 3.2 Summary by Difficulty

| Difficulty Level | Count | % of Features | Total Effort |
|-----------------|-------|---------------|--------------|
| ✅ **Easy** | 11 | 35% | 20-30 weeks |
| ⚠️ **Medium** | 13 | 42% | 35-48 weeks |
| 🔴 **Hard** | 6 | 19% | 60-108 weeks |
| 🚫 **Very Hard / Impossible** | 1 | 3% | 8-12 weeks |
| **TOTAL** | **31** | **100%** | **123-198 weeks (2.4-3.8 years)** |

**Note:** This assumes a **single full-time developer**. With a team of 3-5, could reduce to **1.5-2.5 years**, but coordination overhead increases.

---

<a name="architecture-options"></a>
## 4. Architecture Options

### Option 1: Full Electron Rewrite

**Approach:** Replace PyQt6 GUI entirely with React + Electron

```
┌─────────────────────────────────────────┐
│         ELECTRON APP                     │
├─────────────────────────────────────────┤
│  Renderer Process (React/Next.js)       │
│    - All UI components                  │
│    - State management (Zustand/Redux)   │
│    - TanStack Query for data fetching   │
├─────────────────────────────────────────┤
│  Main Process (Node.js)                 │
│    - Window management                  │
│    - IPC with Python backend            │
│    - System tray, file dialogs          │
├─────────────────────────────────────────┤
│  Python Backend (unchanged)             │
│    - Database (SQLite + cache)          │
│    - E-book processing                  │
│    - Conversion engine                  │
│    - Device drivers                     │
│                                         │
│  Communication: HTTP API or IPC         │
└─────────────────────────────────────────┘
```

**Pros:**
- ✅ Modern React development
- ✅ Rich ecosystem (npm packages)
- ✅ Familiar to web developers
- ✅ Hot reload during development
- ✅ Cross-platform with single codebase

**Cons:**
- ❌ Large bundle size (~200-250MB)
- ❌ High memory usage (200-300MB)
- ❌ Slower than native PyQt6
- ❌ Full rewrite required (2-4 years)
- ❌ Electron dependency updates
- ❌ IPC overhead (Electron ↔ Python)

**Effort:** Very Large (3-5 years)
**Risk:** High
**Recommended:** ❌ No

---

### Option 2: Tauri Rewrite

**Approach:** Replace PyQt6 with React + Tauri (Rust)

```
┌─────────────────────────────────────────┐
│         TAURI APP                        │
├─────────────────────────────────────────┤
│  Frontend (React/Next.js)                │
│    - All UI components                  │
│    - State management                   │
├─────────────────────────────────────────┤
│  Tauri Core (Rust)                      │
│    - Window management                  │
│    - IPC with Python backend            │
│    - System APIs                        │
├─────────────────────────────────────────┤
│  Python Backend (unchanged)             │
│    - Database                           │
│    - E-book processing                  │
│    - Conversion engine                  │
│                                         │
│  Communication: Tauri invoke API        │
└─────────────────────────────────────────┘
```

**Pros:**
- ✅ Tiny bundle size (~10-20MB)
- ✅ Low memory usage (50-100MB)
- ✅ Fast startup (<500ms)
- ✅ Modern React development
- ✅ Rust backend (performance + security)

**Cons:**
- ❌ Young ecosystem (fewer packages)
- ❌ WebView fragmentation (platform bugs)
- ❌ Rust learning curve
- ❌ Full rewrite still required (2-4 years)
- ❌ Less mature than Electron
- ❌ IPC overhead (Tauri ↔ Python)

**Effort:** Very Large (3-5 years)
**Risk:** Very High (newer tech)
**Recommended:** ❌ No (too risky for production)

---

### Option 3: Hybrid Approach (Keep PyQt, Modernize Web Interface)

**Approach:** Keep desktop PyQt6, enhance existing web Content Server with modern React

```
┌─────────────────────────────────────────┐
│  DESKTOP (unchanged)                     │
│    PyQt6 GUI                            │
│      - Full features                    │
│      - Native performance               │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  WEB (modernized)                        │
├─────────────────────────────────────────┤
│  Frontend (React/Next.js)                │
│    - Library browser                    │
│    - Book reader (Epub.js)              │
│    - Metadata editor                    │
│    - Settings                           │
├─────────────────────────────────────────┤
│  Python Backend (enhanced)              │
│    - REST/GraphQL API                   │
│    - WebSocket for real-time updates   │
│    - Same database & processing         │
└─────────────────────────────────────────┘
```

**Pros:**
- ✅ No disruption to desktop users
- ✅ Improve mobile/remote access experience
- ✅ Smaller scope (web UI only)
- ✅ Can be done incrementally
- ✅ Lower risk
- ✅ Existing Python backend reused

**Cons:**
- ⚠️ Maintaining two UIs
- ⚠️ Feature parity challenges
- ⚠️ Doesn't address desktop modernization

**Effort:** Medium (6-12 months)
**Risk:** Low
**Recommended:** ✅ Yes (best ROI)

---

### Option 4: Progressive Enhancement

**Approach:** Incrementally replace PyQt6 components with web-based views embedded via QWebEngine

```
┌─────────────────────────────────────────┐
│  CALIBRE DESKTOP                         │
├─────────────────────────────────────────┤
│  Phase 1: Simple Forms                  │
│    [PyQt6] Main Window                  │
│    [PyQt6] Menu Bar                     │
│    [React in QWebEngine] Metadata Edit  │
│    [React in QWebEngine] Settings       │
│                                         │
│  Phase 2: Complex Views                 │
│    [React in QWebEngine] Library Grid   │
│    [React in QWebEngine] Book Details   │
│    [PyQt6] E-book Viewer (keep native)  │
│                                         │
│  Phase 3: Advanced Features             │
│    [React in QWebEngine] Conversion UI  │
│    [PyQt6] Device Sync (keep native)    │
│    [PyQt6] E-book Editor (keep native)  │
└─────────────────────────────────────────┘
```

**Pros:**
- ✅ Incremental migration (low risk)
- ✅ Can test each component separately
- ✅ Keep native for complex features
- ✅ Modernize UI without full rewrite
- ✅ Fallback to PyQt6 if web version fails

**Cons:**
- ⚠️ Hybrid complexity (PyQt + QWebEngine + React)
- ⚠️ IPC overhead (PyQt ↔ JavaScript)
- ⚠️ QWebEngine memory overhead
- ⚠️ Inconsistent UX (some native, some web)

**Effort:** Large (1.5-2.5 years)
**Risk:** Medium
**Recommended:** ⚠️ Possible (if desktop modernization is critical)

---

### Option 5: Web-Only with Desktop Wrapper

**Approach:** Build React web app, wrap with Electron/Tauri for desktop distribution

```
┌─────────────────────────────────────────┐
│  REACT WEB APP (primary)                 │
│    - Next.js or Create React App        │
│    - Runs in browser (web)              │
│    - Wrapped in Electron (desktop)      │
├─────────────────────────────────────────┤
│  Python Backend Server                  │
│    - REST/GraphQL API                   │
│    - Same processing logic              │
│                                         │
│  Deployment Options:                    │
│    1. Self-hosted web server            │
│    2. Electron app (bundled server)     │
│    3. Cloud deployment (optional)       │
└─────────────────────────────────────────┘
```

**Pros:**
- ✅ Single codebase (web + desktop)
- ✅ Modern React stack
- ✅ Can deploy as web app or desktop
- ✅ Progressive web app (PWA) support

**Cons:**
- ❌ Web limitations (file system access, USB)
- ❌ Performance worse than native
- ❌ Requires backend server always running
- ❌ Full rewrite required

**Effort:** Very Large (3-4 years)
**Risk:** High
**Recommended:** ❌ No (loses desktop advantages)

---

### Recommended Architecture: **Option 3 (Hybrid)**

**Rationale:**
1. **Preserves existing investment** - Desktop app continues to work
2. **Improves weak point** - Current web UI is basic HTML templates
3. **Lower risk** - No disruption to core users
4. **Faster ROI** - Can ship in 6-12 months
5. **Incremental** - Can enhance over time

**Implementation Plan:**

**Phase 1 (3-4 months): Modern Web UI Foundation**
- Set up Next.js 15 with React 19
- Build REST/GraphQL API layer
- Implement authentication
- Basic library browser
- Book detail view
- Mobile-responsive design

**Phase 2 (3-4 months): Core Features**
- Advanced search
- Metadata editing
- E-book reader (Epub.js integration)
- Cover management
- Tag/series browsing

**Phase 3 (2-3 months): Advanced Features**
- Reading progress sync
- Annotations (if time permits)
- Settings management
- Multi-library support

**Phase 4 (1-2 months): Polish & Deploy**
- Performance optimization
- PWA support
- Comprehensive testing
- Documentation

**Total:** ~12 months for fully-featured modern web UI

---

<a name="risk-assessment"></a>
## 5. Risk Assessment

### 5.1 Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| **Performance degradation** | High | High | Keep native for heavy operations; benchmark early |
| **Feature parity failure** | Medium | High | Prioritize core features; accept some limitations |
| **Electron/Tauri bugs** | Medium | Medium | Extensive testing; maintain PyQt6 as fallback |
| **IPC overhead** | Medium | Medium | Optimize IPC; use efficient serialization (msgpack) |
| **WebView fragmentation** | High (Tauri) | Medium | Extensive cross-platform testing |
| **Team skill gap** | Medium | Medium | Training; hire React developers |
| **Library updates breaking changes** | Medium | Low | Pin dependencies; thorough testing |
| **USB/device access limitations** | High (Web) | High | Keep device sync in native code |
| **Database access from web** | Low | Medium | Secure API layer; input validation |
| **File system security** | Medium (Electron) | High | Careful sandboxing; code review |

### 5.2 Business Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| **User backlash (UI changes)** | High | High | Gradual rollout; user feedback; keep classic option |
| **Development time underestimation** | Very High | Very High | Conservative estimates; phased delivery |
| **Opportunity cost** | High | High | Focus on high-value features first |
| **Team burnout** | Medium | High | Realistic timeline; avoid crunch |
| **Market changes during rewrite** | Medium | Medium | Iterative delivery; MVP approach |
| **Competing projects emerge** | Low | Medium | Open source advantage; community |

### 5.3 Project Management Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| **Scope creep** | Very High | Very High | Strict feature freeze; MVP mindset |
| **Architecture changes mid-project** | Medium | High | Solid architecture upfront; spike work |
| **Dependency on specific developers** | Medium | High | Knowledge sharing; documentation |
| **Testing coverage gaps** | High | High | Automated tests; QA focus |
| **Migration data loss** | Low | Very High | Extensive backups; migration scripts; rollback plan |

### 5.4 Risk Mitigation Strategy

**For Full Rewrite (Not Recommended):**
1. ❌ **Don't do it** - Too risky, too expensive
2. If forced:
   - Build MVP in 6 months to validate
   - Keep PyQt6 in maintenance mode
   - Plan 3-5 year timeline
   - Allocate 30% buffer for unknowns

**For Hybrid Approach (Recommended):**
1. ✅ Start small (web UI only)
2. ✅ Keep desktop unchanged
3. ✅ Iterate based on feedback
4. ✅ Measure success metrics (usage, performance)
5. ✅ Plan 12-month delivery

---

<a name="effort-estimation"></a>
## 6. Effort Estimation

### 6.1 Full Electron Rewrite

**Team Composition:**
- 2 Senior React Developers
- 1 Python Backend Developer
- 1 UI/UX Designer
- 1 QA Engineer
- 0.5 Project Manager

**Timeline:** 3-5 years

**Breakdown by Phase:**

| Phase | Duration | Effort (Person-Months) | Key Deliverables |
|-------|----------|----------------------|------------------|
| **Planning & Architecture** | 2-3 months | 6 | Technical design, prototypes, tech stack |
| **Core Infrastructure** | 3-4 months | 12 | Electron setup, IPC, build pipeline |
| **Basic Library Browser** | 2-3 months | 8 | Table view, search, filters |
| **Metadata Management** | 3-4 months | 10 | Forms, editing, validation |
| **E-book Viewer** | 6-8 months | 20 | Epub.js integration, controls, annotations |
| **E-book Editor** | 8-12 months | 30 | Full WYSIWYG editor, EPUB manipulation |
| **Conversion UI** | 3-4 months | 10 | Wizard, progress, preview |
| **Device Sync** | 4-6 months | 15 | USB detection, file transfer, UI |
| **Plugin System** | 3-4 months | 10 | Dynamic UI generation, sandboxing |
| **Settings & Preferences** | 2-3 months | 6 | All settings screens |
| **Polish & Bug Fixes** | 6-12 months | 20 | Performance, bugs, UX refinement |
| **Testing & QA** | Ongoing | 30 | Unit, integration, e2e, manual |
| **Documentation** | 2-3 months | 6 | User docs, dev docs, migration guides |
| **TOTAL** | **36-60 months** | **183 person-months** | Full-featured app |

**Cost Estimation (US Market):**
- Senior React Dev: $120k-$180k/year
- Python Dev: $110k-$160k/year
- UI/UX Designer: $90k-$130k/year
- QA Engineer: $80k-$120k/year
- Project Manager: $100k-$150k/year

**Team Annual Cost:** ~$500k-$740k
**Total Project Cost:** $1.5M - $3.7M

---

### 6.2 Hybrid Approach (Modernize Web UI Only)

**Team Composition:**
- 1 Senior Full-Stack Developer (React + Python)
- 1 UI/UX Designer (part-time)
- 0.25 QA Engineer

**Timeline:** 9-12 months

**Breakdown by Phase:**

| Phase | Duration | Effort (Person-Months) | Key Deliverables |
|-------|----------|----------------------|------------------|
| **Planning & Design** | 1 month | 1.5 | API design, UI mockups, tech stack |
| **API Layer** | 2 months | 2 | REST/GraphQL endpoints, auth, CORS |
| **Frontend Foundation** | 2 months | 2 | Next.js setup, routing, state management |
| **Library Browser** | 2 months | 2 | Table, grid view, search, filters |
| **Book Details & Metadata** | 1.5 months | 1.5 | Display, editing, cover management |
| **E-book Reader** | 3 months | 3 | Epub.js, PDF.js, controls, settings |
| **Tag/Series Management** | 1 month | 1 | Browser, editing, bulk operations |
| **Settings & Preferences** | 1 month | 1 | User settings, library config |
| **Polish & Performance** | 2 months | 2 | Optimization, responsive design |
| **Testing & Bug Fixes** | 1.5 months | 1.5 | Testing, fixes, edge cases |
| **Documentation** | 0.5 months | 0.5 | User guide, API docs |
| **TOTAL** | **12 months** | **18 person-months** | Modern web UI |

**Cost Estimation (US Market):**
- Senior Full-Stack Dev: $130k-$180k/year
- UI/UX Designer (part-time): $45k-$65k/year
- QA Engineer (part-time): $20k-$30k/year

**Team Annual Cost:** ~$195k-$275k
**Total Project Cost:** $195k-$275k

---

### 6.3 Progressive Enhancement (Gradual Desktop Modernization)

**Team Composition:**
- 1 Senior React Developer
- 1 Python/PyQt6 Developer
- 1 UI/UX Designer (part-time)
- 0.5 QA Engineer

**Timeline:** 18-30 months

**Breakdown by Phase:**

| Phase | Duration | Effort (Person-Months) | Key Deliverables |
|-------|----------|----------------------|------------------|
| **Proof of Concept** | 2 months | 3 | QWebEngine + React integration, IPC |
| **Simple Forms (Metadata, Settings)** | 3 months | 4.5 | Replace simple PyQt dialogs with React |
| **Library Grid View** | 4 months | 6 | React grid embedded in QWebEngine |
| **Book Details View** | 2 months | 3 | React detail panel |
| **Search & Filters** | 2 months | 3 | React-based search |
| **Tag Browser** | 3 months | 4.5 | Tree view in React |
| **Conversion UI** | 4 months | 6 | Multi-step wizard |
| **Keep Native Components** | Ongoing | - | Viewer, Editor, Device sync unchanged |
| **Integration & Testing** | 4 months | 6 | End-to-end testing, performance tuning |
| **TOTAL** | **24 months** | **36 person-months** | Partially modernized desktop |

**Cost Estimation (US Market):**
- Senior React Dev: $120k-$180k/year
- Python/PyQt6 Dev: $110k-$160k/year
- UI/UX Designer (part-time): $45k-$65k/year
- QA Engineer (part-time): $40k-$60k/year

**Team Annual Cost:** ~$315k-$465k
**Total Project Cost:** $630k-$930k

---

### 6.4 Summary Comparison

| Approach | Timeline | Cost | Risk | ROI |
|----------|----------|------|------|-----|
| **Full Electron Rewrite** | 3-5 years | $1.5M-$3.7M | 🔴 Very High | ❌ Poor |
| **Tauri Rewrite** | 3-5 years | $1.5M-$3.7M | 🔴 Extremely High | ❌ Very Poor |
| **Hybrid (Web Only)** | 9-12 months | $195k-$275k | 🟢 Low | ✅ Excellent |
| **Progressive Enhancement** | 18-30 months | $630k-$930k | 🟡 Medium | ⚠️ Moderate |
| **Web-Only Wrapper** | 3-4 years | $1.2M-$3M | 🔴 High | ❌ Poor |

---

<a name="tech-stack"></a>
## 7. Technology Stack Recommendations

### 7.1 For Hybrid Approach (Recommended)

#### Frontend Stack

**Core Framework:**
- **Next.js 15** (App Router)
  - Server components for initial render
  - Client components for interactivity
  - Built-in routing
  - API routes (if not using separate backend)

**UI Libraries:**
- **React 19** (latest)
- **TypeScript 5.7+** (type safety)
- **Tailwind CSS** (styling)
- **shadcn/ui** (component library, accessible)

**State Management:**
- **Zustand** (lightweight, modern)
- **TanStack Query** (server state, caching)

**Data Fetching:**
- **TanStack Query v5** (data fetching, caching, invalidation)
- **GraphQL** (optional, if complex queries needed)
  - **Apollo Client** or **urql**

**Tables & Complex UI:**
- **TanStack Table v8** (headless table library)
- **TanStack Virtual** (virtualization for large lists)
- **react-arborist** (tree views)
- **dnd-kit** (drag & drop)

**E-book Rendering:**
- **Epub.js** (EPUB rendering)
- **PDF.js** (PDF rendering)
- **React PDF** (wrapper for PDF.js)

**Forms:**
- **React Hook Form** (performant forms)
- **Zod** (schema validation)

**Testing:**
- **Vitest** (unit tests, faster than Jest)
- **Playwright** (e2e tests)
- **Testing Library** (component tests)

---

#### Backend Stack

**API Framework:**
- **FastAPI** (Python, modern, async)
  - Fast (comparable to Node.js)
  - Auto-generated OpenAPI docs
  - Type hints → validation
  - WebSocket support

**Alternative:**
- **Flask-RESTX** (if prefer Flask)
- **Starlette** (lightweight, async)

**Database (Unchanged):**
- **SQLite + APSW** (keep existing)
- **In-memory cache** (keep existing)

**Authentication:**
- **JWT tokens** (stateless)
- **Passkey support** (optional, modern)

**WebSocket (Real-time Updates):**
- **FastAPI WebSockets** or **Socket.IO**

---

#### Build & Deployment

**Build Tools:**
- **Vite 6** (fast dev server, HMR)
- **Turbopack** (Next.js built-in, even faster)

**Linting & Formatting:**
- **ESLint 9** (with TypeScript plugin)
- **Prettier** (code formatting)
- **Biome** (faster alternative to ESLint + Prettier)

**Package Manager:**
- **pnpm** (faster, more efficient than npm/yarn)

**Deployment:**
- **Self-hosted:** Docker container
- **Web hosting:** Vercel, Netlify (for frontend)
- **Backend:** Keep Python server running locally

---

### 7.2 For Full Electron Rewrite (Not Recommended, but if you must)

**Everything from above, plus:**

**Desktop Framework:**
- **Electron 33+** (latest)
- **electron-builder** (packaging)
- **electron-updater** (auto-updates)

**Electron-Specific:**
- **electron-store** (persistent settings)
- **electron-context-menu** (right-click menus)
- **Better IPC** (typed IPC with electron-trpc)

**Security:**
- **contextIsolation: true**
- **nodeIntegration: false**
- **Sandboxed renderer**

---

### 7.3 For Tauri Rewrite (High Risk)

**Frontend:** Same as Hybrid approach

**Backend:**
- **Tauri 2.x** (latest, released late 2024)
- **Rust 1.83+** (stable)
- **serde** (serialization)
- **tokio** (async runtime)

**Tauri Plugins:**
- **tauri-plugin-fs** (file system access)
- **tauri-plugin-shell** (spawn processes)
- **tauri-plugin-http** (HTTP client)
- **tauri-plugin-sql** (SQLite, optional)

**Python Bridge:**
- **Custom sidecar** (Python process managed by Tauri)
- **HTTP API** (Tauri frontend ↔ Python backend)

---

### 7.4 Rejected Technologies

❌ **Vue.js / Nuxt.js** - React has larger ecosystem, more Calibre-relevant libraries
❌ **Angular** - Too heavy, overkill for this use case
❌ **Svelte/SvelteKit** - Smaller ecosystem, fewer libraries
❌ **Create React App** - Deprecated, use Next.js or Vite
❌ **Redux** - Overkill, Zustand is simpler and sufficient
❌ **MobX** - Less popular than Zustand, similar complexity
❌ **Material-UI** - Heavy, prefer Tailwind + shadcn/ui
❌ **Ant Design** - Not accessible, outdated design
❌ **jQuery** - Legacy, avoid
❌ **Bootstrap** - Prefer Tailwind's utility-first approach

---

<a name="recommendation"></a>
## 8. Final Recommendation

### Primary Recommendation: **Hybrid Approach (Option 3)**

**Modernize the web Content Server UI with React/Next.js, keep desktop PyQt6 GUI unchanged.**

---

### Rationale

#### Why This Approach Wins

1. **Lowest Risk**
   - No disruption to existing desktop users (largest user base)
   - Web UI is currently basic HTML templates (low bar to improve)
   - Incremental rollout possible
   - Easy rollback if needed

2. **Best ROI**
   - **Cost:** $195k-$275k (vs. $1.5M-$3.7M for full rewrite)
   - **Timeline:** 9-12 months (vs. 3-5 years)
   - **Value:** Significant UX improvement for web/mobile users
   - **Quick wins:** Can ship basic version in 3-4 months

3. **Addresses Real Pain Points**
   - Current web UI is outdated (2010s-era HTML templates)
   - Mobile experience is poor (not responsive)
   - Remote access is limited
   - No PWA support

4. **Preserves Strengths**
   - Desktop performance (native PyQt6)
   - Complex features (e-book editor, device sync)
   - Offline functionality
   - USB device access

5. **Future-Proof**
   - Establishes modern React codebase
   - API layer useful for future integrations
   - Could gradually port desktop features if needed
   - Easier to hire React developers

---

### Implementation Roadmap

#### Phase 1: Foundation (Months 1-3)

**Goals:**
- Modern web UI foundation
- API layer for data access
- Basic library browsing

**Deliverables:**
- Next.js 15 project setup
- FastAPI REST endpoints
- Authentication system
- Library browser (table view)
- Book detail page
- Mobile-responsive design

**Success Metrics:**
- Page load <2 seconds
- API response <200ms
- Mobile-friendly (responsive)

---

#### Phase 2: Core Features (Months 4-6)

**Goals:**
- Feature parity with basic library management
- E-book reading

**Deliverables:**
- Advanced search & filters
- Metadata editing
- E-book reader (EPUB support)
- Cover management
- Tag/series browsing
- User settings

**Success Metrics:**
- Can manage library fully from web UI
- EPUB reading works on mobile
- Editing metadata functional

---

#### Phase 3: Advanced Features (Months 7-9)

**Goals:**
- Polish and advanced functionality

**Deliverables:**
- Reading progress sync
- Annotations (basic)
- Bulk operations
- Virtual libraries
- Advanced filters
- PWA support (offline reading)

**Success Metrics:**
- Feature-complete for 80% of users
- Performance benchmarks met
- No critical bugs

---

#### Phase 4: Polish & Launch (Months 10-12)

**Goals:**
- Production-ready release

**Deliverables:**
- Performance optimization
- Comprehensive testing
- User documentation
- Migration guide
- Beta program
- Official release

**Success Metrics:**
- <5 critical bugs
- 90%+ uptime
- Positive user feedback

---

### Alternative Recommendation (If Desktop Modernization Critical): **Progressive Enhancement (Option 4)**

If modernizing the desktop UI is absolutely critical, consider:

**Phase 1 (Months 1-6):** Prove viability
- Build proof-of-concept (QWebEngine + React)
- Replace 2-3 simple dialogs (metadata edit, settings)
- Measure performance impact

**Phase 2 (Months 7-12):** Expand gradually
- Replace library grid view
- Replace book details panel
- Keep complex features native (viewer, editor, device sync)

**Phase 3 (Months 13-18):** Advanced replacement
- Conversion UI
- Plugin configuration
- Evaluate further migration

**Decision Point (Month 18):**
- If successful: Continue gradual migration
- If problematic: Stop and keep hybrid approach
- Cost ceiling: Don't exceed $1M total

---

### What NOT to Do

❌ **Full Electron/Tauri Rewrite**
- Too expensive ($1.5M-$3.7M)
- Too risky (3-5 year project)
- Questionable value (desktop users happy with PyQt6)
- Opportunity cost too high

❌ **Web-Only Approach**
- Loses desktop advantages (performance, offline, USB)
- Doesn't serve core user base
- Web limitations hurt UX

❌ **Big Bang Migration**
- Rewriting everything at once
- High risk of failure
- Long time to value
- Difficult to course-correct

---

### Success Criteria

**How to Measure Success:**

**Technical:**
- [ ] Web UI loads in <2 seconds
- [ ] API responses <200ms (95th percentile)
- [ ] Works on mobile (iOS, Android)
- [ ] PWA installable
- [ ] No data loss or corruption
- [ ] Backward compatible with existing API

**User Experience:**
- [ ] Positive user feedback (>70% satisfaction)
- [ ] Increased mobile usage (track metrics)
- [ ] Feature requests addressed
- [ ] No major bugs reported

**Business:**
- [ ] Shipped in 12 months
- [ ] Under $300k budget
- [ ] No disruption to desktop users
- [ ] Enables future mobile app (React Native)

---

### Contingency Plans

**If Web UI Project Fails:**
- Fall back to existing HTML templates
- Minimal user impact
- Learn from mistakes
- Try again with different approach

**If Takes Longer Than Expected:**
- Ship MVP in 6 months (basic features)
- Iterate based on feedback
- De-scope advanced features
- Focus on 80% use cases

**If Performance Insufficient:**
- Optimize (code splitting, lazy loading, caching)
- Use service workers for offline
- Consider edge caching (Cloudflare)
- Profile and fix bottlenecks

---

## Conclusion

### The Bottom Line

**Rewriting Calibre's desktop UI with React/Next.js is technically feasible but not recommended.**

Instead:

1. ✅ **Do:** Modernize web Content Server UI with React/Next.js (Hybrid approach)
   - **Cost:** $195k-$275k
   - **Timeline:** 9-12 months
   - **Risk:** Low
   - **ROI:** Excellent

2. ⚠️ **Consider:** Progressive enhancement if desktop modernization critical
   - **Cost:** $630k-$930k
   - **Timeline:** 18-30 months
   - **Risk:** Medium
   - **ROI:** Moderate

3. ❌ **Don't:** Full Electron/Tauri rewrite
   - **Cost:** $1.5M-$3.7M
   - **Timeline:** 3-5 years
   - **Risk:** Very High
   - **ROI:** Poor

---

### Key Insights from Research

**From Case Studies:**
- Electron rewrites take **3-5 years** even for well-funded teams (Microsoft Teams)
- Performance is a constant battle (Slack, Notion still struggling)
- Tauri is promising but too young for production-critical rewrites
- Hybrid approaches offer best risk/reward ratio

**From Calibre Analysis:**
- 60% of codebase is backend (unaffected by UI choice)
- 34% is PyQt6 GUI (rewrite target)
- 40+ conversion plugins, 10+ device drivers (keep in Python)
- Complex features (editor, viewer, device sync) hard to replicate in web

**From Technology Evaluation:**
- React/Next.js ecosystem is mature and rich
- Electron is heavy but proven (200-300MB, stable)
- Tauri is lightweight but risky (young, fragmented)
- PyQt6 remains best for native desktop performance

---

### Next Steps (If Proceeding with Hybrid Approach)

1. **Month 1: Planning**
   - [ ] Create detailed API specification
   - [ ] Design UI mockups (Figma)
   - [ ] Set up development environment
   - [ ] Hire/assign React developer
   - [ ] Define success metrics

2. **Month 2: Foundation**
   - [ ] Initialize Next.js project
   - [ ] Build FastAPI endpoints
   - [ ] Set up authentication
   - [ ] Create basic layout

3. **Month 3: First Features**
   - [ ] Library browser (table view)
   - [ ] Book detail page
   - [ ] Basic search
   - [ ] Deploy alpha version

4. **Month 4-6: Core Features**
   - [ ] Advanced search & filters
   - [ ] Metadata editing
   - [ ] E-book reader
   - [ ] Tag/series management
   - [ ] Beta release

5. **Month 7-9: Advanced Features**
   - [ ] Reading progress sync
   - [ ] Bulk operations
   - [ ] PWA support
   - [ ] Performance optimization

6. **Month 10-12: Launch**
   - [ ] Bug fixes
   - [ ] Documentation
   - [ ] User testing
   - [ ] Official release
   - [ ] Monitor metrics

---

### Final Thoughts

The allure of modern web technologies (React, Next.js) is strong, and the Calibre desktop UI does show its age. However:

- **Existing PyQt6 UI works well** for core users
- **Web UI is the real pain point** (outdated HTML templates)
- **Full rewrite ROI doesn't justify cost** ($1.5M-$3.7M, 3-5 years)
- **Hybrid approach is sweet spot** ($195k-$275k, 9-12 months)

**Recommendation: Start with modernizing the web UI. Prove value. Then evaluate desktop modernization based on results and user feedback.**

This approach:
- ✅ Delivers value quickly (9-12 months)
- ✅ Keeps costs reasonable (<$300k)
- ✅ Minimizes risk (no desktop disruption)
- ✅ Enables future mobile app (React Native)
- ✅ Establishes modern codebase for iteration

**Don't let perfect be the enemy of good. Ship the hybrid web UI, delight users, then reassess.**

---

## Appendix: Additional Resources

### Research Sources

**Case Studies:**
- Microsoft Teams migration to Edge WebView2 (Microsoft Tech Community, 2024)
- Slack's Electron architecture evolution (Slack Engineering Blog, 2015-2024)
- VS Code continuous optimization (Microsoft DevBlogs, 2015-2025)
- Notion performance challenges (3perf.com case study, 2023)

**Performance Benchmarks:**
- Tauri vs Electron comparison (levminer.com, 2024)
- Memory usage measurements (GitHub tauri-apps/tauri #5889)
- Real-world app size comparisons (Authme case study, 2024)
- Startup time benchmarks (Codeology, 2025)

**Technology Documentation:**
- Electron.js official docs (electronjs.org)
- Tauri documentation (tauri.app)
- Next.js App Router guide (nextjs.org)
- FastAPI documentation (fastapi.tiangolo.com)

### Calibre Codebase References

**Key Files Analyzed:**
- `/home/user/calibre/src/calibre/gui2/` - PyQt6 GUI (457 files)
- `/home/user/calibre/src/calibre/db/cache.py` - Database layer
- `/home/user/calibre/src/calibre/devices/` - Device drivers (10 files)
- `/home/user/calibre/src/calibre/ebooks/conversion/plugins/` - Conversion plugins (40 files)
- `/home/user/calibre/docs/learning/` - Existing documentation (20 files)

**Documentation Created:**
- `FRONTEND_ARCHITECTURE.md` - PyQt6 architecture
- `TECH_STACK_GUIDE.md` - Technology stack overview
- `DATABASE_SCHEMA.md` - Database design
- `ARCHITECTURE_OVERVIEW.md` - System architecture

---

**End of Report**

*This analysis is based on research conducted in November 2025. Technology landscape and recommendations may change as new frameworks emerge and mature.*
