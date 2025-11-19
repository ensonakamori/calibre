# Complete Modern Architecture for Calibre Replacement

**Document Version:** 1.0
**Date:** November 19, 2025
**Author:** Claude (Sonnet 4.5)
**Status:** Comprehensive Technical Architecture

---

## Executive Summary

This document presents a complete technical architecture for building a modern Calibre replacement with feature parity, modern UI/UX, and excellent performance. The architecture is designed to handle 100k+ books, support 40+ e-book formats, 30+ device types, and maintain privacy-first, local-first principles.

### Key Architecture Decisions

| Component | Technology Choice | Justification |
|-----------|------------------|---------------|
| **Desktop Framework** | Tauri 2.0 | 10-30x smaller bundle, 60-90% less RAM, Rust performance & security |
| **Frontend** | React 19 + Next.js 15 | Mature ecosystem, server components, best developer experience |
| **State Management** | Zustand + TanStack Query | Lightweight, modern, excellent DX |
| **UI Library** | Tailwind CSS + shadcn/ui | Accessible, customizable, beautiful |
| **Backend Core** | Rust (Tauri) + Python (processing) | Hybrid: performance where needed, Python ecosystem for e-books |
| **Database** | better-sqlite3 (Node) / rusqlite (Rust) | Fast, embedded, proven with large databases |
| **E-book Processing** | Rust + Python bridge | Keep calibre's Python libraries, wrap in Rust for performance |
| **Plugin System** | WASM + JavaScript | Sandboxed, secure, cross-platform |

### Architecture Highlights

- **Hybrid Backend**: Rust for performance-critical operations, Python for e-book processing
- **Modern Frontend**: React 19 with Server Components, streaming, and suspense
- **Local-First**: All data stored locally, optional sync to cloud
- **Plugin System**: WebAssembly-based for security and performance
- **Cross-Platform**: Single codebase for Windows, macOS, Linux
- **Performance**: Virtual scrolling, lazy loading, worker threads, streaming
- **Scalable**: Designed for 100k+ books from day one

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Technology Stack Detailed Justification](#2-technology-stack-detailed-justification)
3. [Frontend Architecture](#3-frontend-architecture)
4. [Backend Architecture](#4-backend-architecture)
5. [Database Architecture](#5-database-architecture)
6. [E-book Processing Pipeline](#6-ebook-processing-pipeline)
7. [Device Sync Architecture](#7-device-sync-architecture)
8. [Plugin System Architecture](#8-plugin-system-architecture)
9. [Content Server Architecture](#9-content-server-architecture)
10. [Data Flow Diagrams](#10-data-flow-diagrams)
11. [API Design Patterns](#11-api-design-patterns)
12. [Performance Architecture](#12-performance-architecture)
13. [Security Architecture](#13-security-architecture)
14. [Module Breakdown](#14-module-breakdown)
15. [Deployment & Distribution](#15-deployment--distribution)
16. [Development Workflow](#16-development-workflow)
17. [Testing Strategy](#17-testing-strategy)
18. [Migration Strategy](#18-migration-strategy)

---

## 1. System Architecture Overview

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CALIBRE MODERN (Desktop App)                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                    PRESENTATION LAYER                       │   │
│  │                   (React 19 + Next.js 15)                   │   │
│  ├────────────────────────────────────────────────────────────┤   │
│  │  • Library Browser (Virtual Table)                          │   │
│  │  • Book Details Panel                                       │   │
│  │  • E-book Viewer (Epub.js, PDF.js)                         │   │
│  │  • E-book Editor (Monaco + EPUB tools)                     │   │
│  │  • Metadata Editor (Forms, Bulk Edit)                      │   │
│  │  • Device Sync UI                                          │   │
│  │  • Conversion Wizard                                       │   │
│  │  • Settings & Preferences                                  │   │
│  │  • Plugin Manager                                          │   │
│  └────────────────────────────────────────────────────────────┘   │
│                              ↕ (Tauri IPC)                          │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                   APPLICATION LAYER (Rust)                  │   │
│  ├────────────────────────────────────────────────────────────┤   │
│  │  Window Management  │  Menu System    │  Tray Integration  │   │
│  │  File System Access │  USB Detection  │  HTTP Server       │   │
│  │  IPC Router         │  State Manager  │  Background Jobs   │   │
│  └────────────────────────────────────────────────────────────┘   │
│                              ↕                                      │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                    BUSINESS LOGIC LAYER                     │   │
│  ├────────────────────────────────────────────────────────────┤   │
│  │                                                              │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │   │
│  │  │   Database   │  │   E-book     │  │   Device     │     │   │
│  │  │   Service    │  │  Processing  │  │    Sync      │     │   │
│  │  │   (Rust)     │  │ (Rust+Python)│  │   (Rust)     │     │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘     │   │
│  │                                                              │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │   │
│  │  │   Metadata   │  │  Conversion  │  │   Plugin     │     │   │
│  │  │   Sources    │  │   Engine     │  │   Runtime    │     │   │
│  │  │   (Rust)     │  │   (Python)   │  │   (WASM)     │     │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘     │   │
│  │                                                              │   │
│  └────────────────────────────────────────────────────────────┘   │
│                              ↕                                      │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                      DATA LAYER                             │   │
│  ├────────────────────────────────────────────────────────────┤   │
│  │  • SQLite Database (books, metadata, cache)                │   │
│  │  • File System (book files, covers, configs)               │   │
│  │  • FTS5 Full-Text Search Index                             │   │
│  │  • In-Memory Cache (LRU, frequently accessed data)         │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      EXTERNAL INTERFACES                             │
├─────────────────────────────────────────────────────────────────────┤
│  USB Devices  │  Network APIs  │  File System  │  System Services   │
│  (Kindle,     │  (Amazon,      │  (Import,     │  (Notifications,   │
│   Kobo, etc)  │   Goodreads)   │   Export)     │   Tray, Dialogs)   │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Key Architectural Principles

1. **Separation of Concerns**: Clear boundaries between presentation, logic, and data
2. **Performance First**: Virtual scrolling, lazy loading, worker threads, caching
3. **Local-First**: All data local, optional cloud sync
4. **Security by Default**: Sandboxed plugins, input validation, CSP
5. **Extensibility**: Plugin system for custom functionality
6. **Cross-Platform**: Single codebase, OS-specific optimizations where needed
7. **Backward Compatibility**: Import existing Calibre libraries

---

## 2. Technology Stack Detailed Justification

### 2.1 Desktop Framework: Tauri 2.0

**Choice: Tauri 2.0** (over Electron)

**Justification:**
- **Bundle Size**: 3-10MB vs 100-250MB (Electron)
- **Memory Usage**: 30-40MB idle vs 200-300MB (Electron)
- **Startup Time**: <500ms vs 1-2 seconds (Electron)
- **Performance**: Rust backend for CPU-intensive operations
- **Security**: Rust memory safety, sandboxed WebView
- **Native Integration**: Better OS integration (notifications, tray, dialogs)
- **Maturity**: Tauri 2.0 (released late 2024) is production-ready

**Trade-offs:**
- ⚠️ Smaller ecosystem than Electron
- ⚠️ WebView fragmentation (different rendering on each OS)
- ⚠️ Rust learning curve for contributors

**Mitigation:**
- Keep complex WebView features minimal
- Extensive cross-platform testing
- Provide Rust contribution guidelines

**Alternative Considered: Electron**
- Rejected due to bundle size and memory overhead
- Would work but doesn't align with "modern" and "performant" goals

---

### 2.2 Frontend Framework: React 19 + Next.js 15

**Choice: React 19 + Next.js 15 (App Router)**

**Justification:**
- **React 19**: Latest with Server Components, Actions, use hook
- **Next.js 15**: App Router, Turbopack, streaming, server components
- **Ecosystem**: Largest component library ecosystem
- **Developer Experience**: Best tooling, debugging, documentation
- **Performance**: Server Components reduce JS bundle size
- **Streaming**: Load data progressively for large libraries
- **TypeScript**: Full type safety from DB to UI

**Why Not Pure React?**
- Next.js provides routing, SSR, and tooling out of the box
- Can use in "static export" mode if needed
- Turbopack is 10x faster than Webpack

**Alternatives Considered:**
- Vue 3 + Nuxt: Smaller ecosystem, fewer Calibre-specific libraries
- Svelte/SvelteKit: Immature ecosystem, fewer table/grid libraries
- Angular: Too heavy, overkill for this use case

---

### 2.3 State Management: Zustand + TanStack Query

**Choice: Zustand (client state) + TanStack Query (server state)**

**Justification:**

**Zustand:**
- Lightweight (1KB), simple API
- No boilerplate (unlike Redux)
- TypeScript-first
- DevTools support
- Perfect for UI state (sidebar open, selected books, filters)

**TanStack Query:**
- Best-in-class data fetching and caching
- Automatic background refetching
- Optimistic updates
- Infinite scrolling support (for large libraries)
- DevTools for debugging
- Works perfectly with Tauri IPC

**Alternatives Considered:**
- Redux Toolkit: Too heavy, too much boilerplate
- Jotai/Recoil: Atom-based is overkill for this use case
- MobX: Less popular, similar complexity to Zustand

---

### 2.4 UI Library: Tailwind CSS + shadcn/ui

**Choice: Tailwind CSS + shadcn/ui**

**Justification:**

**Tailwind CSS:**
- Utility-first, highly customizable
- Small production bundle (unused classes purged)
- Consistent design system
- Fast development
- Excellent documentation

**shadcn/ui:**
- Not a component library, copy-paste components
- Built on Radix UI (accessible, unstyled)
- Full control over components
- Beautiful default styling
- Customizable with Tailwind
- TypeScript-first

**Alternatives Considered:**
- Material-UI (MUI): Heavy, opinionated, harder to customize
- Ant Design: Outdated design, accessibility issues
- Chakra UI: Good but larger bundle size
- Custom CSS: Too much work, reinventing the wheel

---

### 2.5 Backend Core: Rust (Tauri) + Python Bridge

**Choice: Hybrid Rust + Python**

**Justification:**

**Rust (Tauri Core):**
- Window management, IPC, file system
- Database queries (rusqlite)
- Device detection (USB, MTP)
- HTTP server for content server
- Performance-critical operations

**Python (E-book Processing):**
- Reuse existing Calibre libraries (lxml, PyMuPDF, etc.)
- EPUB parsing, PDF processing, MOBI handling
- Format conversion (40+ formats)
- Metadata extraction
- Web scraping for metadata sources

**Python Bridge:**
- Rust spawns Python as sidecar process
- Communication via stdin/stdout (JSON-RPC)
- Python keeps running for performance
- Rust handles all UI interactions
- Python handles all e-book processing

**Why Not Full Rust?**
- Python e-book libraries are mature and battle-tested
- Rewriting 40+ format converters in Rust would take years
- Python ecosystem for web scraping, APIs, etc.

**Why Not Full Python?**
- Rust provides better performance for UI operations
- Tauri requires Rust
- Rust better for system-level operations

---

### 2.6 Database: rusqlite (Rust) / better-sqlite3 (Node)

**Choice: rusqlite (primary) + better-sqlite3 (for migrations)**

**Justification:**
- **SQLite**: Embedded, no server, single file, proven with 100k+ records
- **rusqlite**: Fast, safe, integrates with Rust
- **better-sqlite3**: For migration tools from existing Calibre DB
- **FTS5**: Full-text search for book content
- **JSON1**: Store complex metadata (identifiers, custom columns)

**Schema:**
- Keep similar to Calibre for easy migration
- Add indexes for performance
- Use JSONB for flexible metadata
- Separate tables for cache

**Alternatives Considered:**
- PostgreSQL: Overkill, requires server
- DuckDB: Interesting for analytics but less mature
- Custom format: Too much work, SQLite is battle-tested

---

### 2.7 E-book Rendering: Epub.js + PDF.js + React

**Choice: Epub.js (EPUB) + PDF.js (PDF) + Custom (others)**

**Justification:**

**Epub.js:**
- Mature EPUB renderer
- Annotations, highlights, bookmarks
- Custom themes, fonts
- Pagination support
- TypeScript support

**PDF.js:**
- Mozilla-maintained, battle-tested
- Canvas + WebGL rendering
- Text selection, search
- React wrapper (react-pdf)

**Custom Renderers:**
- Plain text: Monaco Editor (syntax highlighting)
- HTML: Sandboxed iframe
- Images (CBZ, CBR): React image gallery with virtualization

**Alternatives Considered:**
- Readium: More complex, overkill
- Custom EPUB renderer: Too much work
- Native renderers: Loses cross-platform benefits

---

### 2.8 Plugin System: WebAssembly (WASM) + JavaScript

**Choice: Hybrid WASM + JavaScript plugins**

**Justification:**

**WASM Plugins:**
- Performance (C, Rust, Go compiled to WASM)
- Sandboxed (can't access file system without permission)
- Language-agnostic (compile from many languages)
- Future-proof

**JavaScript Plugins:**
- Easy to write (lower barrier to entry)
- Access to npm ecosystem
- Sandboxed in separate context
- Can call into WASM for performance

**Plugin API:**
- Defined TypeScript interfaces
- Capability-based security (plugins request permissions)
- IPC-based communication with main app
- Hot reload for development

**Alternatives Considered:**
- Python plugins: Security risk, hard to sandbox
- Native plugins: Platform-specific, security risk
- JavaScript only: Performance limitations

---

## 3. Frontend Architecture

### 3.1 Application Structure

```
src/
├── app/                          # Next.js App Router
│   ├── layout.tsx                # Root layout (providers, theme)
│   ├── page.tsx                  # Main library view
│   ├── book/[id]/page.tsx        # Book details
│   ├── read/[id]/page.tsx        # E-book reader
│   ├── edit/[id]/page.tsx        # E-book editor
│   ├── settings/page.tsx         # Settings
│   └── plugins/page.tsx          # Plugin manager
├── components/                   # React components
│   ├── library/                  # Library browser components
│   │   ├── BookTable.tsx         # Virtual table (TanStack Table)
│   │   ├── BookGrid.tsx          # Cover grid
│   │   ├── BookDetails.tsx       # Details panel
│   │   ├── TagBrowser.tsx        # Tag tree
│   │   └── SearchBar.tsx         # Search + filters
│   ├── reader/                   # E-book reader components
│   │   ├── EpubReader.tsx        # Epub.js wrapper
│   │   ├── PdfReader.tsx         # PDF.js wrapper
│   │   ├── ReaderControls.tsx    # Navigation, settings
│   │   └── Annotations.tsx       # Highlights, notes
│   ├── editor/                   # E-book editor components
│   │   ├── MonacoEditor.tsx      # Code editor
│   │   ├── EpubStructure.tsx     # File tree
│   │   ├── Preview.tsx           # Live preview
│   │   └── StyleEditor.tsx       # CSS editor
│   ├── metadata/                 # Metadata editing
│   │   ├── MetadataForm.tsx      # Single book edit
│   │   ├── BulkEdit.tsx          # Multi-book edit
│   │   ├── CoverManager.tsx      # Cover upload/download
│   │   └── IdentifierManager.tsx # ISBN, ASIN, etc.
│   ├── conversion/               # Format conversion
│   │   ├── ConversionWizard.tsx  # Multi-step wizard
│   │   ├── FormatSelector.tsx    # Input/output formats
│   │   ├── OptionsPanel.tsx      # Conversion options
│   │   └── ProgressTracker.tsx   # Real-time progress
│   ├── devices/                  # Device sync
│   │   ├── DeviceList.tsx        # Connected devices
│   │   ├── DeviceContent.tsx     # Device library view
│   │   ├── SyncManager.tsx       # Sync UI
│   │   └── ConflictResolver.tsx  # Handle conflicts
│   ├── plugins/                  # Plugin system
│   │   ├── PluginList.tsx        # Installed plugins
│   │   ├── PluginConfig.tsx      # Plugin settings
│   │   └── PluginStore.tsx       # Browse/install plugins
│   └── ui/                       # Reusable UI components (shadcn/ui)
│       ├── button.tsx
│       ├── dialog.tsx
│       ├── table.tsx
│       ├── select.tsx
│       └── ...
├── lib/                          # Utilities and helpers
│   ├── tauri.ts                  # Tauri IPC wrappers
│   ├── database.ts               # Database queries (via Tauri)
│   ├── formats.ts                # E-book format utilities
│   ├── utils.ts                  # General utilities
│   └── constants.ts              # App constants
├── hooks/                        # React hooks
│   ├── useBooks.ts               # Fetch/manage books
│   ├── useSearch.ts              # Search functionality
│   ├── useDevices.ts             # Device detection
│   ├── usePlugins.ts             # Plugin management
│   └── useSettings.ts            # App settings
├── stores/                       # Zustand stores
│   ├── library.ts                # Library state (selection, filters)
│   ├── reader.ts                 # Reader state (position, settings)
│   ├── ui.ts                     # UI state (sidebar, theme)
│   └── sync.ts                   # Sync state
└── types/                        # TypeScript types
    ├── book.ts                   # Book, Metadata types
    ├── device.ts                 # Device types
    ├── plugin.ts                 # Plugin types
    └── api.ts                    # API types
```

### 3.2 Key Frontend Components

#### 3.2.1 Library Browser (BookTable)

```typescript
// components/library/BookTable.tsx
import { useVirtualizer } from '@tanstack/react-virtual'
import { useReactTable } from '@tanstack/react-table'
import { useBooks } from '@/hooks/useBooks'

export function BookTable() {
  // Fetch books with TanStack Query (handles caching, refetching)
  const { data, isLoading } = useBooks({
    sort: 'title',
    filter: 'all',
    limit: 1000, // Fetch in chunks
  })

  // Virtual table for performance with 100k+ books
  const table = useReactTable({
    data: data?.books ?? [],
    columns,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
  })

  // Virtual scrolling for rendering only visible rows
  const virtualizer = useVirtualizer({
    count: table.getRowModel().rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 35, // Row height
    overscan: 10, // Render 10 rows above/below viewport
  })

  return (
    <div ref={parentRef} className="h-full overflow-auto">
      <table>
        <thead>
          {table.getHeaderGroups().map(headerGroup => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map(header => (
                <th key={header.id}>{/* Header content */}</th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {virtualizer.getVirtualItems().map(virtualRow => {
            const row = table.getRowModel().rows[virtualRow.index]
            return <tr key={row.id}>{/* Row content */}</tr>
          })}
        </tbody>
      </table>
    </div>
  )
}
```

**Features:**
- Virtual scrolling (renders only visible rows)
- Sortable columns
- Resizable columns
- Multi-select with Shift/Ctrl
- Drag & drop (reorder, bulk operations)
- Context menu (right-click actions)
- Keyboard navigation
- Customizable columns

**Performance:**
- 60 FPS scrolling with 100k books
- Lazy loading (fetch more as you scroll)
- Debounced search/filter
- Memoized row rendering

---

#### 3.2.2 E-book Reader (EpubReader)

```typescript
// components/reader/EpubReader.tsx
import ePub from 'epubjs'
import { useEffect, useRef, useState } from 'react'
import { invoke } from '@tauri-apps/api/core'

export function EpubReader({ bookId }: { bookId: string }) {
  const viewerRef = useRef<HTMLDivElement>(null)
  const bookRef = useRef<any>(null)

  useEffect(() => {
    // Get book file path from Rust backend
    invoke<string>('get_book_path', { bookId }).then(async (path) => {
      // Load EPUB
      const book = ePub(path)
      bookRef.current = book

      // Render in viewer
      const rendition = book.renderTo(viewerRef.current!, {
        width: '100%',
        height: '100%',
        spread: 'auto',
      })

      // Load saved position
      const position = await invoke<string>('get_reading_position', { bookId })
      rendition.display(position || undefined)

      // Save position on change
      rendition.on('relocated', (location) => {
        invoke('save_reading_position', {
          bookId,
          position: location.start.cfi,
        })
      })
    })

    return () => bookRef.current?.destroy()
  }, [bookId])

  return (
    <div className="relative h-full">
      <div ref={viewerRef} className="h-full" />
      <ReaderControls book={bookRef.current} />
      <Annotations book={bookRef.current} bookId={bookId} />
    </div>
  )
}
```

**Features:**
- Page turning (keyboard, swipe, click)
- Font customization (family, size, line height)
- Theme (light, dark, sepia)
- Bookmarks
- Highlights & notes
- Search within book
- Table of contents
- Reading progress
- Full-screen mode

**Performance:**
- Lazy rendering (only current chapter)
- Cached fonts
- WebGL acceleration for PDF
- Web Worker for search

---

#### 3.2.3 Metadata Editor (MetadataForm)

```typescript
// components/metadata/MetadataForm.tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { invoke } from '@tauri-apps/api/core'
import { useMutation, useQueryClient } from '@tanstack/react-query'

const metadataSchema = z.object({
  title: z.string().min(1, 'Title is required'),
  authors: z.array(z.string()),
  publisher: z.string().optional(),
  pubdate: z.string().optional(),
  isbn: z.string().optional(),
  tags: z.array(z.string()),
  series: z.string().optional(),
  series_index: z.number().optional(),
  rating: z.number().min(0).max(5).optional(),
  comments: z.string().optional(),
})

export function MetadataForm({ bookId }: { bookId: string }) {
  const queryClient = useQueryClient()

  const form = useForm({
    resolver: zodResolver(metadataSchema),
  })

  const mutation = useMutation({
    mutationFn: (data: z.infer<typeof metadataSchema>) =>
      invoke('update_metadata', { bookId, metadata: data }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['book', bookId] })
    },
  })

  return (
    <form onSubmit={form.handleSubmit((data) => mutation.mutate(data))}>
      {/* Form fields */}
      <Button type="submit" disabled={mutation.isPending}>
        Save
      </Button>
    </form>
  )
}
```

**Features:**
- Autocomplete for authors, publishers, tags
- Date picker for publication date
- ISBN validation
- Cover upload/download
- Batch editing (multiple books)
- Metadata download from online sources
- Custom columns support
- Undo/redo

---

### 3.3 State Management Architecture

#### 3.3.1 Client State (Zustand)

```typescript
// stores/library.ts
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

interface LibraryState {
  selectedBooks: string[]
  view: 'table' | 'grid' | 'covers'
  sortBy: string
  sortOrder: 'asc' | 'desc'
  filters: {
    search: string
    tags: string[]
    series: string
    authors: string[]
  }

  setSelectedBooks: (ids: string[]) => void
  setView: (view: 'table' | 'grid' | 'covers') => void
  setSortBy: (sortBy: string) => void
  setFilters: (filters: Partial<LibraryState['filters']>) => void
  clearFilters: () => void
}

export const useLibraryStore = create<LibraryState>()(
  devtools((set) => ({
    selectedBooks: [],
    view: 'table',
    sortBy: 'title',
    sortOrder: 'asc',
    filters: {
      search: '',
      tags: [],
      series: '',
      authors: [],
    },

    setSelectedBooks: (ids) => set({ selectedBooks: ids }),
    setView: (view) => set({ view }),
    setSortBy: (sortBy) => set({ sortBy }),
    setFilters: (filters) =>
      set((state) => ({
        filters: { ...state.filters, ...filters }
      })),
    clearFilters: () =>
      set({
        filters: { search: '', tags: [], series: '', authors: [] }
      }),
  }))
)
```

#### 3.3.2 Server State (TanStack Query)

```typescript
// hooks/useBooks.ts
import { useQuery } from '@tanstack/react-query'
import { invoke } from '@tauri-apps/api/core'

export function useBooks(options: {
  sort?: string
  filter?: string
  limit?: number
  offset?: number
}) {
  return useQuery({
    queryKey: ['books', options],
    queryFn: () => invoke<Book[]>('get_books', options),
    staleTime: 5 * 60 * 1000, // 5 minutes
    gcTime: 10 * 60 * 1000, // 10 minutes
    refetchOnWindowFocus: false,
  })
}

export function useBook(id: string) {
  return useQuery({
    queryKey: ['book', id],
    queryFn: () => invoke<Book>('get_book', { id }),
  })
}

export function useUpdateBook() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (data: { id: string; updates: Partial<Book> }) =>
      invoke('update_book', data),
    onSuccess: (_, variables) => {
      queryClient.invalidateQueries({ queryKey: ['book', variables.id] })
      queryClient.invalidateQueries({ queryKey: ['books'] })
    },
  })
}
```

**Benefits:**
- Automatic caching
- Background refetching
- Optimistic updates
- Request deduplication
- Automatic retries
- DevTools for debugging

---

## 4. Backend Architecture

### 4.1 Tauri Core (Rust)

```
src-tauri/
├── src/
│   ├── main.rs                   # Entry point
│   ├── commands/                 # Tauri commands (IPC handlers)
│   │   ├── mod.rs
│   │   ├── books.rs              # Book queries
│   │   ├── metadata.rs           # Metadata operations
│   │   ├── devices.rs            # USB device detection
│   │   ├── conversion.rs         # Conversion queue
│   │   ├── plugins.rs            # Plugin management
│   │   └── settings.rs           # App settings
│   ├── database/                 # Database layer
│   │   ├── mod.rs
│   │   ├── schema.rs             # SQL schema
│   │   ├── queries.rs            # Query builders
│   │   ├── migrations.rs         # Schema migrations
│   │   └── cache.rs              # In-memory cache
│   ├── devices/                  # Device sync
│   │   ├── mod.rs
│   │   ├── usb.rs                # USB detection (rusb)
│   │   ├── mtp.rs                # MTP protocol
│   │   ├── kindle.rs             # Kindle-specific
│   │   ├── kobo.rs               # Kobo-specific
│   │   └── sync.rs               # Sync engine
│   ├── ebook/                    # E-book processing
│   │   ├── mod.rs
│   │   ├── epub.rs               # EPUB utilities
│   │   ├── pdf.rs                # PDF utilities
│   │   ├── covers.rs             # Cover extraction
│   │   └── python_bridge.rs     # Bridge to Python
│   ├── metadata/                 # Metadata sources
│   │   ├── mod.rs
│   │   ├── amazon.rs             # Amazon API
│   │   ├── google.rs             # Google Books API
│   │   ├── goodreads.rs          # Goodreads API
│   │   └── openlibrary.rs        # Open Library API
│   ├── plugins/                  # Plugin system
│   │   ├── mod.rs
│   │   ├── runtime.rs            # WASM runtime
│   │   ├── api.rs                # Plugin API
│   │   └── sandbox.rs            # Security sandbox
│   ├── server/                   # Content server
│   │   ├── mod.rs
│   │   ├── http.rs               # HTTP server (axum)
│   │   ├── opds.rs               # OPDS feed
│   │   └── websocket.rs          # WebSocket for sync
│   └── utils/                    # Utilities
│       ├── mod.rs
│       ├── crypto.rs             # Hashing, encryption
│       ├── file.rs               # File operations
│       └── logger.rs             # Logging
├── Cargo.toml                    # Rust dependencies
└── tauri.conf.json               # Tauri configuration
```

### 4.2 Key Rust Modules

#### 4.2.1 Database Module (database/queries.rs)

```rust
// src-tauri/src/database/queries.rs
use rusqlite::{Connection, Result};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct Book {
    pub id: i64,
    pub title: String,
    pub authors: Vec<String>,
    pub publisher: Option<String>,
    pub pubdate: Option<String>,
    pub isbn: Option<String>,
    pub tags: Vec<String>,
    pub series: Option<String>,
    pub series_index: Option<f64>,
    pub rating: Option<i32>,
    pub comments: Option<String>,
    pub path: String,
    pub format: String,
    pub file_size: i64,
    pub cover_path: Option<String>,
}

pub struct Database {
    conn: Connection,
}

impl Database {
    pub fn new(path: &str) -> Result<Self> {
        let conn = Connection::open(path)?;
        Ok(Self { conn })
    }

    pub fn get_books(
        &self,
        sort_by: &str,
        sort_order: &str,
        limit: usize,
        offset: usize,
    ) -> Result<Vec<Book>> {
        let query = format!(
            "SELECT * FROM books ORDER BY {} {} LIMIT ? OFFSET ?",
            sort_by, sort_order
        );

        let mut stmt = self.conn.prepare(&query)?;
        let books = stmt.query_map([limit, offset], |row| {
            Ok(Book {
                id: row.get(0)?,
                title: row.get(1)?,
                // ... map all fields
            })
        })?;

        books.collect()
    }

    pub fn search_books(&self, query: &str) -> Result<Vec<Book>> {
        // Use FTS5 for full-text search
        let sql = "
            SELECT b.* FROM books b
            JOIN books_fts fts ON b.id = fts.rowid
            WHERE books_fts MATCH ?
            ORDER BY rank
        ";

        let mut stmt = self.conn.prepare(sql)?;
        let books = stmt.query_map([query], |row| {
            // Map row to Book
        })?;

        books.collect()
    }

    pub fn update_book(&self, id: i64, updates: &Book) -> Result<()> {
        // Update book metadata
        self.conn.execute(
            "UPDATE books SET title = ?, authors = ?, ... WHERE id = ?",
            // params
        )?;
        Ok(())
    }
}
```

#### 4.2.2 Device Detection (devices/usb.rs)

```rust
// src-tauri/src/devices/usb.rs
use rusb::{Context, Device, DeviceDescriptor};
use std::time::Duration;

#[derive(Debug, Clone, serde::Serialize)]
pub struct DetectedDevice {
    pub vendor_id: u16,
    pub product_id: u16,
    pub manufacturer: String,
    pub product: String,
    pub serial: String,
    pub device_type: DeviceType,
}

#[derive(Debug, Clone, serde::Serialize)]
pub enum DeviceType {
    Kindle,
    Kobo,
    Nook,
    Android,
    Generic,
}

pub fn detect_devices() -> Result<Vec<DetectedDevice>, rusb::Error> {
    let context = Context::new()?;
    let devices = context.devices()?;

    let mut detected = Vec::new();

    for device in devices.iter() {
        let descriptor = device.device_descriptor()?;

        // Check if it's a known e-reader
        if let Some(device_type) = identify_device(&descriptor) {
            let handle = device.open()?;
            let timeout = Duration::from_secs(1);

            let manufacturer = handle
                .read_manufacturer_string_ascii(&descriptor, timeout)
                .unwrap_or_default();

            let product = handle
                .read_product_string_ascii(&descriptor, timeout)
                .unwrap_or_default();

            let serial = handle
                .read_serial_number_string_ascii(&descriptor, timeout)
                .unwrap_or_default();

            detected.push(DetectedDevice {
                vendor_id: descriptor.vendor_id(),
                product_id: descriptor.product_id(),
                manufacturer,
                product,
                serial,
                device_type,
            });
        }
    }

    Ok(detected)
}

fn identify_device(desc: &DeviceDescriptor) -> Option<DeviceType> {
    // Known vendor/product IDs for e-readers
    match (desc.vendor_id(), desc.product_id()) {
        (0x1949, _) => Some(DeviceType::Kindle),  // Amazon
        (0x2237, _) => Some(DeviceType::Kobo),    // Kobo
        (0x2080, _) => Some(DeviceType::Nook),    // Barnes & Noble
        // Add more device IDs
        _ => None,
    }
}
```

#### 4.2.3 Python Bridge (ebook/python_bridge.rs)

```rust
// src-tauri/src/ebook/python_bridge.rs
use serde::{Deserialize, Serialize};
use std::process::{Command, Stdio};
use std::io::{BufRead, BufReader, Write};

pub struct PythonBridge {
    process: std::process::Child,
}

#[derive(Serialize, Deserialize)]
struct JsonRpcRequest {
    id: u64,
    method: String,
    params: serde_json::Value,
}

#[derive(Serialize, Deserialize)]
struct JsonRpcResponse {
    id: u64,
    result: Option<serde_json::Value>,
    error: Option<String>,
}

impl PythonBridge {
    pub fn new(python_path: &str) -> Result<Self, std::io::Error> {
        let process = Command::new(python_path)
            .arg("-u") // Unbuffered
            .arg("ebook_processor.py")
            .stdin(Stdio::piped())
            .stdout(Stdio::piped())
            .stderr(Stdio::piped())
            .spawn()?;

        Ok(Self { process })
    }

    pub fn convert_epub_to_pdf(
        &mut self,
        input: &str,
        output: &str,
    ) -> Result<(), String> {
        let request = JsonRpcRequest {
            id: 1,
            method: "convert".to_string(),
            params: serde_json::json!({
                "input": input,
                "output": output,
                "input_format": "epub",
                "output_format": "pdf",
            }),
        };

        // Send request to Python
        let mut stdin = self.process.stdin.as_mut().unwrap();
        let json = serde_json::to_string(&request).unwrap();
        writeln!(stdin, "{}", json)?;

        // Read response
        let mut stdout = BufReader::new(self.process.stdout.as_mut().unwrap());
        let mut line = String::new();
        stdout.read_line(&mut line)?;

        let response: JsonRpcResponse = serde_json::from_str(&line)
            .map_err(|e| e.to_string())?;

        if let Some(error) = response.error {
            return Err(error);
        }

        Ok(())
    }

    pub fn extract_metadata(&mut self, path: &str) -> Result<serde_json::Value, String> {
        // Similar JSON-RPC call
    }
}
```

**Python Side (ebook_processor.py):**

```python
# ebook_processor.py
import sys
import json
from calibre.ebooks.conversion.cli import main as convert_main
from calibre.ebooks.metadata.meta import get_metadata

def handle_request(request):
    method = request['method']
    params = request['params']

    try:
        if method == 'convert':
            # Use Calibre's conversion engine
            convert_main([
                params['input'],
                params['output'],
                # ... conversion options
            ])
            return {'result': True}

        elif method == 'extract_metadata':
            metadata = get_metadata(open(params['path'], 'rb'), params['format'])
            return {'result': metadata_to_dict(metadata)}

        else:
            return {'error': f'Unknown method: {method}'}

    except Exception as e:
        return {'error': str(e)}

if __name__ == '__main__':
    # JSON-RPC server
    for line in sys.stdin:
        request = json.loads(line)
        response = handle_request(request)
        response['id'] = request['id']
        print(json.dumps(response), flush=True)
```

**Benefits:**
- Reuse all Calibre's Python libraries
- Rust handles UI and system operations
- Python handles e-book processing
- Clean separation of concerns

---

## 5. Database Architecture

### 5.1 Schema Design

```sql
-- books.sql

CREATE TABLE books (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    sort TEXT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    pubdate DATETIME,
    series_index REAL DEFAULT 1.0,
    author_sort TEXT,
    isbn TEXT,
    lccn TEXT,
    path TEXT NOT NULL,
    flags INTEGER DEFAULT 1,
    uuid TEXT UNIQUE,
    has_cover BOOLEAN DEFAULT 0,
    last_modified DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE authors (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    sort TEXT,
    link TEXT
);

CREATE TABLE books_authors_link (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,
    author INTEGER NOT NULL,
    UNIQUE(book, author),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(author) REFERENCES authors(id) ON DELETE CASCADE
);

CREATE TABLE publishers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    sort TEXT
);

CREATE TABLE books_publishers_link (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,
    publisher INTEGER NOT NULL,
    UNIQUE(book, publisher),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(publisher) REFERENCES publishers(id) ON DELETE CASCADE
);

CREATE TABLE tags (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE books_tags_link (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,
    tag INTEGER NOT NULL,
    UNIQUE(book, tag),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(tag) REFERENCES tags(id) ON DELETE CASCADE
);

CREATE TABLE series (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    sort TEXT
);

CREATE TABLE books_series_link (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,
    series INTEGER NOT NULL,
    UNIQUE(book, series),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(series) REFERENCES series(id) ON DELETE CASCADE
);

CREATE TABLE comments (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,
    text TEXT NOT NULL,
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE
);

CREATE TABLE data (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,
    format TEXT NOT NULL,
    uncompressed_size INTEGER NOT NULL,
    name TEXT NOT NULL,
    UNIQUE(book, format),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE
);

CREATE TABLE identifiers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,
    type TEXT NOT NULL DEFAULT 'isbn',
    val TEXT NOT NULL,
    UNIQUE(book, type),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE
);

CREATE TABLE languages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    lang_code TEXT NOT NULL UNIQUE
);

CREATE TABLE books_languages_link (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,
    lang_code INTEGER NOT NULL,
    item_order INTEGER NOT NULL DEFAULT 0,
    UNIQUE(book, lang_code),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(lang_code) REFERENCES languages(id) ON DELETE CASCADE
);

CREATE TABLE ratings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    rating INTEGER CHECK(rating >= 0 AND rating <= 10)
);

CREATE TABLE books_ratings_link (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,
    rating INTEGER NOT NULL,
    UNIQUE(book, rating),
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE,
    FOREIGN KEY(rating) REFERENCES ratings(id) ON DELETE CASCADE
);

-- Custom columns (user-defined metadata)
CREATE TABLE custom_columns (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    label TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    datatype TEXT NOT NULL,
    mark_for_delete BOOLEAN DEFAULT 0,
    editable BOOLEAN DEFAULT 1,
    display TEXT DEFAULT '{}',
    is_multiple BOOLEAN DEFAULT 0,
    normalized BOOLEAN DEFAULT 0
);

-- Full-text search (FTS5)
CREATE VIRTUAL TABLE books_fts USING fts5(
    title,
    authors,
    tags,
    series,
    publisher,
    comments,
    content=books,
    content_rowid=id
);

-- Triggers to keep FTS in sync
CREATE TRIGGER books_ai AFTER INSERT ON books BEGIN
    INSERT INTO books_fts(rowid, title, authors, tags, series, publisher, comments)
    VALUES (new.id, new.title, '', '', '', '', '');
END;

CREATE TRIGGER books_ad AFTER DELETE ON books BEGIN
    DELETE FROM books_fts WHERE rowid = old.id;
END;

CREATE TRIGGER books_au AFTER UPDATE ON books BEGIN
    UPDATE books_fts SET title = new.title WHERE rowid = new.id;
END;

-- Reading progress and annotations
CREATE TABLE reading_progress (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,
    position TEXT NOT NULL,  -- CFI for EPUB, page for PDF
    progress REAL NOT NULL,  -- 0.0 to 1.0
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE
);

CREATE TABLE annotations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,
    type TEXT NOT NULL CHECK(type IN ('highlight', 'note', 'bookmark')),
    position TEXT NOT NULL,
    text TEXT,
    note TEXT,
    color TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE
);

-- Device sync tracking
CREATE TABLE device_sync (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    device_id TEXT NOT NULL,
    book INTEGER NOT NULL,
    synced_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY(book) REFERENCES books(id) ON DELETE CASCADE
);

-- Indexes for performance
CREATE INDEX idx_books_title ON books(title);
CREATE INDEX idx_books_timestamp ON books(timestamp);
CREATE INDEX idx_books_pubdate ON books(pubdate);
CREATE INDEX idx_books_uuid ON books(uuid);
CREATE INDEX idx_authors_name ON authors(name);
CREATE INDEX idx_tags_name ON tags(name);
CREATE INDEX idx_series_name ON series(name);
CREATE INDEX idx_identifiers_book ON identifiers(book);
CREATE INDEX idx_data_book ON data(book);
CREATE INDEX idx_reading_progress_book ON reading_progress(book);
CREATE INDEX idx_annotations_book ON annotations(book);
```

### 5.2 Performance Optimizations

**Caching Strategy:**

```rust
// src-tauri/src/database/cache.rs
use lru::LruCache;
use std::num::NonZeroUsize;

pub struct DatabaseCache {
    books: LruCache<i64, Book>,
    metadata: LruCache<i64, Metadata>,
    covers: LruCache<i64, Vec<u8>>,
}

impl DatabaseCache {
    pub fn new() -> Self {
        Self {
            books: LruCache::new(NonZeroUsize::new(1000).unwrap()),
            metadata: LruCache::new(NonZeroUsize::new(5000).unwrap()),
            covers: LruCache::new(NonZeroUsize::new(500).unwrap()),
        }
    }

    pub fn get_book(&mut self, id: i64) -> Option<&Book> {
        self.books.get(&id)
    }

    pub fn put_book(&mut self, id: i64, book: Book) {
        self.books.put(id, book);
    }
}
```

**Query Optimizations:**
- Use prepared statements
- Batch inserts for bulk operations
- Analyze query plans with EXPLAIN QUERY PLAN
- Vacuum database periodically
- Use WAL mode for concurrent access

---

## 6. E-book Processing Pipeline

### 6.1 Format Support Matrix

| Format | Read | Write | Conversion | Metadata | Cover |
|--------|------|-------|------------|----------|-------|
| EPUB | ✅ | ✅ | ✅ | ✅ | ✅ |
| MOBI | ✅ | ✅ | ✅ | ✅ | ✅ |
| AZW3 | ✅ | ✅ | ✅ | ✅ | ✅ |
| PDF | ✅ | ✅ | ✅ | ✅ | ✅ |
| DOCX | ✅ | ✅ | ✅ | ✅ | ❌ |
| FB2 | ✅ | ✅ | ✅ | ✅ | ✅ |
| HTML | ✅ | ✅ | ✅ | ❌ | ❌ |
| TXT | ✅ | ✅ | ✅ | ❌ | ❌ |
| RTF | ✅ | ✅ | ✅ | ❌ | ❌ |
| CBZ/CBR | ✅ | ✅ | ✅ | ✅ | ✅ |
| DJVU | ✅ | ❌ | ✅ | ✅ | ✅ |
| LIT | ✅ | ✅ | ✅ | ✅ | ✅ |
| PDB | ✅ | ✅ | ✅ | ✅ | ✅ |
| ... | ... | ... | ... | ... | ... |
| **Total** | **40+** | **30+** | **40+** | **35+** | **30+** |

### 6.2 Conversion Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CONVERSION PIPELINE                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Input File                                                  │
│      ↓                                                       │
│  ┌──────────────────────────────────────────┐              │
│  │  1. Format Detection                      │              │
│  │     - Magic bytes / file extension        │              │
│  │     - Validate file structure             │              │
│  └──────────────────────────────────────────┘              │
│      ↓                                                       │
│  ┌──────────────────────────────────────────┐              │
│  │  2. Input Plugin Selection                │              │
│  │     - Choose appropriate parser           │              │
│  │     - Load plugin (Python or WASM)        │              │
│  └──────────────────────────────────────────┘              │
│      ↓                                                       │
│  ┌──────────────────────────────────────────┐              │
│  │  3. Parse & Extract                       │              │
│  │     - Parse document structure            │              │
│  │     - Extract text, images, styles        │              │
│  │     - Build intermediate OEB format       │              │
│  └──────────────────────────────────────────┘              │
│      ↓                                                       │
│  ┌──────────────────────────────────────────┐              │
│  │  4. Transformations (Pipeline)            │              │
│  │     - Structure detection                 │              │
│  │     - Embed fonts                         │              │
│  │     - Flatten CSS                         │              │
│  │     - Insert jacket (metadata page)       │              │
│  │     - Rescale images                      │              │
│  │     - Insert page breaks                  │              │
│  │     - ... (configurable)                  │              │
│  └──────────────────────────────────────────┘              │
│      ↓                                                       │
│  ┌──────────────────────────────────────────┐              │
│  │  5. Output Plugin Selection               │              │
│  │     - Choose appropriate writer           │              │
│  │     - Load plugin (Python or WASM)        │              │
│  └──────────────────────────────────────────┘              │
│      ↓                                                       │
│  ┌──────────────────────────────────────────┐              │
│  │  6. Generate Output                       │              │
│  │     - Write in target format              │              │
│  │     - Validate output                     │              │
│  │     - Embed metadata                      │              │
│  └──────────────────────────────────────────┘              │
│      ↓                                                       │
│  Output File                                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 Conversion Implementation

**Rust Side (Conversion Queue):**

```rust
// src-tauri/src/commands/conversion.rs
use serde::{Deserialize, Serialize};
use std::sync::{Arc, Mutex};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ConversionJob {
    pub id: String,
    pub input_path: String,
    pub output_path: String,
    pub input_format: String,
    pub output_format: String,
    pub options: serde_json::Value,
    pub status: JobStatus,
    pub progress: f32,
    pub error: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum JobStatus {
    Pending,
    Running,
    Completed,
    Failed,
}

pub struct ConversionQueue {
    jobs: Arc<Mutex<Vec<ConversionJob>>>,
    python_bridge: Arc<Mutex<PythonBridge>>,
}

impl ConversionQueue {
    pub fn new() -> Self {
        Self {
            jobs: Arc::new(Mutex::new(Vec::new())),
            python_bridge: Arc::new(Mutex::new(
                PythonBridge::new("python").unwrap()
            )),
        }
    }

    pub fn add_job(&self, job: ConversionJob) -> String {
        let mut jobs = self.jobs.lock().unwrap();
        jobs.push(job.clone());

        // Start conversion in background thread
        let jobs_clone = self.jobs.clone();
        let bridge_clone = self.python_bridge.clone();
        let job_id = job.id.clone();

        tokio::spawn(async move {
            Self::run_conversion(jobs_clone, bridge_clone, job_id).await;
        });

        job.id
    }

    async fn run_conversion(
        jobs: Arc<Mutex<Vec<ConversionJob>>>,
        bridge: Arc<Mutex<PythonBridge>>,
        job_id: String,
    ) {
        // Update status to running
        {
            let mut jobs = jobs.lock().unwrap();
            if let Some(job) = jobs.iter_mut().find(|j| j.id == job_id) {
                job.status = JobStatus::Running;
            }
        }

        // Run conversion via Python bridge
        let result = {
            let mut bridge = bridge.lock().unwrap();
            bridge.convert_ebook(/* params */)
        };

        // Update status
        {
            let mut jobs = jobs.lock().unwrap();
            if let Some(job) = jobs.iter_mut().find(|j| j.id == job_id) {
                match result {
                    Ok(_) => {
                        job.status = JobStatus::Completed;
                        job.progress = 1.0;
                    }
                    Err(e) => {
                        job.status = JobStatus::Failed;
                        job.error = Some(e);
                    }
                }
            }
        }
    }
}

#[tauri::command]
pub fn start_conversion(
    queue: tauri::State<ConversionQueue>,
    input: String,
    output: String,
    input_format: String,
    output_format: String,
    options: serde_json::Value,
) -> String {
    let job = ConversionJob {
        id: uuid::Uuid::new_v4().to_string(),
        input_path: input,
        output_path: output,
        input_format,
        output_format,
        options,
        status: JobStatus::Pending,
        progress: 0.0,
        error: None,
    };

    queue.add_job(job)
}

#[tauri::command]
pub fn get_conversion_jobs(
    queue: tauri::State<ConversionQueue>,
) -> Vec<ConversionJob> {
    queue.jobs.lock().unwrap().clone()
}
```

---

## 7. Device Sync Architecture

### 7.1 Supported Devices

| Device Type | Protocol | Driver | Sync | Wireless |
|-------------|----------|--------|------|----------|
| Kindle (USB) | USB Mass Storage | ✅ | ✅ | ❌ |
| Kindle (WiFi) | Amazon Cloud | ✅ | ✅ | ✅ |
| Kobo | USB Mass Storage | ✅ | ✅ | ❌ |
| Nook | USB Mass Storage | ✅ | ✅ | ❌ |
| Android | MTP | ✅ | ✅ | ✅ |
| iOS | iTunes/Finder | ⚠️ | ⚠️ | ✅ |
| Generic MTP | MTP | ✅ | ✅ | ❌ |
| Folder Device | File System | ✅ | ✅ | ✅ |

### 7.2 Sync Engine Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                       DEVICE SYNC ENGINE                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────┐              │
│  │  1. Device Detection                      │              │
│  │     - USB hotplug monitoring              │              │
│  │     - Identify device type                │              │
│  │     - Load device driver                  │              │
│  └──────────────────────────────────────────┘              │
│      ↓                                                       │
│  ┌──────────────────────────────────────────┐              │
│  │  2. Library Comparison                    │              │
│  │     - Read device library                 │              │
│  │     - Compare with Calibre library        │              │
│  │     - Detect new/modified/deleted books   │              │
│  └──────────────────────────────────────────┘              │
│      ↓                                                       │
│  ┌──────────────────────────────────────────┐              │
│  │  3. Conflict Resolution                   │              │
│  │     - User chooses sync direction         │              │
│  │     - Merge or overwrite                  │              │
│  │     - Handle duplicates                   │              │
│  └──────────────────────────────────────────┘              │
│      ↓                                                       │
│  ┌──────────────────────────────────────────┐              │
│  │  4. File Transfer                         │              │
│  │     - Format conversion (if needed)       │              │
│  │     - Copy files to device                │              │
│  │     - Update metadata on device           │              │
│  │     - Verify transfer                     │              │
│  └──────────────────────────────────────────┘              │
│      ↓                                                       │
│  ┌──────────────────────────────────────────┐              │
│  │  5. Sync Completion                       │              │
│  │     - Eject device safely                 │              │
│  │     - Update sync log                     │              │
│  │     - Notify user                         │              │
│  └──────────────────────────────────────────┘              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 Device Driver Interface

```rust
// src-tauri/src/devices/mod.rs
use async_trait::async_trait;

#[async_trait]
pub trait DeviceDriver: Send + Sync {
    /// Get device info
    fn get_device_info(&self) -> DeviceInfo;

    /// Check if device is connected
    async fn is_connected(&self) -> bool;

    /// Get list of books on device
    async fn get_books(&self) -> Result<Vec<DeviceBook>, DeviceError>;

    /// Send book to device
    async fn send_book(
        &self,
        book: &Book,
        progress: impl Fn(f32) + Send,
    ) -> Result<(), DeviceError>;

    /// Delete book from device
    async fn delete_book(&self, id: &str) -> Result<(), DeviceError>;

    /// Update metadata on device
    async fn update_metadata(
        &self,
        id: &str,
        metadata: &Metadata,
    ) -> Result<(), DeviceError>;

    /// Eject device safely
    async fn eject(&self) -> Result<(), DeviceError>;
}

// Kindle driver implementation
pub struct KindleDriver {
    mount_point: PathBuf,
}

#[async_trait]
impl DeviceDriver for KindleDriver {
    fn get_device_info(&self) -> DeviceInfo {
        DeviceInfo {
            name: "Kindle".to_string(),
            vendor: "Amazon".to_string(),
            model: self.detect_model(),
            serial: self.read_serial(),
        }
    }

    async fn get_books(&self) -> Result<Vec<DeviceBook>, DeviceError> {
        // Read books from device's documents folder
        let docs_path = self.mount_point.join("documents");
        let mut books = Vec::new();

        for entry in std::fs::read_dir(docs_path)? {
            let entry = entry?;
            let path = entry.path();

            if path.extension().and_then(|s| s.to_str()) == Some("azw3") {
                // Extract metadata
                let metadata = self.extract_metadata(&path)?;
                books.push(DeviceBook {
                    path: path.to_string_lossy().to_string(),
                    metadata,
                });
            }
        }

        Ok(books)
    }

    async fn send_book(
        &self,
        book: &Book,
        progress: impl Fn(f32) + Send,
    ) -> Result<(), DeviceError> {
        // Convert to Kindle format if needed
        let target_format = "azw3";
        let source_path = &book.path;

        let dest_path = self.mount_point
            .join("documents")
            .join(&book.title)
            .with_extension(target_format);

        // If not in azw3 format, convert
        if book.format != target_format {
            self.convert_book(source_path, &dest_path, progress).await?;
        } else {
            self.copy_book(source_path, &dest_path, progress).await?;
        }

        Ok(())
    }

    async fn eject(&self) -> Result<(), DeviceError> {
        // Platform-specific eject
        #[cfg(target_os = "linux")]
        {
            use std::process::Command;
            Command::new("udisksctl")
                .args(&["unmount", "-b", &self.mount_point.to_string_lossy()])
                .output()?;
        }

        #[cfg(target_os = "macos")]
        {
            use std::process::Command;
            Command::new("diskutil")
                .args(&["eject", &self.mount_point.to_string_lossy()])
                .output()?;
        }

        #[cfg(target_os = "windows")]
        {
            // Use Windows API to eject
        }

        Ok(())
    }
}
```

---

## 8. Plugin System Architecture

### 8.1 Plugin Types

1. **Input Plugins**: Parse new e-book formats
2. **Output Plugins**: Write to new e-book formats
3. **Metadata Plugins**: Fetch metadata from new sources
4. **Catalog Plugins**: Generate book catalogs (PDF, EPUB, etc.)
5. **Device Plugins**: Support new e-reader devices
6. **UI Plugins**: Add new UI components or panels
7. **Processing Plugins**: Custom transformations during conversion

### 8.2 Plugin Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      PLUGIN SYSTEM                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────┐              │
│  │         Plugin Runtime (WASM)             │              │
│  │  ┌────────────────────────────────────┐  │              │
│  │  │  Plugin 1 (WASM Module)            │  │              │
│  │  │  - Sandboxed execution             │  │              │
│  │  │  - Limited capabilities            │  │              │
│  │  │  - No direct file system access    │  │              │
│  │  └────────────────────────────────────┘  │              │
│  │  ┌────────────────────────────────────┐  │              │
│  │  │  Plugin 2 (JavaScript)             │  │              │
│  │  │  - QuickJS runtime                 │  │              │
│  │  │  - Isolated context                │  │              │
│  │  │  - Message passing to host         │  │              │
│  │  └────────────────────────────────────┘  │              │
│  └──────────────────────────────────────────┘              │
│                      ↕ (IPC)                                │
│  ┌──────────────────────────────────────────┐              │
│  │         Plugin API (Rust)                 │              │
│  │  - File system access (with permission)   │              │
│  │  - Database access (read-only)            │              │
│  │  - Network access (with permission)       │              │
│  │  - UI extension points                    │              │
│  └──────────────────────────────────────────┘              │
│                      ↕                                       │
│  ┌──────────────────────────────────────────┐              │
│  │         Plugin Manager                    │              │
│  │  - Load/unload plugins                    │              │
│  │  - Permission management                  │              │
│  │  - Plugin discovery                       │              │
│  │  - Version management                     │              │
│  │  - Update checking                        │              │
│  └──────────────────────────────────────────┘              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 Plugin API Definition

```typescript
// types/plugin.ts

export interface PluginManifest {
  name: string
  version: string
  author: string
  description: string
  type: PluginType
  capabilities: Capability[]
  entrypoint: string
}

export enum PluginType {
  Input = 'input',
  Output = 'output',
  Metadata = 'metadata',
  Catalog = 'catalog',
  Device = 'device',
  UI = 'ui',
  Processing = 'processing',
}

export enum Capability {
  FileSystemRead = 'fs:read',
  FileSystemWrite = 'fs:write',
  NetworkAccess = 'network',
  DatabaseRead = 'db:read',
  DatabaseWrite = 'db:write',
  UIExtension = 'ui:extend',
}

export interface PluginAPI {
  // File system (requires fs:read or fs:write capability)
  fs: {
    readFile(path: string): Promise<Uint8Array>
    writeFile(path: string, data: Uint8Array): Promise<void>
    readDir(path: string): Promise<string[]>
  }

  // Database (requires db:read or db:write capability)
  db: {
    query(sql: string, params: any[]): Promise<any[]>
    execute(sql: string, params: any[]): Promise<void>
  }

  // Network (requires network capability)
  net: {
    fetch(url: string, options?: RequestInit): Promise<Response>
  }

  // UI (requires ui:extend capability)
  ui: {
    registerMenuItem(item: MenuItem): void
    registerPanel(panel: Panel): void
    showNotification(message: string): void
  }

  // Metadata
  metadata: {
    getBookMetadata(id: string): Promise<Metadata>
    updateBookMetadata(id: string, metadata: Partial<Metadata>): Promise<void>
  }

  // Conversion
  conversion: {
    convertBook(
      input: string,
      output: string,
      options: ConversionOptions
    ): Promise<void>
  }
}
```

**Example Plugin (WASM):**

```rust
// plugins/goodreads-metadata/src/lib.rs
use serde::{Deserialize, Serialize};
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct GoodreadsPlugin {
    api_key: String,
}

#[derive(Serialize, Deserialize)]
pub struct Metadata {
    title: String,
    authors: Vec<String>,
    description: String,
    rating: f32,
    cover_url: String,
}

#[wasm_bindgen]
impl GoodreadsPlugin {
    #[wasm_bindgen(constructor)]
    pub fn new(api_key: String) -> Self {
        Self { api_key }
    }

    #[wasm_bindgen]
    pub async fn fetch_metadata(&self, isbn: &str) -> Result<JsValue, JsValue> {
        // Call Goodreads API
        let url = format!(
            "https://www.goodreads.com/book/isbn/{}?key={}",
            isbn, self.api_key
        );

        let response = fetch_api(&url).await?;
        let metadata = parse_goodreads_response(&response)?;

        Ok(serde_wasm_bindgen::to_value(&metadata)?)
    }
}

// Helper to call fetch API from WASM
#[wasm_bindgen]
extern "C" {
    async fn fetch_api(url: &str) -> Result<String, JsValue>;
}
```

**Plugin Manifest:**

```json
{
  "name": "Goodreads Metadata",
  "version": "1.0.0",
  "author": "Plugin Author",
  "description": "Fetch book metadata from Goodreads",
  "type": "metadata",
  "capabilities": ["network"],
  "entrypoint": "goodreads_metadata.wasm"
}
```

---

## 9. Content Server Architecture

### 9.1 HTTP Server (Axum)

```rust
// src-tauri/src/server/http.rs
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::Json,
    routing::{get, post},
    Router,
};
use tower_http::cors::CorsLayer;

pub struct ServerState {
    db: Arc<Mutex<Database>>,
}

pub async fn start_server(port: u16, db: Arc<Mutex<Database>>) -> Result<(), std::io::Error> {
    let state = ServerState { db };

    let app = Router::new()
        // Books API
        .route("/api/books", get(get_books))
        .route("/api/books/:id", get(get_book))
        .route("/api/books/:id/download", get(download_book))
        .route("/api/books/:id/cover", get(get_cover))

        // Search
        .route("/api/search", get(search_books))

        // OPDS feed
        .route("/opds", get(opds_root))
        .route("/opds/all", get(opds_all_books))
        .route("/opds/recent", get(opds_recent_books))

        // WebSocket for sync
        .route("/ws", get(websocket_handler))

        .layer(CorsLayer::permissive())
        .with_state(state);

    let listener = tokio::net::TcpListener::bind(format!("0.0.0.0:{}", port))
        .await?;

    axum::serve(listener, app).await
}

async fn get_books(
    State(state): State<ServerState>,
    Query(params): Query<HashMap<String, String>>,
) -> Result<Json<Vec<Book>>, StatusCode> {
    let db = state.db.lock().unwrap();
    let books = db.get_books(/* params */)
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(Json(books))
}

async fn download_book(
    State(state): State<ServerState>,
    Path(id): Path<i64>,
) -> Result<Vec<u8>, StatusCode> {
    let db = state.db.lock().unwrap();
    let book = db.get_book(id)
        .map_err(|_| StatusCode::NOT_FOUND)?;

    let file_data = std::fs::read(&book.path)
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(file_data)
}
```

### 9.2 OPDS Feed

```rust
// src-tauri/src/server/opds.rs
use axum::{http::StatusCode, response::Html};

pub async fn opds_root() -> Result<Html<String>, StatusCode> {
    let feed = r#"
<?xml version="1.0" encoding="UTF-8"?>
<feed xmlns="http://www.w3.org/2005/Atom"
      xmlns:opds="http://opds-spec.org/2010/catalog">
    <id>calibre-modern:root</id>
    <title>Calibre Modern</title>
    <updated>2025-11-19T00:00:00Z</updated>
    <author><name>Calibre Modern</name></author>

    <link rel="self"
          type="application/atom+xml;profile=opds-catalog"
          href="/opds"/>

    <link rel="start"
          type="application/atom+xml;profile=opds-catalog"
          href="/opds"/>

    <entry>
        <title>All Books</title>
        <link rel="subsection"
              type="application/atom+xml;profile=opds-catalog"
              href="/opds/all"/>
        <updated>2025-11-19T00:00:00Z</updated>
        <id>calibre-modern:all</id>
    </entry>

    <entry>
        <title>Recent Books</title>
        <link rel="subsection"
              type="application/atom+xml;profile=opds-catalog"
              href="/opds/recent"/>
        <updated>2025-11-19T00:00:00Z</updated>
        <id>calibre-modern:recent</id>
    </entry>
</feed>
    "#.to_string();

    Ok(Html(feed))
}

pub async fn opds_all_books(
    State(state): State<ServerState>,
) -> Result<Html<String>, StatusCode> {
    let db = state.db.lock().unwrap();
    let books = db.get_books(/* all books */)
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let mut entries = String::new();
    for book in books {
        entries.push_str(&format!(r#"
    <entry>
        <title>{}</title>
        <id>calibre-modern:book:{}</id>
        <updated>{}</updated>
        <author><name>{}</name></author>
        <link rel="http://opds-spec.org/acquisition"
              type="{}"
              href="/api/books/{}/download"/>
        <link rel="http://opds-spec.org/image"
              type="image/jpeg"
              href="/api/books/{}/cover"/>
    </entry>
        "#, book.title, book.id, book.last_modified,
            book.authors.join(", "), book.mime_type, book.id, book.id));
    }

    let feed = format!(r#"
<?xml version="1.0" encoding="UTF-8"?>
<feed xmlns="http://www.w3.org/2005/Atom"
      xmlns:opds="http://opds-spec.org/2010/catalog">
    <id>calibre-modern:all</id>
    <title>All Books</title>
    <updated>2025-11-19T00:00:00Z</updated>
    {}
</feed>
    "#, entries);

    Ok(Html(feed))
}
```

---

## 10. Data Flow Diagrams

### 10.1 Book Import Flow

```
User Drags File → Frontend
                    ↓
              Validate File
                    ↓
              Tauri IPC Call
                    ↓
         Rust: receive_file()
                    ↓
         Copy to Library Folder
                    ↓
         Extract Metadata (Python Bridge)
                    ↓
         Insert into Database
                    ↓
         Extract Cover (Python Bridge)
                    ↓
         Update FTS Index
                    ↓
         Return Book ID
                    ↓
         Frontend: Invalidate Query Cache
                    ↓
         Frontend: Show Success Notification
```

### 10.2 Book Conversion Flow

```
User Selects Book → Click Convert → Frontend
                                      ↓
                                  Show Wizard
                                      ↓
                              Select Output Format
                                      ↓
                              Configure Options
                                      ↓
                              Tauri IPC: start_conversion()
                                      ↓
         Rust: Create Conversion Job
                                      ↓
         Add to Queue (Pending)
                                      ↓
         Spawn Background Task
                                      ↓
         Python Bridge: convert_ebook()
                |                     ↓
                |              Update Job Status (Running)
                |                     ↓
                |              Stream Progress Updates
                |                     ↓
         Python: Load Input Plugin
                |                     ↓
         Python: Parse Input File
                |                     ↓
         Python: Apply Transformations
                |                     ↓
         Python: Load Output Plugin
                |                     ↓
         Python: Generate Output
                |                     ↓
                └──────────> Return Result
                                      ↓
         Update Job Status (Completed/Failed)
                                      ↓
         Frontend: Poll Job Status
                                      ↓
         Frontend: Show Completion Notification
```

### 10.3 Device Sync Flow

```
USB Device Connected → OS Event
                          ↓
         Rust: USB Hotplug Handler
                          ↓
         Detect Device Type
                          ↓
         Load Device Driver
                          ↓
         Read Device Library
                          ↓
         Compare with Calibre Library
                          ↓
         Frontend: Show Sync Dialog
                          ↓
         User Selects Books to Sync
                          ↓
         Tauri IPC: sync_books()
                          ↓
         For Each Book:
           ├─ Check Format
           ├─ Convert if Needed (Python Bridge)
           ├─ Copy to Device
           ├─ Update Progress
           └─ Verify Transfer
                          ↓
         Update Sync Log in Database
                          ↓
         Eject Device Safely
                          ↓
         Frontend: Show Completion
```

---

## 11. API Design Patterns

### 11.1 Tauri IPC Commands

```rust
// src-tauri/src/commands/books.rs

#[tauri::command]
pub async fn get_books(
    db: tauri::State<'_, Arc<Mutex<Database>>>,
    sort_by: Option<String>,
    sort_order: Option<String>,
    limit: Option<usize>,
    offset: Option<usize>,
) -> Result<Vec<Book>, String> {
    let db = db.lock().unwrap();
    db.get_books(
        &sort_by.unwrap_or("title".to_string()),
        &sort_order.unwrap_or("asc".to_string()),
        limit.unwrap_or(100),
        offset.unwrap_or(0),
    )
    .map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn get_book(
    db: tauri::State<'_, Arc<Mutex<Database>>>,
    id: i64,
) -> Result<Book, String> {
    let db = db.lock().unwrap();
    db.get_book(id).map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn search_books(
    db: tauri::State<'_, Arc<Mutex<Database>>>,
    query: String,
) -> Result<Vec<Book>, String> {
    let db = db.lock().unwrap();
    db.search_books(&query).map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn update_book(
    db: tauri::State<'_, Arc<Mutex<Database>>>,
    id: i64,
    updates: serde_json::Value,
) -> Result<(), String> {
    let db = db.lock().unwrap();
    db.update_book(id, &updates).map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn delete_books(
    db: tauri::State<'_, Arc<Mutex<Database>>>,
    ids: Vec<i64>,
) -> Result<(), String> {
    let db = db.lock().unwrap();
    for id in ids {
        db.delete_book(id).map_err(|e| e.to_string())?;
    }
    Ok(())
}

#[tauri::command]
pub async fn import_book(
    db: tauri::State<'_, Arc<Mutex<Database>>>,
    python: tauri::State<'_, Arc<Mutex<PythonBridge>>>,
    path: String,
) -> Result<i64, String> {
    // Copy file to library
    let book_id = generate_book_id();
    let dest_path = get_library_path().join(book_id.to_string());
    std::fs::copy(&path, &dest_path).map_err(|e| e.to_string())?;

    // Extract metadata
    let mut python = python.lock().unwrap();
    let metadata = python
        .extract_metadata(&dest_path.to_string_lossy())
        .map_err(|e| e.to_string())?;

    // Insert into database
    let db = db.lock().unwrap();
    let id = db.insert_book(&metadata).map_err(|e| e.to_string())?;

    Ok(id)
}
```

### 11.2 Frontend API Usage (TypeScript)

```typescript
// lib/tauri.ts
import { invoke } from '@tauri-apps/api/core'

export async function getBooks(options: {
  sortBy?: string
  sortOrder?: 'asc' | 'desc'
  limit?: number
  offset?: number
} = {}) {
  return invoke<Book[]>('get_books', {
    sortBy: options.sortBy,
    sortOrder: options.sortOrder,
    limit: options.limit,
    offset: options.offset,
  })
}

export async function getBook(id: string) {
  return invoke<Book>('get_book', { id: parseInt(id) })
}

export async function searchBooks(query: string) {
  return invoke<Book[]>('search_books', { query })
}

export async function updateBook(id: string, updates: Partial<Book>) {
  return invoke('update_book', { id: parseInt(id), updates })
}

export async function deleteBooks(ids: string[]) {
  return invoke('delete_books', { ids: ids.map((id) => parseInt(id)) })
}

export async function importBook(path: string) {
  return invoke<number>('import_book', { path })
}
```

### 11.3 React Hooks Usage

```typescript
// hooks/useBooks.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import * as api from '@/lib/tauri'

export function useBooks(options: Parameters<typeof api.getBooks>[0] = {}) {
  return useQuery({
    queryKey: ['books', options],
    queryFn: () => api.getBooks(options),
  })
}

export function useBook(id: string) {
  return useQuery({
    queryKey: ['book', id],
    queryFn: () => api.getBook(id),
    enabled: !!id,
  })
}

export function useSearchBooks(query: string) {
  return useQuery({
    queryKey: ['search', query],
    queryFn: () => api.searchBooks(query),
    enabled: query.length > 0,
  })
}

export function useUpdateBook() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: ({ id, updates }: { id: string; updates: Partial<Book> }) =>
      api.updateBook(id, updates),
    onSuccess: (_, variables) => {
      queryClient.invalidateQueries({ queryKey: ['book', variables.id] })
      queryClient.invalidateQueries({ queryKey: ['books'] })
    },
  })
}

export function useDeleteBooks() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (ids: string[]) => api.deleteBooks(ids),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['books'] })
    },
  })
}

export function useImportBook() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (path: string) => api.importBook(path),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['books'] })
    },
  })
}
```

---

## 12. Performance Architecture

### 12.1 Performance Goals

| Operation | Target | Current Calibre |
|-----------|--------|-----------------|
| **App Startup** | <500ms | ~1-2s |
| **Library Load (100k books)** | <1s | ~3-5s |
| **Search (100k books)** | <100ms | ~500ms |
| **Book Details Load** | <50ms | ~100ms |
| **Cover Grid Render** | 60 FPS | ~30 FPS |
| **Table Scroll** | 60 FPS | 60 FPS |
| **EPUB Render** | <200ms | ~500ms |
| **PDF Render (page)** | <100ms | ~200ms |
| **Conversion (EPUB→PDF)** | ~30s/book | ~30s/book |
| **Device Sync (100 books)** | <5min | ~5min |
| **Memory Usage (idle)** | <100MB | ~200-300MB |
| **Bundle Size** | <50MB | ~100-150MB |

### 12.2 Performance Optimizations

#### 12.2.1 Frontend Optimizations

**Virtual Scrolling:**
```typescript
// Use TanStack Virtual for large lists
import { useVirtualizer } from '@tanstack/react-virtual'

export function BookTable({ books }: { books: Book[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: books.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 35,
    overscan: 10, // Render 10 rows above/below
  })

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map((virtualRow) => {
          const book = books[virtualRow.index]
          return (
            <BookRow
              key={book.id}
              book={book}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                transform: `translateY(${virtualRow.start}px)`,
              }}
            />
          )
        })}
      </div>
    </div>
  )
}
```

**Lazy Loading:**
```typescript
// Lazy load heavy components
const EpubReader = lazy(() => import('@/components/reader/EpubReader'))
const PdfReader = lazy(() => import('@/components/reader/PdfReader'))
const BookEditor = lazy(() => import('@/components/editor/BookEditor'))

// Use Suspense for loading states
<Suspense fallback={<LoadingSpinner />}>
  <EpubReader bookId={id} />
</Suspense>
```

**Image Optimization:**
```typescript
// Lazy load covers
import { LazyLoadImage } from 'react-lazy-load-image-component'

<LazyLoadImage
  src={book.coverUrl}
  alt={book.title}
  effect="blur"
  threshold={100}
/>
```

**Memoization:**
```typescript
// Memoize expensive computations
const filteredBooks = useMemo(() => {
  return books.filter((book) => {
    // Complex filtering logic
  })
}, [books, filters])

// Memoize components
const BookRow = memo(({ book }: { book: Book }) => {
  return <tr>{/* Row content */}</tr>
})
```

**Debouncing:**
```typescript
// Debounce search input
import { useDebouncedValue } from '@/hooks/useDebouncedValue'

const [search, setSearch] = useState('')
const debouncedSearch = useDebouncedValue(search, 300)

useEffect(() => {
  // Trigger search only after 300ms of no typing
  performSearch(debouncedSearch)
}, [debouncedSearch])
```

#### 12.2.2 Backend Optimizations

**Database Indexing:**
```sql
-- Index frequently queried columns
CREATE INDEX idx_books_title ON books(title);
CREATE INDEX idx_books_timestamp ON books(timestamp DESC);
CREATE INDEX idx_authors_name ON authors(name);
CREATE INDEX idx_tags_name ON tags(name);

-- Composite indexes for common queries
CREATE INDEX idx_books_author_title ON books_authors_link(author, book);
CREATE INDEX idx_books_tags ON books_tags_link(tag, book);
```

**Query Optimization:**
```rust
// Use prepared statements
let mut stmt = conn.prepare_cached(
    "SELECT * FROM books WHERE title LIKE ? LIMIT ?"
)?;

// Batch inserts
conn.execute("BEGIN TRANSACTION")?;
for book in books {
    conn.execute("INSERT INTO books ...", params)?;
}
conn.execute("COMMIT")?;
```

**Caching:**
```rust
// LRU cache for frequently accessed data
use lru::LruCache;

struct BookCache {
    cache: LruCache<i64, Book>,
}

impl BookCache {
    pub fn get(&mut self, id: i64) -> Option<&Book> {
        self.cache.get(&id)
    }

    pub fn put(&mut self, id: i64, book: Book) {
        self.cache.put(id, book);
    }
}
```

**Parallel Processing:**
```rust
// Use Rayon for parallel operations
use rayon::prelude::*;

let results: Vec<_> = books
    .par_iter()
    .map(|book| process_book(book))
    .collect();
```

**Worker Threads:**
```typescript
// Offload heavy computation to Web Workers
const worker = new Worker('/workers/search.js')

worker.postMessage({ query: searchQuery, books })

worker.onmessage = (e) => {
  setSearchResults(e.data)
}
```

---

## 13. Security Architecture

### 13.1 Security Principles

1. **Principle of Least Privilege**: Plugins get minimal permissions
2. **Defense in Depth**: Multiple layers of security
3. **Input Validation**: All user input sanitized
4. **Sandboxing**: Isolate untrusted code
5. **Encryption**: Sensitive data encrypted at rest and in transit

### 13.2 Security Measures

#### 13.2.1 Tauri Security

```json
// tauri.conf.json
{
  "security": {
    "csp": "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:;",
    "dangerousDisableAssetCspModification": false,
    "freezePrototype": true,
    "dangerousRemoteDomainIpcAccess": [],
    "dangerousDisableIpcSecurityValidation": false
  },
  "allowlist": {
    "fs": {
      "scope": ["$APPLOCALDATA/**", "$APPDATA/**"]
    },
    "shell": {
      "scope": [
        {
          "name": "python",
          "cmd": "python",
          "args": true
        }
      ]
    }
  }
}
```

#### 13.2.2 Plugin Sandboxing

```rust
// src-tauri/src/plugins/sandbox.rs
use wasmer::{Instance, Module, Store};

pub struct PluginSandbox {
    instance: Instance,
    capabilities: Vec<Capability>,
}

impl PluginSandbox {
    pub fn new(wasm_bytes: &[u8], capabilities: Vec<Capability>) -> Result<Self, String> {
        let store = Store::default();
        let module = Module::new(&store, wasm_bytes)
            .map_err(|e| e.to_string())?;

        // Create imports with restricted capabilities
        let import_object = create_restricted_imports(&store, &capabilities);

        let instance = Instance::new(&module, &import_object)
            .map_err(|e| e.to_string())?;

        Ok(Self {
            instance,
            capabilities,
        })
    }

    pub fn call_function(&self, name: &str, params: &[Value]) -> Result<Value, String> {
        // Validate function call against capabilities
        if !self.can_call_function(name) {
            return Err("Permission denied".to_string());
        }

        let function = self.instance
            .exports
            .get_function(name)
            .map_err(|e| e.to_string())?;

        function.call(params).map_err(|e| e.to_string())
    }

    fn can_call_function(&self, name: &str) -> bool {
        // Check if plugin has necessary capability
        match name {
            "read_file" => self.capabilities.contains(&Capability::FileSystemRead),
            "write_file" => self.capabilities.contains(&Capability::FileSystemWrite),
            "fetch" => self.capabilities.contains(&Capability::NetworkAccess),
            _ => true, // Allow safe functions
        }
    }
}
```

#### 13.2.3 Input Validation

```rust
// src-tauri/src/commands/books.rs
use validator::Validate;

#[derive(Validate, Deserialize)]
pub struct UpdateBookRequest {
    #[validate(length(min = 1, max = 500))]
    pub title: Option<String>,

    #[validate(length(max = 13))]
    pub isbn: Option<String>,

    #[validate(range(min = 0, max = 10))]
    pub rating: Option<i32>,

    #[validate(url)]
    pub cover_url: Option<String>,
}

#[tauri::command]
pub async fn update_book(
    db: tauri::State<'_, Arc<Mutex<Database>>>,
    id: i64,
    request: UpdateBookRequest,
) -> Result<(), String> {
    // Validate input
    request.validate().map_err(|e| e.to_string())?;

    // Sanitize inputs
    let title = request.title.as_ref().map(|t| sanitize_html(t));

    // Update database
    let db = db.lock().unwrap();
    db.update_book(id, &request).map_err(|e| e.to_string())
}
```

#### 13.2.4 SQL Injection Prevention

```rust
// Always use parameterized queries
let stmt = conn.prepare("SELECT * FROM books WHERE title = ?")?;
let books = stmt.query_map([&title], |row| {
    // Map row to Book
})?;

// NEVER do this:
// let query = format!("SELECT * FROM books WHERE title = '{}'", title);
```

#### 13.2.5 XSS Prevention

```typescript
// Sanitize user input before rendering
import DOMPurify from 'dompurify'

function BookDescription({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'br', 'b', 'i', 'em', 'strong'],
  })

  return <div dangerouslySetInnerHTML={{ __html: clean }} />
}
```

---

## 14. Module Breakdown

### 14.1 Core Modules

| Module | Responsibility | Language | LOC (Est.) |
|--------|---------------|----------|------------|
| **Frontend** | UI, user interaction | TypeScript/React | 50,000 |
| **Tauri Core** | Window, IPC, system | Rust | 15,000 |
| **Database** | SQLite operations | Rust | 5,000 |
| **E-book Processing** | Format handling | Python | 80,000 (reuse Calibre) |
| **Device Drivers** | USB, sync | Rust | 8,000 |
| **Metadata Sources** | API integrations | Rust | 5,000 |
| **Plugin System** | WASM runtime | Rust | 10,000 |
| **Content Server** | HTTP/OPDS | Rust | 5,000 |
| **Python Bridge** | IPC to Python | Rust | 3,000 |
| **Tests** | Unit, integration | Rust/TS | 20,000 |
| **TOTAL** | | | **201,000** |

### 14.2 Development Priorities (Phases)

#### Phase 1: Core Foundation (Months 1-4)
- ✅ Tauri project setup
- ✅ Database schema & migrations
- ✅ Basic CRUD operations
- ✅ Library browser (table view)
- ✅ Book import
- ✅ Metadata editing
- ✅ Python bridge for conversion
- ✅ Simple e-book viewer (EPUB only)

**Deliverable**: MVP that can manage a library

---

#### Phase 2: Essential Features (Months 5-8)
- ✅ Cover grid view
- ✅ Advanced search (FTS5)
- ✅ Tag browser
- ✅ Series management
- ✅ Bulk editing
- ✅ PDF viewer
- ✅ Basic conversion UI
- ✅ Device detection

**Deliverable**: Feature-complete for basic users

---

#### Phase 3: Advanced Features (Months 9-14)
- ✅ E-book editor
- ✅ Advanced conversion options
- ✅ Device sync (Kindle, Kobo)
- ✅ Metadata download (multiple sources)
- ✅ Reading progress tracking
- ✅ Annotations & highlights
- ✅ Content server (HTTP + OPDS)
- ✅ Plugin system MVP

**Deliverable**: Feature parity with Calibre desktop

---

#### Phase 4: Polish & Optimization (Months 15-18)
- ✅ Performance tuning
- ✅ UI/UX refinement
- ✅ Accessibility improvements
- ✅ Keyboard shortcuts
- ✅ Themes (light/dark/custom)
- ✅ Comprehensive testing
- ✅ Documentation
- ✅ Beta program

**Deliverable**: Production-ready release

---

#### Phase 5: Extensions (Months 19-24)
- ✅ More device drivers
- ✅ More metadata sources
- ✅ Plugin store
- ✅ Advanced plugins
- ✅ Mobile sync
- ✅ Cloud backup (optional)
- ✅ AI features (optional)

**Deliverable**: Extended ecosystem

---

## 15. Deployment & Distribution

### 15.1 Build Process

```bash
# Development build
npm run dev         # Start Next.js dev server
cargo tauri dev     # Start Tauri dev mode (hot reload)

# Production build
npm run build       # Build Next.js frontend
cargo tauri build   # Build Tauri app + bundle

# Outputs:
# - Windows: .exe installer, .msi installer
# - macOS: .dmg, .app bundle
# - Linux: .AppImage, .deb, .rpm
```

### 15.2 Distribution Channels

1. **Official Website**: Download directly
2. **GitHub Releases**: Auto-updates via Tauri updater
3. **Package Managers**:
   - Windows: Chocolatey, Winget
   - macOS: Homebrew Cask
   - Linux: Snap, Flatpak, AUR
4. **App Stores** (future):
   - Microsoft Store
   - Mac App Store (if worth the effort)

### 15.3 Auto-Update Architecture

```rust
// src-tauri/src/main.rs
use tauri_plugin_updater::UpdaterExt;

fn main() {
    tauri::Builder::default()
        .plugin(tauri_plugin_updater::Builder::new().build())
        .setup(|app| {
            let handle = app.handle().clone();
            tauri::async_runtime::spawn(async move {
                let updater = handle.updater().unwrap();

                // Check for updates
                if let Some(update) = updater.check().await.unwrap() {
                    // Download update
                    update.download_and_install().await.unwrap();
                }
            });
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### 15.4 Bundle Sizes (Estimated)

| Platform | Size (Compressed) | Size (Installed) |
|----------|------------------|------------------|
| **Windows** | 25-35 MB | 80-100 MB |
| **macOS** | 20-30 MB | 70-90 MB |
| **Linux** | 15-25 MB | 60-80 MB |

**Comparison to Calibre:**
- Calibre: ~100-150 MB (with Python runtime)
- Calibre Modern: ~25 MB (native WebView, no bundled browser)

---

## 16. Development Workflow

### 16.1 Tech Stack Summary

```yaml
Frontend:
  Framework: React 19
  Meta-Framework: Next.js 15 (App Router)
  Language: TypeScript 5.7+
  State: Zustand + TanStack Query
  UI: Tailwind CSS + shadcn/ui
  Tables: TanStack Table + TanStack Virtual
  Forms: React Hook Form + Zod
  E-book Rendering: Epub.js, PDF.js
  Editor: Monaco Editor
  Testing: Vitest, Playwright, Testing Library
  Build: Turbopack (Next.js built-in)

Backend:
  Desktop Framework: Tauri 2.0
  Language: Rust 1.83+
  Database: rusqlite (SQLite)
  HTTP Server: Axum
  USB: rusb
  E-book Processing: Python 3.11+ (Calibre libraries)
  Python Bridge: JSON-RPC over stdin/stdout
  Async Runtime: Tokio
  Serialization: serde

Plugin System:
  Runtime: Wasmer (WASM)
  Languages: Rust, JavaScript, Go, C (compiled to WASM)
  Sandbox: Capability-based security

Development Tools:
  Version Control: Git
  CI/CD: GitHub Actions
  Package Managers: pnpm (Node), Cargo (Rust), pip (Python)
  Linting: ESLint (TS), Clippy (Rust), Ruff (Python)
  Formatting: Prettier (TS), rustfmt (Rust), Black (Python)
  Documentation: Docusaurus
```

### 16.2 Project Structure

```
calibre-modern/
├── .github/
│   └── workflows/
│       ├── ci.yml                 # CI pipeline
│       ├── release.yml            # Release automation
│       └── test.yml               # Automated tests
├── src/                           # Frontend (Next.js)
│   ├── app/
│   ├── components/
│   ├── hooks/
│   ├── lib/
│   ├── stores/
│   └── types/
├── src-tauri/                     # Backend (Rust)
│   ├── src/
│   │   ├── commands/
│   │   ├── database/
│   │   ├── devices/
│   │   ├── ebook/
│   │   ├── metadata/
│   │   ├── plugins/
│   │   ├── server/
│   │   └── main.rs
│   ├── Cargo.toml
│   └── tauri.conf.json
├── python/                        # Python e-book processing
│   ├── ebook_processor.py
│   ├── conversion/
│   ├── metadata/
│   └── requirements.txt
├── plugins/                       # Sample plugins
│   ├── goodreads-metadata/
│   └── custom-format/
├── tests/                         # Integration tests
│   ├── e2e/
│   └── integration/
├── docs/                          # Documentation
│   ├── architecture.md
│   ├── api.md
│   ├── plugin-development.md
│   └── user-guide.md
├── package.json
├── tsconfig.json
├── next.config.js
├── tailwind.config.js
└── README.md
```

### 16.3 Development Commands

```bash
# Setup
git clone https://github.com/calibre-modern/calibre-modern
cd calibre-modern
pnpm install
cd python && pip install -r requirements.txt

# Development
pnpm dev                    # Start Next.js dev server + Tauri

# Testing
pnpm test                   # Run unit tests (Vitest)
pnpm test:e2e               # Run E2E tests (Playwright)
cargo test                  # Run Rust tests

# Linting
pnpm lint                   # ESLint (TypeScript)
cargo clippy                # Clippy (Rust)
pnpm format                 # Prettier + rustfmt

# Building
pnpm build                  # Build frontend
cargo tauri build           # Build desktop app

# Database migrations
pnpm migrate                # Run database migrations
```

---

## 17. Testing Strategy

### 17.1 Testing Pyramid

```
                    ┌─────────────┐
                    │  E2E Tests  │  (10%)
                    │  Playwright │
                    └─────────────┘
                  ┌───────────────────┐
                  │ Integration Tests  │  (20%)
                  │ Tauri + Database   │
                  └───────────────────┘
              ┌─────────────────────────────┐
              │      Unit Tests              │  (70%)
              │ Vitest (TS) + Cargo (Rust)   │
              └─────────────────────────────┘
```

### 17.2 Test Examples

**Unit Test (React Component):**
```typescript
// components/__tests__/BookTable.test.tsx
import { render, screen } from '@testing-library/react'
import { BookTable } from '@/components/library/BookTable'

describe('BookTable', () => {
  it('renders books correctly', () => {
    const books = [
      { id: 1, title: 'Test Book', authors: ['Author'] },
    ]

    render(<BookTable books={books} />)

    expect(screen.getByText('Test Book')).toBeInTheDocument()
  })

  it('handles empty state', () => {
    render(<BookTable books={[]} />)

    expect(screen.getByText('No books found')).toBeInTheDocument()
  })
})
```

**Unit Test (Rust):**
```rust
// src-tauri/src/database/queries.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_get_books() {
        let db = Database::new(":memory:").unwrap();

        // Insert test data
        db.insert_book(&Book {
            title: "Test Book".to_string(),
            // ...
        }).unwrap();

        // Query
        let books = db.get_books("title", "asc", 10, 0).unwrap();

        assert_eq!(books.len(), 1);
        assert_eq!(books[0].title, "Test Book");
    }
}
```

**Integration Test (Tauri Command):**
```rust
// tests/integration/books.rs
use tauri::test::mock_builder;

#[test]
fn test_import_book() {
    let app = mock_builder().build();

    let result = app
        .tauri_invoke("import_book", json!({
            "path": "/path/to/book.epub"
        }))
        .await;

    assert!(result.is_ok());
}
```

**E2E Test (Playwright):**
```typescript
// tests/e2e/library.spec.ts
import { test, expect } from '@playwright/test'

test('import and view book', async ({ page }) => {
  await page.goto('/')

  // Click import button
  await page.click('[data-testid="import-book"]')

  // Select file (mocked)
  await page.setInputFiles('input[type="file"]', 'test-book.epub')

  // Wait for import to complete
  await expect(page.locator('text=Test Book')).toBeVisible()

  // Click on book
  await page.click('text=Test Book')

  // Verify details panel
  await expect(page.locator('[data-testid="book-details"]')).toBeVisible()
})
```

---

## 18. Migration Strategy

### 18.1 Migrating from Calibre

**Import Existing Library:**

```rust
// src-tauri/src/commands/migration.rs
#[tauri::command]
pub async fn import_calibre_library(
    calibre_library_path: String,
) -> Result<ImportProgress, String> {
    let old_db_path = format!("{}/metadata.db", calibre_library_path);

    // Open Calibre database
    let calibre_db = Connection::open(&old_db_path)
        .map_err(|e| e.to_string())?;

    // Read all books
    let books = read_calibre_books(&calibre_db)?;

    // Import into new database
    let new_db = Database::new(&get_new_db_path())?;

    for (index, book) in books.iter().enumerate() {
        // Copy book files
        let old_path = format!("{}/{}", calibre_library_path, book.path);
        let new_path = copy_book_to_new_library(&old_path)?;

        // Insert metadata
        new_db.insert_book(&book)?;

        // Update progress
        emit_progress(index, books.len());
    }

    Ok(ImportProgress {
        total: books.len(),
        imported: books.len(),
        failed: 0,
    })
}
```

**Migration UI:**
```typescript
// app/migrate/page.tsx
export default function MigratePage() {
  const [libraryPath, setLibraryPath] = useState('')
  const [progress, setProgress] = useState(0)

  const handleMigrate = async () => {
    const result = await invoke('import_calibre_library', {
      calibreLibraryPath: libraryPath,
    })

    // Show progress
  }

  return (
    <div>
      <h1>Import Calibre Library</h1>
      <input
        type="text"
        value={libraryPath}
        onChange={(e) => setLibraryPath(e.target.value)}
        placeholder="/path/to/calibre/library"
      />
      <Button onClick={handleMigrate}>Start Import</Button>
      {progress > 0 && <ProgressBar value={progress} />}
    </div>
  )
}
```

---

## Conclusion

This architecture provides a complete blueprint for building a modern Calibre replacement with:

✅ **Feature Parity**: All Calibre features (library management, conversion, device sync, etc.)
✅ **Modern UI/UX**: React 19, Tailwind CSS, beautiful and responsive
✅ **High Performance**: 10-30x smaller bundle, 60-90% less RAM, faster startup
✅ **Cross-Platform**: Windows, macOS, Linux from single codebase
✅ **Scalable**: Designed for 100k+ books from day one
✅ **Extensible**: WebAssembly-based plugin system
✅ **Secure**: Sandboxed plugins, input validation, capability-based permissions
✅ **Local-First**: All data local, optional cloud sync
✅ **Backward Compatible**: Import existing Calibre libraries

### Estimated Timeline

| Phase | Duration | Team Size | Deliverable |
|-------|----------|-----------|-------------|
| Phase 1 | 4 months | 2-3 devs | MVP (library management) |
| Phase 2 | 4 months | 3-4 devs | Basic features |
| Phase 3 | 6 months | 4-5 devs | Feature parity |
| Phase 4 | 4 months | 4-5 devs | Production ready |
| Phase 5 | 6 months | 3-4 devs | Extended ecosystem |
| **Total** | **24 months** | **3-5 devs** | **Full replacement** |

### Estimated Cost

- **Team**: $500k-$800k/year (3-5 developers)
- **Total**: $1M-$1.6M (2-year project)

### Next Steps

1. **Prototype** (Month 1): Build proof-of-concept (Tauri + React + SQLite)
2. **MVP** (Months 2-4): Library browser, import, basic metadata editing
3. **Alpha** (Months 5-8): Core features, internal testing
4. **Beta** (Months 9-14): Feature complete, public testing
5. **Release** (Months 15-18): Production ready, polish, documentation
6. **Extensions** (Months 19-24): Plugins, advanced features, ecosystem

---

**This architecture is ready for implementation.** All major technical decisions have been made with detailed justifications. The team can start development immediately following this blueprint.

**Document Version:** 1.0
**Last Updated:** November 19, 2025
**Status:** Complete & Ready for Implementation
