---
title: CUBRID OOS Vault
---

# CUBRID OOS (Out-of-row Overflow Storage)

Knowledge base for the CUBRID OOS project — separating large variable-length columns from heap records into dedicated OOS files to reduce unnecessary disk I/O.

## Core Documents

- [OOS Presentation](OOS-Presentation.md) — Architecture overview, record format, CRUD flows, MVCC integration (Marp slides)
- [OOS Test Scenarios](OOS-Test-Scenarios.md) — ACID, crash recovery, MVCC, replication test cases using VARBIT patterns
- [OOS Schedule](oos-schedule.md) — M2 milestone timeline (3/10–4/17), task assignments, weekly plan
- [OOS TODO](oos-todo.md) — Known bugs, open issues (CBRD-26458 unloaddb perf, CBRD-26516 redundant oos_read)

## Reference (CUBRID Internals)

- [Heap Bestspace Algorithm](reference/heap_bestspace_algorithm.md) — How CUBRID picks which heap page to insert into
- [Page Buffer Analysis (EN)](reference/page_buffer_analysis_report.md) — `page_buffer.c` code analysis report
- [Page Buffer Analysis (KO)](reference/page_buffer_analysis_report_ko.md) — 페이지 버퍼 모듈 분석 보고서 (한국어)

## Quick Reference

| Concept | Detail |
|---------|--------|
| OOS trigger | record > `DB_PAGESIZE/8` (2KB) AND column > 512B |
| OOS file type | `FILE_OOS`, one per heap file (1:1) |
| OOS pointer | 8-byte OOS OID (volid, pageid, slotid) in variable area |
| MVCC flags | `OR_MVCC_FLAG_HAS_OOS` (bit 3), `OR_VAR_BIT_OOS` (bit 0) |
| Key source | `heap_file.c`, `oos_file.cpp`, `object_representation.h` |

## JIRA

- [CBRD-26517](http://jira.cubrid.org/browse/CBRD-26517) — Main OOS tracking issue
- [CBRD-26458](http://jira.cubrid.org/browse/CBRD-26458) — unloaddb `heap_next` performance regression
- [CBRD-26516](http://jira.cubrid.org/browse/CBRD-26516) — UPDATE redundant `oos_read` calls
