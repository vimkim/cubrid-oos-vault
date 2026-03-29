# CUBRID External Sort Module — Comprehensive Analysis

**File:** `src/storage/external_sort.c`
**Header:** `src/storage/external_sort.h`
**Line count:** 5,471 lines (`.c`), 165 lines (`.h`)
**Language:** C/C++17 (compiled as C++17 via `c_to_cpp.sh`)
**Generated:** 2026-03-27

---

## 1. File Overview

### Purpose and Role

`external_sort.c` implements CUBRID's **external merge-sort** subsystem — the engine for sorting datasets that exceed available memory. It is invoked whenever the query executor needs to produce ordered output (ORDER BY, GROUP BY, analytic functions) or when the B-tree bulk loader needs to produce a sorted leaf-page stream for index creation.

The module has two distinct algorithmic phases:

1. **Internal (in-phase) sort**: While records fit in memory, accumulate them into an in-memory buffer, sort them using a run-based algorithm (natural merge sort with run detection), and flush each sorted run to a temporary file.
2. **External (ex-phase) merge**: Perform one or more passes of multi-way merge over the temporary files until a single sorted output is produced, then invoke the caller-supplied `put_fn` on every sorted record.

The module also supports:
- **Duplicate elimination** (`SORT_ELIM_DUP`) or **duplicate preservation** (`SORT_DUP`)
- **LIMIT optimization**: stop flushing early once K records have been produced
- **Multi-page (overflow) records**: records that exceed a single slotted page are stored in a separate overflow file and referenced by VPID
- **Transparent Data Encryption (TDE)**: temp files can be encrypted on write and decrypted on read
- **Parallel sort** (SERVER_MODE only): input can be split across multiple worker threads, each sorting a partition; results are merged hierarchically

### Build Modes

The header guard enforces server-or-standalone only:

```c
#if !defined (SERVER_MODE) && !defined (SA_MODE)
#error Belongs to server module
#endif
```

The file itself contains conditional blocks for:

| Guard | Effect |
|---|---|
| `SERVER_MODE` | Enables parallel sort, pthread mutex/cond, `connection_error.h` |
| `SA_MODE` | Single-process path only (no parallel sort) |
| `ENABLE_SYSTEMTAP` | DTrace/SystemTap probes at sort start/end |
| `CUBRID_DEBUG` | Enables slotted-page dump helpers |
| `NDEBUG` | Disables `sort_validate()` consistency check |

---

## 2. Includes & Dependencies

### System Includes
```c
#include <stdio.h>
#include <string.h>
#include <stddef.h>
#include <stdlib.h>
#include <math.h>
#include <functional>       // std::bind (C++)
```

### Internal (Cross-Module) Includes

| Header | Module | Usage |
|---|---|---|
| `config.h` | Build system | Must be first |
| `error_manager.h` | `src/base/` | `er_set()`, `er_errid()`, error codes |
| `system_parameter.h` | `src/base/` | `prm_get_integer_value(PRM_ID_SR_NBUFFERS)` |
| `memory_alloc.h` | `src/base/` | `db_private_alloc/free` |
| `external_sort.h` | self | Own types and `sort_listfile()` prototype |
| `file_manager.h` | `src/storage/` | `file_create_temp`, `file_temp_retire`, `file_alloc`, `file_numerable_find_nth` |
| `page_buffer.h` | `src/storage/` | `pgbuf_copy_from_area`, `pgbuf_copy_to_area` |
| `log_manager.h` | `src/transaction/` | Pulled transitively |
| `disk_manager.h` | `src/storage/` | Disk space management |
| `slotted_page.h` | `src/storage/` | `SCAN_CODE`, alignment constants, `NULL_OFFSET`, `NULL_SLOTID` |
| `overflow_file.h` | `src/storage/` | `overflow_insert`, `overflow_get`, `overflow_get_length` |
| `boot_sr.h` | `src/transaction/` | Server bootstrap |
| `server_support.h` | `src/executables/` | Server utility functions |
| `thread_entry_task.hpp` | `src/thread/` | `cubthread::entry` (C++ thread entry) |
| `thread_manager.hpp` | `src/thread/` | `thread_get_thread_entry_info`, `thread_sleep` |
| `list_file.h` | `src/query/` | `QFILE_LIST_ID`, `qfile_*` scan/open/close |
| `query_manager.h` | `src/query/` | `qmgr_get_old_page`, `qmgr_free_old_page_and_init`, `qmgr_set_dirty_page` |
| `object_representation.h` | `src/object/` | `DB_ALIGN`, `MAX_ALIGNMENT` |
| `px_worker_manager.hpp` | `src/query/` | `parallel_query::worker_manager` |
| `px_callable_task.hpp` | `src/query/` | `parallel_query::callable_task` |
| `px_parallel.hpp` | `src/query/` | `parallel_query::compute_parallel_degree` |
| `px_sort.h` | `src/query/` | `SORT_MAX_PARALLEL`, `RESULT_RUN`, `sort_copy_sort_info()` |
| `connection_error.h` | `src/connection/` | (SERVER_MODE only) |
| `probes.h` | DTrace | (ENABLE_SYSTEMTAP only) |
| `memory_wrapper.hpp` | `src/base/` | **MUST be last** — memory debugging wrapper |

### Reverse Dependencies (Who Includes `external_sort.h`)

- `src/query/list_file.c` — `qfile_sort_list()` wrapper
- `src/query/query_executor.c` — ORDER BY, GROUP BY, analytic window sorts
- `src/storage/btree_load.c` — B-tree index bulk load

---

## 3. Preprocessor & Compilation

### Key Macros Defined in the `.c` File

| Macro | Value | Meaning |
|---|---|---|
| `SORT_MULTIPAGE_FILE_SIZE_ESTIMATE` | `20` | Pages to pre-allocate in the overflow (multipage) temp file |
| `SORT_MAX_HALF_FILES` | `4` | Max number of files in each half (input or output) = max 8 total |
| `SORT_MAX_TOT_FILES` | `8` | Total temp files = `SORT_MAX_HALF_FILES * 2` |
| `SORT_MIN_HALF_FILES` | `2` | Minimum files per half (floor) |
| `SORT_INITIAL_DYN_ARRAY_SIZE` | `30` | Initial capacity of run-size arrays |
| `SORT_EXPAND_DYN_ARRAY_RATIO` | `1.5` | Growth factor for run-size arrays |
| `SORT_MAXREC_LENGTH` | `DB_PAGESIZE - sizeof(SLOTTED_PAGE_HEADER) - sizeof(SLOT)` | Max record length on one page |
| `SORT_SWAP_PTR(a,b)` | pointer swap | Used internally during merge |
| `SORT_CHECK_DUPLICATE(a,b)` | inline dup check | Used in `sort_run_find()` to null-out and link duplicates |

### Key Macros from `external_sort.h`

| Macro | Value | Meaning |
|---|---|---|
| `SORT_PUT_STOP` | `2` | Special return from `put_fn` to stop early |
| `NO_SORT_LIMIT` | `-1` | Sentinel: no LIMIT clause |
| `SORT_RECORD_LENGTH_SIZE` | `sizeof(INT64)` | 8-byte length prefix per record (alignment) |
| `SORT_RECORD_LENGTH(item_p)` | `*((int*)((item_p) - SORT_RECORD_LENGTH_SIZE))` | Read length prefix before data |

---

## 4. Data Structures & Types

### 4.1 `SORT_STATUS` (enum, `external_sort.h`)

Return codes for the caller-supplied `get_fn`:

| Value | Meaning |
|---|---|
| `SORT_REC_DOESNT_FIT` | Record too large for provided buffer; resize and retry |
| `SORT_SUCCESS` | Record successfully placed in the descriptor |
| `SORT_NOMORE_RECS` | Input exhausted; stop the sort |
| `SORT_ERROR_OCCURRED` | Fatal error; `er_set()` has been called |

### 4.2 `SORT_DUP_OPTION` (enum, `external_sort.h`)

| Value | Meaning |
|---|---|
| `SORT_ELIM_DUP` | Eliminate duplicate keys; use `sort_exphase_merge_elim_dup()` |
| `SORT_DUP` | Preserve duplicates; link them via `SORT_REC.next`; use `sort_exphase_merge()` |

### 4.3 `SORT_PARALLEL_TYPE` (enum, `external_sort.h`)

Hint to the parallel sort subsystem about the usage context:

| Value | Usage |
|---|---|
| `SORT_ORDER_BY` | ORDER BY clause — parallel sort fully implemented |
| `SORT_ORDER_WITH_LIMIT` | ORDER BY + LIMIT |
| `SORT_GROUP_BY` | GROUP BY — parallel sort not yet implemented |
| `SORT_ANALYTIC` | Window/analytic functions — not yet implemented |
| `SORT_INDEX_LEAF` | B-tree bulk load — not yet implemented |

### 4.4 Callback Function Types (`external_sort.h`)

```c
typedef SORT_STATUS SORT_GET_FUNC (THREAD_ENTRY *thread_p, RECDES *, void *);
typedef int         SORT_PUT_FUNC (THREAD_ENTRY *thread_p, const RECDES *, void *);
typedef int         SORT_CMP_FUNC (const void *, const void *, void *);
```

- `SORT_GET_FUNC`: Called repeatedly to supply input records. Returns a status code.
- `SORT_PUT_FUNC`: Called for each record in sorted order. Returns `NO_ERROR` or `SORT_PUT_STOP`.
- `SORT_CMP_FUNC`: Comparison function following `strcmp()` convention (`<0`, `0`, `>0`). The third argument is an opaque `cmp_arg`.

### 4.5 `SORT_REC` (struct, `external_sort.h`)

The in-memory representation of a sort record. Layout is a union:

```c
struct SORT_REC {
    SORT_REC *next;           // linked list for same-key duplicates (SORT_DUP mode)
    union {
        struct {
            INT32 pageid;     // original tuple location
            INT16 volid;
            INT16 offset;
            char  body[1];    // sort key bytes start here
        } original;
        int offset[1];        // column offset vector (non-zero = offset from SORT_REC start)
    } s;
};
```

The `offset[]` form is used when `use_original == 0` in `SORTKEY_INFO` — the sort key includes all columns so records can be reconstructed without going back to the original file.

### 4.6 `SUBKEY_INFO` (struct, `external_sort.h`)

Describes one column within a composite sort key:

| Field | Type | Meaning |
|---|---|---|
| `col` | `int` | Column number in the list file tuple |
| `permuted_col` | `int` | Permuted column index |
| `col_dom` | `TP_DOMAIN *` | Column domain |
| `cmp_dom` | `TP_DOMAIN *` | Comparison domain (for median sort across different string domains) |
| `sort_f` | function pointer | Column comparison function (`data_cmpdisk_function_type`) |
| `is_desc` | `int` | Non-zero if this column sorts descending |
| `is_nulls_first` | `int` | NULL ordering |
| `use_cmp_dom` | `bool` | Use `cmp_dom` instead of `col_dom` for comparison |

### 4.7 `SORTKEY_INFO` (struct, `external_sort.h`)

Aggregate sort-key descriptor:

| Field | Type | Meaning |
|---|---|---|
| `nkeys` | `int` | Number of active sort columns |
| `use_original` | `int` | 0 = reconstruct from keys; 1 = go back to original file |
| `key` | `SUBKEY_INFO *` | Points to `default_keys[8]` for small key counts, else `malloc`'d |
| `default_keys[8]` | `SUBKEY_INFO` | Inline storage for ≤8 columns (avoids heap alloc for typical queries) |
| `error` | `int` | Error accumulator for domain conversion in median sort |

### 4.8 `SORT_INFO` (struct, `external_sort.h`)

The context object passed as `get_arg` and `put_arg` for list-file sorts:

| Field | Type | Meaning |
|---|---|---|
| `key_info` | `SORTKEY_INFO` | Sort key specification |
| `s_id` | `QFILE_SORT_SCAN_ID *` | Stateful scan cursor over the input list file |
| `input_file` | `QFILE_LIST_ID *` | Input list file descriptor |
| `output_file` | `QFILE_LIST_ID *` | Output list file descriptor |
| `output_recdes` | `RECDES` | Working buffer for `ls_sort_put_next_short()` |
| `extra_arg` | `void *` | Caller-specific extra state |
| `sort_list_p` | `SORT_LIST *` | Used to open output list file (parallel) |
| `flag` | `int` | Flags for opening output list file |
| `parallelism` | `int` | Parallelism hint |
| `orderby_stats` | `void *` | Pointer to `ORDERBY_STATS` for trace |

### 4.9 `FILE_CONTENTS` (struct, internal)

Run metadata for one temporary file — a circular queue implemented as a dynamic array:

| Field | Type | Meaning |
|---|---|---|
| `num_pages` | `int *` | Dynamic array; element `i` = page count of run `i` |
| `num_slots` | `int` | Total allocated elements in `num_pages` |
| `first_run` | `int` | Index of oldest run (`-1` = empty) |
| `last_run` | `int` | Index of newest run |
| `start_index` | `int` | Page offset for parallel split (`sort_split_last_run`) |

### 4.10 `SORT_PARAM` (struct, internal)

Central control structure for the entire sort operation. One instance per thread (or per parallel worker):

| Field | Type | Meaning |
|---|---|---|
| `temp[SORT_MAX_TOT_FILES]` | `VFID` | VFID of each temporary file (up to 8) |
| `multipage_file` | `VFID` | Overflow temp file for records > `SORT_MAXREC_LENGTH` |
| `file_contents[SORT_MAX_TOT_FILES]` | `FILE_CONTENTS` | Run lists for each temp file |
| `tde_encrypted` | `bool` | Whether TDE applies to temp files |
| `vol_list` | `VOL_LIST` | Temporary volume list (legacy; no longer used) |
| `internal_memory` | `char *` | Single malloc'd buffer for all in-memory sort operations |
| `tot_runs` | `int` | Cumulative runs flushed so far |
| `tot_buffers` | `int` | Size of `internal_memory` in pages |
| `tot_tempfiles` | `int` | Actual number of temp files = `half_files * 2` |
| `half_files` | `int` | Number of temp files per half (input or output) |
| `in_half` | `int` | Which half is currently input (0 or `half_files`) |
| `cmp_fn` | `SORT_CMP_FUNC *` | Comparison function |
| `cmp_arg` | `void *` | Comparison argument |
| `option` | `SORT_DUP_OPTION` | Dup/elim mode |
| `get_fn` | `SORT_GET_FUNC *` | Input function |
| `get_arg` | `void *` | Input argument |
| `put_fn` | `SORT_PUT_FUNC *` | Output function |
| `put_arg` | `void *` | Output argument |
| `tmp_file_pgs` | `int` | Estimated pages per temp file (for initial allocation) |
| `limit` | `int` | LIMIT K (or `NO_SORT_LIMIT`) |
| `total_numrecs` | `unsigned int` | Total records sorted |
| `px_status` | `PX_STATUS` | Parallel worker status |
| `px_result_file_idx` | `int` | Which temp file holds this worker's result |
| `px_orig_thread_p` | `THREAD_ENTRY *` | Parent thread (for copying context to workers) |
| `px_type` | `SORT_PARALLEL_TYPE` | Parallel type hint |
| `ori_sort_param` | `SORT_PARAM *` | Pointer to master sort_param (in workers) |
| `px_parallel_num` | `int` | Number of parallel workers |
| `px_result_run` | `RESULT_RUN *` | Where this worker deposits its result |
| `px_worker_manager` | `parallel_query::worker_manager *` | Thread pool (C++ object) |
| `orderby_stats` | `ORDERBY_STATS` | Performance trace counters |
| `main_error_context` | `cuberr::context *` | Error context for propagating errors to main thread |
| `px_mtx` | `pthread_mutex_t *` | Mutex for signalling completion (SERVER_MODE) |
| `complete_cond` | `pthread_cond_t *` | Condition for signalling completion |

### 4.11 `SLOTTED_PAGE_HEADER` / `SLOT` (structs, internal)

Local restatements of the slotted page format, used by the `sort_spage_*` family which manages in-memory pages formatted identically to disk pages. This avoids calling the full `spage_*` module (which pins buffer pool pages) for purely in-memory work.

- `SLOTTED_PAGE_HEADER`: 8 INT16 fields: `nslots`, `nrecs`, `anchor_flag`, `alignment`, `waste_align`, `tfree`, `cfree`, `foffset`
- `SLOT`: `roffset` (INT16), `rlength` (INT16), `rtype` (INT16)

### 4.12 `SRUN` / `SORT_STACK` (structs, internal)

Used by the in-memory natural merge sort (`sort_run_sort`):

```c
struct srun {
    char  low_high;        // 'L' = otherbase, 'H' = base
    unsigned short tree_depth;  // depth in merge tree; leaf=1
    long start;            // start index in the vector
    long stop;             // stop index (inclusive)
};

struct sort_stack {
    int    top;            // stack pointer (-1 = empty)
    SRUN  *srun;           // dynamically allocated array
};
```

The `SORT_STACK` drives balanced merge decisions in `sort_run_sort`: two adjacent runs with equal `tree_depth` are merged immediately.

### 4.13 `RUN` (struct, internal)

Simple `{ long start; long stop; }` — used as a range descriptor in parallel splitting contexts.

### 4.14 `SORT_REC_LIST` (struct, internal)

Linked list node used in `sort_exphase_merge_elim_dup` to maintain a sorted list of "smallest current records" across active input files:

| Field | Meaning |
|---|---|
| `next` | Next node |
| `rec_pos` | Index into the `smallest_elem_ptr[]` / `long_recdes[]` arrays |
| `is_duplicated` | Whether this position is currently a duplicate |

### 4.15 `VOL_INFO` / `VOL_LIST` (structs, internal)

Legacy temporary volume tracking structures. The comment in `sort_listfile()` explicitly notes: "This volume list will not be used any more." Retained for structure completeness.

---

## 5. Global & Static Variables

The module has **no file-scope global variables**. All state is carried in `SORT_PARAM` (stack-allocated in `sort_listfile()`, then heap-allocated for parallel workers). This design makes the module fully re-entrant.

---

## 6. Function Catalog

### 6.1 Public API

#### `sort_listfile()` — lines 1342–1560
```c
int sort_listfile(THREAD_ENTRY *thread_p, INT16 volid,
                  int est_inp_pg_cnt,
                  SORT_GET_FUNC *get_fn, void *get_arg,
                  SORT_PUT_FUNC *put_fn, void *put_arg,
                  SORT_CMP_FUNC *cmp_fn, void *cmp_arg,
                  SORT_DUP_OPTION option, int limit,
                  bool includes_tde_class,
                  SORT_PARALLEL_TYPE sort_parallel_type);
```

**The sole public entry point.** Performs the complete external sort from input to output.

**Algorithm:**
1. Allocates and initializes `SORT_PARAM` on the stack (`ori_sort_param`).
2. Sets `tot_buffers = MIN(PRM_SR_NBUFFERS, est_inp_pg_cnt + 10%)`, floor 4.
3. Malloc's `internal_memory` = `tot_buffers * DB_PAGESIZE`. Falls back to 4 pages on OOM.
4. Computes `half_files` via `sort_get_num_half_tmpfiles()`.
5. Allocates `FILE_CONTENTS.num_pages` arrays (30 entries each, grows 1.5x).
6. In SERVER_MODE: calls `sort_check_parallelism()`. If `px_parallel_num > 1`, starts parallel path; otherwise single-thread path.
7. Single path: `sort_listfile_internal()`.
8. Parallel path: `sort_start_parallelism()` → `SORT_EXECUTE_PARALLEL` → `SORT_WAIT_PARALLEL` → `sort_end_parallelism()`.
9. Cleanup: calls `sort_return_used_resources()`.

**Error handling:** All errors go to a `cleanup:` label. SystemTap probe `CUBRID_SORT_END` receives record count and error status.

**Callers:**
- `qfile_sort_list()` in `list_file.c` (ORDER BY, DISTINCT)
- `qexec_groupby()` in `query_executor.c` (GROUP BY)
- `qexec_hash_groupby()` in `query_executor.c` (hash GROUP BY spill)
- `qexec_analytic_eval_instnum_pred()` in `query_executor.c` (analytic window)
- `btree_sort_records()` in `btree_load.c` (index bulk load)

---

### 6.2 Core Sort Pipeline Functions

#### `sort_listfile_internal()` — lines 1663–1712
```c
int sort_listfile_internal(THREAD_ENTRY *thread_p, SORT_PARAM *sort_param);
```

Orchestrates the two-phase sort for one thread:
1. Calls `sort_inphase_sort()` to produce runs on input-half temp files.
2. If `tot_runs > 1`: allocates output-half temp files, then calls either `sort_exphase_merge_elim_dup()` or `sort_exphase_merge()` depending on `option`.
3. If only one run was produced (or no runs — data fit entirely in memory), the ex-phase is skipped.

**Callers:** `sort_listfile()` (single path), `sort_listfile_execute()` (parallel workers)

---

#### `sort_inphase_sort()` — lines 1763–2186
```c
static int sort_inphase_sort(THREAD_ENTRY *thread_p, SORT_PARAM *sort_param,
                              SORT_GET_FUNC *get_fn, void *get_arg,
                              unsigned int *total_numrecs);
```

**The internal-sort (run generation) phase.** This is the most complex function.

**Memory layout within `internal_memory`:**

```
[SORT_RECORD_LENGTH_SIZE | record data ... | ... | index ptrs growing downward | output_buffer (last page)]
^item_ptr                                              ^index_buff  ^index_area                            ^
```

Records grow upward from `internal_memory`; index pointers (array of `char*`) grow downward from just before the output buffer page. When they collide, the buffer is full.

**Algorithm:**
```
loop:
    if buffer full:
        status = SORT_REC_DOESNT_FIT
    else:
        status = get_fn(temp_recdes)

    switch(status):
        SORT_NOMORE_RECS: break loop
        SORT_ERROR_OCCURRED: return error
        SORT_REC_DOESNT_FIT:
            if numrecs > 0:
                sort_run_sort() -> sort the index area
                if SORT_ELIM_DUP and still room:
                    compact and continue (avoid flush)
                else:
                    sort_run_flush() -> write run to temp file
                    reset pointers for next run
                    round-robin to next output temp file
            if temp_recdes.length > SORT_MAXREC_LENGTH:
                get_fn() again into long_recdes
                overflow_insert() into multipage_file
                store VPID in index, flush as REC_BIGONE run
        SORT_SUCCESS:
            advance item_ptr by aligned record length
            decrement index_area and index_buff
```

**Special path when input fits in memory entirely (no flush):**
- `tot_runs == 0`: directly calls `put_fn` on the sorted `index_area` without writing to temp files at all.

**Special path when exactly one run was produced:**
- Optimization: if `!once_flushed` (the run was produced on behalf of a single large record), re-reads the single temp file page and outputs directly.

**Error handling:** `goto exit_on_error`. Frees `long_recdes.data` on exit.

---

#### `sort_run_sort()` — lines 1166–1297
```c
static char **sort_run_sort(THREAD_ENTRY *thread_p, SORT_PARAM *sort_param,
                             char **base, long limit, long sort_numrecs,
                             char **otherbase, long *srun_limit);
```

In-memory natural merge sort on the `char**` index array. Returns a pointer to the start of the sorted result (which may be in `base` or `otherbase`).

**Algorithm (bottom-up natural merge):**
1. Allocate `SORT_STACK` with `ceil(log2(limit/2)) + 2` entries.
2. Loop: call `sort_run_find()` twice (identifies two natural runs), then:
   - While stack has two runs of equal `tree_depth`: call `sort_run_merge()` to merge them.
3. If there are already-sorted records from a previous pass (`sort_numrecs > 0`): synthesize one final SRUN for them and merge.
4. Result location: `base + srun[0].start` if `low_high == 'H'`, or `otherbase + srun[0].start` if `'L'`.
5. Updates `*srun_limit` with the final count (accounting for duplicates eliminated).

**Debug build:** Calls `sort_validate()` to verify the result is fully sorted.

---

#### `sort_run_find()` — lines 883–992
```c
static void sort_run_find(char **source, long *top, SORT_STACK *st_p,
                           long limit, SORT_CMP_FUNC *compare, void *comp_arg,
                           SORT_DUP_OPTION option);
```

Finds the longest natural run (ascending or descending) starting at `source[*top]`.

**Algorithm:**
1. Detect direction from first two elements.
2. Extend run in that direction while consecutive elements follow the trend.
3. Track duplicates: call `SORT_CHECK_DUPLICATE` which either links duplicates (`SORT_DUP`) or nulls them out.
4. For descending runs: call `sort_run_flip()` to reverse them in-place.
5. Right-shift non-null elements to eliminate the null-slots left by duplicates.
6. Push the resulting SRUN (start/stop/depth/low_high) onto the stack.

---

#### `sort_run_merge()` — lines 1004–1141
```c
static void sort_run_merge(char **low, char **high, SORT_STACK *st_p,
                            SORT_CMP_FUNC *compare, void *comp_arg,
                            SORT_DUP_OPTION option);
```

Merges the top two runs on the stack. Operates on a double-buffer (`low` = `base`, `high` = `otherbase`), alternating destination based on `low_high` flags.

**Algorithm:**
- **CON (concatenation) optimization**: if `left_max < right_min` (runs are already in order), simply move the left run adjacent to the right run — no element-wise comparison needed. O(left_size) copy.
- **Full merge**: right-to-left merge into the opposite buffer. Standard 2-way merge reading from right ends, writing to a destination that expands leftward. Handles duplicates by linking or eliminating.
- Updates the stack: pops the right run, updates the left run's extent.
- **Repeats** while the new merged run has equal `tree_depth` to the run below it on the stack.

---

#### `sort_exphase_merge_elim_dup()` — lines 2371–3152
```c
static int sort_exphase_merge_elim_dup(THREAD_ENTRY *thread_p, SORT_PARAM *sort_param);
```

External merge phase for `SORT_ELIM_DUP`. Significantly more complex than `sort_exphase_merge()` due to cross-run duplicate tracking via a sorted `SORT_REC_LIST`.

**Algorithm (outer loop):**
```
while active_infiles > 1:
    compute in_sectsize, out_sectsize
    for each run group:
        if last run with one active file:
            copy-through optimization (no merge needed)
        else:
            read first page-set of each input into sections
            build and sort sr_list (initial sorted list of min elements)
            identify duplicates in initial list
            loop: emit minimum record, advance that input:
                refill that input's buffer when exhausted
                maintain sr_list sorted, track last_elem_cmp
                flush output buffer when full
    swap halves (in_half <-> out_half)
```

**Duplicate tracking (`last_elem_cmp`):**
- `> 0`: must do full minimum scan
- `< 0`: last element of current input section is the new minimum — skip full scan
- `= 0`: last element is a duplicate — skip full scan except for duplicate check

This optimization avoids O(k) minimum scan on every record advance when the sorted run allows it.

**Long record handling:** For `REC_BIGONE` slots, calls `sort_retrieve_longrec()` to dereference the VPID and fetch the overflow data, then compares `long_recdes.data` pointers.

---

#### `sort_exphase_merge()` — lines 3295–4040
```c
static int sort_exphase_merge(THREAD_ENTRY *thread_p, SORT_PARAM *sort_param);
```

External merge phase for `SORT_DUP`. Structurally identical to `sort_exphase_merge_elim_dup()` except:
- No duplicate elimination logic
- The `last_elem_is_min` boolean optimization replaces `last_elem_cmp` integer
- Duplicate SORT_REC links from in-phase sort are preserved and output

The last run emitted during the final merge pass is the complete sorted output. On the "very last run" iteration, instead of writing to a temp file, `put_fn` is called directly on each record.

---

### 6.3 Temp File I/O Functions

#### `sort_run_flush()` — lines 2208–2319
```c
static int sort_run_flush(THREAD_ENTRY *thread_p, SORT_PARAM *sort_param,
                           int out_file, int *cur_page, char *output_buffer,
                           char **index_area, int numrecs, int rec_type);
```

Writes a sorted run to a temporary file. Iterates `index_area[0..numrecs-1]`, inserts each record into an output slotted page (`sort_spage_insert`). When the page is full, writes it via `sort_write_area()` and starts a new page. Handles the LIMIT check: stops flushing once `flushed_items >= sort_param->limit`. For `SORT_DUP`, traverses the `SORT_REC.next` chain to also flush all duplicates. Updates `file_contents` run list and increments `tot_runs`.

---

#### `sort_write_area()` — lines 5038–5084
```c
static int sort_write_area(THREAD_ENTRY *thread_p, VFID *vfid,
                            int first_page, INT32 num_pages,
                            char *area_start, bool tde_encrypted);
```

Writes `num_pages` pages from `area_start` to the temp file `vfid` starting at logical page `first_page`. Uses `file_numerable_find_nth()` to resolve each logical page number to a physical VPID, then `pgbuf_copy_from_area()` to write (bypassing the log). If `tde_encrypted`, fetches the TDE algorithm from the file header first.

---

#### `sort_read_area()` — lines 5098–5132
```c
static int sort_read_area(THREAD_ENTRY *thread_p, VFID *vfid,
                           int first_page, INT32 num_pages, char *area_start);
```

Reads `num_pages` pages from the temp file into `area_start`. Uses `file_numerable_find_nth()` + `pgbuf_copy_to_area()`. No TDE parameter — decryption is handled transparently by the page buffer layer.

---

#### `sort_retrieve_longrec()` — lines 2327–2364
```c
static char *sort_retrieve_longrec(THREAD_ENTRY *thread_p, RECDES *address, RECDES *memory);
```

Dereferences a `REC_BIGONE` slot. Calls `overflow_get_length()` to find the required buffer size, realloc's `memory->data` if necessary, then calls `overflow_get()` to fetch the overflow chain into `memory`. Returns `memory->data` on success, `NULL` on error.

---

#### `sort_add_new_file()` — lines 4174–4228
```c
static int sort_add_new_file(THREAD_ENTRY *thread_p, VFID *vfid,
                              int file_pg_cnt_est, bool force_alloc,
                              bool tde_encrypted);
```

Creates a numerable (random-access by page number) temporary file. If `force_alloc`, pre-allocates all `file_pg_cnt_est` pages immediately (for output files, to avoid fragmentation). If `tde_encrypted`, applies the configured TDE algorithm to the file. On any error, retires the partially created file.

---

### 6.4 Run Metadata Functions

#### `sort_run_add_new()` — lines 5367–5400
```c
static int sort_run_add_new(FILE_CONTENTS *file_contents, int num_pages);
```

Appends a new run record to the `FILE_CONTENTS` dynamic array. If `last_run >= num_slots`, `realloc`s the array by the 1.5x ratio. Returns `ER_FAILED` (not an error code) on OOM — the caller must check.

#### `sort_run_remove_first()` — lines 5407–5420
```c
static void sort_run_remove_first(FILE_CONTENTS *file_contents);
```

Advances `first_run`. If it surpasses `last_run`, marks the list empty with `first_run = -1`. This implements the "consume from front" pattern without shifting array elements.

#### `sort_get_num_file_contents()` — lines 5428–5441
```c
static int sort_get_num_file_contents(FILE_CONTENTS *file_contents);
```

Returns `last_run - first_run + 1` or `0` if empty.

---

### 6.5 Buffer Management Functions

#### `sort_get_num_half_tmpfiles()` — lines 5143–5183
```c
static int sort_get_num_half_tmpfiles(int tot_buffers, int input_pages);
```

Computes the optimal number of temp files per half (input or output set). Logic:
1. Start with `half_files = tot_buffers - 1` (maximum fan-in).
2. If `input_pages > 0`, estimate expected runs = `ceil(input_pages / tot_buffers) + 1`.
3. Use the smaller of `exp_num_runs` and `tot_buffers/2` as the fan-in limit.
4. Clamp to `[SORT_MIN_HALF_FILES=2, SORT_MAX_HALF_FILES=4]`.

This avoids under-utilization (too many files for a small input) and over-saturation (too many files creates per-file overhead).

#### `sort_find_inbuf_size()` — lines 5344–5359
```c
static int sort_find_inbuf_size(int tot_buffers, int in_sections);
```

Allocates half of total buffers to output, divides the remaining half evenly among `in_sections` input files. Returns `max(floor(tot_buffers / (in_sections * 2)), 1)`.

#### `sort_checkalloc_numpages_of_outfiles()` — lines 5198–5289
```c
static int sort_checkalloc_numpages_of_outfiles(THREAD_ENTRY *thread_p, SORT_PARAM *sort_param);
```

Before each merge pass, estimates the needed pages for each output file by summing all input runs (round-robining across output files). Pre-allocates using `file_alloc_multiple()`. Destroys output files that will receive no data (via `file_temp_retire()`) — this reclaims disk space for reuse by the allocations that follow.

#### `sort_get_numpages_of_active_infiles()` — lines 5305–5319
```c
static int sort_get_numpages_of_active_infiles(const SORT_PARAM *sort_param);
```

Counts active input files (those with `first_run != -1`) starting from `in_half`. Due to balanced run distribution, once the first empty file is found, all remaining files are also empty.

#### `sort_get_avg_numpages_of_nonempty_tmpfile()` — lines 4051–4074
```c
static int sort_get_avg_numpages_of_nonempty_tmpfile(SORT_PARAM *sort_param);
```

Returns the average page count across all non-empty temp files. Used to estimate output file sizes before creating them.

---

### 6.6 Slotted Page Functions (Internal Sort Page Manager)

These functions manage `DB_PAGESIZE`-sized in-memory pages using the same slot-array layout as the server's `spage_*` module, but without touching the buffer pool:

#### `sort_spage_initialize()` — lines 314–356
Initializes a page with a `SLOTTED_PAGE_HEADER`: zeroes slot/record counts, computes alignment waste, sets `tfree`, `cfree`, `foffset`.

#### `sort_spage_insert()` — lines 583–609
Inserts a record. Calls `sort_spage_find_free()` to locate a slot and space; on success, `memcpy`s the data. Returns `NULL_SLOTID` if the record does not fit (including `> SORT_MAXREC_LENGTH`).

#### `sort_spage_get_record()` — lines 635–685
Retrieves slot `slotid`. In `PEEK` mode: sets `recdes->data` to point directly into the page — zero copy, but the page must not be modified until done. In `COPY` mode: copies data into `recdes->data`, returning `S_DOESNT_FIT` if the area is too small.

#### `sort_spage_get_numrecs()` — lines 363–371
Returns `sphdr->nrecs` directly.

#### `sort_spage_compact()` — lines 401–461
Defragments the page by sorting slots by offset (`sort_spage_offsetcmp` via `qsort`), then `memmove`ing each record to the tightest packing. Recalculates `cfree`, `tfree`, `foffset`. Frees `sortptr` on exit.

#### `sort_spage_find_free()` — lines 476–572
Locates a slot (reusing an existing empty one, or appending a new one) and carves space for `length` bytes. Calls `sort_spage_compact()` if there is enough total space but not enough contiguous space. Returns `NULL_SLOTID` on failure.

#### `sort_spage_offsetcmp()` — lines 379–392
Comparator for `qsort` in `sort_spage_compact()`: compares two `SLOT**` by their `roffset`.

---

### 6.7 Parallel Sort Functions (SERVER_MODE only)

#### `sort_listfile_execute()` — lines 1563–1656
```c
void sort_listfile_execute(cubthread::entry &thread_ref, SORT_PARAM *sort_param);
```

Worker thread entry point. Copies transaction context from the parent thread. For `SORT_ORDER_BY`: opens the split input file scan, calls `sort_listfile_internal()`, closes scan, accumulates trace statistics. Signals completion by locking `px_mtx`, setting `px_status`, signalling `complete_cond`, and unlocking.

#### `sort_check_parallelism()` — lines 4686–4730
```c
int sort_check_parallelism(THREAD_ENTRY *thread_p, SORT_PARAM *sort_param);
```

Decides whether parallel sort is beneficial. Only `SORT_ORDER_BY` is implemented. Calls `parallel_query::compute_parallel_degree()` with the input page count and a parallelism hint. Rejects if: degree < 2, tuple count ≤ degree, or no workers available via `worker_manager::try_reserve_workers()`.

#### `sort_start_parallelism()` — lines 4888–4948
```c
int sort_start_parallelism(THREAD_ENTRY *thread_p, SORT_PARAM *px_sort_param, SORT_PARAM *sort_param);
```

Prepares N parallel workers:
1. `sort_copy_sort_param()` — deep-copies `SORT_PARAM` for each worker (new memory, file_contents arrays).
2. Deep-copies `get_arg`/`put_arg` (`SORT_INFO`) for each worker.
3. Closes the parent's input scan.
4. `sort_split_input_temp_file()` — physically splits the input list file's page chain into N sub-chains.

#### `sort_end_parallelism()` — lines 4958–5020
```c
int sort_end_parallelism(THREAD_ENTRY *thread_p, SORT_PARAM *px_sort_param, SORT_PARAM *sort_param);
```

Post-parallel cleanup:
1. Reopens the original input file scan.
2. Calls `sort_merge_run_for_parallel()` to hierarchically merge all worker result runs.
3. Aggregates min/max trace statistics across workers into the parent's `ORDERBY_STATS`.

#### `sort_copy_sort_param()` — lines 4238–4342
```c
int sort_copy_sort_param(THREAD_ENTRY *thread_p, SORT_PARAM *px_sort_param,
                          SORT_PARAM *sort_param, int parallel_num);
```

`memcpy`s the master `SORT_PARAM` to each worker, then reinitializes heap-allocated fields (new `internal_memory`, new `file_contents[].num_pages`). Bumps `tot_buffers` to at least 8 for parallel workers (reallocs if needed). Sets `px_orig_thread_p` to the calling thread (or its parent if it is itself a worker).

#### `sort_split_input_temp_file()` — lines 4351–4453
```c
int sort_split_input_temp_file(THREAD_ENTRY *thread_p, SORT_PARAM *px_sort_param,
                                SORT_PARAM *sort_param, int parallel_num);
```

Physically splits the input `QFILE_LIST_ID` page chain. Walks the chain page by page, severing links at `splitted_num_page` intervals by null-terminating the NEXT_VPID pointers (and clearing PREV_VPID of the new first page). Creates N `QFILE_LIST_ID` copies with updated first/last VPIDs and approximate tuple/page counts.

#### `sort_merge_run_for_parallel()` — lines 4462–4625+
```c
int sort_merge_run_for_parallel(THREAD_ENTRY *thread_p, SORT_PARAM *px_sort_param,
                                 SORT_PARAM *sort_param, int parallel_num);
```

Hierarchically merges N sorted result runs (one per worker) into a single final run. Uses a tournament-style reduction:
- While `remaining_run > 1`: group runs into batches of `SORT_PX_MERGE_FILES`, launch parallel merge tasks (`sort_merge_nruns_parallel`), wait for completion. Uses `level` to track the tournament depth.

#### `sort_merge_nruns()` — lines 4634–4678
```c
int sort_merge_nruns(THREAD_ENTRY *thread_p, SORT_PARAM *sort_param);
```

Merges `half_files` input runs (already in temp files) into a single output run. Creates output temp files, calls `sort_exphase_merge_elim_dup()` or `sort_exphase_merge()`, saves the result run location, retires non-result temp files.

#### `sort_merge_nruns_parallel()` — lines 4823–4878
```c
void sort_merge_nruns_parallel(cubthread::entry &thread_ref, SORT_PARAM *sort_param);
```

Worker entry point for a single merge task. Copies thread context, calls `sort_merge_nruns()`, reports performance stats, signals completion.

#### `sort_split_last_run()` — lines 3259–3287
```c
void sort_split_last_run(THREAD_ENTRY *thread_p, SORT_PARAM *px_sort_param,
                          SORT_PARAM *sort_param, int parallel_num);
```

Splits the single consolidated result run across worker `file_contents[0]` entries by assigning page ranges. Used to parallelize the final `put_fn` output phase.

#### `sort_put_result_for_parallel()` — lines 4738–4815
```c
void sort_put_result_for_parallel(cubthread::entry &thread_ref, SORT_PARAM *sort_param);
```

Worker that reads its assigned page range from the final result temp file and calls `put_fn` for each record. For the first worker, uses the origin output file; for later workers, opens a new list file. Handles `REC_BIGONE` via `sort_retrieve_longrec()`.

#### `sort_put_result_from_tmpfile()` — lines 3160–3250
```c
int sort_put_result_from_tmpfile(THREAD_ENTRY *thread_p, SORT_PARAM *sort_param, int start_pagenum);
```

Reads pages from the result temp file in `tot_buffers`-page chunks starting at `start_pagenum`, calls `put_fn` on each record. Used by `sort_put_result_for_parallel()`.

---

### 6.8 Utility Functions

#### `sort_return_used_resources()` — lines 4084–4164
```c
static void sort_return_used_resources(THREAD_ENTRY *thread_p, SORT_PARAM *sort_param,
                                        PARALLEL_TYPE parallel_type);
```

Frees all resources: `internal_memory`, all temp files (via `file_temp_retire`), `multipage_file`, all `file_contents.num_pages` arrays. For `PX_THREAD_IN_PARALLEL`, also frees the `SORT_INFO` structures (`get_arg`, `put_arg`) including their scan IDs and list IDs.

#### `sort_run_flip()` — lines 830–845
```c
static void sort_run_flip(char **start, char **stop);
```

Reverses a sub-range of the index array in-place. Used by `sort_run_find()` to convert descending natural runs to ascending.

#### `sort_append()` — lines 853–868
```c
static void sort_append(const void *pk0, const void *pk1);
```

Appends the `SORT_REC` node at `*pk0` to the tail of the list rooted at `*pk1`. Used in `SORT_DUP` mode to link duplicate keys.

#### `sort_validate()` — lines 1725–1750 (NDEBUG only)
```c
static int sort_validate(char **vector, long size, SORT_CMP_FUNC *compare, void *comp_arg);
```

Debug-mode validation: asserts consecutive elements are strictly less-than by calling `compare`. Currently short-circuits with `return NO_ERROR` (disabled via `#if 1`).

#### `sort_print_file_contents()` — lines 5451–5471 (CUBRID_DEBUG only)
Prints run sizes to stdout. For debugging.

#### `sort_spage_dump_*()` — lines 699–817 (CUBRID_DEBUG only)
Three functions dump page header and slot array to stdout.

---

## 7. Key Algorithms & Logic Flows

### 7.1 External Sort Phases — Complete Flow

```
sort_listfile()
  │
  ├─ [Initialization]
  │    malloc internal_memory (tot_buffers * DB_PAGESIZE)
  │    malloc file_contents[SORT_MAX_TOT_FILES].num_pages
  │    compute half_files = sort_get_num_half_tmpfiles()
  │
  ├─ [Parallel check: SERVER_MODE only]
  │    sort_check_parallelism() -> N workers?
  │    If N > 1:
  │      sort_start_parallelism() -> split input, clone params
  │      SORT_EXECUTE_PARALLEL -> N × sort_listfile_execute()
  │      SORT_WAIT_PARALLEL
  │      sort_end_parallelism() -> sort_merge_run_for_parallel()
  │    Else:
  │      sort_listfile_internal()
  │
  └─ sort_return_used_resources()


sort_listfile_internal()
  │
  ├─ Phase 1: sort_inphase_sort()
  │    Loop over input via get_fn:
  │      Accumulate records into internal_memory
  │      When buffer full (or input exhausted):
  │        sort_run_sort() -> natural merge sort in-memory
  │        sort_run_flush() -> write sorted run to temp file
  │          sort_spage_initialize / sort_spage_insert -> slotted page
  │          sort_write_area() -> pgbuf_copy_from_area
  │      Special: record > SORT_MAXREC_LENGTH:
  │        overflow_insert() -> multipage_file
  │        flush as REC_BIGONE single-record run
  │      If all fits in memory: call put_fn directly, skip Phase 2
  │
  └─ Phase 2: sort_exphase_merge[_elim_dup]()
       Outer loop: while active_infiles > 1:
         sort_checkalloc_numpages_of_outfiles()
         Compute in_sectsize, out_sectsize (sort_find_inbuf_size)
         Inner loop: for each run group:
           sort_read_area() -> fill input sections
           k-way merge via SORT_REC_LIST (sorted linked list of minimums)
           When output buffer full:
             sort_write_area() -> flush to output temp file
         Swap in_half / out_half
       Final run: call put_fn on each record directly
```

### 7.2 Multi-Way Merge Algorithm (k-way, k ≤ 4)

The merge engine does not use a heap/priority queue. Instead it uses a small sorted linked list (`SORT_REC_LIST sr_list[SORT_MAX_HALF_FILES]`) of up to 4 elements — acceptable for k ≤ 4.

**Initial setup:**
1. Read first page-set of each input file.
2. Get the first record from each (`smallest_elem_ptr[i]`).
3. Insertion-sort the records to build `sr_list` in ascending order.
4. Find initial duplicates.

**Per-record loop:**
1. Emit the record at `sr_list` head (`min` = `sr_list->rec_pos`).
2. Write to output buffer via `sort_spage_insert()`.
3. Advance: try to get the next record from file `min`'s input section.
4. If the input section's current page is exhausted: refill from disk (`sort_read_area()`).
5. If the entire run is exhausted: mark that input inactive (`smallest_elem_ptr[min].data = NULL`).
6. Re-insert the new element into `sr_list` by insertion (O(k) work).
7. **Optimization (`last_elem_cmp`)**: if the just-advanced record is less than all others (detected by comparing against the last element of the current page), skip the full re-insertion.

**Buffer management during merge:**
- Each active input file gets `in_sectsize` pages of buffering in `internal_memory`.
- The output gets `out_sectsize = tot_buffers - in_sectsize * act_infiles` pages.
- `in_sectsize = max(floor(tot_buffers / (act_infiles * 2)), 1)`.

### 7.3 Run Generation — Natural Merge Sort

The in-memory sort (`sort_run_sort`) exploits pre-existing order in the input data using the "natural runs" approach:

1. **Run detection** (`sort_run_find`): scans forward from the current position, extending a run as long as the ordering is consistent. Flips descending runs. This is O(n) on already-sorted data.
2. **Balanced merge** (`sort_run_merge`): uses a `SORT_STACK` and merges only when two adjacent runs have equal `tree_depth`. This ensures O(n log n) worst case and O(n) best case.
3. **CON optimization**: if the maximum of the left run is less than the minimum of the right run, the left run is simply moved adjacent — O(left_size) rather than O(left+right).
4. **Dual-buffer**: alternates between `base` and `otherbase` arrays, avoiding additional copies. The `low_high` field in each SRUN records which buffer the run currently lives in.

### 7.4 Duplicate Handling

Two modes controlled by `SORT_DUP_OPTION`:

**`SORT_ELIM_DUP`:**
- In `sort_run_find()`: when `cmp == 0`, the pointer at the current position is nulled out (`*(stop) = NULL`).
- In `sort_run_merge()`: when `cmp == 0`, only `right_stop` is written to dest; `left_stop` is dropped.
- Result: only one copy of each key survives to the slotted pages.
- In the external merge phase: the `last_elem_cmp` tracking ensures duplicates spanning page boundaries are still eliminated.

**`SORT_DUP`:**
- In `sort_run_find()`: when `cmp == 0`, `sort_append()` links the duplicate record to the predecessor via `SORT_REC.next`.
- In `sort_run_merge()`: when `cmp == 0`, `sort_append()` links and both input pointers advance.
- `sort_run_flush()`: traverses `SORT_REC.next` chains for each index entry, emitting all linked duplicates in sequence.
- Result: all duplicate keys are emitted, chained together.

### 7.5 Long Record (Overflow) Handling

Records that exceed `SORT_MAXREC_LENGTH = DB_PAGESIZE - sizeof(SLOTTED_PAGE_HEADER) - sizeof(SLOT)` are handled specially:

1. **Detection**: `get_fn` returns `SORT_REC_DOESNT_FIT` with `temp_recdes.length > SORT_MAXREC_LENGTH`.
2. **Re-fetch**: `get_fn` is called again with a larger `long_recdes` buffer.
3. **Storage**: `overflow_insert()` writes the full record to `multipage_file` (a separate temp overflow file), storing the `VPID` of the overflow chain head into `item_ptr`.
4. **Sort key**: The VPID occupies the sort record (4 bytes for pageid + 2 for volid). During merge, when a slot has `type == REC_BIGONE`, `sort_retrieve_longrec()` is called to dereference the VPID before comparison.
5. **Flush**: Long records are flushed as individual single-record runs with `rec_type = REC_BIGONE`.
6. **Output**: In the final pass, `put_fn` receives the retrieved long record via `long_recdes`.

### 7.6 LIMIT Optimization

When `sort_param->limit > 0` (i.e., `ORDER BY ... LIMIT K`):
- `sort_run_flush()` maintains a `flushed_items` counter and sets `should_continue = false` when `flushed_items >= limit`.
- This prunes the output at flush time — records beyond K are never written to temp files.
- The in-memory sort still processes all records in the current buffer; only the flush respects the limit.

### 7.7 TDE Integration

When `includes_tde_class == true`:
- `sort_add_new_file()` calls `file_apply_tde_algorithm()` to configure the temp file for encryption.
- `sort_write_area()` fetches the TDE algorithm from the file and passes `tde_algo` to `pgbuf_copy_from_area()`.
- `sort_read_area()` does not need special handling — decryption is transparent in `pgbuf_copy_to_area()`.
- The multipage overflow file also receives TDE treatment when created.

---

## 8. Concurrency & Thread Safety

### Single-Thread Path

For SA_MODE or when `px_parallel_num == 1`: fully single-threaded. All state is in the stack-allocated `ori_sort_param`. No locks.

### Parallel Path (SERVER_MODE)

The parallel sort introduces concurrency in two places:

**Phase 1 parallelism (inphase sort):**
- N worker threads each run `sort_listfile_internal()` on a disjoint partition of the input.
- Each worker has its own `SORT_PARAM` copy with separate `internal_memory` and temp files.
- The input file pages are physically severed between partitions (null-terminated page chains) before workers start — no shared input state during sort.
- Each worker signals completion via a dedicated `pthread_mutex_t px_mtx` + `pthread_cond_t complete_cond` (stored as pointers in each worker's `SORT_PARAM`; the actual mutex/cond live on `sort_listfile()`'s stack).

**Phase 2 parallelism (merge):**
- `sort_merge_run_for_parallel()` launches merge tasks via the `parallel_query::worker_manager` thread pool.
- Each merge task also signals via the same mutex/cond mechanism.
- The tournament reduction is driven serially by the main thread between rounds.

**Error propagation:**
- When a worker fails, it sets `px_status = PX_ERR_FAILED` and calls `main_error_context->get_current_error_level().swap(cuberr::context::get_thread_local_error())` — moving the error context to the main thread's error slot before signalling.
- The main thread checks the error after `SORT_WAIT_PARALLEL`.

**Transaction context:**
- Workers copy `tran_index`, `conn_entry`, and `m_px_orig_thread_entry` from the parent thread to allow `qmgr_get_old_page()` and similar calls that need transaction identity.

### Thread Safety Guarantees

- **No global state**: the module has zero global/static variables.
- **Temp files**: each worker owns its own temp files (`SORT_MAX_TOT_FILES` VFIDs) — no sharing.
- **Input splitting**: page chain mutation in `sort_split_input_temp_file()` is done before workers launch; no concurrent modification.
- **Output merging**: hierarchical merge is driven by the main thread; worker merge tasks each operate on disjoint sets of input runs.

---

## 9. Memory Management

### `internal_memory` Buffer

Allocated once in `sort_listfile()` via `malloc`:
```
size = tot_buffers * DB_PAGESIZE
```
Where `tot_buffers = MIN(PRM_SR_NBUFFERS, est_inp_pg_cnt * 1.1)`, clamped to [4, PRM_SR_NBUFFERS].

Falls back to `tot_buffers = 4` (minimum) if the large allocation fails. If even the small allocation fails, returns `ER_OUT_OF_VIRTUAL_MEMORY`.

For parallel sort, bumped to minimum 8 pages per worker via `realloc`.

This single buffer serves dual purpose:
1. **In-phase**: holds records (growing upward from base), index pointers (growing downward from output_buffer), and the output slotted page (last page).
2. **Ex-phase**: subdivided into `act_infiles` input sections + output section.

### `file_contents.num_pages`

Allocated via `malloc(SORT_INITIAL_DYN_ARRAY_SIZE * sizeof(int))` for each of `SORT_MAX_TOT_FILES` files. Grown via `realloc` at 1.5× ratio in `sort_run_add_new()`. Freed via `free_and_init()` in `sort_return_used_resources()`.

### `SRUN` stack in `sort_run_sort()`

`db_private_alloc(NULL, cnt * sizeof(SRUN))` where `cnt = ceil(log2(limit/2)) + 2`. Freed via `db_private_free_and_init()` after sorting.

### Long Record Buffer

`malloc`'d and `realloc`'d on demand within `sort_inphase_sort()` when long records are encountered. Freed on exit or on the next flush that replaces it.

### Parallel Worker `get_arg`/`put_arg` (`SORT_INFO`)

Deep-copied via `sort_copy_sort_info()` for each worker. Freed in `sort_return_used_resources()` when `parallel_type == PX_THREAD_IN_PARALLEL`, which also frees the `QFILE_LIST_ID` and `QFILE_LIST_SCAN_ID` sub-objects via `db_private_free_and_init`.

### Conventions

- Always uses `free_and_init()` (never bare `free()`) to nullify pointers after freeing.
- `db_private_alloc(thread_p, ...)` for SRUN stack (tied to thread private pool).
- `malloc()/realloc()/free_and_init()` for all other allocations.
- No C++ RAII — explicit cleanup via `goto cleanup:` / `goto bailout:` / `goto exit_on_error:` patterns.

---

## 10. Error Handling

The module uses CUBRID's C error model throughout:

### Error Codes Used

| Code | When |
|---|---|
| `ER_OUT_OF_VIRTUAL_MEMORY` | Any `malloc`/`realloc` failure |
| `ER_GENERIC_ERROR` | Fatal logic errors (impossible states) |
| `ER_FAILED` | Generic operational failure (low-level) |
| `ER_SORT_TEMP_PAGE_CORRUPTED` | Slot read failure during merge phase |
| `ER_CSS_PTHREAD_MUTEX_INIT` | Mutex init failure |
| `ER_CSS_PTHREAD_COND_INIT` | Condition variable init failure |
| Various file/overflow errors | Propagated from called modules |

### Error Propagation Pattern

```c
// Standard pattern throughout:
error = some_function(...);
if (error != NO_ERROR)
  {
    ASSERT_ERROR ();    // assert in debug, no-op in release
    goto bailout;       // or goto exit_on_error / goto cleanup
  }
```

`ASSERT_ERROR()` verifies that `er_errid() != NO_ERROR` — i.e., an error code has actually been set before jumping.

`ASSERT_ERROR_AND_SET(error)` additionally assigns the result of `er_errid()` to `error`.

### `put_fn` Return Code Handling

`SORT_PUT_STOP` (value 2) is treated as a non-error early termination: the sort loop converts it to `NO_ERROR` and exits cleanly. This allows callers (e.g., GROUP BY aggregation) to signal "enough" without triggering error paths.

### Cleanup Guarantee

All cleanup paths free `long_recdes.data` (in `sort_inphase_sort`) and call `sort_return_used_resources()` (in `sort_listfile`) before returning. The resource cleanup function is null-safe: `if (sort_param == NULL) return;`.

---

## 11. Integration Points

### 11.1 Query Executor — ORDER BY

**Caller:** `qfile_sort_list()` in `src/query/list_file.c` (line ~4413)

```c
sort_listfile(thread_p, NULL_VOLID, estimated_pages,
              get_func, &info, put_func, &info, cmp_func, &info.key_info,
              dup_option, limit, srlist_id->tfile_vfid->tde_encrypted,
              parallel_type);
```

`get_func` = `ls_sort_get_next()`: scans the input list file (`QFILE_LIST_ID`) using a `QFILE_SORT_SCAN_ID` cursor, serializes each tuple's sort key(s) into a `SORT_REC`.

`put_func` = `ls_sort_put_next()` or `ls_sort_put_next_short()`: writes the sorted tuple back to the output `QFILE_LIST_ID`.

`cmp_func` = `qfile_compare_partial_sort_record()` or `qfile_compare_all_sort_record()`: compares `SORT_REC` key data using the `SUBKEY_INFO.sort_f` per-column comparison functions.

`parallel_type` = `SORT_ORDER_BY` or `SORT_ORDER_WITH_LIMIT`.

### 11.2 Query Executor — GROUP BY

**Caller:** `qexec_groupby()` and `qexec_hash_groupby()` in `src/query/query_executor.c`

```c
sort_listfile(thread_p, NULL_VOLID, estimated_pages,
              &qexec_gby_get_next, &gbstate,
              &qexec_gby_put_next, &gbstate,
              gbstate.cmp_fn, &gbstate.key_info,
              SORT_DUP, NO_SORT_LIMIT, ..., SORT_GROUP_BY);
```

GROUP BY always uses `SORT_DUP` — duplicate keys are kept and processed by the aggregation functions in `put_fn`. The `put_fn` (`qexec_gby_put_next`) handles computing aggregate values as successive same-key records arrive in sorted order. No parallel sort for GROUP BY (not yet implemented).

### 11.3 Query Executor — Analytic Functions

**Caller:** `qexec_analytic_eval_instnum_pred()` in `query_executor.c`

```c
sort_listfile(thread_p, NULL_VOLID, estimated_pages,
              &qexec_analytic_get_next, &analytic_state,
              &qexec_analytic_put_next, &analytic_state,
              analytic_state.cmp_fn, &analytic_state.key_info,
              SORT_DUP, NO_SORT_LIMIT, ..., SORT_ANALYTIC);
```

Window/analytic function evaluation requires records sorted by PARTITION BY + ORDER BY keys. The sorted output is consumed by the analytic evaluation engine. No parallel sort for analytic (not yet implemented).

### 11.4 B-tree Bulk Load

**Caller:** `btree_sort_records()` in `src/storage/btree_load.c`

```c
return sort_listfile(thread_p, sort_args->hfids[0].vfid.volid, 0,
                     &btree_sort_get_next, sort_args,
                     out_func, out_args,
                     compare_driver, sort_args,
                     SORT_DUP, NO_SORT_LIMIT,
                     includes_tde_class, SORT_INDEX_LEAF);
```

`btree_sort_get_next()`: feeds `SORT_REC` entries from a heap scan, each containing the index key + OID.
`compare_driver()`: wraps `btree_compare_key()`.
`out_func`: writes the sorted (key, OID) pairs into leaf pages of the B-tree being built.

The volume hint (`sort_args->hfids[0].vfid.volid`) directs temp files to the same volume as the heap. No parallelism for index builds (not yet implemented).

### 11.5 File Manager Integration

The sort module uses `file_create_temp_numerable()` (not the regular `file_create_temp()`) so that logical page numbers can be resolved by `file_numerable_find_nth()`. This is a random-access abstraction over the file's physical page chain.

Write: `pgbuf_copy_from_area()` — bypasses the log, writes directly to the buffer pool with TDE.
Read: `pgbuf_copy_to_area()` — reads from buffer pool (decryption is transparent).

---

## 12. Complexity & Metrics

### Code Metrics

| Metric | Value |
|---|---|
| Total lines (`.c`) | 5,471 |
| Total lines (`.h`) | 165 |
| Static functions | ~30 |
| Non-static (public/package-visible) functions | ~15 (most parallel helpers) |
| Exported functions | 1 (`sort_listfile`) |
| Maximum function length | `sort_exphase_merge_elim_dup()`: ~780 lines |
| Second longest | `sort_inphase_sort()`: ~424 lines |
| Third longest | `sort_exphase_merge()`: ~745 lines |

### Algorithm Complexity

| Phase | Time | Space |
|---|---|---|
| In-memory natural merge sort | O(n log n) worst, O(n) best (sorted input) | O(n) — the `internal_memory` buffer |
| External merge (one pass) | O(n) I/O reads + O(n) I/O writes | O(B) in-memory — B = `tot_buffers` pages |
| External merge (total passes) | O(n log_{k} n) where k = `half_files` ≤ 4 | O(B) in-memory |
| In-memory min selection (k-way) | O(k) per record, k ≤ 4 → effectively O(1) | O(k) |

### Configuration Parameters

| Parameter | Default/Role |
|---|---|
| `PRM_ID_SR_NBUFFERS` | Sort buffer size in pages (controls `tot_buffers`) |
| `SORT_MAX_HALF_FILES = 4` | Max 4-way merge |
| `SORT_MIN_HALF_FILES = 2` | Minimum 2 files per half |
| `DB_PAGESIZE` | Page size (determines record size limits and buffer granularity) |
| `PRM_ID_TDE_DEFAULT_ALGORITHM` | TDE cipher selection |

---

## 13. Notable Patterns & Idioms

### 13.1 No-Log I/O

Temp files are written/read entirely through `pgbuf_copy_from_area()` / `pgbuf_copy_to_area()` — which bypass WAL logging. Crash recovery is not needed for sort temp files; they are dropped on server restart.

### 13.2 Private Slotted Page Implementation

Rather than calling `spage_insert()` (which would require buffer pool pins and logging), the module reimplements the slotted page format locally in `sort_spage_*()`. This allows purely in-memory page construction before flushing in bulk — a critical performance optimization since each sort page may contain dozens of records inserted sequentially.

### 13.3 Index Array Pattern

During in-phase sort, records are stored compactly in `internal_memory` (variable-length, forward-growing) while a separate index array of `char *` pointers (reverse-growing from the end of the buffer) provides O(1) access by position. Only pointers are moved during sorting — record bytes are never copied. This is the classic "pointer sort" pattern for variable-length records.

The dual-buffer (`base` / `otherbase`) natural merge sort extends this: the `SRUN` metadata tracks which buffer each run currently lives in, allowing the merge to alternate between buffers without additional copies on the CON (concatenation) fast path.

### 13.4 Dynamic Array Queue (FILE_CONTENTS)

Run sizes are tracked in a dynamic array used as a queue: `first_run` and `last_run` are indices (not pointers), advancing through the array. This avoids pointer aliasing issues with `realloc` and gives O(1) enqueue and dequeue. The array grows at 1.5× when full — the SORT_INITIAL_DYN_ARRAY_SIZE of 30 runs is sufficient for typical queries.

### 13.5 Conditional Compilation Layers

The codebase uses four layers of conditional compilation:
- `SERVER_MODE` / `SA_MODE`: deployment mode
- `ENABLE_SYSTEMTAP`: optional DTrace integration
- `CUBRID_DEBUG`: extra debug output to stdout
- `NDEBUG`: disables `sort_validate()` (called even in debug builds, but with a `#if 1` short-circuit leaving it always disabled)

### 13.6 Error Context Propagation

Parallel workers use the `cuberr::context` mechanism to transport error information from a worker thread's error context to the main thread's error context:
```cpp
sort_param->main_error_context->get_current_error_level()
    .swap(cuberr::context::get_thread_local_error());
```
This is the standard CUBRID C++ error propagation pattern for worker threads.

### 13.7 SORT_PUT_STOP as Flow Control

The value `2` for `SORT_PUT_STOP` (as opposed to `NO_ERROR = 0` and negative error codes) allows `put_fn` to signal "stop iterating" without setting an error. The sort engine treats it as a clean early exit. This is documented in the header and is a deliberate API contract with callers.

### 13.8 Overflow File Lazy Creation

The `multipage_file` is created lazily (in `sort_inphase_sort`) only when the first long record is encountered. For most queries with typical-length data, the overflow file is never created, saving a file creation syscall.

### 13.9 Parallel Sort Constraints

Parallel sort is only fully implemented for `SORT_ORDER_BY`. The other types (`SORT_GROUP_BY`, `SORT_ANALYTIC`, `SORT_INDEX_LEAF`) return `ER_FAILED` from `sort_start_parallelism()` — effectively falling back to single-thread mode. The check in `sort_check_parallelism()` returns 1 for these types before even trying.

---

## Appendix: Function Summary Table

| Function | Lines | Visibility | Description |
|---|---|---|---|
| `sort_listfile` | 1342 | **public** | Main entry point |
| `sort_listfile_execute` | 1563 | package (SERVER_MODE) | Parallel worker thread entry |
| `sort_listfile_internal` | 1663 | package | Two-phase sort orchestrator |
| `sort_inphase_sort` | 1763 | `static` | Run generation phase |
| `sort_run_sort` | 1166 | `static` | In-memory natural merge sort |
| `sort_run_find` | 883 | `static` | Natural run detection |
| `sort_run_merge` | 1004 | `static` | Two-run merge in dual buffer |
| `sort_run_flip` | 830 | `static` | Reverse a subarray in-place |
| `sort_append` | 853 | `static` | Link duplicate SORT_RECs |
| `sort_run_flush` | 2208 | `static` | Flush sorted run to temp file |
| `sort_retrieve_longrec` | 2327 | `static` | Dereference REC_BIGONE overflow |
| `sort_exphase_merge_elim_dup` | 2371 | `static` | External merge, elim duplicates |
| `sort_exphase_merge` | 3295 | `static` | External merge, keep duplicates |
| `sort_put_result_from_tmpfile` | 3160 | package | Read result temp file, call put_fn |
| `sort_split_last_run` | 3259 | package | Split final run for parallel output |
| `sort_get_avg_numpages_of_nonempty_tmpfile` | 4051 | `static` | Average pages per non-empty file |
| `sort_return_used_resources` | 4084 | `static` | Free all sort resources |
| `sort_add_new_file` | 4174 | `static` | Create numerable temp file |
| `sort_copy_sort_param` | 4238 | package (SERVER_MODE) | Deep-copy SORT_PARAM for workers |
| `sort_split_input_temp_file` | 4351 | package (SERVER_MODE) | Physically split input page chain |
| `sort_merge_run_for_parallel` | 4462 | package (SERVER_MODE) | Hierarchical merge of N worker results |
| `sort_merge_nruns` | 4634 | package (SERVER_MODE) | Merge N runs into one |
| `sort_check_parallelism` | 4686 | package (SERVER_MODE) | Decide parallel degree |
| `sort_put_result_for_parallel` | 4738 | package (SERVER_MODE) | Parallel output worker |
| `sort_merge_nruns_parallel` | 4823 | package (SERVER_MODE) | Parallel merge task entry |
| `sort_start_parallelism` | 4888 | package (SERVER_MODE) | Initialize parallel sort |
| `sort_end_parallelism` | 4958 | package (SERVER_MODE) | Finalize parallel sort |
| `sort_write_area` | 5038 | `static` | Write pages to temp file |
| `sort_read_area` | 5098 | `static` | Read pages from temp file |
| `sort_get_num_half_tmpfiles` | 5143 | `static` | Compute optimal file fan-in |
| `sort_checkalloc_numpages_of_outfiles` | 5198 | `static` | Pre-allocate output file pages |
| `sort_get_numpages_of_active_infiles` | 5305 | `static` | Count active input files |
| `sort_find_inbuf_size` | 5344 | `static` | Compute per-input buffer allocation |
| `sort_run_add_new` | 5367 | `static` | Append run to FILE_CONTENTS |
| `sort_run_remove_first` | 5407 | `static` | Consume oldest run |
| `sort_get_num_file_contents` | 5428 | `static` | Count runs in FILE_CONTENTS |
| `sort_spage_initialize` | 314 | `static` | Init in-memory slotted page |
| `sort_spage_get_numrecs` | 363 | `static` | Get record count from page header |
| `sort_spage_offsetcmp` | 379 | `static` | Slot comparator for compaction |
| `sort_spage_compact` | 401 | `static` | Defragment in-memory page |
| `sort_spage_find_free` | 476 | `static` | Find free space in page |
| `sort_spage_insert` | 583 | `static` | Insert record into in-memory page |
| `sort_spage_get_record` | 635 | `static` | Read record from in-memory page |
| `sort_validate` | 1725 | `static` | Debug: verify sort correctness |
| `sort_print_file_contents` | 5451 | `static` | Debug: print run list |
| `sort_spage_dump_sptr` | 699 | `static` | Debug: dump slot array |
| `sort_spage_dump_hdr` | 731 | `static` | Debug: dump page header |
| `sort_spage_dump` | 754 | `static` | Debug: dump full page |
