# extendible_hash.md 학습 질문

아래 20개의 질문은 CUBRID의 `extendible_hash.c` 구현을 깊이 이해하기 위한 아키텍처 학습용 질문입니다.

---

## 기초 (Foundational)

1. Extendible hashing에서 global depth와 local depth의 의미는 각각 무엇이며, 두 값이 다를 수 있는 이유는 무엇인가?

2. `EHID` 구조체가 `VFID`(directory file ID)와 `PAGEID`(root page ID) 두 필드를 모두 보유하는 이유는 무엇인가? `VFID`만으로는 충분하지 않은가?

3. `EHASH_DIR_HEADER`의 `local_depth_count[0..32]` 배열은 어떤 값을 저장하며, directory shrink 조건 판단에 어떻게 활용되는가?

4. Bucket 페이지에서 slot 0이 항상 `EHASH_BUCKET_HEADER`로 예약되어 있는 이유는 무엇이며, 실제 `(key, OID)` 레코드는 어느 slot부터 시작하는가?

5. `FIND_OFFSET(hash_key, depth)` 매크로가 hash key의 상위 `depth` 비트를 directory index로 변환하는 방식을 설명하라. `GETBITS` 매크로에서 `pos=1`을 사용하는 이유는 무엇인가?

---

## 중급 (Intermediate)

6. `ehash_insert_helper`가 처음에 `S_LOCK`으로 시도한 뒤 bucket이 가득 찼을 때 `X_LOCK`으로 재시도하는 optimistic locking 패턴을 사용하는 이유는 무엇인가? 이 패턴이 없을 경우 발생하는 성능 문제는?

7. `ehash_dir_locate` 함수가 directory의 첫 번째 페이지(page 0)와 이후 페이지들을 다르게 처리하는 이유는 무엇인가? `EHASH_NUM_FIRST_PAGES`와 `EHASH_NUM_NON_FIRST_PAGES`가 서로 다른 값을 가지는 구조적 이유를 설명하라.

8. `ehash_expand_directory`에서 directory를 확장할 때 복사를 왜 "역방향(backward)"으로 수행해야 하는가? 정방향 복사를 사용했을 때 발생하는 문제는 무엇인가?

9. `ehash_find_first_bit_position`이 bucket split 시 `new_local_depth`를 결정하기 위해 기존 레코드들의 hash key를 XOR-accumulate하는 알고리즘을 설명하라. 모든 레코드의 hash key가 동일한 경우(hash collision) 어떻게 처리되는가?

10. `RVEH_CONNECT_BUCKET` 로그 레코드에서 `EHASH_REPETITION` 구조체(`VPID + count`)를 사용하는 이유는 무엇인가? directory doubling 시 개별 `EHASH_DIR_RECORD`마다 로그를 남기는 방식과 비교했을 때의 이점은?

11. `ehash_delete`가 delete 완료 후 페이지 latch를 모두 해제한 뒤에야 `ehash_merge`를 호출하는 이유는 무엇인가? 이 방식이 도입하는 race condition은 무엇이며, 어떻게 처리되는가?

12. `ehash_hash_string_type`이 단순한 단일 hash function 대신 `mht_1strhash`, `mht_2strhash`, `mht_3strhash` 세 가지를 조합하여 32-bit pseudo-key를 구성하는 이유는 무엇인가?

13. `ehash_check_merge_possible`에서 두 bucket의 결합 크기가 `EHASH_OVERFLOW_THRESHOLD`(90% of page size)를 초과하면 merge를 거부하는 이유는 무엇인가? threshold를 100%로 설정하지 않는 설계 의도는?

---

## 고급 (Advanced)

14. `log_sysop_start/commit/abort`로 감싸는 structural change(split, expand, merge, shrink)와 개별 레코드 변경을 `log_append_undoredo_data2`로 직접 로깅하는 방식을 혼용하는 기준은 무엇인가? 어떤 연산이 sysop 없이 처리되는가?

15. `ehash_split_bucket`이 bucket의 before-image를 undo로, after-image를 redo로 통째로 로깅(`RVEH_REPLACE`)하는 전략을 택한 이유는 무엇인가? 레코드 단위의 세밀한 로깅과 비교했을 때 trade-off는?

16. `ehash_shrink_directory`의 forward copy pass에서 source index `i * times`를 destination index `i`로 복사하는 로직이 정확히 어떤 directory 상태를 가정하고 있는가? 이 로직이 `ehash_expand_directory`의 backward copy와 대칭적으로 동작함을 설명하라.

17. `ehash_insert_to_bucket`에서 key가 이미 존재할 경우 OID를 in-place로 교체(`RVEH_REPLACE`)하는데, undo log에 original record 전체를 저장하고 redo log에 `(offset, new_OID)`만 저장하는 비대칭 로그 설계의 이유는 무엇인가?

18. `ehash_rv_insert_undo`가 단순 `ehash_rv_delete`를 호출하면서 bucket merge를 트리거하지 않는 이유는 무엇인가? 이 결정이 recovery 정확성에 미치는 영향은?

19. `EHASH_HASH_KEY`가 현재 32-bit `unsigned int`로 정의되어 있고 `GETBITS` 매크로에 `TODO: M2 64-bit` 주석이 남겨진 이유를 분석하라. hash key를 64-bit로 확장할 경우 directory 구조, 기존 on-disk 포맷, 그리고 `~0UL` 마스크 계산에 어떤 변경이 필요한가?

20. `ehash_map`이 directory를 순회하지 않고 bucket 파일 페이지를 순차적으로 순회하는 방식을 선택한 이유는 무엇인가? directory에 중복 포인터(같은 bucket을 가리키는 multiple entries)가 존재하는 상황에서 이 방식이 중복 처리를 피할 수 있는 이유를 설명하라.
