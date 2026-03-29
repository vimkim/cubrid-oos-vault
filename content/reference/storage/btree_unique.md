# CUBRID B-Tree Unique Constraint — Comprehensive Analysis Report

**Source files analyzed:**
- `/home/vimkim/gh/cb/develop/src/storage/btree_unique.cpp` — 243 lines
- `/home/vimkim/gh/cb/develop/src/storage/btree_unique.hpp` — 109 lines

**Generated:** 2026-03-27
**Tooling used:** clangd LSP (lsp_document_symbols, lsp_find_references, lsp_hover), direct source reads, Grep cross-reference

---

## 1. File Overview

| Property | Value |
|----------|-------|
| Implementation file | `src/storage/btree_unique.cpp` |
| Header file | `src/storage/btree_unique.hpp` |
| Total lines (combined) | 352 (243 + 109) |
| Language standard | C++17 (compiled as C++ via `c_to_cpp.sh`) |
| License | Apache 2.0 (Search Solution Corp / CUBRID Corp) |
| Compile-time guards | None — no `SERVER_MODE`/`SA_MODE`/`CS_MODE` guards; the types are used server-side only in practice |
| Build modes | Included in all three binary targets (server, standalone, client-side library) via the storage module |
| Memory wrapper | Explicitly excluded (`#if 0` guard around `memory_wrapper.hpp`) due to `placement new` usage in `construct()` |

**Purpose:**

This module defines two tightly coupled statistics-tracking classes that support CUBRID's unique constraint enforcement for B-tree indexes:

1. `btree_unique_stats` — a lightweight counter triplet `(rows, keys, nulls)` for a single B-tree index. Used to accumulate deltas as rows are inserted or deleted during a DML statement, and to detect uniqueness violations by asserting `rows == keys + nulls`.

2. `multi_index_unique_stats` — a `std::map<BTID, btree_unique_stats>` that tracks stats across all unique indexes touched during a multi-row DML operation (bulk insert, bulk update, bulk delete). Owned by the transaction descriptor (`LOG_TDES::m_multiupd_stats`) and by the heap scan cache (`HEAP_SCANCACHE::m_index_stats`).

The uniqueness invariant is: after all row mutations of a batch complete, every key in every unique index must appear exactly once. Null values are exempt from the uniqueness check (SQL standard: NULLs are not equal to each other). The invariant expressed in code is:

```cpp
bool is_unique() const { return m_rows == m_keys + m_nulls; }
```

This is deferred enforcement — individual intermediate states within a multi-row transaction may be temporarily non-unique; the check fires at statement end.

---

## 2. Includes & Dependencies

### Header file (`btree_unique.hpp`) includes

| Include | Type | Provides |
|---------|------|----------|
| `storage_common.h` | Internal (storage module) | `BTID`, `VFID`, `HFID`, `NULL_PAGEID`, `BTID_IS_NULL()` macro |
| `<cstdint>` | System C++ | `std::int64_t` for `stat_type` alias |
| `<map>` | System C++ | `std::map` backing `container_type` |
| `class string_buffer` (forward decl) | Internal (base module) | String formatting; avoids including full header |

### Implementation file (`btree_unique.cpp`) includes

| Include | Type | Provides |
|---------|------|----------|
| `btree_unique.hpp` | Own header | Class definitions (always first own-header rule) |
| `string_buffer.hpp` | Internal (base module) | `string_buffer` class — variadic printf-style buffer; used in `to_string()` |
| `<utility>` | System C++ | `std::move` used in move-assignment operator |
| `memory_wrapper.hpp` | Internal (disabled with `#if 0`) | Heap memory monitoring — excluded because of placement new |

### Reverse dependencies (files that include `btree_unique.hpp`)

| File | Role |
|------|------|
| `src/storage/btree.h` | Forward-declares `btree_unique_stats`; embeds pointer in `BTREE_INSERT_HELPER` and `BTREE_DELETE_HELPER` structs, and in function signatures |
| `src/storage/btree.c` | Primary consumer — accumulates stats during insert/delete operations |
| `src/storage/heap_file.h` | Embeds `multi_index_unique_stats *m_index_stats` in `HEAP_SCANCACHE` |
| `src/storage/heap_file.c` | Creates/destroys `multi_index_unique_stats` via `new`/`delete`; calls `add_empty()` |
| `src/transaction/log_impl.h` | Embeds `multi_index_unique_stats m_multiupd_stats` directly in `LOG_TDES` |
| `src/transaction/log_tran_table.c` | Calls `construct()`, `clear()`, and overloaded `logtb_tran_update_unique_stats()` |
| `src/transaction/locator_sr.c` | End-of-batch violation check loop, stats accumulation from scan caches |
| `src/query/query_executor.c` | Embeds `m_unique_stats` in `UPDDEL_CLASS_INFO_INTERNAL`; drives uniqueness check |
| `src/loaddb/load_server_loader.cpp` | Bulk load path — checks uniqueness of each index after loading |

---

## 3. Preprocessor & Compilation

### Memory wrapper exclusion (lines 28–39, `.cpp`)

```cpp
#if 0
// ... placement new usage prevents memory monitoring
#include "memory_wrapper.hpp"
#endif
```

The `construct()` method uses placement new (`new (this) multi_index_unique_stats()`) directly on the object's memory. CUBRID's `memory_wrapper.hpp` overrides `operator new` globally; placement new bypasses the normal allocation path, so tracking would miss the allocation. The developer comment acknowledges this and treats the file as a permanent exception from heap monitoring until placement new is removed.

### No build-mode guards

Unlike most CUBRID storage/transaction code, `btree_unique.hpp` contains no `#if defined(SERVER_MODE)` guards. The types are compile-time available in all modes. At runtime, `multi_index_unique_stats` is only populated on the server side.

---

## 4. Data Structures & Types

### 4.1 `btree_unique_stats` (class, `btree_unique.hpp` lines 34–67)

A value-semantic statistics counter for one B-tree unique index. Holds signed 64-bit integers (can go negative transiently during rollback/undo paths).

#### Type alias

```cpp
using stat_type = std::int64_t;
```

Resolved by LSP as `long` on Linux/x86-64. 64-bit so that large tables (> 2^31 rows) are supported without overflow.

#### Private fields

| Field | Type | Semantic meaning |
|-------|------|-----------------|
| `m_rows` | `stat_type` (`int64_t`) | Total number of OIDs (object identifiers) in the index — both key-bearing and null. Invariant: `m_rows == m_keys + m_nulls` when unique. |
| `m_keys` | `stat_type` | Count of non-null key entries. In CUBRID's unique index model, each distinct key should map to exactly one OID. |
| `m_nulls` | `stat_type` | Count of null-key entries. NULL values are exempt from uniqueness checks per SQL standard; each NULL gets its own index entry. |

The comment `// m_rows = m_keys + m_nulls` (line 63 of header) documents the invariant explicitly. After a multi-row operation, if `m_rows != m_keys + m_nulls`, there is a duplicate non-null key, and `ER_BTREE_UNIQUE_FAILED` is raised.

**Sign semantics:** Values may be negative during intermediate states (e.g., if keys are deleted before all inserts are counted). The final check happens only after all row mutations complete.

### 4.2 `multi_index_unique_stats` (class, `btree_unique.hpp` lines 69–107)

A container that maps each `BTID` (B-tree identifier) to a `btree_unique_stats` delta. Used to track all unique index changes in one multi-row DML statement.

#### Inner struct: `btid_comparator`

```cpp
struct btid_comparator {
    bool operator()(const BTID &a, const BTID &b) const {
        return a.root_pageid < b.root_pageid ||
               (a.root_pageid == b.root_pageid && a.vfid.volid < b.vfid.volid);
    }
};
```

A strict weak ordering comparator for `BTID` values in `std::map`. Compares by `root_pageid` first, then by `vfid.volid`. Note: `vfid.fileid` is not used in the comparison — two BTIDs with the same `root_pageid` and `volid` but different `fileid` would be treated as equal. In practice, `root_pageid` is unique per index in the volume namespace, so this is safe.

#### Type alias

```cpp
using container_type = std::map<BTID, btree_unique_stats, btid_comparator>;
```

LSP hover confirms the expanded type: `class std::map<btid, btree_unique_stats, multi_index_unique_stats::btid_comparator>`.

#### Private field

| Field | Type | Semantic meaning |
|-------|------|-----------------|
| `m_stats_map` | `container_type` | Maps each touched BTID to its accumulated `(rows, keys, nulls)` delta. Initialized with empty entries for all indexes of a class at scan-cache setup time. |

#### Copy-assignment policy

The copy-assignment operator `operator=(const multi_index_unique_stats &)` is explicitly `= delete`. This is a deliberate design choice: copying the entire map is expensive, and the API instead provides:
- `copy_to(dest)` — explicit deep copy (used in transaction descriptor cloning)
- `operator=(multi_index_unique_stats &&)` — move assignment (cheap, transfers map ownership)
- `operator+=(const multi_index_unique_stats &)` — accumulation (merge another map's stats)

### 4.3 `BTID` (from `storage_common.h`)

```c
typedef struct btid BTID;
struct btid {
    VFID vfid;        // volume+file ID (volid + fileid)
    INT32 root_pageid; // root page of the B-tree
};
```

The primary key for indexing into `m_stats_map`. `BTID_IS_NULL(btid)` checks `vfid.fileid == NULL_FILEID`.

---

## 5. Global & Static Variables

**None.** Neither file defines any global or static variables. Both classes are entirely instance-based. This makes the module trivially thread-safe at the class-definition level — safety depends entirely on how instances are owned (see Section 8).

---

## 6. Function/Method Catalog

### 6.1 `btree_unique_stats` — Constructors

---

#### `btree_unique_stats(stat_type keys, stat_type nulls = 0)` (lines 41–46)

**Signature:**
```cpp
btree_unique_stats::btree_unique_stats(stat_type keys, stat_type nulls = 0)
    : m_rows(keys + nulls), m_keys(keys), m_nulls(nulls)
```

**Description:** Primary constructor. Sets all three counters. `m_rows` is derived as `keys + nulls`, enforcing the invariant from construction. The `nulls` parameter defaults to 0.

**Algorithm:** Member-initializer list only. O(1).

**Error handling:** None. No invalid state possible — all `int64_t` values are valid.

**Callers:** Used in `btree.c` when creating a temporary `incr` delta object (e.g., line 27253), and in `add_empty()` to zero-initialize a map slot.

---

#### `btree_unique_stats()` (lines 48–51)

**Signature:**
```cpp
btree_unique_stats::btree_unique_stats()
    : btree_unique_stats(0, 0)
```

**Description:** Default constructor. Delegates to the primary constructor with all-zeros. Produces a zeroed stats object: `m_rows=0, m_keys=0, m_nulls=0`.

**Algorithm:** Delegating constructor — O(1).

**Callers:** `add_empty()` (line 178), and implicitly when declaring local `btree_unique_stats incr;` before incrementing.

---

### 6.2 `btree_unique_stats` — Accessors (getters)

---

#### `stat_type get_key_count() const` (lines 53–57)

**Signature:** `btree_unique_stats::stat_type btree_unique_stats::get_key_count() const`

**Description:** Returns `m_keys` — the count of non-null index entries. Used by `logtb_tran_update_unique_stats()` in `log_tran_table.c` (line 3635) to extract the key delta for the old C-style API that takes separate `n_keys`, `n_oids`, `n_nulls` parameters.

**Algorithm:** Trivial getter. O(1), no branches.

---

#### `stat_type get_row_count() const` (lines 59–63)

**Signature:** `btree_unique_stats::stat_type btree_unique_stats::get_row_count() const`

**Description:** Returns `m_rows` — total OID count (keys + nulls). Used in the same bridge function alongside `get_key_count()` and `get_null_count()`.

**Algorithm:** Trivial getter. O(1).

---

#### `stat_type get_null_count() const` (lines 65–69)

**Signature:** `btree_unique_stats::stat_type btree_unique_stats::get_null_count() const`

**Description:** Returns `m_nulls` — count of null-key entries. Used in the C-bridge call.

**Algorithm:** Trivial getter. O(1).

---

### 6.3 `btree_unique_stats` — Mutators

---

#### `void insert_key_and_row()` (lines 71–76)

**Signature:** `void btree_unique_stats::insert_key_and_row()`

**Description:** Increments both `m_keys` and `m_rows` by 1. Called when a non-null key is physically inserted into a unique index. Maintains `m_rows == m_keys + m_nulls` invariant.

**Algorithm:**
```cpp
++m_keys;
++m_rows;
```
O(1). No null-key side effect.

**Callers:** `btree.c:27277` — inside `btree_key_append_object_unique()` when `!insert_helper->is_null` and the operation is an insert (not MVCC delete).

---

#### `void insert_null_and_row()` (lines 78–83)

**Signature:** `void btree_unique_stats::insert_null_and_row()`

**Description:** Increments both `m_nulls` and `m_rows` by 1. Called when a null-key entry is inserted into a unique index. NULL values do not violate uniqueness — they are counted separately.

**Algorithm:**
```cpp
++m_nulls;
++m_rows;
```
O(1).

**Callers:** `btree.c:27273` — same call site as above but when `insert_helper->is_null` is true.

---

#### `void add_row()` (lines 85–89)

**Signature:** `void btree_unique_stats::add_row()`

**Description:** Increments only `m_rows` by 1 without touching `m_keys` or `m_nulls`. This temporarily breaks the `m_rows == m_keys + m_nulls` invariant — it is used when a duplicate OID is added to an existing key slot (multi-version scenario where more than one visible OID maps to the same key). The increment to `m_rows` captures that a duplicate exists; when is_unique() is later called, the inequality will signal a violation.

**Algorithm:**
```cpp
++m_rows;
```
O(1). The asymmetry with `m_keys`/`m_nulls` is intentional.

**Callers:** Not directly called from within btree_unique.cpp. Referenced in `btree.c` for the case where an additional OID is appended to an existing unique key slot (overcounting rows to force the is_unique() check to fail).

---

#### `void delete_key_and_row()` (lines 91–96)

**Signature:** `void btree_unique_stats::delete_key_and_row()`

**Description:** Decrements both `m_keys` and `m_rows` by 1. Called when a non-null key entry is physically removed from a unique index.

**Algorithm:**
```cpp
--m_keys;
--m_rows;
```
O(1). Values may go negative during undo/rollback — this is expected.

**Callers:** `btree.c:27265` (insert path, MVCC delete purpose), `btree.c:30838` (delete path).

---

#### `void delete_null_and_row()` (lines 98–103)

**Signature:** `void btree_unique_stats::delete_null_and_row()`

**Description:** Decrements both `m_nulls` and `m_rows` by 1. Called when a null-key entry is removed.

**Algorithm:**
```cpp
--m_nulls;
--m_rows;
```
O(1).

**Callers:** `btree.c:27261` (MVCC delete of null key), `btree.c:30834` (physical delete of null key).

---

#### `void delete_row()` (lines 105–109)

**Signature:** `void btree_unique_stats::delete_row()`

**Description:** Decrements only `m_rows` by 1. The counterpart to `add_row()` — used when a duplicate OID is removed from a key slot that already has other OIDs. Does not decrement `m_keys` because the key itself still exists.

**Algorithm:**
```cpp
--m_rows;
```
O(1).

---

### 6.4 `btree_unique_stats` — Query methods

---

#### `bool is_zero() const` (lines 111–115)

**Signature:** `bool btree_unique_stats::is_zero() const`

**Description:** Returns true if both `m_keys == 0` and `m_nulls == 0`. Note: does NOT check `m_rows`. Used as a short-circuit optimization in `logtb_tran_update_unique_stats()` (log_tran_table.c:3631) — if no net change occurred, the function returns early without touching the transaction's log or stats hash table.

**Algorithm:**
```cpp
return m_keys == 0 && m_nulls == 0;
```
O(1). Intentionally ignores `m_rows` — if only duplicate OIDs were added/removed (add_row/delete_row) but no key-level change happened, the caller still needs to re-examine.

**Callers:** `log_tran_table.c:3631` — early-exit optimization in the C++ overload of `logtb_tran_update_unique_stats`.

---

#### `bool is_unique() const` (lines 117–121)

**Signature:** `bool btree_unique_stats::is_unique() const`

**Description:** The core uniqueness predicate. Returns true iff `m_rows == m_keys + m_nulls`. This holds when every non-null key has exactly one OID (m_keys OIDs in key slots) plus m_nulls null OIDs. If any non-null key has two or more visible OIDs, `m_rows > m_keys + m_nulls`, and this returns false.

**Algorithm:**
```cpp
return m_rows == m_keys + m_nulls;
```
O(1). This is the single expression that determines whether a unique constraint violation exists.

**Callers (LSP find-references confirms 3 call sites):**

| Call site | Context |
|-----------|---------|
| `locator_sr.c:6688` | End of multi-object update — iterates `m_multiupd_stats.get_map()`, checks each index |
| `query_executor.c:10847` | `qexec_process_unique_stats()` — end-of-statement check for UPDATE/DELETE |
| `query_executor.c:10897` | Partition unique stats check |
| `query_executor.c:12977` | Insert into partitioned table |
| `load_server_loader.cpp:1100` | Bulk load completion check |

---

### 6.5 `btree_unique_stats` — Operators

---

#### `btree_unique_stats& operator=(const btree_unique_stats &us)` (lines 123–131)

**Signature:** Copy-assignment from another `btree_unique_stats`.

**Description:** Deep copies all three fields. Returns `*this` for chaining.

**Algorithm:** Three field assignments. O(1).

---

#### `void operator+=(const btree_unique_stats &us)` (lines 133–139)

**Signature:** Accumulates another stats object into this one.

**Description:** Component-wise addition of `m_rows`, `m_keys`, `m_nulls`. Used when merging deltas — e.g., `(*insert_helper->unique_stats_info) += incr` (btree.c:27293), or when accumulating a scan cache's stats into a transaction-level map.

**Algorithm:**
```cpp
m_rows  += us.m_rows;
m_keys  += us.m_keys;
m_nulls += us.m_nulls;
```
O(1). Critical hot path — called once per row mutation for multi-row operations.

---

#### `void operator-=(const btree_unique_stats &us)` (lines 141–147)

**Signature:** Subtracts another stats object from this one.

**Description:** Component-wise subtraction. Used in rollback/undo paths where previously accumulated deltas need to be reversed.

**Algorithm:**
```cpp
m_rows  -= us.m_rows;
m_keys  -= us.m_keys;
m_nulls -= us.m_nulls;
```
O(1).

---

#### `void to_string(string_buffer &strbuf) const` (lines 149–153)

**Signature:** Formats the stats into a `string_buffer` for debugging/logging.

**Description:** Produces `"oids=N keys=N nulls=N"` using `string_buffer`'s variadic printf-style call operator. Note the field name: `m_rows` is printed as `oids` (matching CUBRID's legacy OID-centric terminology for B-tree entries).

**Algorithm:**
```cpp
strbuf("oids=%d keys=%d nulls=%d", m_rows, m_keys, m_nulls);
```
O(1). Formatting only — no allocation within this method (string_buffer manages its own buffer).

---

### 6.6 `multi_index_unique_stats` — Lifecycle

---

#### `void construct()` (lines 155–159)

**Signature:** `void multi_index_unique_stats::construct()`

**Description:** In-place construction using placement new. Intended for use when `multi_index_unique_stats` is embedded directly in C structs (like `LOG_TDES`) that are allocated with C-style `malloc` or arena allocation, bypassing normal C++ constructor invocation. Calling `construct()` explicitly triggers the default constructor on the already-allocated memory.

**Algorithm:**
```cpp
new (this) multi_index_unique_stats();
```
This is the reason `memory_wrapper.hpp` is excluded from this file.

**Callers:** `log_tran_table.c:1646` — called during transaction descriptor initialization (`logtb_initialize_tdes()`), where `LOG_TDES` is a C struct allocated in a shared-memory array.

---

#### `void destruct()` (lines 161–165)

**Signature:** `void multi_index_unique_stats::destruct()`

**Description:** Explicit destructor call. Counterpart to `construct()`. Triggers `~multi_index_unique_stats()` (= default), which destroys `m_stats_map` and deallocates the `std::map`'s internal nodes. Required when the containing C struct is freed without going through `delete`.

**Algorithm:**
```cpp
this->~multi_index_unique_stats();
```

**Callers:** Called when transaction descriptors are torn down in the same C-lifetime management code that calls `construct()`. The pair (`construct()` / `destruct()`) is CUBRID's idiom for managing C++ objects embedded in C structs.

---

### 6.7 `multi_index_unique_stats` — Map management

---

#### `void add_index_stats(const BTID &index, const btree_unique_stats &us)` (lines 167–172)

**Signature:**
```cpp
void multi_index_unique_stats::add_index_stats(const BTID &index, const btree_unique_stats &us)
```

**Description:** Accumulates `us` into the slot for `index` in `m_stats_map`. If `index` is not yet in the map, `std::map::operator[]` default-constructs a zero `btree_unique_stats` first, then `operator+=` adds `us`. Asserts that `index` is not null.

**Algorithm:**
```cpp
assert(!BTID_IS_NULL(&index));
m_stats_map[index] += us;
```
O(log N) where N is the number of unique indexes being tracked. For typical OLTP loads (N < 20), effectively O(1).

**Callers:** Not called from within this file; called by external code accumulating per-index stats.

---

#### `void add_empty(const BTID &index)` (lines 174–179)

**Signature:**
```cpp
void multi_index_unique_stats::add_empty(const BTID &index)
```

**Description:** Inserts a zero-initialized `btree_unique_stats` for `index` into the map. Used at scan-cache setup time to pre-populate the map with all indexes of the scanned class, so that `get_stats_of()` lookups never need to insert new slots.

**Algorithm:**
```cpp
assert(!BTID_IS_NULL(&index));
m_stats_map[index] = btree_unique_stats();
```
O(log N).

**Callers:** `heap_file.c:7033` — called in a loop over `classrepr->n_indexes` during `heap_scancache_start_modify()`.

---

#### `void clear()` (lines 181–185)

**Signature:** `void multi_index_unique_stats::clear()`

**Description:** Removes all entries from `m_stats_map`. Called at the end of a multi-row operation (after committing stats to the transaction log) to reset for the next batch, and on error paths.

**Algorithm:**
```cpp
m_stats_map.clear();
```
O(N) where N is the number of entries (destructs each `btree_unique_stats` node, though those destructors are trivial).

**Callers:**
- `log_tran_table.c:1520` — transaction reset
- `locator_sr.c:6702` — after successful end-of-multi-update
- `locator_sr.c:6715` — on error, to avoid stale data
- `log_manager.c:5396`, `5511` — log manager rollback/abort paths

---

#### `const container_type& get_map() const` (lines 187–191)

**Signature:**
```cpp
const multi_index_unique_stats::container_type& multi_index_unique_stats::get_map() const
```

**Description:** Returns a const reference to the internal `m_stats_map`. Used for iteration — callers range-for over the map to check each index's `is_unique()` and submit stats to the log via `logtb_tran_update_unique_stats()`.

**Algorithm:** Single return. O(1).

**Callers:** Multiple sites in `locator_sr.c`, `query_executor.c`, `load_server_loader.cpp` — all following the pattern:
```cpp
for (const auto &it : stats.get_map()) {
    if (!it.second.is_unique()) { /* raise error */ }
    logtb_tran_update_unique_stats(..., it.first, it.second, true);
}
```

---

#### `bool empty() const` (lines 193–197)

**Signature:** `bool multi_index_unique_stats::empty() const`

**Description:** Returns true if the map has no entries. Quick guard before iterating.

**Algorithm:**
```cpp
return m_stats_map.empty();
```
O(1).

---

#### `btree_unique_stats& get_stats_of(const BTID &index)` (lines 199–204)

**Signature:**
```cpp
btree_unique_stats& multi_index_unique_stats::get_stats_of(const BTID &index)
```

**Description:** Returns a mutable reference to the stats for a specific index. Asserts the index is not null. Uses `std::map::operator[]`, which will default-insert a zero entry if the key is absent (though callers typically guarantee `add_empty()` was called first). The non-const reference allows callers to call `insert_key_and_row()` etc. directly on the stored object.

**Algorithm:**
```cpp
assert(!BTID_IS_NULL(&index));
return m_stats_map[index];
```
O(log N).

**Callers:**
- `locator_sr.c:7901` — `unique_stat_info = &scan_cache->m_index_stats->get_stats_of(index->btid)` — gets pointer to the map slot, then the B-tree insert/delete code writes into it via the pointer.
- `locator_sr.c:8525` — same pattern for updates.

---

#### `void to_string(string_buffer &strbuf) const` (lines 206–221)

**Signature:**
```cpp
void multi_index_unique_stats::to_string(string_buffer &strbuf) const
```

**Description:** Formats the entire map as a nested JSON-like string for debugging. Produces: `{{btid=ROOT|VOL|FILE, oids=N keys=N nulls=N}, ...}`. Iterates the map and calls `btree_unique_stats::to_string()` for each entry, adding comma separators between entries.

**Algorithm:** Linear iteration over `m_stats_map`. O(N).

**Format example output:**
```
{{btid=123|1|456, oids=5 keys=4 nulls=1}, {btid=789|1|012, oids=3 keys=3 nulls=0}}
```

---

### 6.8 `multi_index_unique_stats` — Operators and copy

---

#### `multi_index_unique_stats& operator=(multi_index_unique_stats &&other)` (lines 223–228)

**Signature:** Move-assignment operator.

**Description:** Moves `other.m_stats_map` into `this->m_stats_map` using `std::move`. After this call, `other.m_stats_map` is in a valid but unspecified (empty) state.

**Algorithm:**
```cpp
m_stats_map = std::move(other.m_stats_map);
return *this;
```
O(1) — pointer/node transfer, no element copies.

**Note:** Copy-assignment `operator=(const multi_index_unique_stats &)` is `= delete`. The deliberate asymmetry forces callers to use either move assignment or the explicit `copy_to()` method.

---

#### `void operator+=(const multi_index_unique_stats &other)` (lines 230–238)

**Signature:**
```cpp
void multi_index_unique_stats::operator+=(const multi_index_unique_stats &other)
```

**Description:** Merges all entries from `other.m_stats_map` into `this->m_stats_map` using `add_index_stats` semantics. For each entry in `other`, accumulates it into the corresponding slot in `this` (creating the slot if absent via `operator[]` default-insert).

**Algorithm:**
```cpp
for (const auto &it : other.m_stats_map) {
    m_stats_map[it.first] += it.second;
}
```
O(M log N) where M = entries in `other`, N = entries in `this`. In practice, both maps have the same set of BTIDs (one per unique index on a class).

**Callers:**
- `locator_sr.c:6677` — `tdes->m_multiupd_stats += *scan_cache->m_index_stats` — aggregates scan cache stats into transaction-level stats at end of each batch.
- `query_executor.c:10820` — `(*info) += (*scan_cache->m_index_stats)`.

---

#### `void copy_to(multi_index_unique_stats &dest) const` (lines 240–244)

**Signature:**
```cpp
void multi_index_unique_stats::copy_to(multi_index_unique_stats &dest) const
```

**Description:** Deep-copies `m_stats_map` into `dest.m_stats_map`. The explicit method exists because copy-assignment is deleted. Used when cloning a transaction descriptor.

**Algorithm:**
```cpp
dest.m_stats_map = m_stats_map;
```
O(N) — copies all N map nodes.

**Callers:** `log_tran_table.c:6273` — inside `log_tdes::copy_to()`, which performs a full transaction descriptor deep-copy (used for 2PC coordinator/participant mirroring and transaction table snapshot).

---

## 7. Key Algorithms & Logic Flows

### 7.1 Single-row DML (immediate stats update)

For single-row inserts/deletes (`!BTREE_IS_MULTI_ROW_OP(op_type)`):

1. **btree.c** creates a local `btree_unique_stats incr` (zero-initialized).
2. Calls `incr.insert_key_and_row()` or `incr.delete_key_and_row()` (or null variants).
3. Calls `logtb_tran_update_unique_stats(thread_p, *btid, incr, true)`.
4. This C++ overload (log_tran_table.c:3628) calls `incr.is_zero()` as an early exit; if not zero, bridges to the old C API: `logtb_tran_update_unique_stats(thread_p, &btid, incr.get_key_count(), incr.get_row_count(), incr.get_null_count(), write_to_log)`.
5. The old API updates the transaction's per-BTID stats hash table (separate from `m_multiupd_stats`) and writes a compensating log record.
6. Violation is detected immediately (one row at a time), not deferred.

### 7.2 Multi-row DML (deferred stats update — the primary path)

For bulk operations (`BTREE_IS_MULTI_ROW_OP(op_type)` — MULTI_ROW_INSERT, MULTI_ROW_UPDATE, MULTI_ROW_DELETE):

```
HEAP_SCANCACHE::m_index_stats (multi_index_unique_stats*)
    ^-- add_empty() for each index at scan-cache start (heap_file.c)
    ^-- get_stats_of(btid) returns mutable ref -> used as unique_stat_info pointer

btree.c insert/delete path:
    local incr = btree_unique_stats()
    incr.insert_key_and_row() / delete_key_and_row()  [or null variants]
    (*unique_stats_info) += incr     // accumulates into scan cache's map slot

At end of each object batch (locator_sr.c):
    tdes->m_multiupd_stats += *scan_cache->m_index_stats
    [repeats for each scan cache in the operation]

At END_MULTI_UPDATE flag:
    for each (btid, stats) in tdes->m_multiupd_stats.get_map():
        if !stats.is_unique():
            BTREE_SET_UNIQUE_VIOLATION_ERROR -> ER_BTREE_UNIQUE_FAILED
        else:
            logtb_tran_update_unique_stats(btid, stats, write_to_log=true)
    tdes->m_multiupd_stats.clear()
```

**Key design insight:** The deferred model allows multi-row operations to transiently violate uniqueness (e.g., UPDATE that moves all rows from key=1 to key=2 must delete all key=1 entries before inserting key=2 entries). The check fires only after all mutations are applied, when the final state can be evaluated.

**HA exception:** `BTREE_INSERT_HELPER::is_ha_enabled` — when HA is active, even transient violations are forbidden; the multi-row deferred path is bypassed for stricter immediate checking.

### 7.3 Uniqueness check predicate deep-dive

`is_unique()` relies on the invariant `m_rows == m_keys + m_nulls`:

- **Unique, no duplicates:** m_rows=5, m_keys=4, m_nulls=1 → 5==5 → true
- **Duplicate non-null key:** Two OIDs share key K. m_rows=6, m_keys=4, m_nulls=1 → 6≠5 → false
- **Null values (always pass):** m_rows=3, m_keys=0, m_nulls=3 → 3==3 → true
- **Net zero change (short-circuit):** m_rows=0, m_keys=0, m_nulls=0 → is_zero() returns true, caller skips entirely

Why `add_row()` / `delete_row()` exist: When MVCC is active and an update occurs, there is a window where both the old and new OID are visible in the same unique index slot. `add_row()` increments `m_rows` without incrementing `m_keys`, temporarily making `m_rows > m_keys + m_nulls`. This signals "more OIDs than expected for this key." The correction happens when the old OID is cleaned up (MVCC delete), calling `delete_key_and_row()` which restores the invariant.

### 7.4 Query executor unique stats flow

`qexec_process_unique_stats()` (query_executor.c:10833):

1. If the internal class has an active scan cache, call `qexec_update_btree_unique_stats_info()` to merge `m_scancache.m_index_stats` into `internal_class->m_unique_stats`.
2. Iterate `m_unique_stats.get_map()`.
3. For each `(btid, stats)`: call `stats.is_unique()`. If false, raise `ER_BTREE_UNIQUE_FAILED`.
4. If true, call `logtb_tran_update_unique_stats()` to persist the delta.

This is called from both `qexec_end_mainblock_iterations()` (UPDATE processing) and the insert mainblock, providing uniform violation detection across DML types.

---

## 8. Concurrency & Thread Safety

### Per-instance ownership model

**`btree_unique_stats` objects are not thread-safe.** They are always owned by one of:
- A local variable in a btree operation function (stack-allocated, single-thread)
- A slot inside `multi_index_unique_stats::m_stats_map` (protected by the owning container's access pattern)
- A pointed-to slot retrieved via `get_stats_of()` (callers must not share this pointer across threads)

**`multi_index_unique_stats` objects are not thread-safe.** They are owned by:
- `LOG_TDES::m_multiupd_stats` — one per transaction descriptor. CUBRID transaction descriptors are thread-local in the sense that each worker thread operates on its own transaction at any given time. The `m_multiupd_stats` is accessed only by the thread currently executing that transaction.
- `HEAP_SCANCACHE::m_index_stats` — heap allocated via `new`, owned by the scan cache. Scan caches are per-thread during scan execution.

### No locks in this module

The module contains zero synchronization primitives. All safety is provided by the caller's ownership discipline:
- The transaction table (`LOG_TDES` array) uses a global latch at higher levels.
- Scan caches are thread-confined during their lifetime.

### `m_multiupd_stats` lifetime safety

`LOG_TDES::m_multiupd_stats` uses `construct()`/`destruct()` because `LOG_TDES` is a C-allocated struct. The transaction table initialization calls `construct()` once per descriptor slot, and teardown calls `destruct()`. Between these calls, `clear()` resets it between multi-row operations. This pattern is safe because transaction descriptors are not shared across threads while active.

---

## 9. Memory Management

### `btree_unique_stats`

- **No dynamic allocation.** Three `int64_t` fields. Trivially destructible.
- Stack-allocated as local `incr` variables in btree.c.
- Embedded by value in `m_stats_map` nodes.

### `multi_index_unique_stats`

- **`m_stats_map` (std::map):** Standard library tree, allocates nodes on the C++ heap via `std::allocator`. Each node holds `{BTID key, btree_unique_stats value, color, parent/left/right pointers}`. Node count = number of unique indexes tracked.
- **`heap_file.c` usage:** `scan_cache->m_index_stats = new multi_index_unique_stats()` — raw `new`. Freed with `delete scan_cache->m_index_stats` (line 7028) and `delete`/null at `heap_scancache_end()` (line 7247–7248). The comment in the header ("does this really belong to scan cache??") suggests this ownership is under question.
- **`LOG_TDES` embedding:** `m_multiupd_stats` is embedded by value (not pointer) in the struct. It is constructed/destructed with the explicit `construct()`/`destruct()` pair because the parent struct is C-allocated. `m_stats_map`'s internal heap nodes are managed normally by the STL.
- **No `free_and_init()` usage** — this module is C++ only and uses RAII/STL allocation for the map. The project-wide rule of `free_and_init()` applies to C-style `malloc`/`free`, not to `operator new`/`delete` used here.

### Placement new

`construct()` uses `new (this) multi_index_unique_stats()` — placement new on `this`. This invokes the default constructor in-place, resetting `m_stats_map` to an empty state. It does not allocate memory; the memory for the object itself was already provided by the `LOG_TDES` allocation.

---

## 10. Error Handling

### No error codes in this module

`btree_unique_stats` and `multi_index_unique_stats` themselves never call `er_set()` and never return error codes. They are pure data manipulation. The uniqueness-violation error (`ER_BTREE_UNIQUE_FAILED`) is raised by callers after calling `is_unique()`.

### Error propagation pattern in callers

```cpp
// locator_sr.c pattern:
for (const auto &it : tdes->m_multiupd_stats.get_map()) {
    if (!it.second.is_unique()) {
        BTREE_SET_UNIQUE_VIOLATION_ERROR(thread_p, NULL, NULL, &class_oid, &it.first, NULL);
        error_code = ER_BTREE_UNIQUE_FAILED;
        goto error;  // jump to cleanup that calls tdes->m_multiupd_stats.clear()
    }
    error_code = logtb_tran_update_unique_stats(thread_p, it.first, it.second, true);
    if (error_code != NO_ERROR) {
        ASSERT_ERROR();
        goto error;
    }
}
```

### Assert usage

Both `add_index_stats()`, `add_empty()`, and `get_stats_of()` contain `assert(!BTID_IS_NULL(&index))`. These are debug-mode only checks (`assert` is a no-op in release builds unless `NDEBUG` is not defined). They guard against programming errors where a null BTID is accidentally used as a map key.

### Negative counter tolerance

The use of `int64_t` (signed) for all three counters is deliberate. During MVCC undo/rollback, deletes may be recorded before inserts are rolled back, producing transient negative counts. The final check at transaction end is the authoritative one.

---

## 11. Integration Points

### 11.1 `src/storage/btree.c` (primary consumer)

`btree.c` is the primary call site where `btree_unique_stats` mutations occur. Two internal helper structs carry a pointer to the stats accumulator:

```c
// BTREE_INSERT_HELPER (line 667):
btree_unique_stats *unique_stats_info; // non-null for multi-row ops

// BTREE_DELETE_HELPER (line 773):
btree_unique_stats *unique_stats_info; // same
```

The stats update logic (lines 27248–27310 for inserts, 30826–30860 for deletes) follows the same pattern:
1. Create local `btree_unique_stats incr`.
2. Call the appropriate mutator based on null/non-null and insert/delete.
3. If `BTREE_IS_MULTI_ROW_OP`: `(*unique_stats_info) += incr` — accumulate into the scan cache slot.
4. Else: `logtb_tran_update_unique_stats(thread_p, *btid, incr, true)` — immediate log entry.

The `unique_stats_info` pointer in the helper structs points into `multi_index_unique_stats::m_stats_map` via `get_stats_of(btid)` — so the map slot is updated directly without additional indirection.

### 11.2 `src/transaction/log_impl.h` + `log_tran_table.c`

`LOG_TDES::m_multiupd_stats` (log_impl.h:511) accumulates the transaction-level delta for multi-row operations. The C++ overloads of `logtb_tran_update_unique_stats()` (log_tran_table.c:3628, 3640) provide a clean typed API:

```cpp
// Single BTID overload: bridges to C API
int logtb_tran_update_unique_stats(THREAD_ENTRY *thread_p,
    const BTID &btid, const btree_unique_stats &ustats, bool write_to_log);

// Multi-BTID overload: iterates the map
int logtb_tran_update_unique_stats(THREAD_ENTRY *thread_p,
    const multi_index_unique_stats &multi_stats, bool write_to_log);
```

The single-BTID overload uses `is_zero()` for early exit, then extracts the three counters via the getter methods.

### 11.3 `src/transaction/locator_sr.c`

The server-side locator (`locator_force_for_multi_update()`) is the orchestrator for bulk DML:
- Gets `unique_stat_info = &scan_cache->m_index_stats->get_stats_of(index->btid)` — direct reference into the map slot.
- Passes this pointer into btree insert/delete functions.
- At batch end: `tdes->m_multiupd_stats += *scan_cache->m_index_stats` — merges into transaction-level aggregator.
- At `END_MULTI_UPDATE`: checks `is_unique()` for each index and logs approved stats.

### 11.4 `src/storage/heap_file.c` + `heap_file.h`

`HEAP_SCANCACHE::m_index_stats` (heap_file.h:155) is a heap-allocated `multi_index_unique_stats *`. Lifecycle:
- `heap_scancache_start_modify()` with a multi-row op type: `new multi_index_unique_stats()`, then `add_empty()` for each index.
- `heap_scancache_end()`: `delete m_index_stats`, set to NULL.
- For single-row op types: remains NULL (stats go directly to transaction table).

### 11.5 `src/query/query_executor.c`

`UPDDEL_CLASS_INFO_INTERNAL::m_unique_stats` (query_executor.c:380) is a by-value `multi_index_unique_stats` embedded in the UPDATE/DELETE internal class info. `qexec_update_btree_unique_stats_info()` merges scan cache stats into it, and `qexec_process_unique_stats()` performs the final check and log commit.

### 11.6 `src/loaddb/load_server_loader.cpp`

The bulk loader checks uniqueness at the end of loading each class's data, iterating `m_scancache.m_index_stats->get_map()` and calling `is_unique()`. Violations abort the load with `m_error_handler.on_failure()`.

---

## 12. Complexity & Metrics

| Metric | Value |
|--------|-------|
| Total lines (cpp+hpp) | 352 |
| Classes defined | 2 (`btree_unique_stats`, `multi_index_unique_stats`) |
| Methods total | 29 (17 in `btree_unique_stats`, 12 in `multi_index_unique_stats`) |
| Global variables | 0 |
| Static variables | 0 |
| Dynamic allocations in this file | 1 (placement new in `construct()`) |
| External files that include btree_unique.hpp | 8 direct (btree.h, btree.c, heap_file.h, heap_file.c, log_impl.h, log_tran_table.c, locator_sr.c, query_executor.c, load_server_loader.cpp) |
| Total LSP references to `btree_unique_stats` | 62 |
| Total LSP references to `multi_index_unique_stats` | 37 |
| Average method complexity | O(1) for all `btree_unique_stats` methods; O(log N) for map operations in `multi_index_unique_stats`; O(N) for iteration methods |
| Cyclomatic complexity | 1 for all `btree_unique_stats` methods (no branches); 2–3 for `multi_index_unique_stats` iteration methods |

---

## 13. Notable Patterns & Idioms

### 13.1 Placement new for C-struct-embedded C++ objects

The `construct()`/`destruct()` pair is a CUBRID-wide idiom for managing C++ objects that are embedded by value in C structs allocated outside the C++ runtime. It appears in `multi_index_unique_stats`, `mvcc_info`, `log_postpone_cache`, and other types embedded in `LOG_TDES`. This avoids the need to heap-allocate each sub-object while still getting proper construction/destruction semantics.

### 13.2 Deleted copy-assignment with explicit `copy_to()`

`multi_index_unique_stats` deletes `operator=(const &)` but provides `copy_to(dest)`. This is an intentional API design that:
- Prevents accidental expensive map copies in hot paths (e.g., `stats_a = stats_b` would silently copy N map nodes).
- Forces callers to explicitly name the operation (`copy_to`) to acknowledge the cost.
- Allows move assignment for cheap transfers.

### 13.3 `is_zero()` short-circuit before logging

The `logtb_tran_update_unique_stats()` C++ overload calls `ustats.is_zero()` before doing any work. This is a hot-path optimization: if a row was inserted then immediately deleted within the same batch (net zero change), no log record is written and the hash table lookup is skipped entirely.

### 13.4 Deferred constraint checking (ACID relaxation within a statement)

The entire architecture implements "statement-level" uniqueness checking rather than "row-level" checking. This is necessary for:
- `UPDATE t SET k = k + 1` — all rows shift; intermediate states have both old and new keys simultaneously.
- `INSERT INTO t SELECT ...` — bulk insert where duplicates resolve as a batch.

This is standard SQL behavior (deferred constraints within a statement) and the `multi_index_unique_stats` accumulator is the mechanism that makes it work in CUBRID.

### 13.5 `stat_type = int64_t` for large table support

Using `int64_t` rather than `int` or `uint32_t` ensures:
- Tables with more than 2^31 rows (2.1 billion) are handled correctly.
- Negative values during undo paths are representable without overflow.

### 13.6 `btid_comparator` incompleteness note

The `btid_comparator` struct compares `root_pageid` and `vfid.volid` but omits `vfid.fileid`. This is technically a partial ordering that could treat distinct BTIDs as equal if they share the same root page ID and volume ID but have different file IDs. In practice, root page IDs are globally unique within the volume namespace, so two distinct B-trees will never share a root page ID. The comparator is therefore correct in practice even though it is not a full structural comparison.

### 13.7 `string_buffer` diagnostic interface

Both classes implement `to_string(string_buffer &)` rather than returning a `std::string`. `string_buffer` is CUBRID's growable printf-style buffer that avoids heap allocation for short strings (stack-local storage with overflow to heap). Using it as an output parameter avoids string copies and integrates with CUBRID's structured logging infrastructure.

### 13.8 Memory wrapper exclusion

The `#if 0` block around `#include "memory_wrapper.hpp"` with its explanatory comment is an unusual pattern in CUBRID. Most files must include `memory_wrapper.hpp` as the last header for heap tracking. The explicit exclusion here, with a developer note explaining the reason (placement new bypasses the tracker), demonstrates awareness of the memory monitoring system and care not to create phantom tracking holes.

---

## Summary

`btree_unique.cpp` and `btree_unique.hpp` implement a compact, high-performance statistics accumulation layer for CUBRID's unique index constraint enforcement. The two classes — `btree_unique_stats` (per-index counter triplet) and `multi_index_unique_stats` (BTID-keyed map of per-index stats) — together implement deferred uniqueness checking for multi-row DML operations. The design is O(1) per row mutation, O(N) at check time where N is the number of unique indexes. It integrates with the transaction descriptor (`LOG_TDES`), heap scan cache (`HEAP_SCANCACHE`), the B-tree engine (`btree.c`), and the query executor (`query_executor.c`) through a well-defined typed API. The module has zero global state, zero locks, and zero error returns — all safety and error detection is delegated to callers.
