# AGENTS.md — CUBRID OOS Vault (Quartz v4)

A Quartz v4 static site hosting documentation for CUBRID OOS (Out-of-row Overflow Storage). Content is Obsidian-flavored Markdown in `content/`. The Quartz framework (TypeScript, Preact, unified/remark/rehype) builds it into a website.

## Build / Lint / Test Commands

```bash
npm run check             # Type-check (tsc --noEmit) + Prettier format check
npm run format            # Auto-fix formatting with Prettier
npm test                  # Run all tests (tsx --test — Node.js native test runner)
npx quartz build          # Build static site (content/ → public/)
npx quartz build --serve  # Build + dev server (port 8080, hot-reload via WS on 3001)
```

### Running a Single Test

```bash
npx tsx --test quartz/util/path.test.ts                                    # One file
npx tsx --test --test-name-pattern="slugify" quartz/util/path.test.ts      # Filter by name
```

### CI Pipeline (ci.yaml)

Runs on PRs and pushes to `v4`. Multi-platform (Windows, macOS, Ubuntu):

1. `npm ci` → 2. `npm run check` → 3. `npm test` → 4. `npx quartz build --bundleInfo -d docs`

## Project Structure

```
quartz.config.ts          # Site config (plugins, theme, metadata — title: "CUBRID OOS")
quartz.layout.ts          # Page layout (component arrangement)
content/                  # Markdown content (Obsidian vault) — OOS documentation lives here
  index.md                # Landing page
  OOS.md                  # Main OOS specification document (Korean)
  OOS-Test-Scenarios.md   # User-level test scenarios for OOS verification
quartz/
  components/             # Preact components (PascalCase .tsx)
  plugins/                # transformers/, filters/, emitters/ (camelCase .ts)
  util/                   # Shared utilities (camelCase .ts, tests co-located as .test.ts)
  styles/                 # Global SCSS styles
```

## Code Style (Quartz Framework)

### Formatting (Prettier — `.prettierrc`)

- Print width: 100 | No semicolons | Trailing commas: all | 2-space indent

### TypeScript (`tsconfig.json`)

- Strict mode (`strict: true`), no unused locals/parameters
- ESM modules (`"type": "module"`), JSX via Preact (`jsxImportSource: "preact"`)
- No ESLint — type-checking + Prettier only

### Naming & Imports

- Components: PascalCase `.tsx` | Plugins/utils: camelCase `.ts` | Tests: `.test.ts` co-located
- Always relative imports — no path aliases
- Named imports for utils/types, default imports for components
- Constants: `camelCase` (not UPPER_CASE — project convention)

### Key Patterns

- Components: factory functions with `satisfies QuartzComponentConstructor`
- Plugins: `Options + defaultOptions` merge pattern, three types (transformer/filter/emitter)
- Path types: branded strings (`FullSlug`, `SimpleSlug`, `FilePath`, `RelativeURL`)
- Errors: plain `Error` throws; fatal uses `trace()` from `quartz/util/trace.ts`
- Testing: `node:test` + `node:assert`, run via `tsx --test`

### Environment

- Node.js ≥ 22, npm ≥ 10.9.2 (enforced via `engines` + `.npmrc`)
- Tool versions: `mise.toml` (node = latest)

---

## OOS Content Domain

All OOS documentation is written in Korean. Content follows Obsidian Markdown conventions.

### What is OOS?

OOS (Out-of-row Overflow Storage) separates large variable-length columns from heap records into dedicated OOS files, reducing unnecessary disk I/O when only small columns are queried.

- **AS-IS**: CUBRID stores entire rows contiguously in slotted/overflow pages — reading any column reads the whole record.
- **TO-BE**: Large variable columns are stored separately in OOS files; heap records hold only 8-byte OOS OIDs pointing to the actual values.

### Core Terminology

| Term         | Definition                                                                   |
| ------------ | ---------------------------------------------------------------------------- |
| OOS Record   | Column data split from heap record, stored in OOS file                       |
| OOS File     | FILE_OOS type file, 1:1 with heap file (one per table)                       |
| OOS Page     | Slotted page within OOS file (size = DB_PAGESIZE)                            |
| OOS OID      | 8-byte pointer (volid, pageid, slotid) stored in heap record's variable area |
| HAS_OOS flag | Record-level flag — true if any column is OOS                                |
| IS_OOS flag  | Per-column flag in variable offset table entry                               |
| OOS Resolve  | Replacing OOS OIDs with actual values (expands record size)                  |

### OOS Trigger Conditions (Milestone 1)

1. Record threshold: `header + payload + mvcc_extra > DB_PAGESIZE / 8`
2. Column condition: `is_variable && column_size > 512 bytes`

Example with DB_PAGESIZE=16K: record must exceed 2K AND column must exceed 512B.

### Key Implementation Details

**Record format changes:**

- Variable offset table: lower 2 bits repurposed as flags (`OR_VAR_BIT_OOS = 0x1`)
- MVCC header: `OR_MVCC_FLAG_HAS_OOS = 0x08` (bit 3); size lookup uses lower 3 bits only

**Insert path:** `heap_attrinfo_determine_disk_layout()` → identify OOS candidates → `oos_insert()` per column → store OOS OIDs in heap record

**Read path:** Check `OR_IS_OOS(offset)` → `oos_read()` → `heap_record_replace_oos_oids_with_values_if_exists()` to reconstruct full record

**Update:** Always creates new OOS records (no OID reuse in Milestone 1). Old OOS OIDs are deleted immediately via `oos_delete()` and values are resolved back into the undo log record.

**Delete:** OOS OIDs cannot be resolved in-place (heap page size limit). OOS deletion deferred to vacuum (future milestone).

**Large values:** Chunked across multiple OOS pages with linked-list chain (reverse insertion order, each chunk header stores next OID).

### Source Code References

- Heap insert/read + OOS integration: `src/storage/heap_file.c`
- OOS file operations (insert/read/delete/chunk): `src/storage/oos_file.cpp`
- Record format macros: `object_representation.h`
- MVCC flag constant: `object_representation_constants.h`

### Milestone 1 Scope

**Included:** FILE_OOS/PAGE_OOS types, OOS insert/read (single + multi-chunk), heap↔OOS integration, replication support, recovery support.

**Excluded:** `oos_file_destroy`, compaction, bestspace optimization, OOS value deduplication on update, vacuum optimization for OOS.

---

## OOS Architecture — Deep Reference for Content Authors

This section provides detailed context that content authors (human or AI) need when writing or extending OOS documentation. Understand these internals before writing any OOS content.

### CUBRID Storage Fundamentals (Context for OOS)

CUBRID uses a **slotted page** model for heap storage. Key concepts:

- **Heap file**: One per table. Contains slotted pages. Each page holds multiple records via a slot directory.
- **Overflow page**: When a single record exceeds one page, CUBRID chains overflow pages. OOS replaces the need for overflow in the variable-column dimension.
- **MVCC (Multi-Version Concurrency Control)**: CUBRID uses in-place MVCC — records carry insert/delete transaction IDs in their MVCC header. Readers see a consistent snapshot without locks.
- **WAL (Write-Ahead Logging)**: All modifications are logged before being applied to pages. Recovery replays the log to restore consistency after crash.
- **Vacuum**: Background process that reclaims space from records no longer visible to any active transaction.

### OOS Data Flow — End to End

```
INSERT with OOS:
  heap_insert()
    → heap_attrinfo_determine_disk_layout()   // decide which columns → OOS
    → oos_insert() per OOS column              // write OOS records, get OOS OIDs
    → build heap record with OOS OIDs          // 8-byte OID replaces actual value
    → spage_insert() into heap page            // write compact heap record
    → WAL: log both heap insert + OOS inserts

SELECT with OOS:
  heap_get() / scan
    → read heap record from page
    → check OR_MVCC_FLAG_HAS_OOS
    → if set: heap_record_replace_oos_oids_with_values_if_exists()
      → for each IS_OOS column: oos_read() → fetch actual value
      → reconstruct full record (size expands significantly)
    → return to query executor

UPDATE with OOS (Milestone 1):
  heap_update()
    → resolve ALL old OOS OIDs to actual values → embed in undo log recdes
    → oos_delete() for each old OOS OID (immediate physical delete)
    → oos_insert() for new OOS columns → get new OOS OIDs
    → write updated heap record with new OOS OIDs
    → WAL: log undo (with resolved values) + redo (with new OOS OIDs)

DELETE with OOS (Milestone 1):
  heap_delete()
    → add MVCC delete ID to record
    → OOS OIDs remain in heap record (NOT resolved, NOT deleted)
    → vacuum will eventually handle OOS cleanup (future milestone)
```

### Record Binary Layout

```
Heap Record (on disk):
┌──────────────────┬──────────────────┬───────┬────────────────────────────────┐
│  MVCC Header     │  Variable Offset │ Fixed │  Variable Area                 │
│  (flags+txn ids) │  Table (VOT)     │ Cols  │  (values or 8-byte OOS OIDs)   │
└──────────────────┴──────────────────┴───────┴────────────────────────────────┘

VOT Entry (per variable column):
  [offset_value (30 bits) | RESERVED (1 bit) | IS_OOS (1 bit)]

  IS_OOS = 1  →  variable area contains OOS OID (8 bytes) at this offset
  IS_OOS = 0  →  variable area contains actual value at this offset

MVCC Header Flags (5 bits total):
  bit 0: has insert ID
  bit 1: has delete ID
  bit 2: has prev version LSA
  bit 3: HAS_OOS (new for OOS)
  bit 4: reserved

  ⚠ MVCC header size lookup uses only lower 3 bits (idx & 0x07)
     HAS_OOS (bit 3) does NOT affect header size — it's metadata only.
```

### Multi-Chunk OOS Chain

When a column value exceeds one OOS page, it's split into chunks stored as a linked list:

```
Insertion order (reverse):
  chunk_3 (tail of value) → inserted first, next_oid = NULL
  chunk_2 (middle)        → inserted second, next_oid = chunk_3 OID
  chunk_1 (head of value) → inserted last, next_oid = chunk_2 OID

  Heap record stores OOS OID pointing to chunk_1.

Read order (forward):
  Follow chain: chunk_1 → chunk_2 → chunk_3 → reassemble value

Each chunk record:
┌──────────────────┬─────────────────────────┐
│ next OOS OID     │ chunk data              │
│ (8 bytes, or     │ (up to max_chunk_size)  │
│  NULL if last)   │                         │
└──────────────────┴─────────────────────────┘
```

### Recovery & Replication Invariants

These invariants MUST hold — test scenarios should verify each one:

1. **WAL completeness**: Every OOS insert/delete is logged. After crash + recovery, OOS state matches the last committed transaction state.
2. **Undo correctness**: Update undo log contains fully-resolved OOS values (no OOS OIDs in undo records). Rollback restores the actual previous column values.
3. **No orphan OOS records after update**: On update, old OOS records are physically deleted before new ones are created. If crash occurs mid-update, recovery (undo) must clean up any partially-written new OOS records.
4. **Delete safety**: Deleted records retain OOS OIDs in heap. OOS records are NOT deleted at delete time — they remain accessible until vacuum.
5. **Replication log completeness**: Replication log must contain enough information to replay OOS operations on replica. OOS OIDs in replication log point to the correct OOS pages on the replica.
6. **OOS file ↔ heap file 1:1**: Each heap file has at most one OOS file. The OOS VFID is stored in the heap header page. If the OOS VFID is NULL, no OOS records exist for that table.

### Known Limitations (Milestone 1)

When writing about OOS, explicitly call out these limitations — readers must understand the current scope:

| Limitation                          | Impact                                  | Future Fix                   |
| ----------------------------------- | --------------------------------------- | ---------------------------- |
| No `oos_file_destroy`               | OOS files grow indefinitely             | Milestone 2+                 |
| No across-page compaction           | Fragmentation over time                 | Future milestone             |
| Bestspace = last-insert-page only   | Hotspot on single page, wasted space    | Global bestspace tracking    |
| No OOS OID reuse on update          | Extra I/O even when OOS value unchanged | Deduplication in update path |
| Delete doesn't clean OOS            | Orphan OOS records until vacuum         | Vacuum integration           |
| PEEK mode unsupported for OOS reads | Always COPY semantics, extra memcpy     | Future optimization          |
| `S_DOESNT_FIT` handling incomplete  | Caller must handle buffer overflow      | Upper-layer fixes            |

---

## Content Writing Guidelines

### Inline Code Adjacent to Korean

When inline code (`` `xxx` ``) is immediately adjacent to Korean characters, always add a space between them. This is required because pandoc's Jira conversion renders `` `xxx` `` as `{{xxx}}`, and Jira's `{{...}}` monospace markup does not render correctly when directly touching Korean text.

- ❌ `` `oos_read`는 `` → ✅ `` `oos_read` 는 ``
- ❌ `` `unloaddb`에서 `` → ✅ `` `unloaddb` 에서 ``
- ❌ `한국어`code `` → ✅ `한국어` `code`

Apply this rule to all `.md` files in `content/`.

### Language & Format

- Write in **Korean** (all existing content is Korean)
- Use **Obsidian-flavored Markdown** (wikilinks `[[...]]`, callouts `> [!note]` supported via Quartz plugins)
- Place new documents in `content/` — Quartz ignores `private/`, `templates/`, `.obsidian/`
- Use horizontal rules (`---`) to separate major sections

### Document Structure Pattern

Follow the existing OOS.md structure:

```markdown
# Title

Brief 1-2 sentence summary.

- Key bullet points

---

## Section

### Subsection

Description paragraph.

코드 레퍼런스

- `src/storage/relevant_file.c`

---
```

### Code Examples

- **SQL examples**: Use fenced code blocks with `sql` language tag. Show both the input and expected behavior.
- **C/C++ code**: Use `c` or `cpp` language tags. Include the source file path as comment or reference.
- **Pseudocode / Record layouts**: Use plain fenced blocks (no language tag) with ASCII art.

```sql
-- GOOD: Show setup + operation + expected result
create table tbl (id int, vc1 varchar, vc2 varchar);
insert into tbl values (1, REPEAT('a', 1700), REPEAT('b', 600));
-- vc1, vc2 둘 다 OOS로 저장됨 (record > 2K, 각 column > 512B)
```

```c
// GOOD: Include source path reference
// src/storage/oos_file.cpp
int oos_insert(THREAD_ENTRY *thread_p, VFID *vfid, RECDES *recdes, OID *oos_oid)
{
  // ... key logic
}
```

### Terminology Consistency

Always use the exact terms from the Core Terminology table above. Do NOT use:

- ❌ "OOS pointer" → ✅ "OOS OID"
- ❌ "OOS storage" (ambiguous) → ✅ "OOS 파일" or "OOS 레코드"
- ❌ "moved to OOS" → ✅ "OOS로 분리 저장"
- ❌ "overflow storage" → ✅ "OOS (Out-of-row Overflow Storage)"

### Content Quality Checklist

Before publishing any OOS content, verify:

- [ ] All SQL examples are syntactically correct for CUBRID
- [ ] All source code references point to actual CUBRID source paths
- [ ] Trigger conditions (record > DB_PAGESIZE/8, column > 512B) are stated correctly
- [ ] Milestone 1 limitations are explicitly mentioned where relevant
- [ ] Binary layout descriptions match the actual implementation (VOT flags, MVCC header bits)
- [ ] JIRA references (e.g., `CBRD-26517`) are included where applicable
- [ ] Korean text is natural and uses consistent technical vocabulary

### What NOT to Write

- Do not speculate about future milestone implementations beyond what's documented
- Do not describe `oos_file_destroy`, vacuum-OOS integration, or OOS deduplication as if they exist — they are NOT implemented in Milestone 1
- Do not describe PEEK mode support for OOS reads — it is not supported
- Do not imply OOS OIDs can be shared between records — each OOS OID is referenced by exactly one record
