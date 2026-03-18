# OOS Project Knowledge Base

## What is OOS?

**OOS (Out-of-row Overflow Storage)** separates large variable-length columns from heap records into dedicated OOS files to reduce unnecessary disk I/O. Instead of reading a full 3KB record for a 4-byte ID, only small columns are read from the heap.

### Trigger Conditions (M1)
- Record size > DB_PAGESIZE/8 (typically 2KB for 16KB pages)
- Column is variable-length AND column size > 512 bytes

### Core Terminology
- **OOS Record**: Column data stored in OOS file, separated from heap
- **OOS File**: FILE_OOS type, 1:1 with heap file (one per table)
- **OOS OID**: 8-byte pointer (volid, pageid, slotid) stored in heap record's variable area
- **HAS_OOS flag**: MVCC header bit 3 (`OR_MVCC_FLAG_HAS_OOS = 0x08`)
- **IS_OOS flag**: VOT entry bit 0 (`OR_VAR_BIT_OOS = 0x1`)
- **OOS Resolve**: Converting OOS OIDs back to actual values

## Record Format

```
Heap Record: [MVCC header] [VOT] [Fixed columns] [Variable area (values or OOS OIDs)]
```

- Variable offset table lower 2 bits repurposed as flags
- MVCC header size lookup uses only lower 3 bits to avoid overflow
- OOS file uses slotted page format with multi-chunk support (linked-list chain)

## CRUD Operation Flows

### INSERT
`heap_insert()` → `heap_attrinfo_determine_disk_layout()` (identify OOS candidates) → `heap_oos_find_vfid()` → `oos_insert()` per column → build heap record with OOS OIDs → `spage_insert()`

### SELECT (OOS Resolve)
`heap_get()` → check `OR_MVCC_FLAG_HAS_OOS` → `heap_record_replace_oos_oids_with_values_if_exists()` → per IS_OOS column: `oos_read()` → reconstruct full record

### UPDATE (Always New OID)
1. `oos_insert()` for new OOS columns → new OOS OIDs
2. Write updated heap record with new OOS OIDs
3. Old heap record (with old OOS OIDs) saved to undo log as-is
4. Old OOS records remain — old transactions may still access them via MVCC undo
5. When vacuum removes the old heap record, `oos_delete()` cleans up old OOS records together

**Invariant**: One OOS OID is referenced by exactly one record (heap page or undo log).

### DELETE
Add MVCC delete ID. OOS OIDs remain in heap record (NOT resolved, NOT deleted). Vacuum handles cleanup later.

## Key Source Files
- `src/storage/heap_file.c` — Heap insert/read/update integration with OOS
- `src/storage/oos_file.cpp` — OOS file operations (insert/read/delete/chunk management)
- `object_representation.h` — Record format macros
- `object_representation_constants.h` — MVCC flag constants

## Recovery & Replication Invariants
1. Every OOS insert/delete is WAL-logged; crash recovery restores OOS state
2. Update undo log retains OOS OIDs as-is (old OOS records must stay alive for MVCC readers)
3. Old OOS records are deleted by vacuum together with the old heap record, not during update
4. Deleted records retain OOS OIDs until vacuum reclaims them
5. Replication log has sufficient info to replay on replica
6. Each heap file has at most 1 OOS file; VFID stored in heap header

## Known Bugs & Limitations

| Issue | Impact | Fix Target |
|---|---|---|
| No `oos_file_destroy` | OOS files grow indefinitely | M2 |
| Bestspace = last-insert-page only | Hotspot on single page | M2 |
| DELETE doesn't clean OOS | Orphan records until vacuum | M2 |
| No OOS OID reuse on update | Extra I/O when value unchanged | M3 |
| Ordered fix deadlock risk | Two tx's accessing OOS pages in different order | M4 |
| No PEEK mode for OOS | Always COPY semantics, extra memcpy | Future |
| No across-page compaction | Fragmentation over time | Future |
| `S_DOESNT_FIT` handling incomplete | Caller must handle buffer overflow | Upper-layer |
| unloaddb 1.6-1.7x slower | `heap_attrinfo_start` called per `heap_next` | CBRD-26458 |
| UPDATE calls `oos_read` 3x redundantly | OOS resolve in wrong context | CBRD-26516 |

## Milestones
- **M1** (Feb 2026, DONE): Basic POC — insert/read/update/delete, WAL, recovery, replication
- **M2** (3/10–4/17, IN PROGRESS): Drop table, bestspace optimization, in-page compaction, vacuum integration
- **M3** (4/20–5/29, PLANNED): OOS OID reuse on update (deduplication)
- **M4** (TBD): Ordered fix deadlock handling, monitoring tools

## JIRA Issues
- **CBRD-26517**: Main OOS tracking issue
- **CBRD-26458**: unloaddb `heap_next` performance regression
- **CBRD-26516**: UPDATE redundant `oos_read` calls

## Writing Conventions

### Inline Code Adjacent to Korean Text
Always add a space between inline code and Korean characters:
- OK: `` `oos_read` 는 ``
- BAD: `` `oos_read`는 `` (Jira `{{}}` rendering breaks without space)

### Terminology
- OOS OID (not "pointer")
- OOS 파일 (not "OOS storage")
- OOS로 분리 저장 (not "moved to OOS")
- Use exact thresholds (512B, DB_PAGESIZE/8)
