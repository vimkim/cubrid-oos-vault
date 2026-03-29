# Double Write Buffer — Comprehensive Analysis Report

**Source:** `src/storage/double_write_buffer.cpp`
**Header:** `src/storage/double_write_buffer.hpp`
**Generated:** 2026-03-27
**Analyzer:** Executor (claude-sonnet-4-6)

---

## 1. File Overview

| Property | Value |
|---|---|
| **File path** | `src/storage/double_write_buffer.cpp` |
| **Header** | `src/storage/double_write_buffer.hpp` |
| **Line count** | 4168 lines |
| **Language** | C++17 (compiled as C++ despite `.cpp` extension) |
| **Purpose** | Crash-safe write path: every page is written to the DWB file first, then to its final data volume location |
| **Build modes** | `SERVER_MODE` (cub_server) and `SA_MODE` (standalone); excluded from `CS_MODE` (client-only) |

### Purpose and Motivation

The double-write buffer (DWB) protects against *partial-page writes* — a crash scenario where the OS writes only part of a 16 KB database page before power loss. Because CUBRID pages are larger than the hardware sector size, a torn write can corrupt a page in the data volume. The WAL log contains redo records that reference the *before* state of a page; if the page is partially overwritten with new content, neither the old nor the new state is recoverable from the log alone.

The DWB addresses this by creating a two-phase write protocol:
1. A complete page is written to a dedicated DWB volume file and `fsync`-ed.
2. The same page is written to its final location in the data volume.

At restart, if a data page is found to be corrupted (detected by checksum), the recovery code can restore it from the DWB file before replaying the WAL.

---

## 2. Includes and Dependencies

### Header Includes (double_write_buffer.hpp)

| Header | Purpose |
|---|---|
| `file_io.h` | `FILEIO_PAGE`, `VPID`, `fileio_*` I/O primitives |
| `log_lsa.hpp` | `LOG_LSA` type — Log Sequence Address |

### Implementation Includes (double_write_buffer.cpp)

| Header | Module | Purpose |
|---|---|---|
| `<assert.h>` | System | `assert()` macro |
| `<math.h>` | System | `log()` — used to compute `log2_num_block_pages` |
| `double_write_buffer.hpp` | Self | Forward declarations and public interface |
| `system_parameter.h` | base | `prm_get_integer_value`, `prm_get_bool_value` — read `PRM_ID_DWB_SIZE`, `PRM_ID_DWB_BLOCKS`, `PRM_ID_ENABLE_DWB_FLUSH_THREAD`, `PRM_ID_PB_SYNC_ON_NFLUSH` |
| `thread_daemon.hpp` | thread | `cubthread::daemon` — background flush/sync daemon threads |
| `thread_entry_task.hpp` | thread | `cubthread::entry_task`, `cubthread::entry_callable_task` |
| `thread_lockfree_hash_map.hpp` | thread | `cubthread::lockfree_hashmap<K,V>` — lock-free VPID→slot hash |
| `thread_manager.hpp` | thread | `cubthread::get_manager()` — daemon creation/destruction |
| `log_append.hpp` | transaction | `log_Gl.append.get_nxio_lsa()` — WAL protocol check (debug only) |
| `log_impl.h` | transaction | `logpb_need_wal()` — WAL enforcement check (debug only) |
| `log_volids.hpp` | transaction | `LOG_DBDWB_VOLID` — special volume ID for DWB file |
| `boot_sr.h` | transaction | `boot_db_full_name()`, `BO_IS_FLUSH_DAEMON_AVAILABLE()` |
| `perf_monitor.h` | monitor | `PERF_UTIME_TRACKER`, `perfmon_add_stat`, `PSTAT_*` constants |
| `porting_inline.hpp` | base | `STATIC_INLINE`, `ALWAYS_INLINE`, `ATOMIC_*` atomics, `UINT64` |
| `memory_wrapper.hpp` | base | **Must be last** — memory allocation instrumentation |

### Reverse Dependencies (files that include the DWB header)

| File | Role |
|---|---|
| `src/storage/page_buffer.c` | Primary caller — wraps every dirty-page flush through DWB |
| `src/storage/file_io.c` | Calls `dwb_synchronize`, `dwb_add_page`, `dwb_flush_force` |
| `src/storage/disk_manager.c` | Calls DWB on volume extend/flush paths |
| `src/transaction/boot_sr.c` | Calls `dwb_load_and_recover_pages`, `dwb_create`, `dwb_destroy` at server startup/shutdown |
| `src/transaction/log_page_buffer.c` | Calls `dwb_flush_force` before checkpoint log sync |
| `src/base/system_parameter.c` | Reads DWB-related system parameters |

---

## 3. Preprocessor and Compilation

### Mode Guards

```c
#if defined(SERVER_MODE)
  // daemon thread pointers, thread sleep, wakeup calls
  static cubthread::daemon *dwb_flush_block_daemon = NULL;
  static cubthread::daemon *dwb_file_sync_helper_daemon = NULL;
#endif

#if !defined(SERVER_MODE)
  // Stubs that neutralize pthread calls in SA_MODE
  #define pthread_mutex_init(a, b)
  #define pthread_mutex_destroy(a)
  #define pthread_mutex_lock(a)   0
  #define pthread_mutex_unlock(a)
#endif

#if !defined(NDEBUG)
  // dwb_debug_check_dwb() — duplicate detection in ordered slots
  // WAL assertion checks during flush
#endif
```

In `SA_MODE` (standalone), the pthread mutex macros expand to nothing or zero, so all lock/unlock calls are eliminated by the preprocessor. This means the DWB works in single-threaded standalone mode without any mutex overhead. Daemon threads are also disabled in `SA_MODE`.

The `CS_MODE` guard appears in reverse dependencies (`file_io.c`, `page_buffer.c`) to skip DWB calls on the client side — the DWB is a server/standalone concern only.

### Key Compile-Time Constants

| Macro | Value | Meaning |
|---|---|---|
| `DWB_MIN_SIZE` | `512 * 1024` (512 KB) | Minimum DWB buffer size |
| `DWB_MAX_SIZE` | `32 * 1024 * 1024` (32 MB) | Maximum DWB buffer size |
| `DWB_MIN_BLOCKS` | `1` | Minimum number of blocks |
| `DWB_MAX_BLOCKS` | `32` | Maximum number of blocks |
| `DWB_SLOTS_HASH_SIZE` | `1000` | Hash table bucket count |
| `DWB_SLOTS_FREE_LIST_SIZE` | `100` | Free-list block size for lock-free hash |

All size/count values **must be powers of 2** (enforced by `assert(IS_POWER_OF_2(...))`).

### Position-With-Flags Bit Layout

The central synchronization variable `position_with_flags` (64-bit) encodes:

```
Bits 63–32: Block status flags (1 bit per block, bit 63 = block 0, bit 32 = block 31)
Bit 31:     DWB_MODIFY_STRUCTURE — structure modification in progress
Bit 30:     DWB_CREATE — DWB is created and active
Bits 29–0:  Current write position (slot index, wraps at DWB_NUM_TOTAL_PAGES-1)
```

---

## 4. Data Structures and Types

### 4.1 `DWB_SLOT` (public — defined in .hpp)

```c
struct double_write_slot
{
  FILEIO_PAGE *io_page;         // Pointer into block's write_buffer (never heap-allocated alone)
  VPID vpid;                    // (volid, pageid) of the contained page; NULL_VPID = empty/invalid slot
  LOG_LSA lsa;                  // Page LSA copied from io_page->prv.lsa at slot assignment
  bool ensure_metadata;         // If true, fileio_synchronize must sync filesystem metadata
  unsigned int position_in_block; // Slot index within its block [0, num_block_pages)
  unsigned int block_no;        // Block index [0, num_blocks)
};
```

**Key invariants:**
- `io_page` always points into `block->write_buffer + position_in_block * IO_PAGESIZE`; it is never freed independently.
- `VPID_ISNULL(&slot->vpid)` means the slot carries no valid page data and should be skipped during flush.
- `lsa` is copied at `dwb_set_slot_data()` time and used for hash deduplication (newer LSA wins).

### 4.2 `DWB_WAIT_QUEUE_ENTRY`

```c
struct double_write_wait_queue_entry
{
  void *data;                   // Points to a THREAD_ENTRY* (the waiting thread)
  DWB_WAIT_QUEUE_ENTRY *next;   // Intrusive singly-linked list
};
```

Entries are pooled: freed entries go to `free_list` and are reused before `malloc`. The `data` field is deliberately not cleared on free for debug convenience.

### 4.3 `DWB_WAIT_QUEUE`

```c
struct double_write_wait_queue
{
  DWB_WAIT_QUEUE_ENTRY *head;      // First entry (FIFO dequeue from head)
  DWB_WAIT_QUEUE_ENTRY *tail;      // Last entry (enqueue at tail)
  DWB_WAIT_QUEUE_ENTRY *free_list; // Recycled entries (LIFO stack)
  int count;                       // Active entry count
  int free_count;                  // Recycled entry count
};
```

Initializer: `DWB_WAIT_QUEUE_INITIALIZER` = `{NULL, NULL, NULL, 0, 0}`.

Two instances exist: one per `DWB_BLOCK` (for block-completion waits) and one in `DOUBLE_WRITE_BUFFER` (for structure-modification waits).

### 4.4 `FLUSH_VOLUME_STATUS` (enum)

```c
typedef enum {
  VOLUME_NOT_FLUSHED,
  VOLUME_FLUSHED_BY_DWB_FILE_SYNC_HELPER_THREAD,
  VOLUME_FLUSHED_BY_DWB_FLUSH_THREAD
} FLUSH_VOLUME_STATUS;
```

Used for coordination between the flush-block thread and the file-sync-helper thread. CAS operations on this field determine which thread "wins" the right to call `fileio_synchronize` for a given volume.

### 4.5 `FLUSH_VOLUME_INFO`

```c
struct flush_volume_info
{
  int vdes;                         // Volume file descriptor
  volatile int num_pages;           // Pages written to this volume in current block (decremented by syncer)
  volatile bool all_pages_written;  // True when dwb_write_block completes all writes to this volume
  volatile bool metadata;           // Whether fsync must flush metadata (inode/directory changes)
  volatile FLUSH_VOLUME_STATUS flushed_status; // Who is responsible for syncing
};
```

One `FLUSH_VOLUME_INFO` per distinct volume touched in a DWB block. Sized to `num_block_pages` entries per block (upper bound). The `count_flush_volumes_info` field tracks how many are actually populated in a flush cycle.

### 4.6 `DWB_BLOCK`

```c
struct double_write_block
{
  FLUSH_VOLUME_INFO *flush_volumes_info;      // Array of volume flush info, size = max_to_flush_vdes
  volatile unsigned int count_flush_volumes_info; // How many volumes currently being tracked
  unsigned int max_to_flush_vdes;             // Capacity of flush_volumes_info array

  pthread_mutex_t mutex;           // Protects wait_queue
  DWB_WAIT_QUEUE wait_queue;       // Threads waiting for this block to complete flush

  char *write_buffer;              // Contiguous buffer: num_block_pages * IO_PAGESIZE bytes
  DWB_SLOT *slots;                 // Array of num_block_pages slots; slot[i].io_page = write_buffer + i*IO_PAGESIZE
  volatile unsigned int count_wb_pages; // How many pages have been committed to this block

  unsigned int block_no;           // Immutable block index
  volatile UINT64 version;         // Incremented each time the block completes a flush cycle
  volatile bool all_pages_written; // Set true when dwb_write_block finishes; signals file_sync_helper
};
```

**Key design points:**
- `write_buffer` and `slots` are allocated together during `dwb_create_blocks` and freed in `dwb_finalize_block`.
- `count_wb_pages` is incremented atomically by `dwb_add_page`. When it reaches `DWB_BLOCK_NUM_PAGES`, the block is full and ready for flush.
- `version` starts at 0 and is incremented by `ATOMIC_INC_64(&block->version, 1ULL)` at the end of every successful `dwb_flush_block`. It is used to establish flush ordering.
- `all_pages_written` is a fence between `dwb_write_block` (writer) and `dwb_file_sync_helper` (reader).

### 4.7 `DWB_SLOTS_HASH_ENTRY`

```c
struct dwb_slots_hash_entry
{
  VPID vpid;                    // Hash key — the page identifier
  DWB_SLOTS_HASH_ENTRY *stack;  // Lock-free freelist linkage
  DWB_SLOTS_HASH_ENTRY *next;   // Hash bucket chain linkage
  pthread_mutex_t mutex;        // Per-entry mutex (LF_EM_USING_MUTEX mode)
  UINT64 del_id;                // Lock-free delete transaction ID
  DWB_SLOT *slot;               // The most-recently-inserted slot for this VPID
  // Constructor/destructor init/destroy the mutex
};
```

The lock-free hash uses `LF_ENTRY_DESCRIPTOR` to wire up the allocator, key comparison, and hash function callbacks. The per-entry mutex enables safe compare-and-swap of `slot` under the `lockfree_hashmap::find_or_insert` protocol (the caller holds the mutex until it releases via `pthread_mutex_unlock`).

`dwb_hashmap_type` is aliased as `cubthread::lockfree_hashmap<VPID, dwb_slots_hash_entry>`.

### 4.8 `DOUBLE_WRITE_BUFFER` (the global singleton)

```c
struct double_write_buffer
{
  bool logging_enabled;                 // Runtime logging toggle (PRM_ID_DWB_LOGGING)

  DWB_BLOCK *blocks;                    // Array of num_blocks DWB_BLOCK structs
  unsigned int num_blocks;              // Total blocks (power of 2, [1,32])
  unsigned int num_pages;               // Total pages = num_blocks * num_block_pages
  unsigned int num_block_pages;         // Pages per block (power of 2)
  unsigned int log2_num_block_pages;    // log2(num_block_pages) — used for block-no extraction from position

  volatile unsigned int blocks_flush_counter; // Currently 0 or 1 (only one flush allowed at a time)
  volatile unsigned int next_block_to_flush;  // Next block index the flush daemon should flush

  pthread_mutex_t mutex;                // Protects global wait_queue
  DWB_WAIT_QUEUE wait_queue;            // Threads waiting for structure modification to end

  UINT64 volatile position_with_flags; // THE central synchronization word (see bit layout above)

  dwb_hashmap_type slots_hashmap;       // VPID → DWB_SLOT* hash (most-recent version per page)
  int vdes;                             // File descriptor for the DWB volume file

  DWB_BLOCK *volatile file_sync_helper_block; // Block being processed by file-sync-helper daemon (NULL = none)
};
```

There is exactly one instance: `static DOUBLE_WRITE_BUFFER dwb_Global;` (file-static, zero-initialized by C++).

The `position_with_flags` field is the heart of lock-free coordination. All state transitions (slot acquisition, block start/end, structure modification, creation) go through atomic CAS operations on this single 64-bit word.

---

## 5. Global and Static Variables

| Variable | Type | Scope | Description |
|---|---|---|---|
| `dwb_Volume_name` | `char[PATH_MAX]` | File-global (extern) | Full path of the DWB volume file, set in `dwb_create` via `fileio_make_dwb_name` |
| `dwb_Global` | `static DOUBLE_WRITE_BUFFER` | File-static | The singleton DWB instance |
| `dwb_flush_block_daemon` | `static cubthread::daemon*` | File-static, SERVER_MODE only | Daemon that calls `dwb_flush_next_block` every 1 ms |
| `dwb_file_sync_helper_daemon` | `static cubthread::daemon*` | File-static, SERVER_MODE only | Daemon that calls `dwb_file_sync_helper` every 10 ms |
| `slots_entry_Descriptor` | `static LF_ENTRY_DESCRIPTOR` | File-static | Wires the lock-free hash to the `DWB_SLOTS_HASH_ENTRY` allocator callbacks |

### Logging Macros

```c
#define dwb_Log  dwb_Global.logging_enabled
#define dwb_check_logging()  (dwb_Log = prm_get_bool_value(PRM_ID_DWB_LOGGING))
#define dwb_log(...)         if (dwb_Log) _er_log_debug(ARG_FILE_LINE, "DWB: " __VA_ARGS__)
#define dwb_log_error(...)   if (dwb_Log) _er_log_debug(ARG_FILE_LINE, "DWB ERROR: " __VA_ARGS__)
```

Logging is gated by the `PRM_ID_DWB_LOGGING` system parameter and only calls `_er_log_debug` when enabled — zero overhead in production.

---

## 6. Function Catalog

This section documents every function in declaration order. All functions are `STATIC_INLINE` (file-private inlined) unless otherwise noted.

---

### 6.1 Wait Queue Functions

#### `dwb_init_wait_queue`
```c
STATIC_INLINE void dwb_init_wait_queue(DWB_WAIT_QUEUE *wait_queue)
```
- **Lines:** 448–457
- **Visibility:** Static inline
- **Purpose:** Zero-initializes a wait queue (head, tail, free_list = NULL; counts = 0).
- **Callers:** `dwb_initialize_block`, `dwb_create_internal`
- **Algorithm:** Straight field assignment; no locking needed (called during initialization).

---

#### `dwb_make_wait_queue_entry`
```c
STATIC_INLINE DWB_WAIT_QUEUE_ENTRY* dwb_make_wait_queue_entry(DWB_WAIT_QUEUE *wait_queue, void *data)
```
- **Lines:** 466–492
- **Purpose:** Allocate (or recycle from free_list) a wait queue entry and set its `data` pointer.
- **Algorithm:** If `free_list` is non-null, pops the head entry from it (O(1)). Otherwise, `malloc`s a new entry. Sets `data` and `next=NULL`.
- **Error handling:** On `malloc` failure, sets `ER_OUT_OF_VIRTUAL_MEMORY` and returns `NULL`.
- **Callers:** `dwb_block_add_wait_queue_entry`

---

#### `dwb_block_add_wait_queue_entry`
```c
STATIC_INLINE DWB_WAIT_QUEUE_ENTRY* dwb_block_add_wait_queue_entry(DWB_WAIT_QUEUE *wait_queue, void *data)
```
- **Lines:** 503–528
- **Purpose:** Append a new entry to the tail of the wait queue.
- **Note:** Caller is responsible for holding the queue's mutex.
- **Algorithm:** Calls `dwb_make_wait_queue_entry`, links into tail (or sets head==tail if empty), increments `count`.
- **Callers:** `dwb_wait_for_block_completion`, `dwb_wait_for_strucure_modification`

---

#### `dwb_block_disconnect_wait_queue_entry`
```c
STATIC_INLINE DWB_WAIT_QUEUE_ENTRY* dwb_block_disconnect_wait_queue_entry(DWB_WAIT_QUEUE *wait_queue, void *data)
```
- **Lines:** 538–590
- **Purpose:** Remove and return an entry from the wait queue by its `data` pointer (or the head if `data==NULL`).
- **Algorithm:** O(n) linear scan when `data != NULL`; O(1) when `data == NULL`. Repairs head/tail pointers and decrements `count`.
- **Callers:** `dwb_remove_wait_queue_entry`, `dwb_signal_waiting_threads`

---

#### `dwb_block_free_wait_queue_entry`
```c
STATIC_INLINE void dwb_block_free_wait_queue_entry(DWB_WAIT_QUEUE *wait_queue,
    DWB_WAIT_QUEUE_ENTRY *wait_queue_entry, int (*func)(void *))
```
- **Lines:** 600–618
- **Purpose:** Return an entry to the free_list (and optionally apply a callback on it, e.g., to wake the thread).
- **Algorithm:** Calls `func(wait_queue_entry)` if non-null, then pushes entry onto free_list stack.
- **Note:** `data` field is preserved for debug inspection after recycling.

---

#### `dwb_remove_wait_queue_entry`
```c
STATIC_INLINE void dwb_remove_wait_queue_entry(DWB_WAIT_QUEUE *wait_queue,
    pthread_mutex_t *mutex, void *data, int (*func)(void *))
```
- **Lines:** 631–651
- **Purpose:** Find, disconnect, and free a wait queue entry under optional mutex protection.
- **Algorithm:** Optionally locks mutex, calls `dwb_block_disconnect_wait_queue_entry` then `dwb_block_free_wait_queue_entry`, unlocks.
- **Callers:** `dwb_signal_waiting_threads`, `dwb_wait_for_block_completion` (timeout path), `dwb_wait_for_strucure_modification` (timeout path)

---

#### `dwb_signal_waiting_threads`
```c
STATIC_INLINE void dwb_signal_waiting_threads(DWB_WAIT_QUEUE *wait_queue, pthread_mutex_t *mutex)
```
- **Lines:** 660–679
- **Purpose:** Wake all threads in a wait queue.
- **Algorithm:** Under optional mutex, drains the queue by calling `dwb_remove_wait_queue_entry(NULL, dwb_signal_waiting_thread)` in a loop until the queue is empty.
- **Callers:** `dwb_destroy_wait_queue`, `dwb_signal_block_completion`, `dwb_signal_structure_modificated`

---

#### `dwb_destroy_wait_queue`
```c
STATIC_INLINE void dwb_destroy_wait_queue(DWB_WAIT_QUEUE *wait_queue, pthread_mutex_t *mutex)
```
- **Lines:** 688–717
- **Purpose:** Signal all waiting threads and then free all recycled entries in the free_list.
- **Algorithm:** Calls `dwb_signal_waiting_threads` first, then walks `free_list` calling `free_and_init` on each.
- **Callers:** `dwb_finalize_block`, `dwb_destroy_internal`

---

### 6.2 Configuration Helpers

#### `dwb_power2_ceil`
```c
STATIC_INLINE void dwb_power2_ceil(unsigned int min, unsigned int max, unsigned int *p_value)
```
- **Lines:** 729–756
- **Purpose:** Clamp `*p_value` to `[min, max]` and round up to the next power of 2.
- **Algorithm:** If below min, set to min. If above max, set to max. Otherwise, left-shift from min until exceeding value.
- **Callers:** `dwb_load_buffer_size`, `dwb_load_block_count`

---

#### `dwb_load_buffer_size`
```c
STATIC_INLINE bool dwb_load_buffer_size(unsigned int *p_double_write_buffer_size)
```
- **Lines:** 766–782
- **Purpose:** Read `PRM_ID_DWB_SIZE`; return false (disable DWB) if 0; otherwise snap to valid power-of-2 in [512K, 32M].
- **Callers:** `dwb_create_internal`

---

#### `dwb_load_block_count`
```c
STATIC_INLINE bool dwb_load_block_count(unsigned int *p_num_blocks)
```
- **Lines:** 792–808
- **Purpose:** Read `PRM_ID_DWB_BLOCKS`; return false if 0; otherwise snap to power-of-2 in [1, 32].
- **Callers:** `dwb_create_internal`

---

### 6.3 Structure Modification Guards

#### `dwb_starts_structure_modification`
```c
STATIC_INLINE int dwb_starts_structure_modification(THREAD_ENTRY *thread_p, UINT64 *current_position_with_flags)
```
- **Lines:** 819–912
- **Visibility:** Static inline
- **Purpose:** Acquire the "structure modification lock" by atomically setting `DWB_MODIFY_STRUCTURE` in `position_with_flags`. Flushes all in-progress blocks before returning.
- **Algorithm:**
  1. CAS loop: atomically set `MODIFY_STRUCTURE` bit; fail if already set (only one modifier allowed).
  2. In `SERVER_MODE`: spin-wait until `blocks_flush_counter == 0` and both daemons are idle.
  3. Steal `file_sync_helper_block` if set (flush it locally).
  4. Find all blocks with `BLOCK_WRITE_STARTED` flag; sort by version (oldest first); call `dwb_flush_block` on each.
  5. Assert no blocks remain in-progress; return the new `position_with_flags`.
- **Error handling:** Returns `ER_FAILED` if already modifying. Propagates `dwb_flush_block` errors.
- **Callers:** `dwb_create`, `dwb_recreate`, `dwb_destroy`

---

#### `dwb_ends_structure_modification`
```c
STATIC_INLINE void dwb_ends_structure_modification(THREAD_ENTRY *thread_p, UINT64 current_position_with_flags)
```
- **Lines:** 921–935
- **Purpose:** Release the structure modification lock by clearing `DWB_MODIFY_STRUCTURE` and waking all blocked threads.
- **Algorithm:** Clears bit via `ATOMIC_TAS_64`, then calls `dwb_signal_structure_modificated`.
- **Callers:** `dwb_create`, `dwb_recreate`, `dwb_destroy`

---

### 6.4 Slot and Block Initialization

#### `dwb_initialize_slot`
```c
STATIC_INLINE void dwb_initialize_slot(DWB_SLOT *slot, FILEIO_PAGE *io_page,
    unsigned int position_in_block, unsigned int block_no)
```
- **Lines:** 946–960
- **Purpose:** Bind a slot to its fixed memory location within the block's write_buffer. Sets `io_page`, copies `vpid` and `lsa` from the page, sets `position_in_block` and `block_no`.
- **Note:** Called once at block creation time. `io_page` points into the block's `write_buffer` and is never changed thereafter.

---

#### `dwb_initialize_block`
```c
STATIC_INLINE void dwb_initialize_block(DWB_BLOCK *block, unsigned int block_no,
    unsigned int count_wb_pages, char *write_buffer, DWB_SLOT *slots,
    FLUSH_VOLUME_INFO *flush_volumes_info, unsigned int count_flush_volumes_info,
    unsigned int max_to_flush_vdes)
```
- **Lines:** 975–995
- **Purpose:** Fully initialize a `DWB_BLOCK` struct: set all fields, initialize mutex and wait queue, set version=0, `all_pages_written=false`.

---

#### `dwb_create_blocks`
```c
STATIC_INLINE int dwb_create_blocks(THREAD_ENTRY *thread_p, unsigned int num_blocks,
    unsigned int num_block_pages, DWB_BLOCK **p_blocks)
```
- **Lines:** 1006–1120
- **Visibility:** Static inline
- **Purpose:** Allocate all DWB blocks with their write buffers, slot arrays, and flush volume info arrays. Wire up each slot to its position within the write buffer.
- **Algorithm:**
  1. `malloc` the block array (`num_blocks * sizeof(DWB_BLOCK)`).
  2. For each block: `malloc` `write_buffer` (`num_block_pages * IO_PAGESIZE`), `slots` array, `flush_volumes_info` array.
  3. For each block, for each page slot: call `fileio_initialize_res` on the page header, then `dwb_initialize_slot`.
  4. Call `dwb_initialize_block`.
- **Error handling:** Full `goto exit_on_error` cleanup on any allocation failure — frees all partially-allocated arrays before returning `ER_OUT_OF_VIRTUAL_MEMORY`.
- **Callers:** `dwb_create_internal`, `dwb_load_and_recover_pages` (recovery path creates one temporary block)

---

#### `dwb_finalize_block`
```c
STATIC_INLINE void dwb_finalize_block(DWB_BLOCK *block)
```
- **Lines:** 1128–1148
- **Purpose:** Free all heap memory owned by a block: `slots`, `write_buffer`, `flush_volumes_info`. Destroy the wait queue and mutex.
- **Callers:** `dwb_destroy_internal`, `dwb_create_blocks` (on error), `dwb_load_and_recover_pages` (cleanup)

---

### 6.5 Lifecycle Functions (Public API)

#### `dwb_create_internal`
```c
STATIC_INLINE int dwb_create_internal(THREAD_ENTRY *thread_p, const char *dwb_volume_name,
    UINT64 *current_position_with_flags)
```
- **Lines:** 1160–1251
- **Visibility:** Static inline (called only from `dwb_create` and `dwb_recreate`)
- **Purpose:** Format and open the DWB volume file, create blocks, initialize the global hashmap, set the `DWB_CREATE` flag.
- **Algorithm:**
  1. Load buffer size and block count from parameters; return `NO_ERROR` (DWB disabled) if either is 0.
  2. `fileio_format` — creates the DWB file with `LOG_DBDWB_VOLID`.
  3. `fileio_synchronize_all` — flush dirty pages before DWB is active.
  4. `dwb_create_blocks` — allocate in-memory structures.
  5. Set all `dwb_Global` fields.
  6. `slots_hashmap.init` — initialize lock-free hash with `DWB_SLOTS_HASH_SIZE=1000` buckets.
  7. Atomically set `DWB_CREATE` flag via CAS on `position_with_flags`.
- **Error handling:** On failure after file creation, dismounts and unformats the file; finalizes any partial blocks.

---

#### `dwb_create` (Public)
```c
int dwb_create(THREAD_ENTRY *thread_p, const char *dwb_path_p, const char *db_name_p)
```
- **Lines:** 2922–2956
- **Visibility:** `extern` (public)
- **Purpose:** Idempotent DWB creation: acquires structure modification lock, checks if already created, builds volume name, calls `dwb_create_internal`.
- **Callers:** `boot_sr.c` at server startup; `dwb_load_and_recover_pages` after recovery.

---

#### `dwb_recreate` (Public)
```c
int dwb_recreate(THREAD_ENTRY *thread_p)
```
- **Lines:** 2964–2998
- **Visibility:** `extern` (public)
- **Purpose:** Destroy and re-create the DWB with current parameter values (used when `PRM_ID_DWB_SIZE` or `PRM_ID_DWB_BLOCKS` changes at runtime).
- **Algorithm:** Acquires modification lock; if currently created, calls `dwb_destroy_internal`; then `dwb_create_internal`.

---

#### `dwb_destroy_internal`
```c
STATIC_INLINE void dwb_destroy_internal(THREAD_ENTRY *thread_p, UINT64 *current_position_with_flags)
```
- **Lines:** 1471–1510
- **Purpose:** Tear down all DWB in-memory structures and the volume file. Clear the `DWB_CREATE` flag.
- **Algorithm:** Destroy global wait queue, finalize each block, free `blocks` array, destroy hashmap, dismount and unformat volume file. Clear `DWB_CREATE` bit via CAS.

---

#### `dwb_destroy` (Public)
```c
int dwb_destroy(THREAD_ENTRY *thread_p)
```
- **Lines:** 3400–3431
- **Visibility:** `extern` (public)
- **Purpose:** Public teardown: lock modification, call `dwb_destroy_internal`, unlock, then destroy daemons.
- **Callers:** `boot_sr.c` at shutdown.

---

#### `dwb_is_created` (Public)
```c
bool dwb_is_created(void)
```
- **Lines:** 2906–2912
- **Visibility:** `extern` (public)
- **Purpose:** Atomically read `DWB_CREATE` bit from `position_with_flags`.
- **Note:** Lock-free; safe to call from any thread without mutex.

---

#### `dwb_get_volume_name` (Public)
```c
char *dwb_get_volume_name(void)
```
- **Lines:** 3437–3448
- **Visibility:** `extern` (public)
- **Purpose:** Return `dwb_Volume_name` if DWB is created, otherwise NULL.

---

### 6.6 Slot Hash Functions

#### `dwb_slots_hash_entry_alloc`
```c
static void *dwb_slots_hash_entry_alloc(void)
```
- **Lines:** 1258–1273
- **Purpose:** Allocate a new `DWB_SLOTS_HASH_ENTRY` via `malloc` and initialize its mutex.
- **Callback:** Registered in `slots_entry_Descriptor.f_alloc`.

---

#### `dwb_slots_hash_entry_free`
```c
static int dwb_slots_hash_entry_free(void *entry)
```
- **Lines:** 1280–1294
- **Purpose:** Destroy the per-entry mutex and `free()` the entry. Note: uses raw `free()` not `free_and_init` — this is intentional since the pointer itself is being freed.
- **Callback:** Registered in `slots_entry_Descriptor.f_free`.

---

#### `dwb_slots_hash_entry_init`
```c
static int dwb_slots_hash_entry_init(void *entry)
```
- **Lines:** 1301–1315
- **Purpose:** Reset an entry's VPID to null and slot pointer to NULL (called when recycling from freelist).
- **Callback:** Registered in `slots_entry_Descriptor.f_init`.

---

#### `dwb_slots_hash_key_copy`
```c
static int dwb_slots_hash_key_copy(void *src, void *dest)
```
- **Lines:** 1323–1333
- **Purpose:** Copy a `VPID` key using `VPID_COPY`.

---

#### `dwb_slots_hash_compare_key`
```c
static int dwb_slots_hash_compare_key(void *key1, void *key2)
```
- **Lines:** 1342–1351
- **Purpose:** Return 0 if `VPID_EQ`, -1 otherwise. Used by the hash table for key equality.

---

#### `dwb_slots_hash_key`
```c
static unsigned int dwb_slots_hash_key(void *key, int hash_table_size)
```
- **Lines:** 1360–1366
- **Purpose:** Compute hash index from VPID.
- **Algorithm:** `(pageid | (volid << 24)) % hash_table_size`. Mixes volume ID into the high bits of pageid to distribute across volumes.

---

#### `dwb_slots_hash_insert`
```c
STATIC_INLINE int dwb_slots_hash_insert(THREAD_ENTRY *thread_p, VPID *vpid, DWB_SLOT *slot, bool *inserted)
```
- **Lines:** 1377–1461
- **Purpose:** Insert or update the "latest slot for this VPID" in the hash. Implements the deduplication policy.
- **Algorithm:**
  1. `find_or_insert` — atomically find or insert an entry; returns with the entry's mutex held.
  2. If already present (`!*inserted`):
     - If existing LSA is newer: unlock and return (keep old slot, discard new — old is better).
     - If existing LSA equals new LSA:
       - Same block: invalidate the earlier position's slot (avoid duplicate within block).
       - Different blocks: debug assertion that older block has lower version.
     - Merge `ensure_metadata` flags: `slot->ensure_metadata |= existing->ensure_metadata`.
  3. Replace entry's `slot` pointer with the new slot.
  4. Unlock entry mutex; set `*inserted = true`.
- **Invariant:** The hash always holds the slot with the highest LSA for each VPID.
- **Callers:** `dwb_add_page`

---

#### `dwb_slots_hash_delete`
```c
STATIC_INLINE int dwb_slots_hash_delete(THREAD_ENTRY *thread_p, DWB_SLOT *slot)
```
- **Lines:** 1880–1926
- **Purpose:** Remove the hash entry for a slot's VPID if the entry still points to that exact slot.
- **Algorithm:**
  1. If slot VPID is null, return (not in hash).
  2. `slots_hashmap.find(vpid)` — returns with entry mutex held.
  3. If `entry->slot == slot` (this is the canonical slot), call `erase_locked`.
  4. If `entry->slot != slot` (a newer slot replaced this one), just unlock.
- **Callers:** `dwb_write_block` (after writing pages to data volume)

---

### 6.7 Core Write and Flush Functions

#### `dwb_compare_slots`
```c
static int dwb_compare_slots(const void *arg1, const void *arg2)
```
- **Lines:** 1778–1832
- **Purpose:** Comparator for `qsort`. Orders slots by: `volid` ascending, then `pageid` ascending, then `lsa.pageid` ascending, then `lsa.offset` ascending.
- **Effect:** Groups writes by volume and within volume by sequential page number — maximizes sequential I/O.

---

#### `dwb_block_create_ordered_slots`
```c
STATIC_INLINE int dwb_block_create_ordered_slots(DWB_BLOCK *block, DWB_SLOT **p_dwb_ordered_slots,
    unsigned int *p_ordered_slots_length)
```
- **Lines:** 1842–1871
- **Purpose:** Create a sorted copy of a block's slot array (including a sentinel null slot at the end).
- **Algorithm:** `malloc` array of `count_wb_pages + 1`, `memcpy` from block's slots, `dwb_init_slot` on sentinel, `qsort` with `dwb_compare_slots`.
- **Output:** `*p_ordered_slots_length = count_wb_pages + 1` (includes sentinel).
- **Callers:** `dwb_flush_block`, `dwb_load_and_recover_pages`

---

#### `dwb_compare_vol_fd`
```c
static int dwb_compare_vol_fd(const void *v1, const void *v2)
```
- **Lines:** 1935–1944
- **Purpose:** Integer comparison for volume file descriptors (used for binary search / deduplication in flush area). Returns `(*v1) - (*v2)`.

---

#### `dwb_add_volume_to_block_flush_area`
```c
STATIC_INLINE FLUSH_VOLUME_INFO *dwb_add_volume_to_block_flush_area(THREAD_ENTRY *thread_p,
    DWB_BLOCK *block, int vol_fd, bool ensure_metadata)
```
- **Lines:** 1957–1989
- **Purpose:** Register a new volume in the block's flush area. Initializes its `FLUSH_VOLUME_INFO` entry and increments `count_flush_volumes_info` atomically.
- **Note:** Uses `ATOMIC_INC_32` for the count increment to prevent code reordering (one writer, multiple readers scenario).
- **Callers:** `dwb_write_block`

---

#### `dwb_write_block`
```c
STATIC_INLINE int dwb_write_block(THREAD_ENTRY *thread_p, DWB_BLOCK *block,
    DWB_SLOT *p_dwb_ordered_slots, unsigned int ordered_slots_length,
    bool file_sync_helper_can_flush, bool remove_from_hash)
```
- **Lines:** 2004–2176
- **Visibility:** Static inline
- **Purpose:** Write all non-null slots in sorted order to their final data volume locations. Optionally wake the file-sync-helper daemon. Optionally remove flushed slots from the hash.
- **Algorithm:**
  1. Iterate sorted slots; skip null VPIDs.
  2. On volume boundary (`volid` changes): mark previous volume `all_pages_written=true`, get new `vol_fd`, add to flush area.
  3. `fileio_write(vol_fd, slot->io_page, slot->vpid.pageid, IO_PAGESIZE, NO_COMPENSATE_WRITE)`.
  4. In `SERVER_MODE`: every `PB_SYNC_ON_NFLUSH` writes, CAS-set `file_sync_helper_block` and wake the helper daemon.
  5. After all writes: if `file_sync_helper_block==NULL`, try to wake helper one more time.
  6. `perfmon_add_stat(PSTAT_PB_NUM_IOWRITES)`.
  7. If `remove_from_hash`: call `dwb_slots_hash_delete` for each written slot.
- **Error handling:** Returns `ER_FAILED` if `fileio_write` returns NULL.
- **Callers:** `dwb_flush_block`, `dwb_load_and_recover_pages`

---

#### `dwb_flush_block`
```c
STATIC_INLINE int dwb_flush_block(THREAD_ENTRY *thread_p, DWB_BLOCK *block,
    bool file_sync_helper_can_flush, UINT64 *current_position_with_flags)
```
- **Lines:** 2189–2455
- **Visibility:** Static inline
- **Purpose:** The core DWB flush operation. Writes the block to the DWB volume, syncs it, then writes to data volumes, syncs data volumes. Resets the block for reuse.
- **Algorithm (the double-write protocol core):**
  1. Increment `blocks_flush_counter` (assert ≤ 1 — only one flush at a time).
  2. Create sorted slot copy via `dwb_block_create_ordered_slots`.
  3. Remove duplicate slots (adjacent same VPID: keep higher LSA, null out lower; propagate `ensure_metadata`).
  4. Debug: assert WAL protocol (no page with LSA past `nxio_lsa`).
  5. In `SERVER_MODE`: wait for any prior `file_sync_helper_block` to complete (ensuring previous block is fully flushed before writing next block to DWB file). If helper unavailable, flush volumes directly.
  6. Reset `count_flush_volumes_info=0`, `all_pages_written=false`.
  7. **Phase 1:** `fileio_write_pages(dwb_Global.vdes, block->write_buffer, ...)` — write entire block to DWB file.
  8. **Phase 2:** `fileio_synchronize(dwb_Global.vdes, ...)` — fsync the DWB file.
  9. **Phase 3:** `dwb_write_block(...)` — write pages to their data volume locations.
  10. **Phase 4:** For each volume in `flush_volumes_info`: if not yet flushed by helper, CAS-claim and call `fileio_synchronize`.
  11. Set `block->all_pages_written = true`.
  12. Reset `count_wb_pages=0`; increment `block->version`.
  13. CAS-clear the `BLOCK_WRITE_STARTED` bit for this block in `position_with_flags`.
  14. Advance `next_block_to_flush` atomically.
  15. Signal all waiting threads via `dwb_signal_block_completion`.
- **Error handling:** Asserts `false` and returns `ER_FAILED` on fileio failures.
- **Callers:** `dwb_flush_next_block`, `dwb_starts_structure_modification`, `dwb_add_page` (SA_MODE / no daemon)

---

#### `dwb_acquire_next_slot`
```c
STATIC_INLINE int dwb_acquire_next_slot(THREAD_ENTRY *thread_p, bool can_wait, DWB_SLOT **p_dwb_slot)
```
- **Lines:** 2465–2598
- **Visibility:** Static inline
- **Purpose:** Atomically reserve the next slot in the DWB ring buffer. Handles block boundaries, waiting for flushed blocks, and structure modification.
- **Algorithm (CAS retry loop):**
  1. Read `position_with_flags`.
  2. If `NOT_CREATED_OR_MODIFYING`: handle structure modification wait or graceful "DWB disabled" return.
  3. Extract `current_block_no` and `position_in_current_block` from position.
  4. If `position_in_current_block == 0` (first slot of a new block):
     - If block's write-started bit is set: the previous cycle of this block isn't flushed yet. If `can_wait`, call `dwb_wait_for_block_completion`; else return NULL slot.
     - Set `BLOCK_WRITE_STARTED` bit for this block in the new position.
  5. Compute `new_position_with_flags = position + 1` (wrapping at `DWB_NUM_TOTAL_PAGES`).
  6. CAS `position_with_flags`; on failure retry from step 1.
  7. Set `*p_dwb_slot = &block->slots[position_in_current_block]`, null the slot VPID.
- **Concurrency:** Multiple threads can acquire slots simultaneously via CAS. The block's write-started bit is set atomically by the thread that gets `position_in_current_block == 0`.

---

#### `dwb_set_slot_data`
```c
STATIC_INLINE void dwb_set_slot_data(THREAD_ENTRY *thread_p, DWB_SLOT *dwb_slot,
    FILEIO_PAGE *io_page_p, bool ensure_metadata)
```
- **Lines:** 2609–2631
- **Purpose:** Copy the page data into the slot's `io_page` buffer. Update `vpid`, `lsa`, `ensure_metadata` on the slot.
- **Algorithm:** If `io_page_p->prv.pageid != NULL_PAGEID`: `memcpy(slot->io_page, io_page_p, IO_PAGESIZE)`. Else: call `fileio_initialize_res` (treat as empty page).
- **Callers:** `dwb_set_data_on_next_slot`

---

#### `dwb_init_slot`
```c
STATIC_INLINE void dwb_init_slot(DWB_SLOT *slot)
```
- **Lines:** 2639–2647
- **Purpose:** Mark a slot as empty: `io_page=NULL`, null VPID, null LSA. Used for sentinel slot in sorted array.
- **Callers:** `dwb_block_create_ordered_slots`

---

#### `dwb_get_next_block_for_flush`
```c
STATIC_INLINE void dwb_get_next_block_for_flush(THREAD_ENTRY *thread_p, unsigned int *block_no)
```
- **Lines:** 2656–2671
- **Purpose:** Return the index of the next full block ready for flush, or `DWB_NUM_TOTAL_BLOCKS` if none ready.
- **Algorithm:** Check `blocks[next_block_to_flush].count_wb_pages == DWB_BLOCK_NUM_PAGES`. Simple comparison; no locking.
- **Callers:** `dwb_flush_next_block`

---

### 6.8 Public Write Interface

#### `dwb_set_data_on_next_slot` (Public)
```c
int dwb_set_data_on_next_slot(THREAD_ENTRY *thread_p, FILEIO_PAGE *io_page_p,
    bool can_wait, bool ensure_metadata, DWB_SLOT **p_dwb_slot)
```
- **Lines:** 2683–2709
- **Visibility:** `extern` (public)
- **Purpose:** Acquire the next slot and copy page data into it. First half of the two-phase `page_buffer.c` write path.
- **Algorithm:** `dwb_acquire_next_slot` → if slot obtained, `dwb_set_slot_data`.
- **Callers:** `page_buffer.c:pgbuf_bcb_flush_with_wal` (called with `can_wait=false` — non-blocking), `dwb_add_page`

---

#### `dwb_add_page` (Public)
```c
int dwb_add_page(THREAD_ENTRY *thread_p, FILEIO_PAGE *io_page_p, VPID *vpid,
    bool ensure_metadata, DWB_SLOT **p_dwb_slot)
```
- **Lines:** 2723–2826
- **Visibility:** `extern` (public)
- **Purpose:** Complete the DWB write: insert slot into hash, increment block page count, trigger flush if block full.
- **Algorithm:**
  1. If `*p_dwb_slot == NULL`: call `dwb_set_data_on_next_slot` with `can_wait=true`.
  2. If `!VPID_ISNULL(vpid)`: call `dwb_slots_hash_insert`. If not inserted (older slot won), invalidate this slot.
  3. `ATOMIC_INC_32(&block->count_wb_pages, 1)`.
  4. If `count_wb_pages < DWB_BLOCK_NUM_PAGES`: return (not yet full).
  5. If `SERVER_MODE` and flush daemon available: wake `dwb_flush_block_daemon` and return.
  6. Else: call `dwb_flush_block` directly (SA_MODE or daemon unavailable).
- **Callers:** `page_buffer.c:pgbuf_bcb_flush_with_wal`, `file_io.c:fileio_write`

---

### 6.9 Synchronization

#### `dwb_synchronize` (Public)
```c
int dwb_synchronize(THREAD_ENTRY *thread_p, int vol_fd, const char *vlabel)
```
- **Lines:** 2839–2899
- **Visibility:** `extern` (public)
- **Purpose:** Synchronize a volume, routing through DWB if appropriate.
- **Algorithm:**
  1. If `fileio_fsync_pending()`: skip (already queued).
  2. In non-CS_MODE: if volume is permanent, call `dwb_flush_force` first.
  3. If `dwb_flush_force` didn't complete everything: fall back to direct `fsync(vol_fd)`.
- **Callers:** `file_io.c:fileio_synchronize`, `file_io.c:fileio_synchronize_all`

---

### 6.10 Recovery

#### `dwb_check_data_page_is_sane`
```c
static int dwb_check_data_page_is_sane(THREAD_ENTRY *thread_p, DWB_BLOCK *rcv_block,
    DWB_SLOT *p_dwb_ordered_slots, int *p_num_recoverable_pages)
```
- **Lines:** 3088–3182
- **Purpose:** During recovery, for each slot in the DWB, read the corresponding data page and check if it is corrupted. If corrupted in data volume but intact in DWB, count it as recoverable. If both are corrupted, log an error and fail.
- **Algorithm:**
  1. For each slot with non-null VPID:
     - Get volume fd; check page ID is within volume bounds (page may have been written to DWB but not yet to data volume during a prior flush).
     - `fileio_read` the page from the data volume.
     - `fileio_page_check_corruption` on the data page.
     - If data page is clean: null out slot (no need to restore).
     - If data page is corrupted: check DWB page; if DWB page also corrupted: `assert(false)`, return `ER_FAILED`.
     - Otherwise: increment `num_recoverable_pages`.
- **Callers:** `dwb_load_and_recover_pages`

---

#### `dwb_load_and_recover_pages` (Public)
```c
int dwb_load_and_recover_pages(THREAD_ENTRY *thread_p, const char *dwb_path_p, const char *db_name_p)
```
- **Lines:** 3196–3392
- **Visibility:** `extern` (public)
- **Purpose:** Called at startup recovery. Reads the existing DWB file, identifies corrupted data pages, restores them from DWB, then rebuilds the DWB with fresh parameters.
- **Algorithm:**
  1. If DWB volume file does not exist: skip recovery, just create a new DWB.
  2. Mount the DWB file; read number of pages.
  3. Validate `num_dwb_pages > 0 && IS_POWER_OF_2(num_dwb_pages)` — if not, skip recovery (partial write detected).
  4. Allocate one temporary `DWB_BLOCK` via `dwb_create_blocks`.
  5. `fileio_read_pages` — bulk read entire DWB into the block's write buffer.
  6. Set VPID/LSA for each slot from its page header.
  7. Create sorted slot array, remove duplicates (same logic as normal flush but without LSA ordering guarantee since partial DWB writes are possible).
  8. Debug: `dwb_debug_check_dwb` — verify no valid duplicates remain.
  9. `dwb_check_data_page_is_sane` — identify pages to recover.
  10. If any recoverable pages: `dwb_write_block(remove_from_hash=false)` + fsync each volume.
  11. Dismount old DWB file, `fileio_unformat` (delete it).
  12. `dwb_create` — build fresh DWB per current parameters.
- **Error handling:** Does NOT delete old DWB file on error (preserves evidence for crash analysis).

---

### 6.11 Flush Force

#### `dwb_flush_force` (Public)
```c
int dwb_flush_force(THREAD_ENTRY *thread_p, bool *all_sync)
```
- **Lines:** 3511–3755
- **Visibility:** `extern` (public)
- **Purpose:** Force-flush all pending DWB content. Called from `dwb_synchronize` and checkpoint paths. Fills the current partial block with null pages to trigger flush if needed.
- **Algorithm (complex state machine):**
  1. Read `position_with_flags`. If not created or block-status == 0 and no helper block: nothing to do.
  2. Find the most-recently-written block (highest version among all `BLOCK_WRITE_STARTED` blocks).
  3. Record `initial_num_pages` and `max_pages_to_add = DWB_BLOCK_NUM_PAGES - initial_num_pages`.
  4. Loop `check_flushed_blocks`:
     - If another thread is currently flushing the initial block, wait via `dwb_wait_for_block_completion`.
     - Re-read `position_with_flags`.
     - If initial block's write-started bit is now clear (block flushed): go to `wait_for_file_sync_helper_block`.
     - If block was overwritten (version changed): go to `wait_for_file_sync_helper_block`.
     - If position hasn't changed and we haven't added enough null pages yet:
       - `dwb_add_page(iopage=zeroed page, vpid=NULL_VPID)` — inserts a filler page to advance position.
       - Increment `count_added_pages`.
     - Loop back.
  5. `wait_for_file_sync_helper_block`: in SERVER_MODE, spin until `file_sync_helper_block != initial_block`.
  6. Set `*all_sync = true`.
- **Design note:** The null-page injection technique is a clever way to fill a partial block and trigger a flush without introducing a special "force flush" code path in the normal write flow.

---

### 6.12 Block Flush Daemon

#### `dwb_flush_next_block`
```c
static int dwb_flush_next_block(THREAD_ENTRY *thread_p)
```
- **Lines:** 3456–3502
- **Purpose:** Entry point for the flush-block daemon. Checks if DWB is active, finds the next full block, flushes it, and loops.
- **Algorithm:** `goto start` loop: check created+not-modifying, call `dwb_get_next_block_for_flush`, if block found call `dwb_flush_block(file_sync_helper_can_flush=true)`, loop back (`goto start`) to check for more.
- **Callers:** `dwb_flush_block_daemon_task::execute`

---

#### `dwb_file_sync_helper`
```c
static int dwb_file_sync_helper(THREAD_ENTRY *thread_p)
```
- **Lines:** 3763–3955
- **Purpose:** Concurrent file synchronization helper. Processes the `file_sync_helper_block` while `dwb_write_block` is still writing, overlapping I/O writes and fsyncs.
- **Algorithm:**
  1. Get `block = dwb_Global.file_sync_helper_block`; return if NULL.
  2. `do-while(can_flush_volume)` loop:
     - Iterate `flush_volumes_info[start_flush_volume..count_flush_volumes_info)`.
     - For each volume: CAS-claim ownership via `flushed_status` field.
     - If `num_pages >= PB_SYNC_ON_NFLUSH` OR `all_pages_written==true`: proceed to sync.
     - Otherwise: mark `need_wait`, track `first_partial_flushed_volume`.
     - `ATOMIC_TAS_32(&num_pages, 0)` — take the page count; `fileio_synchronize`.
     - Advance `start_flush_volume` or loop back to partially-flushed volume.
     - If not enough data yet and no new volumes arrived: `thread_sleep(1)` (wait for more pages to be written).
  3. At end: `ATOMIC_TAS_ADDR(&dwb_Global.file_sync_helper_block, NULL)` — release the block.
- **Callers:** `dwb_file_sync_helper_execute` (daemon task wrapper)

---

### 6.13 Read Interface

#### `dwb_read_page` (Public)
```c
int dwb_read_page(THREAD_ENTRY *thread_p, const VPID *vpid, void *io_page, bool *success)
```
- **Lines:** 3966–4003
- **Visibility:** `extern` (public)
- **Purpose:** Serve a page read from in-memory DWB instead of disk if the page is in the hash (most-recent version).
- **Algorithm:**
  1. If DWB not created: `*success=false`, return.
  2. `slots_hashmap.find(vpid)` — returns with entry mutex held.
  3. If found and slot VPID still matches: `memcpy(io_page, slot->io_page, IO_PAGESIZE)`, `*success=true`.
  4. Unlock entry mutex.
- **Callers:** `page_buffer.c` during `pgbuf_read_page` when loading a page from disk.

---

### 6.14 Daemon Infrastructure (SERVER_MODE only)

#### `dwb_flush_block_daemon_task::execute`
- **Lines:** 4022–4042
- **Purpose:** Daemon task body for the flush-block daemon. Checks `BO_IS_FLUSH_DAEMON_AVAILABLE()` and `PRM_ID_ENABLE_DWB_FLUSH_THREAD`, calls `dwb_flush_next_block`. Tracks performance with `PSTAT_DWB_FLUSH_BLOCK_COND_WAIT`.

#### `dwb_file_sync_helper_execute`
- **Lines:** 4050–4063
- **Purpose:** Free-function daemon task body for the file-sync-helper. Same availability check, calls `dwb_file_sync_helper`.

#### `dwb_flush_block_daemon_init`
- **Lines:** 4068–4075
- **Purpose:** Create `dwb_flush_block_daemon` with a 1 ms looper and a `dwb_flush_block_daemon_task`.

#### `dwb_file_sync_helper_daemon_init`
- **Lines:** 4080–4087
- **Purpose:** Create `dwb_file_sync_helper_daemon` with a 10 ms looper and an `entry_callable_task` wrapping `dwb_file_sync_helper_execute`.

#### `dwb_daemons_init` (Public, SERVER_MODE)
- **Lines:** 4092–4097
- **Purpose:** Initialize both daemons. Called from `boot_sr.c` after DWB creation.

#### `dwb_daemons_destroy` (Public, SERVER_MODE)
- **Lines:** 4102–4107
- **Purpose:** Destroy both daemons via `cubthread::get_manager()->destroy_daemon()`.

---

### 6.15 Waiting and Signaling

#### `dwb_wait_for_block_completion`
```c
STATIC_INLINE int dwb_wait_for_block_completion(THREAD_ENTRY *thread_p, unsigned int block_no)
```
- **Lines:** 1549–1632
- **Purpose:** Suspend the current thread until a specific DWB block finishes its flush cycle.
- **Algorithm (SERVER_MODE only):**
  1. Lock `dwb_block->mutex`; lock thread entry.
  2. Re-check `BLOCK_WRITE_STARTED` — if already cleared (flushed), unlock and return.
  3. Add thread to block's `wait_queue`.
  4. `thread_suspend_timeout_wakeup_and_unlock_entry` with 20 ms timeout.
  5. On timeout: remove from queue, return `ER_CSS_PTHREAD_COND_TIMEDOUT`.
  6. On shutdown interrupt: remove from queue, set `ER_INTERRUPTED`.
  7. On normal wakeup: assert `THREAD_DWB_QUEUE_RESUMED`, return `NO_ERROR`.
- **SA_MODE:** Returns `NO_ERROR` immediately (no blocking needed).

---

#### `dwb_signal_waiting_thread`
```c
STATIC_INLINE int dwb_signal_waiting_thread(void *data)
```
- **Lines:** 1640–1664
- **Purpose:** Wake a single suspended thread stored in a wait queue entry.
- **Algorithm:** Cast `data` to `DWB_WAIT_QUEUE_ENTRY*`, extract `THREAD_ENTRY*`, lock thread entry, if status is `THREAD_DWB_QUEUE_SUSPENDED` call `thread_wakeup_already_had_mutex(THREAD_DWB_QUEUE_RESUMED)`.

---

#### `dwb_signal_block_completion`
```c
STATIC_INLINE void dwb_signal_block_completion(THREAD_ENTRY *thread_p, DWB_BLOCK *dwb_block)
```
- **Lines:** 1673–1680
- **Purpose:** Wake all threads waiting on a block completion. Calls `dwb_signal_waiting_threads` on the block's wait queue.

---

#### `dwb_signal_structure_modificated`
```c
STATIC_INLINE void dwb_signal_structure_modificated(THREAD_ENTRY *thread_p)
```
- **Lines:** 1688–1693
- **Purpose:** Wake all threads waiting on the global structure-modification queue.

---

#### `dwb_set_status_resumed`
```c
STATIC_INLINE int dwb_set_status_resumed(void *data)
```
- **Lines:** 1518–1540
- **Purpose:** Mark a thread's `resume_status` as `THREAD_DWB_QUEUE_RESUMED` without waking it. Used as the `func` callback when removing a thread from the wait queue during timeout cleanup.

---

#### `dwb_wait_for_strucure_modification`
```c
STATIC_INLINE int dwb_wait_for_strucure_modification(THREAD_ENTRY *thread_p)
```
- **Lines:** 1701–1770
- **Purpose:** Suspend the current thread until the DWB structure modification completes.
- **Algorithm (SERVER_MODE):** Mirrors `dwb_wait_for_block_completion` but uses the global `dwb_Global.wait_queue` and mutex; 10 ms timeout.

---

### 6.16 Debug Function

#### `dwb_debug_check_dwb` (NDEBUG-only)
```c
static int dwb_debug_check_dwb(THREAD_ENTRY *thread_p, DWB_SLOT *p_dwb_ordered_slots, unsigned int num_dwb_pages)
```
- **Lines:** 3011–3075
- **Purpose:** Scan sorted slots for same-VPID adjacent entries (duplicates). If both are non-corrupted and non-zero, `assert(false)` — indicates a logic error.
- **Algorithm:** Sequential scan; for adjacent equal VPIDs, check corruption status of both. If both look valid and non-zero, assert failure.

---

## 7. Key Algorithms and Logic Flows

### 7.1 The Double-Write Protocol (Normal Flush Path)

The complete sequence for a single dirty page flush:

```
[page_buffer.c: pgbuf_bcb_flush_with_wal()]
    1. dwb_set_data_on_next_slot(io_page, can_wait=false, &dwb_slot)
       → dwb_acquire_next_slot() — CAS on position_with_flags → reserve slot
       → dwb_set_slot_data()     — memcpy io_page into slot->io_page
    2. ... (write to WAL if needed) ...
    3. dwb_add_page(io_page, vpid, &dwb_slot)
       → dwb_slots_hash_insert() — register newest slot for this VPID
       → ATOMIC_INC_32(count_wb_pages)
       → if count_wb_pages == DWB_BLOCK_NUM_PAGES:
            SERVER_MODE: wake dwb_flush_block_daemon
            SA_MODE:     dwb_flush_block() directly

[dwb_flush_block_daemon: dwb_flush_next_block()]
    4. dwb_flush_block(block, file_sync_helper_can_flush=true)
       a. dwb_block_create_ordered_slots() — sort by (volid, pageid, lsa)
       b. Remove duplicates (null-out stale same-VPID slots)
       c. Wait for prior file_sync_helper_block to drain
       d. fileio_write_pages(DWB_vdes, block->write_buffer, 0, N, IO_PAGESIZE)
          [*** WRITE BLOCK TO DWB FILE ***]
       e. fileio_synchronize(DWB_vdes) — FSYNC DWB FILE
          [*** DWB IS NOW CRASH-SAFE ***]
       f. dwb_write_block(remove_from_hash=true)
          → for each sorted slot: fileio_write(vol_fd, slot->io_page, pageid)
            → [every PB_SYNC_ON_NFLUSH writes]: wake file_sync_helper_daemon
          → after all writes: try to wake file_sync_helper one more time
          → remove each slot from VPID hash
       g. For each volume with num_pages > 0 (not yet synced by helper):
            CAS-claim flushed_status → fileio_synchronize(vol_fd)
       h. block->all_pages_written = true
          [signals file_sync_helper that it can finish its remaining syncs]
       i. count_wb_pages = 0; version++
       j. CAS-clear BLOCK_WRITE_STARTED bit in position_with_flags
       k. Advance next_block_to_flush
       l. dwb_signal_block_completion() — wake threads waiting on this block
```

**Invariant:** If the system crashes after step (e) but before step (g), the DWB file contains the complete, valid page content. Recovery reads the DWB and restores any corrupted data pages.

### 7.2 Slot Allocation — Lock-Free Ring Buffer

```
position_with_flags layout:
  [63:32] = block write-started bitmask
  [31]    = MODIFY_STRUCTURE
  [30]    = CREATE
  [29:0]  = current write position (0..num_pages-1)

Acquiring slot N at position P in block B:
  1. Read position_with_flags atomically
  2. Compute B = P >> log2_num_block_pages
         slot_in_B = P & (num_block_pages - 1)
  3. if slot_in_B == 0: set BLOCK_WRITE_STARTED[B] in new value
  4. new_position = (P == num_pages-1) ? 0 : P+1  (ring wrap)
     (preserve flags in high bits)
  5. CAS(position_with_flags, old, new); retry on failure
```

The CAS ensures exactly one thread "claims" each position. The BLOCK_WRITE_STARTED bit is set exactly once per block cycle, by the thread that claims `position_in_block == 0`.

### 7.3 VPID Hash Deduplication

When the same page is dirtied multiple times before the block is flushed (common in write-heavy workloads), the hash ensures only the most recent version is written to disk:

```
Hash insert (vpid, new_slot):
  find_or_insert(vpid, &entry)  // holds entry->mutex on return
  if entry already existed:
    if entry->slot->lsa > new_slot->lsa:
      // Old slot is newer! Drop new_slot silently.
      unlock; return
    if entry->slot->lsa == new_slot->lsa && same block:
      // Duplicate in same block (e.g., unlogged update)
      null out the earlier position's slot VPID
    new_slot->ensure_metadata |= entry->slot->ensure_metadata  // preserve flag
  entry->slot = new_slot
  unlock
```

At flush time (`dwb_flush_block`), the sorted duplicate removal provides a second line of defense: adjacent same-VPID slots in the sorted array have the older one nulled out.

### 7.4 Recovery Algorithm

```
dwb_load_and_recover_pages():
  1. Open old DWB file (if exists)
  2. Validate: num_pages > 0 && IS_POWER_OF_2(num_pages)
     (Non-power-of-2 means partial DWB write at crash time — skip recovery)
  3. Read all DWB pages into temp block
  4. Sort by (volid, pageid, lsa); remove duplicates
     - Same LSA, earlier position wins (it was written later in a partial flush)
  5. For each slot:
     a. Read data page from volume
     b. Check corruption (checksum)
     c. If data page OK: skip (no recovery needed)
     d. If data page corrupted + DWB page OK: mark for recovery
     e. If both corrupted: fatal error
  6. Write all recovery pages to data volumes; fsync
  7. Delete old DWB file
  8. Create fresh DWB per current parameters
```

**The "non-power-of-2" heuristic:** If a crash occurs during `fileio_write_pages` of the DWB block (step 7d in the flush protocol), the DWB file may have been partially written. After a crash, the file will have an inconsistent mix of pages from the current and previous flush cycle. A partial flush writes an incomplete block — the resulting page count in the file would not be a power of 2 (since normal sizes are always powers of 2). This heuristic safely skips recovery when the DWB itself is in a partially-written state, relying on WAL for consistency instead.

### 7.5 `dwb_flush_force` — Partial Block Drain

When `fileio_synchronize` is called for a volume (e.g., during checkpoint), the DWB must ensure all pending writes for that volume are completed. If the current block is only partially full, it may never trigger an automatic flush.

The solution: inject null pages (VPID=NULL) to fill the block:
```
max_pages_to_add = DWB_BLOCK_NUM_PAGES - initial_num_pages
while block not yet flushed and count_added_pages < max_pages_to_add:
  dwb_add_page(zero_page, NULL_VPID)   // does NOT insert into hash
  count_added_pages++
  → this will eventually fill the block → trigger flush
```

Null VPID pages are simply skipped during `dwb_write_block` (they don't correspond to any real page), so they have zero data-volume I/O cost.

---

## 8. Concurrency and Thread Safety

### 8.1 The Central Lock: `position_with_flags`

`position_with_flags` is a 64-bit `volatile` value used as the primary lock-free synchronization word. All state transitions use `ATOMIC_CAS_64` (compare-and-swap). The encoding packs:
- Current write position (bits 29–0)
- DWB creation/modification state (bits 31–30)
- Per-block write-in-progress flags (bits 63–32)

This design allows lock-free slot acquisition by multiple concurrent threads via a CAS retry loop, without any mutex for the common case.

### 8.2 Thread Roles

| Thread/Context | Concurrent Behavior |
|---|---|
| Worker threads (multiple) | Each calls `dwb_set_data_on_next_slot` + `dwb_add_page` concurrently; slot acquisition is lock-free via CAS on `position_with_flags` |
| `dwb_flush_block_daemon` | Single instance; flushes full blocks; protected by `blocks_flush_counter` (max 1 flush at a time) |
| `dwb_file_sync_helper_daemon` | Concurrent with `dwb_flush_block_daemon`; claims volume fsync rights via CAS on `flushed_status` |
| Structure modifier (single) | `dwb_starts_structure_modification` serializes via the `MODIFY_STRUCTURE` bit |

### 8.3 Mutex Usage

| Mutex | Scope | Protected Resource |
|---|---|---|
| `dwb_Global.mutex` | Global | `dwb_Global.wait_queue` (structure-modification waiters) |
| `dwb_block->mutex` (per block) | Per block | `block->wait_queue` (block-completion waiters) |
| `dwb_slots_hash_entry->mutex` (per entry) | Per hash entry | `entry->slot` pointer swap; held briefly during hash insert/delete |

Pthread mutexes are **no-ops in SA_MODE** (replaced by preprocessor stubs). In SERVER_MODE they are real `pthread_mutex_t` objects initialized in constructors.

### 8.4 Atomic Operations Used

| Operation | Macro | Purpose |
|---|---|---|
| Lock-free read | `ATOMIC_INC_64(&x, 0)` | Non-tearing 64-bit read |
| CAS | `ATOMIC_CAS_64(&x, old, new)` | Compare-and-swap on `position_with_flags` |
| CAS | `ATOMIC_CAS_32(&x, old, new)` | On `flushed_status`, `next_block_to_flush`, `blocks_flush_counter` |
| Atomic pointer swap | `ATOMIC_TAS_ADDR(&ptr, new)` | `file_sync_helper_block` hand-off |
| Atomic increment | `ATOMIC_INC_32(&x, delta)` | `count_wb_pages`, `count_flush_volumes_info`, `blocks_flush_counter` |
| Atomic exchange | `ATOMIC_TAS_32(&x, val)` | Take-and-reset `num_pages` before fsync |
| Atomic increment | `ATOMIC_INC_64(&block->version, 1)` | Block version bump after flush |

### 8.5 `file_sync_helper_block` Coordination

This pointer mediates between the flush-block thread and the file-sync-helper:

```
dwb_write_block (flush thread):
  while writing pages:
    every N pages: ATOMIC_CAS_ADDR(&file_sync_helper_block, NULL, block)
                   if succeeded: wake file_sync_helper_daemon
  at end: try once more if block still NULL

dwb_file_sync_helper (helper thread):
  block = file_sync_helper_block
  ... fsync volumes until all done ...
  ATOMIC_TAS_ADDR(&file_sync_helper_block, NULL)  // release

dwb_flush_block (next block):
  while file_sync_helper_block != NULL: thread_sleep(1)
  // ensures previous block's data is fully on disk before next DWB write
```

### 8.6 Block Version Ordering

Each block has a monotonically increasing `version` counter. This is used in `dwb_starts_structure_modification` to determine flush order: blocks must be flushed oldest-first (lowest version) to maintain correct recovery semantics.

Version also detects "block was overwritten" in `dwb_flush_force`: if `current_block_version != initial_block_version` for the same `block_no`, the slot contents have been replaced by a newer flush cycle.

### 8.7 Thread Wait Protocol

Threads waiting for a block to flush or structure modification to complete use a custom wait queue rather than condition variables:

1. Lock the relevant mutex.
2. Re-check the condition (double-checked locking).
3. If still waiting: add self (`THREAD_ENTRY*`) to the queue.
4. Unlock mutex.
5. `thread_suspend_timeout_wakeup_and_unlock_entry` (suspends with a timeout).
6. On wakeup: check `resume_status == THREAD_DWB_QUEUE_RESUMED`.
7. On timeout: remove self from queue.

This avoids `pthread_cond_wait`'s overhead and integrates with CUBRID's thread management layer.

---

## 9. Memory Management

### 9.1 Allocation Patterns

| Allocation | Site | Freed |
|---|---|---|
| `DWB_BLOCK` array | `dwb_create_blocks` via `malloc` | `dwb_destroy_internal` via `free_and_init` |
| `block->write_buffer` | `dwb_create_blocks` via `malloc` | `dwb_finalize_block` via `free_and_init` |
| `block->slots` array | `dwb_create_blocks` via `malloc` | `dwb_finalize_block` via `free_and_init` |
| `block->flush_volumes_info` array | `dwb_create_blocks` via `malloc` | `dwb_finalize_block` via `free_and_init` |
| `DWB_WAIT_QUEUE_ENTRY` | `dwb_make_wait_queue_entry` via `malloc` | Recycled in `free_list` pool; freed by `dwb_destroy_wait_queue` via `free_and_init` |
| `DWB_SLOTS_HASH_ENTRY` | `dwb_slots_hash_entry_alloc` via `malloc` | `dwb_slots_hash_entry_free` via raw `free()` |
| Ordered slots temp array | `dwb_block_create_ordered_slots` via `malloc` | `dwb_flush_block` via `free_and_init`; `dwb_load_and_recover_pages` via `free_and_init` |
| Recovery temp block | `dwb_load_and_recover_pages` via `dwb_create_blocks` | `dwb_load_and_recover_pages` cleanup via `dwb_finalize_block` + `free_and_init` |

**Important:** The `DWB_SLOT.io_page` pointer is NOT heap-allocated independently. It points into `block->write_buffer`. Freeing the write buffer automatically frees all associated page memory.

### 9.2 No RAII

Following CUBRID conventions, no RAII or smart pointers are used. All memory is managed with explicit `malloc`/`free_and_init` and `goto exit_on_error` patterns. This matches the broader C-style memory discipline of the engine.

### 9.3 Memory Sizing

For a typical configuration (`DWB_SIZE=2M`, `DWB_BLOCKS=2`):
- `num_blocks = 2`, `num_block_pages = 64` (each block = 64 × 16KB = 1MB)
- Per block: `write_buffer = 1MB`, `slots = 64 × sizeof(DWB_SLOT) = ~2KB`, `flush_volumes_info = 64 × sizeof(FLUSH_VOLUME_INFO) = ~1.5KB`
- Total in-memory: ~2MB write buffers + minor overhead
- Hash: 1000 buckets + freelists (bounded by `DWB_SLOTS_FREE_LIST_SIZE=100`)

---

## 10. Error Handling

### 10.1 Error Code Patterns

All functions follow the CUBRID convention:
- Return `NO_ERROR` (= 0) on success.
- Return negative error codes (`ER_FAILED`, `ER_OUT_OF_VIRTUAL_MEMORY`, `ER_INTERRUPTED`, etc.) on failure.
- Call `er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_CODE, ...)` before returning errors.

### 10.2 Critical Assertions

The code uses `assert(false)` in places that should be logically impossible:
- `dwb_flush_block`: if `fileio_write_pages` (DWB file write) fails — this is a critical I/O error, treated as unrecoverable.
- `dwb_create_internal`: if CAS to set CREATE flag fails — impossible since modify-structure flag guards this.
- `dwb_flush_force`: if `next_block_to_flush` cannot advance — only one thread should advance this.

### 10.3 Recovery from Errors

| Error Condition | Behavior |
|---|---|
| `malloc` failure during `dwb_create_blocks` | Full `goto exit_on_error` rollback; all partial allocations freed |
| `fileio_format` failure | Return error before any in-memory structures created |
| `fileio_write_pages` (DWB file) failure | `assert(false)`, return `ER_FAILED` — data integrity at risk |
| `fileio_write` (data volume) failure | `assert(false)`, return `ER_FAILED` |
| `dwb_wait_for_block_completion` timeout | Retry from `goto start` in caller; non-fatal |
| Shutdown interrupt during wait | Return `ER_INTERRUPTED`; caller propagates cleanly |
| Both DWB page and data page corrupted | Log error, return `ER_FAILED` in recovery — unrecoverable |

### 10.4 WAL Protocol Enforcement

In debug builds, `dwb_flush_block` checks:
```c
if (s1->io_page->prv.pageid != NULL_PAGEID && logpb_need_wal(&s1->io_page->prv.lsa))
{
  nxio_lsa = log_Gl.append.get_nxio_lsa();
  assert(LSA_ISNULL(&nxio_lsa));  // WAL must be satisfied by flush time
}
```
This ensures the Write-Ahead Log protocol is not violated: a data page cannot be written to disk with an LSA that has not yet been persisted in the log file (unless the log has been fully flushed, indicated by `nxio_lsa` being null).

---

## 11. Integration Points

### 11.1 `page_buffer.c` — Primary Consumer

Two call sites in `pgbuf_bcb_flush_with_wal` (approximate line 10463):

```c
// Step 1: Acquire slot before marking page dirty (non-blocking)
uses_dwb = dwb_is_created() && !is_temp;
if (uses_dwb) {
  error = dwb_set_data_on_next_slot(thread_p, iopage, /*can_wait=*/false, false, &dwb_slot);
  if (dwb_slot != NULL) iopage = NULL;  // slot owns the data now
}

// Step 2: Add to DWB after WAL write
if (uses_dwb) {
  error = dwb_add_page(thread_p, iopage, &bufptr->vpid, false, &dwb_slot);
  if (dwb_slot == NULL) {
    // DWB disabled meanwhile — write directly
    write_mode = FILEIO_WRITE_DEFAULT_WRITE;
  }
}
```

Also: `dwb_read_page` is called in the page-load path to serve cached DWB pages before going to disk.

### 11.2 `file_io.c` — Synchronization Interception

`dwb_synchronize` is called from `fileio_synchronize` (the standard fsync path) to route through DWB:

```c
// fileio_synchronize (file_io.c ~line 2912):
#if !defined(CS_MODE)
  if (dwb_synchronize(thread_p, vol_fd, vlabel) != vol_fd)
    return NULL_VOLDES;
#else
  if (fsync(vol_fd) != 0) ...
#endif
```

Also: `fileio_write` (file_io.c ~line 4014) checks `dwb_is_created()` to skip its own fsync and route through `dwb_add_page` instead.

### 11.3 `boot_sr.c` — Lifecycle Management

```c
// Server startup (boot_restart_server):
dwb_load_and_recover_pages(thread_p, dwb_path, db_name);
...
dwb_daemons_init();   // SERVER_MODE only

// Server shutdown (boot_shutdown_server):
dwb_flush_force(thread_p, &all_sync);
dwb_destroy(thread_p);
```

### 11.4 `log_page_buffer.c` — Checkpoint Sync

Before writing a checkpoint log record, `dwb_flush_force` ensures all pending DWB pages are written to their data volumes, maintaining the invariant that the checkpoint LSA is consistent with the data pages on disk.

### 11.5 Volume File

The DWB volume is a regular file with special volume ID `LOG_DBDWB_VOLID`. It is formatted with `fileio_format`, mounted with `fileio_mount`, and its name follows the pattern `{path}/{dbname}.dwb` (via `fileio_make_dwb_name` in `file_io.c`).

---

## 12. Complexity and Metrics

### 12.1 File Metrics

| Metric | Value |
|---|---|
| Total lines | 4168 |
| Public functions | 11 |
| Static inline functions | ~35 |
| Static non-inline functions | ~10 |
| Data structures defined | 8 (in .cpp) + 1 (DWB_SLOT in .hpp) |
| Macros (operational) | ~30 |
| Preprocessor conditional blocks | ~25 (`SERVER_MODE`, `NDEBUG`, `CS_MODE`) |

### 12.2 Algorithmic Complexity

| Operation | Complexity | Notes |
|---|---|---|
| `dwb_acquire_next_slot` | O(1) amortized | CAS retry loop; rare contention |
| `dwb_slots_hash_insert` | O(1) amortized | Lock-free hash with per-entry mutex |
| `dwb_block_create_ordered_slots` | O(N log N) | qsort on N = `count_wb_pages` ≤ `DWB_BLOCK_NUM_PAGES` |
| `dwb_flush_block` | O(N log N + V) | N = pages, V = distinct volumes; dominated by I/O |
| `dwb_load_and_recover_pages` | O(N log N + P) | N = DWB pages, P = pages to check on disk |
| `dwb_flush_force` | O(N) | Injects at most `DWB_BLOCK_NUM_PAGES` null pages |
| `dwb_starts_structure_modification` | O(B × N log N) | B = blocks to flush, N = pages per block; expensive operation, rare |
| Wait queue operations | O(1) / O(n) | Add/free = O(1); disconnect by pointer = O(n) in worst case |

### 12.3 I/O Pattern Analysis

A single block flush generates:
1. One `fileio_write_pages` call: writes N contiguous pages to the DWB file (sequential I/O to DWB volume)
2. One `fileio_synchronize` on DWB file (fsync)
3. N individual `fileio_write` calls sorted by (vol, page): approximately sequential within each volume
4. One `fileio_synchronize` per distinct volume touched (V fsyncs)

Total: 1 DWB write + 1 DWB fsync + N data writes + V data fsyncs

---

## 13. Notable Patterns and Idioms

### 13.1 Packed 64-bit Flags Word

The entire DWB state machine (creation status, modification status, per-block write flags, current position) is encoded in a single 64-bit word. This enables lock-free multi-state CAS operations where a single atomic operation simultaneously claims a slot and records the block's write-started status. Compare to traditional designs requiring separate locks for position and block state.

### 13.2 `STATIC_INLINE __attribute__((ALWAYS_INLINE))`

Heavily used for all hot-path functions. Combined with C++17 inlining semantics, this eliminates function call overhead for the inner loops. All wait-queue operations, slot initialization, and hash operations are inlined.

### 13.3 Free-List Pooling for Wait Queue Entries

Wait queue entries are pooled to avoid `malloc`/`free` in the critical wakeup path. This is important because `dwb_signal_block_completion` is called from the flush thread's critical path, where latency matters.

### 13.4 Sentinal Slot in Sorted Array

`dwb_block_create_ordered_slots` appends an extra null slot after the sorted data. This allows the duplicate-removal loop in `dwb_flush_block` to compare `s1` and `s2 = s1+1` without bounds checking — `s2` will have null VPID and `VPID_EQ(s1->vpid, NULL_VPID)` will be false.

### 13.5 Null-Page Injection for Forced Flush

`dwb_flush_force` injects pages with `NULL_VPID` to fill a partial block. These pages are skipped by `dwb_write_block` (no data volume I/O), but they count toward `count_wb_pages` to trigger the block-full condition and thus the flush. This keeps the flush trigger logic in `dwb_add_page` clean and avoids special-casing forced flushes.

### 13.6 Two-Phase Page Write Split

The public interface splits DWB insertion into two phases:
- `dwb_set_data_on_next_slot` (non-blocking, called before WAL write) — reserves a slot and copies data.
- `dwb_add_page` (blocking, called after WAL write) — commits to hash and potentially triggers flush.

This split allows `page_buffer.c` to copy the page content before the WAL write (`logpb_flush_pages`), so the slot holds a stable snapshot that won't be modified by concurrent writers. If `can_wait=false` and no slot is available, the caller can fall back to direct disk write.

### 13.7 CAS-Based Flush Ownership

The `flushed_status` field in `FLUSH_VOLUME_INFO` uses CAS to determine which thread (flush daemon or file-sync helper) is responsible for a given volume's fsync. This avoids double-fsyncing the same volume while allowing both threads to make progress on different volumes concurrently — a form of work stealing.

### 13.8 Recovery Heuristic: Power-of-2 Validation

Rather than storing a DWB header with a magic number or version, the recovery code uses the structural invariant that valid DWB sizes are always powers of 2 to detect a partially-written DWB file. This is simple and robust — a partial write that leaves an invalid page count is safely skipped, falling back to WAL-only recovery.

### 13.9 SA_MODE Pthread Stub Elimination

The `#if !defined(SERVER_MODE)` block at line 312–317 redefines all pthread mutex functions as no-ops. This is a compile-time elimination — in SA_MODE, all mutex code is completely absent from the binary, not just skipped at runtime. This keeps the standalone library lean.

### 13.10 `ensure_metadata` Flag Propagation

When a slot is replaced in the hash (newer LSA wins), the `ensure_metadata` flag from the old slot is OR'd into the new slot. This ensures that if any version of a page required a metadata sync, the final flush will include it — even if the page version that originally set the flag was superseded.

---

## Summary

The CUBRID double-write buffer is a sophisticated crash-recovery mechanism that uses a lock-free ring buffer architecture for high-throughput page writes. Its central design innovations are:

1. **Single 64-bit packed flags word** for all synchronization state, enabling lock-free slot acquisition by multiple concurrent threads.
2. **Per-VPID hash with LSA-based deduplication** ensuring only the most recent page version is written to disk, eliminating redundant I/O.
3. **Concurrent flush + fsync** via the file-sync-helper daemon that overlaps page writes and volume syncs.
4. **Null-page injection** in `dwb_flush_force` to drain partial blocks without special-casing the flush trigger.
5. **Power-of-2 recovery heuristic** that gracefully handles partially-written DWB files at crash recovery.

The implementation supports three build configurations: `SERVER_MODE` (full daemon threads, blocking waits), `SA_MODE` (direct flush, no threads, no mutexes), and it is excluded entirely from `CS_MODE` (client-only library).
