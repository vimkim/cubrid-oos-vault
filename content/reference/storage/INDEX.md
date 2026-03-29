# CUBRID `src/storage/` — Master Index Report

**Generated:** 2026-03-28
**Source reports:** 26 per-file analysis reports in `reports/storage/`
**Module AGENTS.md:** `src/storage/AGENTS.md`

---

## Table of Contents

1. [Module Overview](#1-module-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [File Inventory](#3-file-inventory)
4. [Dependency Matrix](#4-dependency-matrix)
5. [Key Cross-File Call Graphs](#5-key-cross-file-call-graphs)
6. [Core Protocols](#6-core-protocols)
7. [Module Statistics](#7-module-statistics)
8. [Navigation Guide](#8-navigation-guide)

---

## 1. Module Overview

### Purpose in the CUBRID Architecture

`src/storage/` is the **physical storage engine layer** of CUBRID. It sits between the query execution layer (which produces logical plans) and the raw OS filesystem. Every read or write of a user row, index entry, catalog record, or LOB object ultimately flows through this module.

The module is compiled exclusively into server-side binaries:

| Build mode | Binary | Description |
|------------|--------|-------------|
| `SERVER_MODE` | `cub_server` | Multi-user server process — the primary target |
| `SA_MODE` | `cubridsa` | Standalone (client+server in-process), used by `csql -S` and utilities |
| `CS_MODE` | `cubridcs` | Client library — **does not include** most storage files; only utility headers |

A handful of files (`storage_common`, `oid`, `byte_order`, `es*`, `record_descriptor`, `statistics_cl`) compile in all three modes because they define shared types or client-callable utility functions.

### Key Abstractions

| Type | Fields | Purpose |
|------|--------|---------|
| `VPID` | `volid`, `pageid` | Volume + page address — the atomic disk unit |
| `VFID` | `volid`, `fileid` | Volume + file identifier (a file is a collection of pages) |
| `OID` | `volid`, `pageid`, `slotid` | Object identifier — permanent row address (heap record) |
| `HFID` | `vfid`, `hpgid` | Heap file identifier (one per table) |
| `BTID` | `vfid`, `root_pageid` | B-tree identifier (one per index) |
| `RECDES` | `data`, `length`, `area_size`, `type` | Record descriptor — universal in-memory record container |
| `LOG_LSN` | `pageid`, `offset` | Log Sequence Number — WAL position marker |
| `MVCCID` | (64-bit int) | MVCC transaction identifier for visibility checking |

### Design Philosophy and Layering

The storage module enforces a strict layered architecture:

1. **No raw disk I/O above `file_io.c`** — all page reads/writes go through `page_buffer.c` which calls `file_io.c` internally.
2. **No direct page access above `slotted_page.c`** — heap, B-tree, and catalog always use `spage_*` functions to manage records within pages.
3. **WAL before write** — any modification to a page must log an undo/redo record (via `log_append.hpp`) before calling `pgbuf_set_dirty()`.
4. **Buffer pool as the single page cache** — pages are never held in secondary caches; `pgbuf_fix()` / `pgbuf_unfix()` are the only access points.
5. **Sector-based disk allocation** — `disk_manager.c` allocates space in 64-page sectors; `file_manager.c` sub-allocates individual pages within sectors.

---

## 2. Architecture Diagram

### Vertical Layering (top to bottom = higher to lower abstraction)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          SQL / Query Layer                               │
│   src/query/query_executor.c  src/query/scan_manager.c                   │
│   src/optimizer/  src/parser/                                            │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │ calls heap_next(), btree_range_search(),
                            │ catalog_get_representation(), stats_get_*()
┌───────────────────────────▼──────────────────────────────────────────────┐
│                    High-Level Storage Managers                            │
│                                                                          │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────────┐   │
│  │  heap_file.c    │  │    btree.c        │  │  system_catalog.c    │   │
│  │  (heap manager) │  │  (B-tree index)   │  │  catalog_class.c     │   │
│  │  27K lines      │  │  37K lines        │  │  (schema metadata)   │   │
│  └────────┬────────┘  └────────┬──────────┘  └──────────┬───────────┘  │
│           │                    │                          │              │
│  ┌────────▼────────┐  ┌────────▼──────────┐             │              │
│  │ overflow_file.c │  │  btree_load.c     │             │              │
│  │ (large records) │  │  (bulk load)      │             │              │
│  └────────┬────────┘  └────────────────────┘             │              │
│           │                                               │              │
│  ┌────────▼─────────────────────────────────────────────▼──────────┐   │
│  │              extendible_hash.c   external_sort.c                 │   │
│  │              statistics_sr.c     compactdb_sr.c                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │ spage_get_record(), spage_insert(), spage_update()
┌───────────────────────────▼──────────────────────────────────────────────┐
│                      Page-Level Layer                                     │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    slotted_page.c                                │    │
│  │          (slot directory, record insert/update/delete)           │    │
│  │                       5,291 lines                               │    │
│  └──────────────────────────────┬──────────────────────────────────┘    │
└─────────────────────────────────┼────────────────────────────────────────┘
                                  │ pgbuf_fix() / pgbuf_unfix()
┌─────────────────────────────────▼────────────────────────────────────────┐
│                      Buffer Pool Layer                                    │
│                                                                          │
│  ┌────────────────────────────────┐   ┌───────────────────────────────┐ │
│  │       page_buffer.c            │   │   double_write_buffer.cpp     │ │
│  │  (LRU, fix/unfix, latch mgmt)  │◄──│   (crash-safe write path)     │ │
│  │       16,931 lines             │   │       4,168 lines             │ │
│  └──────────────┬─────────────────┘   └───────────────────────────────┘ │
│                 │ tde_encrypt/decrypt_data_page()                        │
│  ┌──────────────▼─────────────────┐                                     │
│  │           tde.c                │                                     │
│  │  (Transparent Data Encryption)  │                                     │
│  └──────────────┬─────────────────┘                                     │
└─────────────────┼────────────────────────────────────────────────────────┘
                  │ fileio_read() / fileio_write()
┌─────────────────▼────────────────────────────────────────────────────────┐
│                       File / Disk I/O Layer                               │
│                                                                          │
│  ┌─────────────────────────┐   ┌──────────────────┐                     │
│  │      file_io.c          │   │  disk_manager.c   │                     │
│  │  (volume lifecycle,     │   │  (sector bitmap,  │                     │
│  │   page r/w, backup)     │   │   volume header)  │                     │
│  │     12,109 lines        │   │   6,828 lines     │                     │
│  └─────────────────────────┘   └──────────────────┘                     │
│                                                                          │
│  NOTE: file_manager.c (not yet analyzed) sits between page_buffer and   │
│  disk_manager: it manages file segments and maps sectors to pages.       │
└──────────────────────────────────────────────────────────────────────────┘

External Storage (LOB) — parallel path, bypasses buffer pool:
┌──────────────────────────────────────────────────────────────┐
│  es.c (dispatcher) → es_posix.c / es_owfs.c                  │
│  es_common.c (URI utilities, hash, unique ID)                 │
└──────────────────────────────────────────────────────────────┘

Foundation Types (included by everything):
┌──────────────────────────────────────────────────────────────┐
│  storage_common.h/c  oid.c/h  byte_order.h/c                  │
│  record_descriptor.hpp/cpp  btree_unique.cpp                  │
└──────────────────────────────────────────────────────────────┘
```

### Subsystem Grouping

```
B-tree indexing ────── btree.c, btree_load.c, btree_unique.cpp
Heap storage ──────── heap_file.c, overflow_file.c
Page management ───── page_buffer.c, slotted_page.c
File/Disk I/O ─────── file_io.c, disk_manager.c
Crash safety ─────── double_write_buffer.cpp
System catalog ────── system_catalog.c, catalog_class.c
Statistics ─────────── statistics_sr.c, statistics_cl.c
External storage ───── es.c, es_common.c, es_posix.c, es_owfs.c
Sorting ────────────── external_sort.c
Hashing ────────────── extendible_hash.c
Encryption ─────────── tde.c
Utilities ──────────── oid.c, storage_common.c, byte_order.c,
                       record_descriptor.cpp, compactdb_sr.c
```

---

## 3. File Inventory

### B-tree Indexing

| File | Lines (`.c`/`.cpp`) | Role | Layer | Report |
|------|---------------------|------|-------|--------|
| `btree.c` | 36,683 | B+-tree index manager: insert, delete, range scan, split/merge, recovery, statistics, online build | High-level | [btree.md](btree.md) |
| `btree_load.c` | 5,200 | B-tree bulk loader: sorted-pair stream → balanced B+-tree; node header management shared with `btree.c` | High-level | [btree_load.md](btree_load.md) |
| `btree_unique.cpp` | 243 | Deferred uniqueness stats: `btree_unique_stats` and `multi_index_unique_stats` classes; tracks (rows, keys, nulls) per index per batch | Foundation | [btree_unique.md](btree_unique.md) |

### Heap Storage

| File | Lines (`.c`) | Role | Layer | Report |
|------|--------------|------|-------|--------|
| `heap_file.c` | 26,759 | Heap file manager: record CRUD, MVCC header management, sequential scan, best-page heuristic, class repr cache, overflow coordination, WAL recovery | High-level | [heap_file.md](heap_file.md) |
| `overflow_file.c` | ~1,223 | Overflow record manager: singly-linked page chains for records too large for one page; insert/get/update/delete with WAL | High-level | [overflow_file.md](overflow_file.md) |

### Page Management

| File | Lines (`.c`) | Role | Layer | Report |
|------|--------------|------|-------|--------|
| `page_buffer.c` | 16,931 | Buffer pool: 3-zone LRU, fix/unfix/latch, dirty tracking, flush, DWB integration, TDE integration, ordered-fix, page quota | Buffer pool | [page_buffer.md](page_buffer.md) |
| `slotted_page.c` | 5,291 | Slotted page format: slot directory, record insert/update/delete/compact, saved-space for undo, scan helpers, vacuum integration | Page-level | [slotted_page.md](slotted_page.md) |

### File / Disk I/O

| File | Lines (`.c`/`.cpp`) | Role | Layer | Report |
|------|---------------------|------|-------|--------|
| `file_io.c` | 12,109 | Lowest-level disk I/O: volume lifecycle, page r/w, volume descriptor registry, file locking, backup/restore engine, flush-control token bucket, `fsync` | Disk I/O | [file_io.md](file_io.md) |
| `disk_manager.c` | 6,828 | Disk volume manager: sector-bitmap (STAB), volume header, sector reserve/unreserve, volume expansion, disk cache, consistency check | Disk I/O | [disk_manager.md](disk_manager.md) |
| `double_write_buffer.cpp` | 4,168 | Crash-safe write buffer: two-phase write (DWB file then data volume) to prevent torn-page corruption; background flush/sync daemon | Buffer pool | [double_write_buffer.md](double_write_buffer.md) |

### System Catalog

| File | Lines (`.c`) | Role | Layer | Report |
|------|--------------|------|-------|--------|
| `system_catalog.c` | 5,991 | Catalog manager: stores/retrieves `DISK_REPR`, `CLS_INFO`, `BTREE_STATS`; primary metadata store for query optimizer | High-level | [system_catalog.md](system_catalog.md) |
| `catalog_class.c` | 5,823 | Class meta-catalog: maintains system tables (`_db_class`, `_db_attribute`, `_db_index`, etc.) in sync with schema DDL; bridges OR format to relational catalog | High-level | [catalog_class.md](catalog_class.md) |

### Statistics

| File | Lines (`.c`) | Role | Layer | Report |
|------|--------------|------|-------|--------|
| `statistics_sr.c` | 1,496 | Server-side stats: collects NDV, heap counts, B-tree metrics on `UPDATE STATISTICS`; stores to catalog; serves serialized blobs to client | High-level | [statistics_sr.md](statistics_sr.md) |
| `statistics_cl.c` | 651 | Client-side stats: deserializes blobs from server into `CLASS_STATS`; provides live `COUNT(DISTINCT)` NDV path; consumed by query optimizer | Foundation (CS_MODE) | [statistics_cl.md](statistics_cl.md) |

### External Storage (LOB)

| File | Lines (`.c`) | Role | Layer | Report |
|------|--------------|------|-------|--------|
| `es.c` | 583 | External storage dispatcher: routes LOB operations (create/read/write/copy/rename/delete) to POSIX or OWFS backend by URI prefix | ES layer | [es.md](es.md) |
| `es_common.c` | 111 | LOB shared utilities: URI type detection, content-addressed filename hash, time-based unique number generator | ES foundation | [es_common.md](es_common.md) |
| `es_posix.c` | 964 | POSIX filesystem LOB backend: manages LOB files in a `file://` directory tree; thread-safe `rand_r` for filename generation | ES backend | [es_posix.md](es_posix.md) |
| `es_owfs.c` | 925 | OwFS (OneWriteFS) LOB backend: distributed append-only FS bridge; stub-only unless built with `CUBRID_OWFS` | ES backend | [es_owfs.md](es_owfs.md) |

### Sorting

| File | Lines (`.c`) | Role | Layer | Report |
|------|--------------|------|-------|--------|
| `external_sort.c` | 5,471 | External merge-sort: in-memory run generation + multi-way merge over temp files; duplicate elimination, LIMIT opt, parallel sort (`SERVER_MODE`), TDE temp file support | High-level | [external_sort.md](external_sort.md) |

### Hashing

| File | Lines (`.c`) | Role | Layer | Report |
|------|--------------|------|-------|--------|
| `extendible_hash.c` | 5,513 | On-disk extendible hash: dynamic `(key, OID)` hash with directory + bucket files; used for classname table, catalog cross-reference, and query temp hashes | High-level | [extendible_hash.md](extendible_hash.md) |

### Encryption

| File | Lines (`.c`) | Role | Layer | Report |
|------|--------------|------|-------|--------|
| `tde.c` | 1,720 | Transparent Data Encryption: AES-256-CTR / ARIA-256-CTR page encryption; two-level key hierarchy (master key in flat file, data keys in heap); OpenSSL backend | Buffer pool | [tde.md](tde.md) |

### Foundation / Utilities

| File | Lines (`.c`/`.cpp`) | Role | Layer | Report |
|------|---------------------|------|-------|--------|
| `storage_common.c` / `.h` | 378 / 1,240 | Foundational types: `VPID`, `OID`, `HFID`, `BTID`, `RECDES`, `OPERATOR_TYPE`, `MVCCID`, `XASL_ID`; page-size globals; included by ~131 files | Foundation | [storage_common.md](storage_common.md) |
| `oid.c` | 388 | OID operations: system-class OID cache, comparison/hash functions, RR sentinel, temporary OID machinery, C++ `std::hash` specialization | Foundation | [oid.md](oid.md) |
| `byte_order.c` / `.h` | 199 / 187 | Portable byte-order conversion: `htons/ntohs/htonl/ntohl` + 64-bit and float/double variants; `swap64` and `MOVING_VAN` union; consumed by `object_representation.h` OR macros | Foundation | [byte_order.md](byte_order.md) |
| `record_descriptor.cpp` | 360 | RAII C++ wrapper around `recdes`: ownership state machine, auto-resize on `S_DOESNT_FIT`, slotted-page I/O helpers, `cubpacking::packable_object` conformance | Foundation | [record_descriptor.md](record_descriptor.md) |
| `compactdb_sr.c` | 773 | `compactdb` utility server side: nullify dead OID references, rewrite stale representations, trigger `spage_compact()` per page, drop old catalog representations | High-level | [compactdb_sr.md](compactdb_sr.md) |

> **Note:** `file_manager.c` (file segment and page allocation manager, sits between `disk_manager` and `page_buffer`) is present in the source tree but was not included in the 26-report analysis set. It is referenced extensively by heap, B-tree, catalog, and sort modules via `file_manager.h`.

---

## 4. Dependency Matrix

The table below shows which storage files directly include / call which other storage files. An `X` means a direct `#include` (and therefore functional dependency). Read rows as "this file depends on column file."

```
                        storage  oid  byte  slotted  page   file   disk   heap   btree  ovflw  file   sys    cat    ext    ext    tde    dbl    btree  btree  stat   stat   es.*   ext    rec
                        _common      _order  _page   _buf   _io    _mgr   _file  .c     _file  _mgr   _cat   _cls   _sort  _hash        _wbuf  _load  _uniq  _sr    _cl           _sort  _desc
heap_file.c               X      X           X       X             X(tr)          X      X      X      X      X
btree.c                   X      X           X       X                            X             X      X(tr)         X                          X      X             X
btree_load.c              X      X           X       X                    X       X             X                                     X                              X
overflow_file.c           X              X   X       X             X(tr)   X                   X
slotted_page.c            X      X           X       X
page_buffer.c             X              X           X      X      X                                                               X             X                        X
file_io.c                 X                                        X                                                                                                                X(es)
disk_manager.c            X                          X(tr)  X      X
system_catalog.c          X              X   X       X             X(tr)   X      X(tr)         X             X                    X
catalog_class.c           X      X       X   X       X             X(tr)   X      X      X(tr)  X      X
external_sort.c           X                  X(tr)   X(tr)         X(tr)   X             X
extendible_hash.c         X              X   X       X                                          X(tr)
statistics_sr.c           X              X           X(tr)         X(tr)   X      X      X(tr)  X      X             X
statistics_cl.c           X              X                                                                     X
tde.c                     X                          X(tr)  X                     X
double_write_buffer.cpp   X                          X(tr)  X
compactdb_sr.c            X              X   X(tr)         X(tr)   X(tr)   X      X(tr)
es.c                      X                                                                                                               X(pos)  X(ow)
es_posix.c                X                          X(tr)
es_owfs.c                 X
es_common.c               X
btree_unique.cpp          X
record_descriptor.cpp     X                  X
```

Key: `X` = direct include, `X(tr)` = transitive/indirect include, `X(pos)` = es_posix backend, `X(ow)` = es_owfs backend.

### Most-depended-upon files (most frequently included by others in this module)

| File | Approximate dependents within storage/ |
|------|----------------------------------------|
| `storage_common.h` | ~all 26 files |
| `slotted_page.h` | heap_file, btree, overflow_file, system_catalog, extendible_hash, external_sort, compactdb_sr |
| `page_buffer.h` | heap_file, btree, overflow_file, slotted_page, file_io, disk_manager, extendible_hash, external_sort, tde, system_catalog |
| `heap_file.h` | btree_load, catalog_class, system_catalog, statistics_sr, tde, compactdb_sr |
| `file_manager.h` | heap_file, btree, btree_load, system_catalog, external_sort (all need page allocation) |

---

## 5. Key Cross-File Call Graphs

### 5.1 Page Lifecycle: Allocation → Buffer Pool → Slotted Page → Heap / B-tree

```
DML (INSERT row)
  heap_file.c : heap_insert_logical()
    → file_manager.c : file_alloc()          -- get a VPID for the new/existing page
        → disk_manager.c : disk_reserve_sectors()  -- reserve sectors if needed
    → page_buffer.c : pgbuf_fix(vpid, NEW_PAGE, PGBUF_LATCH_WRITE)
        → file_io.c : fileio_read()          -- load page from disk if not cached
        [page pinned in BCB, latch held]
    → slotted_page.c : spage_insert(page, &recdes, &slotid)
        -- finds free space, writes record, updates slot directory
    → log_append.hpp : log_append_undoredo_recdes2()
        -- WAL: write undo+redo log record BEFORE marking dirty
    → page_buffer.c : pgbuf_set_dirty(page, DONT_FREE)
    → page_buffer.c : pgbuf_unfix(page)
        -- page remains in buffer pool, may be flushed later
```

### 5.2 Record Lifecycle: Insert → Heap → Slotted Page → Overflow (if needed)

```
heap_insert_logical()
  ├── [record fits in one page]
  │     → spage_insert()         -- direct in-page record
  │
  └── [record too large for one page]
        → overflow_file.c : overflow_insert(thread_p, ovf_vfid, recdes)
              → file_manager.c : file_alloc_multiple()   -- N overflow pages
              → page_buffer.c : pgbuf_fix() × N          -- fix each page
              → (copy data across linked chain, set next_vpid in each header)
              → log_append_undo_data() per page           -- WAL each page
              → pgbuf_set_dirty() + pgbuf_unfix() × N
          (heap record stores first-overflow VPID as the data pointer)
```

### 5.3 Index Lifecycle: Create → btree_load → btree Operations

```
CREATE INDEX (xbtree_load_index):
  btree_load.c : btree_load_index()
    → heap_file.c : heap_next()             -- sequential heap scan
    → external_sort.c : sort_listfile()     -- sort (key, OID) pairs
    → btree_load.c : btree_construct_indexes()
          → file_manager.c : file_create()  -- allocate B-tree file
          → page_buffer.c : pgbuf_fix(NEW_PAGE) × many
          → slotted_page.c : spage_insert() -- insert leaf entries
          → log_append_redo_data()          -- redo-only (bulk load, no undo)
          → pgbuf_set_dirty() + pgbuf_unfix()

Runtime index operation (btree_insert_mvcc_delid_into_page):
  btree.c : btree_insert()
    → page_buffer.c : pgbuf_fix(root)       -- fix root
    → btree.c : btree_split_node() (if needed)
          → page_buffer.c : pgbuf_fix(new_page, NEW_PAGE)
          → slotted_page.c : spage_insert() -- new node entries
          → log_append_undoredo_data2()     -- WAL split
    → slotted_page.c : spage_insert/update()
    → pgbuf_set_dirty() + pgbuf_unfix()
```

### 5.4 Transaction Integration: Storage ↔ WAL / Logging

```
Every page modification follows this protocol:

  storage_module.c
    1. pgbuf_fix(vpid, PGBUF_LATCH_WRITE)     -- acquire page
    2. [modify in-memory page contents]
    3. log_append_undoredo_data2(...)          -- WAL: log BEFORE marking dirty
         (log_append.hpp → log_manager.c → log_page_buffer.c)
    4. pgbuf_set_dirty(page, DONT_FREE)        -- mark page dirty
    5. pgbuf_unfix(page)                       -- release latch

Flush path (checkpoint / bgflush daemon):
  page_buffer.c : pgbuf_flush_victim_candidate()
    → double_write_buffer.cpp : dwb_add_page()  -- write to DWB file first
    → file_io.c : fileio_write()                -- write to data volume
    → (DWB slot freed)

Recovery (crash restart):
  boot_sr.c : boot_restart_server()
    → double_write_buffer.cpp : dwb_load_and_recover_pages()
         -- restore torn pages from DWB before log replay
    → log_recovery.c : log_recovery()
         -- replay redo records to bring pages forward
         -- apply undo records to roll back incomplete transactions
```

### 5.5 Statistics Flow: Collection → Storage → Optimizer

```
UPDATE STATISTICS ON t:
  statistics_sr.c : xstats_update_statistics()
    → heap_file.c : heap_get_class_name()
    → btree.c : btree_get_stats()              -- per-index metrics
    → system_catalog.c : catalog_update_class_statistics()
         → slotted_page.c : spage_update()     -- write to catalog pages
         → log_append_undoredo_recdes2()        -- WAL

Query compilation (client side):
  schema_manager.c : sm_get_class_with_statistics()
    → statistics_cl.c : stats_get_statistics()
         → network_interface_cl.c : stats_get_statistics_from_server()
              [RPC → statistics_sr.c : xstats_get_statistics_from_server()]
    → optimizer/query_graph.c : qo_estimate_statistics()
         -- uses CLASS_STATS.ATTR_STATS[i].ndv, BTREE_STATS[j].keys, etc.
```

---

## 6. Core Protocols

### 6.1 Buffer Pool Protocol (pgbuf_fix / pgbuf_unfix)

Every access to any database page goes through the buffer pool:

```c
/* Fix (pin) a page — MUST unfix when done */
PAGE_PTR page = pgbuf_fix (thread_p, &vpid, OLD_PAGE,
                            PGBUF_LATCH_READ,   /* or PGBUF_LATCH_WRITE */
                            PGBUF_UNCONDITIONAL_LATCH);
if (page == NULL) { /* error */ }

/* ... use page ... */

/* If modified: */
pgbuf_set_dirty (thread_p, page, DONT_FREE);

/* Release latch and allow eviction */
pgbuf_unfix (thread_p, page);
/* page pointer is now invalid — never use after unfix */
```

Rules:
- Every `pgbuf_fix()` call has exactly one matching `pgbuf_unfix()` call. Verified at runtime by `resource_tracker` in debug builds.
- Latch modes: `PGBUF_LATCH_READ` (shared, concurrent reads OK), `PGBUF_LATCH_WRITE` (exclusive, no concurrent access).
- Fix modes for new pages: `NEW_PAGE` (formats the page in-place, no disk read), `OLD_PAGE` (loads from disk or cache), `OLD_PAGE_PREVENT_DEALLOC` (used during deallocation undo).
- `pgbuf_ordered_fix()` must be used for heap pages to enforce latch ordering and avoid deadlocks.

### 6.2 WAL Protocol (Log Before Write)

CUBRID implements strict Write-Ahead Logging:

```c
/* Canonical pattern in any page modification function: */

/* Step 1: modify the page in memory */
spage_insert (thread_p, page, &recdes, &slotid);

/* Step 2: append log record BEFORE marking the page dirty */
log_append_undoredo_recdes2 (thread_p, RVHF_INSERT,
                              &hfid->vfid, page,
                              NULL,    /* undo data (none for insert) */
                              &recdes  /* redo data */);

/* Step 3: mark dirty (will be flushed later) */
pgbuf_set_dirty (thread_p, page, DONT_FREE);
```

The invariant is: **a page can never reach disk with changes that have no log record**. The double-write buffer reinforces this by ensuring pages are written completely (no torn writes) even on OS crash.

### 6.3 Latch Ordering Rules

To prevent latch-order deadlocks, the engine enforces a strict acquisition order:

```
Disk/file level (no latch):  disk_manager, file_manager
  ↓
Buffer pool latch order (must acquire in this order when holding multiple):
  1. Root page of a B-tree (if splitting)
  2. Parent B-tree node
  3. Child B-tree node (leaf)
  4. Overflow page (of the leaf's key)

For heap files:
  1. Header page (HFID.hpgid) — acquired with pgbuf_ordered_fix()
  2. Data pages in VPID order (volid, pageid ascending)
  3. Overflow pages last

Cross-subsystem rule:
  - Never hold a heap page latch while acquiring a log buffer latch.
  - Never hold a buffer pool mutex while calling disk_manager.
```

Violation of latch order causes deadlock. `pgbuf_ordered_fix()` implements automatic ordering for heap header pages.

### 6.4 MVCC Visibility in Storage

CUBRID uses multi-version concurrency control. Each heap record has an embedded MVCC header:

```
MVCC header fields (in RECDES data prefix):
  - insert_id  (MVCCID): transaction that inserted this version
  - delete_id  (MVCCID): transaction that deleted this version (0 = live)
  - prev_version_lsa (LOG_LSA): LSA of previous version in undo log
```

The visibility check pattern:

```c
/* In heap_file.c: heap_get_visible_version() */
MVCC_REC_HEADER mvcc_header;
heap_get_mvcc_rec_header_from_overflow (thread_p, page, recdes, &mvcc_header);

if (mvcc_satisfies_snapshot (thread_p, &mvcc_header, snapshot)) {
    /* record is visible to this transaction */
} else if (MVCC_IS_HEADER_DELID_VALID (&mvcc_header)) {
    /* record deleted — follow prev_version_lsa for older version */
}
```

Key functions:
- `mvcc_satisfies_snapshot()` — in `src/transaction/mvcc.c`; checks `insert_id < snapshot.lowest_active_mvccid` and `delete_id` not in snapshot.
- `HEAP_UPDATE_IS_MVCC_OP` macro — controls whether heap updates create new MVCC versions (always true in `SERVER_MODE`, always false in `SA_MODE`).
- `heap_get_visible_version()` — heap's entry point for MVCC-aware record reads.
- Vacuum (`src/query/vacuum.c`) — background process that physically removes dead MVCC versions by calling `spage_vacuum_slot()`.

### 6.5 Unique Index Enforcement (btree_unique)

Uniqueness is checked **deferred** at statement end, not per-row:

```cpp
/* During DML batch: accumulate stats */
btree_unique_stats stats;
stats.add_row(+1);   /* increment for insert */
stats.add_key(+1);

/* At statement end (locator_sr.c): */
if (!stats.is_unique()) {
    /* rows != keys + nulls => uniqueness violation */
    er_set(ER_ERROR_SEVERITY, ..., ER_UNIQUE_VIOLATION, ...);
}
```

This allows a statement like `INSERT INTO t VALUES (1),(1)` to fail atomically rather than partially.

---

## 7. Module Statistics

### Lines of Code by File

| File | Source lines | Header lines | Language |
|------|-------------|--------------|----------|
| `btree.c` | 36,683 | 926 | C (→C++17) |
| `heap_file.c` | 26,759 | 724 | C (→C++17) |
| `page_buffer.c` | 16,931 | 499 | C (→C++17) |
| `file_io.c` | 12,109 | 637 | C (→C++17) |
| `disk_manager.c` | 6,828 | 159 | C (→C++17) |
| `system_catalog.c` | 5,991 | 208 | C (→C++17) |
| `catalog_class.c` | 5,823 | 50 | C (→C++17) |
| `extendible_hash.c` | 5,513 | 59 | C (→C++17) |
| `external_sort.c` | 5,471 | 165 | C (→C++17) |
| `slotted_page.c` | 5,291 | 177 | C (→C++17) |
| `btree_load.c` | 5,200 | 328 | C (→C++17) |
| `double_write_buffer.cpp` | 4,168 | (hpp) | C++17 |
| `tde.c` | 1,720 | 214 | C (→C++17) |
| `statistics_sr.c` | 1,496 | — | C (→C++17) |
| `overflow_file.c` | ~1,223 | 77 | C (→C++17) |
| `es_owfs.c` | 925 | 40 | C (→C++17) |
| `es_posix.c` | 964 | 59 | C (→C++17) |
| `compactdb_sr.c` | 773 | — | C (→C++17) |
| `statistics_cl.c` | 651 | 153 | C (→C++17) |
| `es.c` | 583 | 52 | C (→C++17) |
| `storage_common.c` | 378 | 1,240 | C (→C++17) |
| `oid.c` | 388 | 239 | C (→C++17) |
| `record_descriptor.cpp` | 360 | 181 | C++17 |
| `btree_unique.cpp` | 243 | 109 | C++17 |
| `byte_order.c` | 199 | 187 | C (→C++17) |
| `es_common.c` | 111 | 53 | C (→C++17) |
| **Total** | **~148,783** | | |

### Summary Metrics

| Metric | Value |
|--------|-------|
| Total files analyzed | 26 (of ~28 in the directory) |
| Total source lines | ~148,800 |
| Language breakdown | 23 C files compiled as C++17; 3 native C++ files (`double_write_buffer.cpp`, `record_descriptor.cpp`, `btree_unique.cpp`) |
| Largest file | `btree.c` at 36,683 lines |
| Smallest file | `es_common.c` at 111 lines |
| Files server-only (`SERVER_MODE` + `SA_MODE`) | 18 |
| Files all-mode (`CS_MODE` too) | 8 (`storage_common`, `oid`, `byte_order`, `record_descriptor`, `btree_unique`, `statistics_cl`, `es*`) |
| Files with C++ class definitions | `btree_unique.cpp`, `record_descriptor.cpp`, `double_write_buffer.cpp` |
| Not-yet-analyzed file | `file_manager.c` (no report in `reports/storage/`) |

---

## 8. Navigation Guide

### "I want to fix X" → which files to look at

| Task | Primary file | Secondary files |
|------|-------------|-----------------|
| Fix buffer pool bug (page not flushed, BCB corruption) | `page_buffer.c` | `double_write_buffer.cpp`, `file_io.c` |
| Fix torn-page / crash recovery issue | `double_write_buffer.cpp` | `page_buffer.c`, `file_io.c`, `disk_manager.c` |
| Fix heap scan returning wrong rows | `heap_file.c` | `slotted_page.c`, `overflow_file.c`, `mvcc.c` |
| Fix heap insert / update / delete | `heap_file.c` | `slotted_page.c`, `overflow_file.c`, `btree.c` (index maintenance) |
| Fix index scan wrong results | `btree.c` | `slotted_page.c`, `page_buffer.c` |
| Fix index unique constraint violation | `btree_unique.cpp` | `btree.c`, `heap_file.c`, `log_tran_table.c` |
| Fix CREATE INDEX performance / correctness | `btree_load.c` | `external_sort.c`, `heap_file.c`, `btree.c` |
| Fix page slot corruption | `slotted_page.c` | `heap_file.c`, `btree.c` |
| Fix disk space not reclaimed | `disk_manager.c` | `file_manager.c`, `heap_file.c` |
| Fix overflow record corruption | `overflow_file.c` | `heap_file.c`, `slotted_page.c` |
| Fix catalog not reflecting schema | `catalog_class.c` | `system_catalog.c`, `heap_file.c` |
| Fix optimizer using wrong cardinality | `statistics_sr.c` | `statistics_cl.c`, `btree.c`, `system_catalog.c` |
| Fix LOB (BLOB/CLOB) read/write | `es.c` → `es_posix.c` | `es_common.c`, `elo.c` (object layer) |
| Fix backup/restore | `file_io.c` | `page_buffer.c`, `double_write_buffer.cpp` |
| Fix TDE encryption/decryption | `tde.c` | `page_buffer.c`, `file_io.c` |
| Fix ORDER BY / GROUP BY sort overflow | `external_sort.c` | `file_manager.c`, `overflow_file.c` |
| Fix compactdb utility | `compactdb_sr.c` | `heap_file.c`, `slotted_page.c`, `system_catalog.c` |
| Add a new page type | `storage_common.h` | `slotted_page.c`, `page_buffer.c`, `file_manager.c` |

### Common Debugging Starting Points

| Symptom | Entry point function | File |
|---------|---------------------|------|
| Page I/O failure | `fileio_read()` / `fileio_write()` | `file_io.c` |
| Buffer pool latch deadlock | `pgbuf_fix()` latch acquisition | `page_buffer.c` |
| Heap record not found | `heap_get_visible_version()` | `heap_file.c` |
| Index key not found | `btree_find_key()` | `btree.c` |
| Unique constraint spurious error | `btree_unique_stats::is_unique()` | `btree_unique.cpp` |
| Catalog data stale | `catalog_get_representation()` | `system_catalog.c` |
| Statistics not updated | `xstats_update_statistics()` | `statistics_sr.c` |
| LOB file not created | `es_posix_write_file()` | `es_posix.c` |
| Sort running out of temp space | `sort_run_flush()` | `external_sort.c` |
| Disk full error | `disk_reserve_sectors()` | `disk_manager.c` |
| TDE page decrypt failure | `tde_decrypt_data_page()` | `tde.c` |

### Key Entry Point Functions by Subsystem

| Subsystem | Entry points |
|-----------|-------------|
| **Buffer pool** | `pgbuf_fix()`, `pgbuf_unfix()`, `pgbuf_set_dirty()`, `pgbuf_ordered_fix()`, `pgbuf_flush_with_wal()` |
| **Heap** | `heap_insert_logical()`, `heap_update_logical()`, `heap_delete_logical()`, `heap_next()`, `heap_get_visible_version()` |
| **B-tree** | `btree_insert()`, `btree_delete()`, `btree_find_key()`, `btree_range_search()`, `xbtree_find_unique()` |
| **B-tree bulk load** | `btree_load_index()`, `btree_construct_indexes()` |
| **Slotted page** | `spage_insert()`, `spage_get_record()`, `spage_update()`, `spage_delete()`, `spage_compact()` |
| **Overflow** | `overflow_insert()`, `overflow_get()`, `overflow_update()`, `overflow_delete()` |
| **File I/O** | `fileio_read()`, `fileio_write()`, `fileio_format()`, `fileio_open()`, `fileio_mount()` |
| **Disk manager** | `disk_reserve_sectors()`, `disk_unreserve_ordered_sectors()`, `disk_format()` |
| **DWB** | `dwb_add_page()`, `dwb_flush_force()`, `dwb_load_and_recover_pages()` |
| **System catalog** | `catalog_get_representation()`, `catalog_get_class_info()`, `catalog_insert()`, `catalog_update()` |
| **Class catalog** | `catcls_insert_instance()`, `catcls_update_instance()`, `catcls_delete_instance()` |
| **Statistics (server)** | `xstats_update_statistics()`, `xstats_get_statistics_from_server()` |
| **Statistics (client)** | `stats_get_statistics()`, `stats_dump()` |
| **External storage** | `es_create_file()`, `es_write_file()`, `es_read_file()`, `es_delete_file()` |
| **External sort** | `sort_listfile()` |
| **Extendible hash** | `ehash_create()`, `ehash_search()`, `ehash_insert()`, `ehash_delete()` |
| **TDE** | `tde_initialize()`, `tde_encrypt_data_page()`, `tde_decrypt_data_page()` |
| **OID** | `oid_compare()`, `oid_hash()`, `oid_is_system_class()` |
| **Compactdb** | `boot_compact_db()`, `boot_heap_compact_pages()` |

---

*End of index. For deep-dive information on any file, see the corresponding per-file report linked in Section 3.*
