# record_descriptor — Comprehensive Analysis Report

> Generated: 2026-03-27
> Scope: `src/storage/record_descriptor.hpp` + `src/storage/record_descriptor.cpp`

---

## Table of Contents

1. [File Overview](#1-file-overview)
2. [Includes & Dependencies](#2-includes--dependencies)
3. [Preprocessor & Compilation](#3-preprocessor--compilation)
4. [Data Structures & Types](#4-data-structures--types)
5. [Global & Static Variables](#5-global--static-variables)
6. [Function / Method Catalog](#6-function--method-catalog)
7. [Key Algorithms & Logic Flows](#7-key-algorithms--logic-flows)
8. [Concurrency & Thread Safety](#8-concurrency--thread-safety)
9. [Memory Management](#9-memory-management)
10. [Error Handling](#10-error-handling)
11. [Integration Points](#11-integration-points)
12. [Complexity & Metrics](#12-complexity--metrics)
13. [Notable Patterns & Idioms](#13-notable-patterns--idioms)

---

## 1. File Overview

| Property | Value |
|---|---|
| Header | `src/storage/record_descriptor.hpp` |
| Implementation | `src/storage/record_descriptor.cpp` |
| Header guard | `_RECORD_DESCRIPTOR_HPP_` (traditional `#ifndef` style, no `#pragma once`) |
| Language | C++17 (compiled via `c_to_cpp.sh` pipeline; `.cpp` extension, C++17 standard) |
| Line count (header) | 181 lines |
| Line count (impl) | 360 lines |
| Total | ~541 lines |
| Purpose | Provides `record_descriptor`, a RAII C++ wrapper around the legacy C struct `recdes` (RECORD DESCRIPTOR). Adds safe buffer management, ownership tracking, slotted-page I/O helpers, data mutation APIs, and serialisation support. |
| Build modes | All three: `SERVER_MODE` (cub_server), `SA_MODE` (standalone), and `CS_MODE` (client library). There are no preprocessor guards restricting usage to any single mode. |
| LSP-reported size | 64 bytes, alignment 8 bytes |

### Purpose in the Engine

`recdes` is CUBRID's universal record container — a plain C struct used throughout the engine from heap I/O to query execution. It carries a raw `char *data` pointer, an allocated area size, a data length, and a 16-bit record type tag. Raw `recdes` requires manual buffer management and is error-prone.

`record_descriptor` wraps `recdes` with:
- Automatic heap buffer management via `cubmem::extensible_block`
- An explicit ownership/source state machine (`data_source` enum)
- Safe mutation methods that enforce write-permission checks
- Slotted-page I/O helpers (`peek`, `copy`, `get`) that auto-resize on `S_DOESNT_FIT`
- Serialisation conformance through `cubpacking::packable_object`

It is the modern preferred alternative to directly manipulating `recdes` + raw `malloc`/`free` in C++ code.

---

## 2. Includes & Dependencies

### 2.1 Header (`record_descriptor.hpp`)

| Include | Module | Purpose |
|---|---|---|
| `"mem_block.hpp"` | `src/base` | `cubmem::extensible_block`, `cubmem::block_allocator`, `cubmem::stack_block<S>`, `cubmem::PRIVATE_BLOCK_ALLOCATOR`, `cubmem::STANDARD_BLOCK_ALLOCATOR` |
| `"memory_alloc.h"` | `src/base` | Private heap allocation infrastructure (`db_private_alloc`, `HL_HEAPID`) |
| `"memory_private_allocator.hpp"` | `src/base` | `cubmem::PRIVATE_BLOCK_ALLOCATOR` constant; private allocator templates |
| `"packable_object.hpp"` | `src/base` | Base class `cubpacking::packable_object` (pure-virtual pack/unpack interface) |
| `"storage_common.h"` | `src/storage` | `recdes` struct, `PGSLOTID`, `PAGE_PTR`, `PEEK`/`COPY` constants, `REC_HOME`, `REC_UNKNOWN`, record-type enum |
| `<cinttypes>` | system | `std::int16_t`, `std::size_t` |
| Forward decl `cubthread::entry` | `src/thread` | Thread entry pointer used in slotted-page calls; avoids pulling in thread headers |

### 2.2 Implementation (`record_descriptor.cpp`)

| Include | Module | Purpose |
|---|---|---|
| `"record_descriptor.hpp"` | self | Own header |
| `"error_code.h"` | `src/base` | `NO_ERROR`, `ER_FAILED` error code constants |
| `"memory_alloc.h"` | `src/base` | Memory allocation support |
| `"packer.hpp"` | `src/base` | `cubpacking::packer` and `cubpacking::unpacker` concrete types |
| `"slotted_page.h"` | `src/storage` | `spage_get_record()`, `S_SUCCESS`, `S_DOESNT_FIT`, `SCAN_CODE` |
| `<cstring>` | system | `std::memcpy`, `std::memmove` |
| `"memory_wrapper.hpp"` | `src/base` | **Must be last include** — overrides `malloc`/`free` for leak detection (project convention) |

### 2.3 Reverse Dependencies (Files That Include `record_descriptor.hpp`)

LSP find-references confirms 58 cross-file references across 9 source files:

| File | Module | Usage pattern |
|---|---|---|
| `src/storage/heap_file.c` | storage | Output parameter in `heap_attrinfo_transform_to_disk*()` — receives a newly built disk record |
| `src/storage/heap_file.h` | storage | Forward declaration + function signatures using `record_descriptor *` |
| `src/transaction/locator_sr.c` | transaction | `build_record()` local variable; multi-record insert (`std::vector<record_descriptor>`) |
| `src/transaction/locator_sr.h` | transaction | Function signature with `std::vector<record_descriptor>` parameter |
| `src/loaddb/load_server_loader.cpp` | loaddb | `new_recdes` local variable for bulk-loading records |
| `src/loaddb/load_server_loader.hpp` | loaddb | Member or parameter type |
| `src/query/serial.c` | query | `new_recdesc` local variable for serial object updates |
| `src/storage/slotted_page.c` | storage | Indirect — `spage_get_record` is called by `record_descriptor::get` |
| `src/storage/record_descriptor.cpp` | self | Implementation |

---

## 3. Preprocessor & Compilation

### 3.1 Header Guards

```cpp
#ifndef _RECORD_DESCRIPTOR_HPP_
#define _RECORD_DESCRIPTOR_HPP_
// ...
#endif // !_RECORD_DESCRIPTOR_HPP_
```

Follows the project-wide convention: `_FILENAME_HPP_` style, no `#pragma once`.

### 3.2 `memory_wrapper.hpp` Convention

The implementation file places `memory_wrapper.hpp` as the absolute last include, preceded by the mandatory comment:

```cpp
// XXX: SHOULD BE THE LAST INCLUDE HEADER
#include "memory_wrapper.hpp"
```

This is a hard project rule. `memory_wrapper.hpp` overrides `malloc`/`free` for memory tracking and must not precede other system headers.

### 3.3 No Mode Guards

There are no `#if defined(SERVER_MODE)`, `#if defined(SA_MODE)`, or `#if defined(CS_MODE)` guards anywhere in these files. `record_descriptor` is unconditionally available in all build configurations. This contrasts with many other storage files that are server-side only.

### 3.4 `EXPORT_IMPORT`

Not used. `record_descriptor` is not exported from a DLL boundary; it is a header-only-for-templates + implementation-file class.

---

## 4. Data Structures & Types

### 4.1 The Underlying C Struct: `recdes`

Defined in `src/storage/storage_common.h:219`:

```c
typedef struct recdes RECDES;
struct recdes
{
  int area_size;   /* Allocated buffer size (negative when peeking a page slot) */
  int length;      /* Actual data length (does NOT include type/length overhead) */
  INT16 type;      /* Record type tag — REC_HOME, REC_BIGONE, etc. */
  char *data;      /* Raw byte pointer to record content */
};
```

Key semantic: `area_size < 0` signals the record data is inside a slotted-page buffer (peek mode, not owned). The initialiser macro is `RECDES_INITIALIZER { 0, -1, REC_UNKNOWN, NULL }`.

### 4.2 Record Type Enum (from `storage_common.h`)

| Value | Constant | Meaning |
|---|---|---|
| 0 | `REC_UNKNOWN` | Uninitialised / unknown |
| 1 | `REC_ASSIGN_ADDRESS` | Address-only record, no content |
| 2 | `REC_HOME` | Normal home record |
| 3 | `REC_NEWHOME` | Relocated record at new position |
| 4 | `REC_RELOCATION` | Pointer to new home (forward) |
| 5 | `REC_BIGONE` | Large object descriptor |
| 6 | `REC_MARKDELETED` | Deleted, slot not reusable |
| 7 | `REC_DELETED_WILL_REUSE` | Deleted, slot reusable |

### 4.3 `record_get_mode` Enum

```cpp
enum class record_get_mode
{
  PEEK_RECORD = PEEK,   // = true  (bool constant in storage_common.h)
  COPY_RECORD = COPY    // = false (bool constant in storage_common.h)
};
```

An explicit scoped enum aliasing the legacy `bool` PEEK/COPY constants. Provides type safety; cast to `int` is performed internally before passing to `spage_get_record`.

### 4.4 `record_descriptor` Class

```cpp
class record_descriptor : public cubpacking::packable_object
```

**Size**: 64 bytes, alignment 8 (per LSP hover). Layout:

| Field | Type | Size (est.) | Role |
|---|---|---|---|
| `m_recdes` | `recdes` | 16 bytes (4+4+2+pad+8 ptr) | The underlying C record descriptor |
| `m_own_data` | `cubmem::extensible_block` | ~32 bytes (block + allocator ptr) | Owned heap buffer for record data |
| `m_data_source` | `data_source` (enum class, int) | 4 bytes + padding | Source/ownership state |
| vtable ptr (from `packable_object`) | pointer | 8 bytes | Virtual dispatch for pack/unpack |

### 4.5 `data_source` Private Enum

```cpp
enum class data_source
{
  INVALID,    // No data assigned; object uninitialised
  PEEKED,     // data points into a slotted page buffer (read-only, not owned)
  COPIED,     // data was copied from page or another record (owned, mutable)
  NEW,        // data is a freshly allocated buffer (owned, mutable)
  IMMUTABLE   // data points to an external const buffer (not owned, not mutable)
};
```

This state machine is the core invariant of the class. The valid transitions are:

```
INVALID -> COPIED   (set_recdes with non-empty rec)
INVALID -> NEW      (resize_buffer when INVALID)
INVALID -> IMMUTABLE (set_data)
INVALID -> PEEKED   (get with PEEK_RECORD mode)
INVALID -> COPIED   (get with COPY_RECORD mode)
INVALID -> COPIED   (unpack)
PEEKED  -> [no mutation allowed]
COPIED  -> COPIED   (remains after further copies)
NEW     -> NEW      (remains after resize)
IMMUTABLE -> [no mutation allowed]
```

Mutation methods (`modify_data`, `delete_data`, `insert_data`, `move_data`, `set_record_length`, `set_type`, `get_data_for_modify`) all call `check_changes_are_permitted()`, which asserts `is_mutable()`. Only `COPIED` and `NEW` states are mutable.

### 4.6 `cubmem::extensible_block`

The backing buffer type. Key properties (from `mem_block.hpp`):

- Wraps a `cubmem::block` (pointer + size pair) and a `const block_allocator *`
- `extend_to(n)`: grows to at least `n` bytes, preserving content
- `freemem()`: immediately releases memory
- `release_ptr()`: relinquishes ownership (sets internal ptr to null, returns raw pointer)
- `get_ptr()` / `get_size()`: accessors
- **Not copyable** (copy ctor/assign deleted); move-only
- Default allocator: `STANDARD_BLOCK_ALLOCATOR` (uses `new[]`/`delete[]`)
- Destructor calls `m_allocator->m_dealloc_f(m_block)` automatically

### 4.7 `cubmem::block_allocator`

A pair of `std::function` objects:
- `alloc_func`: `void(block &b, size_t size)` — allocate/reallocate
- `dealloc_func`: `void(block &b)` — deallocate

Pre-defined allocators:
- `STANDARD_BLOCK_ALLOCATOR` — `new[]`/`delete[]`
- `EXPONENTIAL_STANDARD_BLOCK_ALLOCATOR` — doubles capacity on growth
- `CSTYLE_BLOCK_ALLOCATOR` — `malloc`/`realloc`/`free`
- `PRIVATE_BLOCK_ALLOCATOR` — CUBRID private heap (`db_private_alloc`)

The default constructor of `record_descriptor` uses `PRIVATE_BLOCK_ALLOCATOR`.

---

## 5. Global & Static Variables

`record_descriptor.cpp` and `record_descriptor.hpp` define **no** global or static variables of their own. All state is per-instance.

The allocator constants referenced (`cubmem::PRIVATE_BLOCK_ALLOCATOR`, `cubmem::STANDARD_BLOCK_ALLOCATOR`, `cubmem::CSTYLE_BLOCK_ALLOCATOR`) are defined in `src/base/mem_block.cpp` and declared `extern` in `mem_block.hpp`.

---

## 6. Function / Method Catalog

This is the exhaustive catalog of every constructor, destructor, and method defined in the class.

---

### 6.1 Constructor: Default / Allocator

```cpp
record_descriptor(const cubmem::block_allocator &alloc = cubmem::PRIVATE_BLOCK_ALLOCATOR);
```

**Location**: `hpp:67`, `cpp:47-56`

**Description**: Primary constructor. Initialises an empty `record_descriptor` with no data.

**Algorithm**:
1. Default-initialises `m_recdes` via member initialiser list.
2. Constructs `m_own_data` with the provided allocator (default: private heap).
3. Sets `m_data_source = data_source::INVALID`.
4. Explicitly sets `m_recdes.area_size = 0`, `m_recdes.length = 0`, `m_recdes.type = REC_HOME`, `m_recdes.data = NULL`.

**Parameters**: `alloc` — the block allocator to use for internal buffer growth. Callers may pass `cubmem::STANDARD_BLOCK_ALLOCATOR`, `cubmem::CSTYLE_BLOCK_ALLOCATOR`, or `cubmem::PRIVATE_BLOCK_ALLOCATOR`.

**Error handling**: None; cannot fail.

**Callers**: All call sites that construct a `record_descriptor` without existing data (most common case). Also called via delegating constructors.

---

### 6.2 Constructor: From `recdes`

```cpp
record_descriptor(const recdes &rec,
                  const cubmem::block_allocator &alloc = cubmem::PRIVATE_BLOCK_ALLOCATOR);
```

**Location**: `hpp:73`, `cpp:58-63`

**Description**: Constructs by copying the content of an existing C-style `recdes`. Delegates to the default constructor, then calls `set_recdes(rec)`.

**Algorithm**:
1. Delegates to `record_descriptor(alloc)` (sets up empty state).
2. Calls `set_recdes(rec)` which deep-copies data if `rec.length != 0`.

**Error handling**: None; `set_recdes` uses `assert` for preconditions.

**Callers**: Code that has an existing `recdes` from a C API and wants C++ wrapper semantics.

---

### 6.3 Constructor: Move

```cpp
record_descriptor(record_descriptor &&other);
```

**Location**: `hpp:75`, `cpp:65-74`

**Description**: Move constructor. Transfers ownership of buffer from `other`.

**Algorithm**:
1. Copies `m_recdes` by value (trivial struct copy).
2. Move-assigns `m_own_data` via `std::move` (transfers heap buffer ownership, `other.m_own_data` becomes empty).
3. Copies `m_data_source`.
4. Clears `other`: sets `m_data_source = INVALID`, `m_recdes.data = NULL`, `m_recdes.type = REC_UNKNOWN`.

**Error handling**: None; noexcept-safe because `extensible_block` move is noexcept.

**Callers**: Returning `record_descriptor` from functions, inserting into `std::vector<record_descriptor>` (as in `locator_sr.c:13780`).

**Notable**: No copy constructor is defined (implicitly deleted because `extensible_block` is move-only). This enforces that `record_descriptor` values cannot be accidentally copied.

---

### 6.4 Constructor: From Raw Data

```cpp
record_descriptor(const char *data, std::size_t size);
```

**Location**: `hpp:70`, `cpp:76-80`

**Description**: Constructs an IMMUTABLE record referencing an external byte buffer. Delegates to default constructor then calls `set_data(data, size)`.

**Algorithm**:
1. Delegates to `record_descriptor()` (default, uses `PRIVATE_BLOCK_ALLOCATOR`).
2. Calls `set_data(data, size)` → `m_data_source = IMMUTABLE`.

**Error handling**: None.

**Use case**: Wrapping a constant data buffer (e.g., a compile-time struct) in a record without copying. Mutation is forbidden after construction.

---

### 6.5 Destructor

```cpp
~record_descriptor(void);
```

**Location**: `hpp:68`, `cpp:82-84`

**Description**: Trivial destructor — body is empty. All cleanup is performed automatically by `m_own_data`'s destructor (`extensible_block::~extensible_block()` calls `m_allocator->m_dealloc_f(m_block)`).

**Memory release**: Only happens if `m_own_data` owns the buffer (i.e., `set_external_buffer` was NOT used with a caller-managed buffer — in that case `m_own_data.freemem()` was called first). PEEKED records have `m_recdes.data` pointing into a page buffer, not into `m_own_data`, so nothing is freed.

---

### 6.6 `set_recdes`

```cpp
void set_recdes(const recdes &rec);
```

**Location**: `hpp:77`, `cpp:86-103`

**Description**: Deep-copies record content from an existing `recdes` into this object.

**Precondition (assert)**: `m_data_source == data_source::INVALID` — can only be called on a freshly constructed object.

**Algorithm**:
1. Copies `rec.type` to `m_recdes.type`.
2. If `rec.length != 0`:
   a. Sets `m_recdes.area_size = rec.length` (allocate exactly as much as needed).
   b. Calls `m_own_data.extend_to(area_size)` to allocate.
   c. Sets `m_recdes.data = m_own_data.get_ptr()`.
   d. `std::memcpy` copies `rec.length` bytes from `rec.data`.
   e. Sets `m_data_source = data_source::COPIED`.
3. If `rec.length == 0`, no buffer is allocated; `m_data_source` stays `INVALID`.

**Error handling**: None (returns void, uses assert for precondition).

**Callers**: The `recdes`-based constructor. Also callable directly to initialise from a C-style record.

---

### 6.7 `peek`

```cpp
int peek(cubthread::entry *thread_p, PAGE_PTR page, PGSLOTID slotid);
```

**Location**: `hpp:80`, `cpp:105-109`

**Description**: Read a record from a slotted page in PEEK mode (zero-copy — `m_recdes.data` will point directly into the page buffer).

**Algorithm**: Delegates to `get(thread_p, page, slotid, record_get_mode::PEEK_RECORD)`.

**Post-condition**: `m_data_source == PEEKED`. Mutation of record data is forbidden until the object is re-initialised.

**Callers**: Found in 2 locations (declaration + definition). External callers use this for read-only record inspection where copy overhead is undesirable.

**Locking note**: The caller must hold a page latch for the lifetime of any access to `m_recdes.data` after a `peek` call. This is not enforced by the class — it is a caller responsibility.

---

### 6.8 `copy`

```cpp
int copy(cubthread::entry *thread_p, PAGE_PTR page, PGSLOTID slotid);
```

**Location**: `hpp:83`, `cpp:111-115`

**Description**: Read a record from a slotted page in COPY mode — the record data is copied into the object's owned buffer.

**Algorithm**: Delegates to `get(thread_p, page, slotid, record_get_mode::COPY_RECORD)`.

**Post-condition**: `m_data_source == COPIED`. Record data is fully owned; page latch may be released.

---

### 6.9 `get` (internal unified I/O)

```cpp
int get(cubthread::entry *thread_p, PAGE_PTR page, PGSLOTID slotid, record_get_mode mode);
```

**Location**: `hpp:86`, `cpp:117-148`

**Description**: Core page-read implementation. Handles both PEEK and COPY, including automatic buffer resize on `S_DOESNT_FIT`.

**Algorithm**:
1. Cast `mode` to `int` for the C API.
2. Call `spage_get_record(thread_p, page, slotid, &m_recdes, mode_to_int)` → returns `SCAN_CODE`.
3. If `S_SUCCESS`: call `update_source_after_get(mode)`, return `NO_ERROR`.
4. If `S_DOESNT_FIT` (only in COPY mode):
   a. Assert that `m_recdes.length < 0` (spage contract: negative length = required size).
   b. Assert that mode is `COPY_RECORD`.
   c. Compute `required_size = (size_t)(-m_recdes.length)`.
   d. Call `resize_buffer(required_size)` to grow the internal buffer.
   e. Retry `spage_get_record` once.
   f. If `S_SUCCESS` on retry: update source, return `NO_ERROR`.
5. On any other failure: `assert(false)`, return `ER_FAILED`.

**Error handling**: Only two expected outcomes: success or `ER_FAILED`. The `assert(false)` on unexpected failures is a debug-mode crash guard. In release builds, `ER_FAILED` propagates to the caller who should check `er_errid()` for a set error code (set by `spage_get_record`).

**Key interaction**: `spage_get_record` is defined in `src/storage/slotted_page.h:135`:
```c
extern SCAN_CODE spage_get_record(THREAD_ENTRY *thread_p, PAGE_PTR pgptr,
                                  PGSLOTID slotid, RECDES *recdes, int ispeeking);
```
When `area_size > 0` and the record fits, `COPY` mode copies into `recdes.data`. When the buffer is too small, `spage_get_record` sets `recdes.length = -(required_size)` and returns `S_DOESNT_FIT`.

---

### 6.10 `resize_buffer`

```cpp
void resize_buffer(std::size_t required_size);
```

**Location**: `hpp:128`, `cpp:150-170`

**Description**: Ensures the internal buffer is at least `required_size` bytes. Allocates if not already large enough.

**Precondition (assert)**: `m_data_source == INVALID || is_mutable()` — cannot resize a PEEKED or IMMUTABLE record.

**Algorithm**:
1. If `m_recdes.area_size > 0 && required_size <= (size_t)m_recdes.area_size`: return early (already large enough).
2. Call `m_own_data.extend_to(required_size)` (allocates/reallocates via the block allocator).
3. Update `m_recdes.data = m_own_data.get_ptr()` (pointer may have changed after realloc).
4. Update `m_recdes.area_size = (int)required_size`.
5. If `m_data_source == INVALID`: transition to `data_source::NEW`.

**Note**: The `m_recdes.data` pointer is always refreshed after `extend_to` because reallocation may move the buffer.

---

### 6.11 `update_source_after_get`

```cpp
void update_source_after_get(record_get_mode mode);
```

**Location**: `hpp:141`, `cpp:172-188`

**Description**: Private helper that updates `m_data_source` after a successful `spage_get_record` call.

**Algorithm**: Switch on `mode`:
- `PEEK_RECORD` → `m_data_source = PEEKED`
- `COPY_RECORD` → `m_data_source = COPIED`
- default → `assert(false)`, set `INVALID`

---

### 6.12 `get_recdes`

```cpp
const recdes &get_recdes(void) const;
```

**Location**: `hpp:89`, `cpp:190-195`

**Description**: Returns a const reference to the underlying `recdes` struct.

**Precondition (assert)**: `m_data_source != INVALID`.

**Use case**: Passing to C APIs that accept `const RECDES *` (e.g., heap attribute reading functions).

---

### 6.13 `get_data`

```cpp
const char *get_data(void) const;
```

**Location**: `hpp:91`, `cpp:197-202`

**Description**: Returns a const pointer to the raw record data bytes.

**Precondition (assert)**: `m_data_source != INVALID`.

---

### 6.14 `get_size`

```cpp
std::size_t get_size(void) const;
```

**Location**: `hpp:92`, `cpp:204-210`

**Description**: Returns the current record data length as `size_t`.

**Precondition (assert)**: `m_data_source != INVALID` AND `m_recdes.length > 0`.

**Note**: The second assert means `get_size` is not safe to call on a zero-length record even if it is valid. This is intentional — callers should not expect a meaningful size from a logically empty record.

---

### 6.15 `get_data_for_modify`

```cpp
char *get_data_for_modify(void);
```

**Location**: `hpp:93`, `cpp:212-218`

**Description**: Returns a mutable pointer to the record data buffer.

**Precondition**: Calls `check_changes_are_permitted()` → asserts `is_mutable()` (state must be `COPIED` or `NEW`).

**Use case**: When the caller needs to directly write into the record buffer after having allocated space via `resize_buffer`.

---

### 6.16 `set_data`

```cpp
void set_data(const char *data, std::size_t size);
```

**Location**: `hpp:96`, `cpp:220-227`

**Description**: Makes the record reference an external constant buffer without copying.

**Algorithm**:
1. Sets `m_data_source = IMMUTABLE`.
2. Casts away constness: `m_recdes.data = const_cast<char *>(data)` (comment notes: "status will protect against changes").
3. Sets `m_recdes.length = (int)size`.
4. Does NOT set `m_recdes.area_size` — it remains 0 from construction.

**Error handling**: None.

**Post-condition**: Object is in IMMUTABLE state; all mutation methods will assert-fail.

---

### 6.17 `set_data_to_object` (template)

```cpp
template <typename T>
void set_data_to_object(const T &t);
```

**Location**: `hpp:97-98`, `hpp:173-178` (inline template definition)

**Description**: Convenience template that calls `set_data` with the raw byte representation of object `t`.

**Algorithm**:
```cpp
set_data(reinterpret_cast<const char *>(&t), sizeof(t));
```

**Use case**: Making a record point to a POD struct or fixed-size object, e.g., for serial objects in `src/query/serial.c`.

**Warning**: Only safe for POD/trivially-copyable types. No static_assert enforces this.

---

### 6.18 `set_record_length`

```cpp
void set_record_length(std::size_t length);
```

**Location**: `hpp:100`, `cpp:229-235`

**Description**: Updates `m_recdes.length` to reflect the actual used portion of the buffer.

**Preconditions (assert)**:
1. `is_mutable()` — must be in COPIED or NEW state.
2. `m_recdes.area_size >= 0 && length <= (size_t)m_recdes.area_size` — new length must fit in allocated buffer.

**Use case**: After directly writing into the buffer via `get_data_for_modify`, the caller must update the length.

---

### 6.19 `set_type`

```cpp
void set_type(std::int16_t type);
```

**Location**: `hpp:101`, `cpp:237-242`

**Description**: Sets the record type tag (`REC_HOME`, `REC_BIGONE`, etc.).

**Precondition**: `check_changes_are_permitted()` — must be mutable.

---

### 6.20 `set_external_buffer` (char* overload)

```cpp
void set_external_buffer(char *buf, std::size_t buf_size);
```

**Location**: `hpp:130`, `cpp:244-251`

**Description**: Points the record at a caller-managed heap buffer, releasing any previously owned buffer.

**Algorithm**:
1. `m_own_data.freemem()` — releases the internal owned buffer (if any).
2. Sets `m_recdes.data = buf` and `m_recdes.area_size = (int)buf_size`.
3. Sets `m_data_source = NEW` (mutable, but not owned by `m_own_data`).

**Warning**: After this call, the record is in NEW state and is mutable, but the buffer is NOT owned by `m_own_data`. The caller retains responsibility for freeing `buf`. This is an escape hatch for performance-sensitive code.

---

### 6.21 `set_external_buffer` (stack_block template overload)

```cpp
template <std::size_t S>
void set_external_buffer(cubmem::stack_block<S> &membuf);
```

**Location**: `hpp:131-132`, `hpp:163-171` (inline template)

**Description**: Points the record at a stack-allocated buffer.

**Algorithm**:
1. `m_own_data.freemem()`.
2. Sets `m_recdes.area_size = membuf.SIZE` (compile-time constant `S`).
3. Sets `m_recdes.data = membuf.get_ptr()`.
4. Sets `m_data_source = NEW`.

**Warning**: Same as char* overload — stack lifetime. Safe only while the `stack_block` remains in scope. Suitable for temporary record construction on the stack to avoid heap allocation.

---

### 6.22 `release_buffer`

```cpp
void release_buffer(char *&data, std::size_t &size);
```

**Location**: `hpp:133`, `cpp:354-359`

**Description**: Transfers ownership of the internal buffer to the caller.

**Algorithm**:
1. `size = m_own_data.get_size()`.
2. `data = m_own_data.release_ptr()` — sets `m_own_data`'s internal pointer to null.

**Post-condition**: `m_own_data` no longer owns the buffer; `data` now points to the (heap-allocated) buffer. The caller is responsible for deallocation. `m_recdes.data` still points to the now-detached buffer.

**Use case**: Transferring a built record buffer to a C API that will free it, or handing it to another owner.

---

### 6.23 `move_data`

```cpp
void move_data(std::size_t dest_offset, std::size_t source_offset);
```

**Location**: `hpp:117`, `cpp:253-284`

**Description**: Shifts the tail of the record data from `source_offset` to `dest_offset`, adjusting record length. This is the C++ replacement for the `RECORD_MOVE_DATA` macro.

**Precondition**: `check_changes_are_permitted()`.

**Algorithm**:
1. If `dest_offset == source_offset`: no-op, return.
2. `rec_size = get_size()`.
3. Assert `rec_size >= source_offset`.
4. `memmove_size = rec_size - source_offset` (bytes from `source_offset` to end).
5. `new_size = rec_size + dest_offset - source_offset`.
6. If `dest_offset > source_offset`: growing — call `resize_buffer(new_size)` first.
7. If `memmove_size > 0`: `std::memmove(data + dest_offset, data + source_offset, memmove_size)`.
8. `m_recdes.length = (int)new_size`.

**Diagram**:
```
Before (source_offset < dest_offset — growing):
  [prefix][TAIL DATA...]
           ^src          ^end

After:
  [prefix][GAP      ][TAIL DATA...]
                      ^dest         ^new end
```

**Use case**: Opening a hole in the record to insert data; shrinking a record by moving tail data forward.

---

### 6.24 `modify_data`

```cpp
void modify_data(std::size_t offset, std::size_t old_size, std::size_t new_size, const char *new_data);
```

**Location**: `hpp:108`, `cpp:286-297`

**Description**: Replaces `old_size` bytes at `offset` with `new_size` bytes from `new_data`. C++ replacement for `RECORD_REPLACE_DATA` macro.

**Precondition**: `check_changes_are_permitted()`.

**Algorithm**:
1. Call `move_data(offset + new_size, offset + old_size)` — shifts tail to make room for (or close gap after) the replacement.
2. If `new_size > 0`: `std::memcpy(data + offset, new_data, new_size)` — write new content.

**Use case**: In-place field update within a record without reallocating.

---

### 6.25 `delete_data`

```cpp
void delete_data(std::size_t offset, std::size_t data_size);
```

**Location**: `hpp:111`, `cpp:299-306`

**Description**: Removes `data_size` bytes from `offset`, closing the gap.

**Precondition**: `check_changes_are_permitted()`.

**Algorithm**: Calls `move_data(offset, offset + data_size)` — moves tail forward over the deleted region.

---

### 6.26 `insert_data`

```cpp
void insert_data(std::size_t offset, std::size_t new_size, const char *new_data);
```

**Location**: `hpp:114`, `cpp:308-312`

**Description**: Inserts `new_size` bytes from `new_data` at `offset`, shifting existing content right.

**Algorithm**: Calls `modify_data(offset, 0, new_size, new_data)` — old_size=0 means pure insertion.

---

### 6.27 `check_changes_are_permitted` (private)

```cpp
void check_changes_are_permitted(void) const;
```

**Location**: `hpp:138`, `cpp:314-318`

**Description**: Debug guard — asserts `is_mutable()`.

**Algorithm**: `assert(is_mutable())`. In release builds, this is a no-op (asserts are compiled out with `NDEBUG`).

---

### 6.28 `is_mutable` (private)

```cpp
bool is_mutable() const;
```

**Location**: `hpp:139`, `cpp:320-324`

**Description**: Returns true if the record data may be modified.

**Algorithm**:
```cpp
return m_data_source == data_source::COPIED || m_data_source == data_source::NEW;
```

`PEEKED` and `IMMUTABLE` are explicitly read-only. `INVALID` is not mutable either — you must initialise first.

---

### 6.29 `pack`

```cpp
void pack(cubpacking::packer &packer) const override;
```

**Location**: `hpp:119`, `cpp:326-331`

**Description**: Serialises the record into the packer's output buffer for network transmission or disk logging.

**Algorithm**:
1. `packer.pack_short(m_recdes.type)` — serialise 16-bit record type.
2. `packer.pack_buffer_with_length(m_recdes.data, m_recdes.length)` — serialise length-prefixed data.

**Override**: From `cubpacking::packable_object`.

---

### 6.30 `unpack`

```cpp
void unpack(cubpacking::unpacker &unpacker) override;
```

**Location**: `hpp:120`, `cpp:333-343`

**Description**: Deserialises a record from the unpacker's input buffer.

**Algorithm**:
1. Force `m_data_source = COPIED` (required so `resize_buffer` does not reject the call — it checks `INVALID || is_mutable()`).
2. `unpacker.unpack_short(m_recdes.type)`.
3. `unpacker.peek_unpack_buffer_length(m_recdes.length)` — reads the length prefix without consuming the data.
4. `resize_buffer(m_recdes.length)` — allocates owned buffer large enough.
5. `unpacker.unpack_buffer_with_length(m_recdes.data, m_recdes.length)` — copies data into the owned buffer.

**Note**: Step 1 sets `COPIED` before the buffer exists to satisfy `resize_buffer`'s precondition. This is the one place where the state is set before the data is actually valid; it is immediately corrected in steps 3-5.

---

### 6.31 `get_packed_size`

```cpp
std::size_t get_packed_size(cubpacking::packer &packer, std::size_t curr_offset) const override;
```

**Location**: `hpp:121`, `cpp:345-352`

**Description**: Returns the number of bytes required to serialise this record.

**Algorithm**:
1. `entry_size = packer.get_packed_short_size(curr_offset)` — size of the type field.
2. `entry_size += packer.get_packed_buffer_size(m_recdes.data, m_recdes.length, entry_size)` — size of length-prefixed data.
3. Return `entry_size`.

**Use case**: Called before `pack` to pre-allocate the output buffer.

---

## 7. Key Algorithms & Logic Flows

### 7.1 Record Read Flow (COPY mode)

```
Caller                record_descriptor          spage_get_record
  |                         |                          |
  |-- copy(thd, page, sid) ->|                          |
  |                    get(mode=COPY)                   |
  |                         |-- spage_get_record() ---->|
  |                         |<-- S_DOESNT_FIT (length=-N)|
  |                    resize_buffer(N)                  |
  |                    m_own_data.extend_to(N)            |
  |                    m_recdes.data = new ptr            |
  |                         |-- spage_get_record() ---->|
  |                         |<-- S_SUCCESS              |
  |                    update_source(COPIED)             |
  |<-- NO_ERROR ------------|                           |
```

The auto-resize-and-retry loop is the critical feature: the first call probes the required size (by getting `S_DOESNT_FIT` with the negative length), then the second call succeeds.

### 7.2 Record Read Flow (PEEK mode)

```
Caller                record_descriptor          spage_get_record
  |                         |                          |
  |-- peek(thd, page, sid) ->|                          |
  |                    get(mode=PEEK)                   |
  |                         |-- spage_get_record() ---->|
  |                         |<-- S_SUCCESS (data->page buffer)
  |                    update_source(PEEKED)            |
  |<-- NO_ERROR ------------|                           |
```

No allocation occurs. `m_recdes.data` points into the page buffer. The page latch must be held by the caller.

### 7.3 Data Modification Flow

All mutations follow this pattern:

```
modify_data(offset, old_sz, new_sz, new_data)
  └─> check_changes_are_permitted()    [assert is_mutable]
  └─> move_data(offset+new_sz, offset+old_sz)
        └─> check_changes_are_permitted()
        └─> [if growing] resize_buffer(new_size)
        └─> std::memmove(...)
        └─> m_recdes.length = new_size
  └─> std::memcpy(data+offset, new_data, new_sz)
```

`delete_data` and `insert_data` are convenience wrappers around `move_data` and `modify_data` respectively.

### 7.4 Serialisation Flow

```
[Sender]
  get_packed_size(packer, offset)
    -> packer.get_packed_short_size(offset)
    -> packer.get_packed_buffer_size(data, length, ...)
    -> total_bytes
  pack(packer)
    -> packer.pack_short(type)
    -> packer.pack_buffer_with_length(data, length)

[Receiver]
  unpack(unpacker)
    -> m_data_source = COPIED
    -> unpacker.unpack_short(type)
    -> unpacker.peek_unpack_buffer_length(length)
    -> resize_buffer(length)
    -> unpacker.unpack_buffer_with_length(data, length)
```

---

## 8. Concurrency & Thread Safety

### 8.1 Thread Safety of the Object

`record_descriptor` instances are **not thread-safe**. There is no internal synchronisation (no mutex, no atomic fields). Each instance is intended to be owned by a single thread.

### 8.2 Thread Entry Parameter

The `peek`, `copy`, and `get` methods accept `cubthread::entry *thread_p` and pass it directly to `spage_get_record`. This:
- Identifies the calling thread for page latch tracking.
- Is required by the server's page buffer manager to associate latch ownership.
- Is irrelevant to the internal state of `record_descriptor` itself.

### 8.3 Page Latch Dependency (PEEK mode)

After `peek()` succeeds, `m_recdes.data` points into a page buffer. The page must remain latched (at least S-latch) while the data is accessed. `record_descriptor` does not manage this latch — the caller must:

1. Latch the page before calling `peek`.
2. Keep the latch while reading through the descriptor.
3. Release the latch when done.

In COPY mode, the page latch can be released after `copy()` returns because the data is in the owned buffer.

### 8.4 Allocator Thread Affinity

The default `PRIVATE_BLOCK_ALLOCATOR` allocates from CUBRID's thread-private heap (`db_private_alloc`). This heap is per-thread — using a `record_descriptor` across threads is undefined behaviour if the default allocator is used. Callers that need cross-thread records should use `cubmem::STANDARD_BLOCK_ALLOCATOR` or `cubmem::CSTYLE_BLOCK_ALLOCATOR`.

---

## 9. Memory Management

### 9.1 Ownership Model

| State | Buffer owned by | Who frees |
|---|---|---|
| `INVALID` | Nobody | N/A |
| `PEEKED` | Page buffer manager | Page unfix / buffer eviction |
| `COPIED` | `m_own_data` (extensible_block) | `~record_descriptor()` |
| `NEW` | `m_own_data` (if not via set_external_buffer) | `~record_descriptor()` |
| `NEW` (via `set_external_buffer`) | External caller | Caller |
| `IMMUTABLE` | External caller | Caller |

### 9.2 `extensible_block` Lifecycle

- Construction: `m_own_data` is constructed with the given allocator but holds no memory (block.dim=0, block.ptr=NULL).
- First allocation: happens in `resize_buffer` → `m_own_data.extend_to(n)`.
- Subsequent resizes: `extend_to` is idempotent if `n <= current_size`. Calls `extend_by(delta)` when growth needed, which calls `m_allocator->m_alloc_f(block, new_total)`. The allocator is expected to `realloc`-style (preserve content).
- Destruction: `~extensible_block` calls `m_allocator->m_dealloc_f(m_block)` — releases the buffer.
- Move: `operator=(extensible_block&&)` transfers `block.ptr` and `block.dim`, zeroes the source.

### 9.3 `release_buffer` Semantics

After `release_buffer(data, size)`:
- `m_own_data.m_block.ptr = NULL`, `m_own_data.m_block.dim = 0`.
- `m_recdes.data` still points to the (now-detached) buffer.
- The `~record_descriptor` destructor will call `m_own_data`'s destructor, which calls `dealloc_f` on a null block — this is safe (`freemem` / `delete[] NULL` is a no-op).
- Caller must free `data` using the matching deallocation function for the original allocator.

### 9.4 No `free_and_init` Usage

`record_descriptor` does not use the project-wide `free_and_init` macro because it uses C++ RAII (`extensible_block`) rather than raw C `free`. This is appropriate for a C++ class.

### 9.5 `set_external_buffer` Warning

When `set_external_buffer(buf, buf_size)` is called:
1. `m_own_data.freemem()` releases any existing owned buffer.
2. `m_recdes.data = buf` — now points at caller's buffer.
3. `m_data_source = NEW` — object appears mutable.
4. `m_own_data` is now empty; its destructor is harmless.

If the caller then calls `resize_buffer`, `m_own_data.extend_to` allocates a **new** buffer and `m_recdes.data` is updated — the external `buf` is abandoned (not freed). This is a potential memory leak. Callers must be careful.

---

## 10. Error Handling

### 10.1 Return Codes

Methods that interact with storage return `int`:
- `NO_ERROR` (0) on success.
- `ER_FAILED` (-1) on failure.

All other methods return `void` — they cannot fail in ways that are recoverable (they use assertions instead).

### 10.2 Assertion Strategy

`check_changes_are_permitted()` and nearly all preconditions use `assert()`. In a debug build, assertion failure aborts the process with a stack trace. In a release build (`NDEBUG`), these are no-ops — **there is no runtime check protecting against mutating a PEEKED record in release mode**. Correctness depends on callers respecting the ownership contract.

Key assert sites:
- `set_recdes`: `m_data_source == INVALID`
- `resize_buffer`: `m_data_source == INVALID || is_mutable()`
- `get` (on S_DOESNT_FIT): `m_recdes.length < 0` and `mode == COPY_RECORD`
- `get` (on other failure): `assert(false)` — unexpected state
- `get_recdes`, `get_data`, `get_size`: `m_data_source != INVALID`
- `get_size`: additionally `m_recdes.length > 0`
- `move_data`: `rec_size >= source_offset`
- `set_record_length`: `length <= (size_t)m_recdes.area_size`

### 10.3 No `er_set` in This Layer

`record_descriptor` does not call `er_set`. Error codes are set by `spage_get_record` inside `get()`. The wrapper surfaces `ER_FAILED` without additional annotation. Callers must inspect `er_errid()` for details.

---

## 11. Integration Points

### 11.1 `slotted_page.c` / `slotted_page.h`

**Primary interface**: `spage_get_record`

```c
SCAN_CODE spage_get_record(THREAD_ENTRY *thread_p, PAGE_PTR pgptr,
                           PGSLOTID slotid, RECDES *recdes, int ispeeking);
```

`record_descriptor::get` is a thin safe wrapper around this function. The slotted page layer manages the physical layout of variable-length records on fixed-size database pages. It handles:
- Slot directory (slot ID → offset/length mapping on the page)
- Peeking (returning pointer into the page) vs copying
- Reporting `S_DOESNT_FIT` with the required size in `recdes.length`

`record_descriptor` insulates callers from the raw `RECDES *` interface, the PEEK/COPY integer constants, and the `S_DOESNT_FIT` retry loop.

### 11.2 `heap_file.c`

`heap_file.c` includes `record_descriptor.hpp` and uses it as an output parameter for high-level record operations:

**`heap_attrinfo_transform_to_disk` (line 763, 11898, 11918)**:
```c
SCAN_CODE heap_attrinfo_transform_to_disk(THREAD_ENTRY *thread_p,
    HEAP_CACHE_ATTRINFO *attr_info, RECDES *old_recdes,
    record_descriptor *new_recdes);
```
This function transforms attribute values into a disk-format record, writing into the `record_descriptor`'s managed buffer. Using `record_descriptor` here allows the function to grow the output buffer automatically without the caller pre-sizing it.

**`heap_attrinfo_transform_to_disk_except_lob`** (line 11918): same pattern, excludes LOB attributes.

**`heap_attrinfo_transform_to_disk_internal`** (line 12333): internal implementation shared by both above.

The use of `record_descriptor *` as the output type (rather than raw `RECDES *`) is the modern convention in this codebase — heap-file functions that write new records use `record_descriptor` for auto-sizing.

### 11.3 `locator_sr.c`

Two usage patterns:
1. **Single record**: `build_record()` at line 7367 declares a local `record_descriptor` with `CSTYLE_BLOCK_ALLOCATOR` for building a new record.
2. **Batch insert**: `locator_insert_uniq_multiobj_ordered_updates` at line 13780 accepts `const std::vector<record_descriptor> &recdes` — demonstrating that `record_descriptor` works in standard containers (via move semantics, since copy is deleted).

### 11.4 `load_server_loader.cpp`

Bulk loader constructs a `record_descriptor new_recdes(cubmem::STANDARD_BLOCK_ALLOCATOR)` at line 692 for building records during `loaddb` bulk loading operations. The standard allocator is chosen instead of private to avoid private-heap thread-affinity issues during the loader's multi-threaded operation.

### 11.5 `serial.c`

Serial object updates (line 924) declare:
```c
record_descriptor new_recdesc;
```
Using the default constructor (PRIVATE allocator). Used to build the updated serial counter record.

### 11.6 Serialisation Infrastructure (`packer.hpp`)

`record_descriptor` implements `cubpacking::packable_object`, making it usable in CUBRID's network/log serialisation framework. Key packer methods used:

| Method | Purpose |
|---|---|
| `pack_short(value)` | Pack 16-bit type field |
| `pack_buffer_with_length(ptr, len)` | Pack variable-length data with length prefix |
| `get_packed_short_size(offset)` | Compute aligned size of a short at given offset |
| `get_packed_buffer_size(ptr, len, offset)` | Compute aligned size of length-prefixed buffer |
| `unpack_short(value)` | Unpack 16-bit type field |
| `peek_unpack_buffer_length(len)` | Read length prefix without consuming data |
| `unpack_buffer_with_length(ptr, len)` | Unpack variable-length data |

---

## 12. Complexity & Metrics

### 12.1 Code Metrics

| Metric | Value |
|---|---|
| Total source lines | ~541 (181 + 360) |
| Number of public methods | 22 |
| Number of private methods | 3 |
| Number of constructors | 4 + 1 destructor |
| Template methods | 2 (`set_data_to_object<T>`, `set_external_buffer<S>`) |
| Virtual methods | 3 (`pack`, `unpack`, `get_packed_size`) |
| Lines of actual logic in .cpp | ~230 (rest is boilerplate, comments, blank lines) |
| Cyclomatic complexity (estimate) | Low: most methods have 1-3 branches |
| Highest complexity method | `get()` — 4 branches (success, S_DOESNT_FIT, retry success, failure) |

### 12.2 Method Complexity

| Method | Cyclomatic Complexity | Branches |
|---|---|---|
| `get` | 5 | S_SUCCESS, S_DOESNT_FIT + retry_success, S_DOESNT_FIT + retry_fail, other_fail |
| `move_data` | 4 | dest==src, growing (resize), memmove needed, length update |
| `modify_data` | 2 | delegates + memcpy if new_size > 0 |
| `resize_buffer` | 3 | already large, INVALID→NEW transition |
| `unpack` | 1 | linear |
| `peek`, `copy` | 1 | pure delegation |

### 12.3 Dependencies

- Directly calls into: `spage_get_record` (1 external storage function), `cubpacking::packer/unpacker` (6 methods), `cubmem::extensible_block` (5 methods), `std::memcpy`, `std::memmove`
- Zero dependency on query, transaction, or buffer-pool layers — deliberately narrow scope

---

## 13. Notable Patterns & Idioms

### 13.1 C++ Class Wrapping a C Struct

`record_descriptor` is the canonical example in this codebase of the "C++ wrapper over C struct" pattern. The C struct `recdes` is retained as `m_recdes` for direct passing to C APIs (`spage_get_record`, `heap_attrinfo_*`), while the wrapper adds RAII and safety. The comment block at the top of both files documents the wrapped struct explicitly — a codebase convention.

### 13.2 State Machine as `enum class`

The `data_source` private enum encodes object lifecycle as a state machine, enforced at every mutation point via `check_changes_are_permitted`. This pattern replaces the `area_size < 0` convention of the raw `recdes` struct (where negative `area_size` signalled "peeked from page") with explicit, named states.

### 13.3 Delegating Constructors

All constructors except the primary default delegate to it or to each other, ensuring initialisation logic is not duplicated:

```
record_descriptor(const char*, size_t)
  └─> record_descriptor()    [default]
      └─> set_data(...)

record_descriptor(const recdes&, alloc)
  └─> record_descriptor(alloc)   [default]
      └─> set_recdes(...)

record_descriptor(record_descriptor&&)
  [does NOT delegate — full move logic inline]
```

### 13.4 Move-Only Semantics

No copy constructor or copy-assignment operator is defined. Because `cubmem::extensible_block` deletes its copy operations, the compiler implicitly deletes `record_descriptor`'s copy operations too. This is intentional: you cannot accidentally duplicate a record with its buffer. You must either `set_recdes` to explicit-copy or `std::move` to transfer.

### 13.5 `const_cast` for IMMUTABLE State

In `set_data`, `const char *data` is cast to `char *`:
```cpp
m_recdes.data = const_cast<char *>(data);
```
The comment explicitly notes that the `data_source` state protects against writes. This pattern acknowledges that `recdes.data` is `char *` (the C struct cannot be changed), while the object's state machine enforces the const contract. It is a controlled, documented `const_cast`.

### 13.6 Macro Replacement via Methods

`modify_data` and `move_data` are explicitly documented as replacements for the C macros `RECORD_REPLACE_DATA` and `RECORD_MOVE_DATA` defined in `storage_common.h`. The C++ methods are safer because they:
- Enforce mutability via assert
- Handle buffer growth automatically
- Use typed `size_t` arguments instead of unchecked C macro parameters

### 13.7 Two-Phase Unpack (Peek-Length Pattern)

`unpack` uses a two-step approach: first call `peek_unpack_buffer_length` to read the length without consuming input, then allocate, then actually unpack. This avoids double-allocation and ensures the buffer is exactly the right size before the data copy.

### 13.8 Template + `reinterpret_cast` for Type Punning

`set_data_to_object<T>` uses:
```cpp
set_data(reinterpret_cast<const char *>(&t), sizeof(t));
```
This is the standard C++ idiom for treating an object as a byte array. It is correct for POD types and technically implementation-defined for non-POD, but in this codebase (a C/C++ database engine), this pattern is pervasive and assumed safe on the target platforms (Linux x86-64).

### 13.9 Default Parameter with Named Constant

```cpp
record_descriptor(const cubmem::block_allocator &alloc = cubmem::PRIVATE_BLOCK_ALLOCATOR);
```

The default parameter is a named global constant, not a magic value. This makes the default policy explicit and allows callers that need a different allocator to override it cleanly.

### 13.10 Absence of Exceptions

No exceptions are thrown or caught. Errors propagate via:
- Return codes (`int`) for storage I/O failures
- `assert` for contract violations
- `er_set` (by callee `spage_get_record`) for detailed error codes

This is consistent with the project-wide anti-pattern rule: "Never use C++ exceptions in engine code."

---

*End of report. Total length: ~600+ lines.*
