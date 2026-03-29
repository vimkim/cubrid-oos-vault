# page_buffer_qa.md 학습 질문

답변: [[page_buffer_qa-answers]]

아래 질문들은 CUBRID page buffer 내부 구조를 깊이 이해하기 위한 아키텍처 학습용 질문입니다.

---

## 기초 (1-5)

1. `PGBUF_BCB` 구조체가 `PGBUF_IOPAGE_BUFFER` 구조체와 분리된 배열로 관리되는 이유는 무엇인가요? 두 구조체를 하나로 합치면 캐시 성능에 어떤 영향이 생길까요?

2. `pgbuf_fix()` 함수의 역할을 설명하고, 호출 시 반드시 전달해야 하는 인자들(`VPID`, `fetch_mode`, `latch_mode`, `condition`)이 각각 무엇을 의미하는지 설명해 주세요.

3. CUBRID page buffer의 hash table 버킷 수가 2^20(약 100만 개)로 고정된 이유는 무엇이며, `pgbuf_hash_func_mirror()` 함수가 bit-reversal 기법을 사용하는 이유는 무엇인가요?

4. `pgbuf_unfix()` 호출 후 페이지 포인터를 다시 사용하면 안 되는 이유는 무엇인가요? unfix 이후 해당 BCB에 어떤 일이 일어날 수 있나요?

5. CUBRID page buffer에서 WAL(Write-Ahead Logging) 원칙이란 무엇이며, 페이지를 디스크에 flush하기 전에 반드시 수행해야 하는 작업은 무엇인가요?

---

## 중급 (6-13)

6. BCB의 `atomic_latch` 필드가 64비트 atomic 하나로 `latch_mode`, `waiter_exists`, `fcnt`를 모두 담는 설계의 핵심 이유는 무엇인가요? `pgbuf_lockfree_fix_ro()` 함수에서 이 설계가 어떻게 lock-free fast path를 가능하게 하는지 설명해 주세요.

7. BCB의 `flags` 필드가 zone 정보(LRU zone, LRU list index)와 상태 플래그(DIRTY, FLUSHING_TO_DISK 등)를 하나의 `volatile int`에 함께 담는 이유는 무엇인가요? 이 두 정보를 별도 필드로 분리하면 어떤 문제가 생길까요?

8. `PGBUF_HOLDER` 시스템이 존재하는 이유는 무엇인가요? BCB의 `fcnt`만으로 fix 횟수를 추적하면 어떤 정보를 알 수 없게 되나요?

9. 3-zone LRU에서 zone 1(hot), zone 2(warm/buffer), zone 3(cold)으로 구분하는 이유는 무엇이며, Aout list(2Q algorithm)가 새로운 페이지 로드 시 LRU 배치 위치를 결정하는 데 어떤 역할을 하나요?

10. `pgbuf_search_hash_chain()` 함수가 two-phase locking 패턴(Phase 1: mutex 없이 탐색 → Phase 2: hash_mutex 획득 후 탐색)을 사용하는 이유는 무엇인가요? 두 mutex를 동시에 보유하지 않는 것이 왜 중요한가요?

11. `OLD_PAGE_PREVENT_DEALLOC`과 `OLD_PAGE_MAYBE_DEALLOCATED` fetch mode의 차이는 무엇이며, 각각 어떤 상황에서 사용되나요?

12. `pgbuf_promote_read_latch()` 함수가 `ER_PAGE_LATCH_PROMOTE_FAIL`을 반환할 수 있는 두 가지 조건은 무엇이며, 호출자는 promotion 실패 시 어떻게 대응해야 하나요?

13. 4개의 background daemon(`Page_maintenance`, `Page_flush`, `Page_post_flush`, `Flush_control`)이 각자 담당하는 역할은 무엇이며, 이들이 하나의 daemon으로 통합되지 않고 분리된 이유는 무엇이라고 생각하나요?

---

## 고급 (14-20)

14. `pgbuf_lockfree_fix_ro()` 함수에서 VPID가 6바이트(pageid 4바이트 + volid 2바이트)이기 때문에 단일 atomic load가 보장되지 않는다는 torn read 문제가 있습니다. x86에서 실질적 위험이 낮은 이유를 설명하고, ARM 같은 약한 메모리 모델(weak memory model) 플랫폼에서 이 문제가 실제 위험으로 이어질 수 있는 이유를 설명해 주세요.

15. CUBRID page buffer가 페이지 latch deadlock을 방지하지 않는다고 선언하고(`"We do not guarantee that there is no deadlock between page latches"`), 대신 300초 timeout으로 deadlock을 감지하는 설계를 선택한 이유는 무엇인가요? 이 trade-off의 장단점을 설명해 주세요.

16. `pgbuf_ordered_fix()` 함수가 deadlock 방지를 위해 기존 페이지를 unfix한 뒤 재fix하는 과정에서, 그 사이에 페이지 내용이 변경될 수 있습니다. `page_was_unfixed` 플래그와 `pgbuf_bcb_register_avoid_deallocation()` 호출이 이 문제를 어떻게 각각 다른 방식으로 보호하는지 설명해 주세요.

17. `CAST_PGPTR_TO_BFPTR` 매크로가 type-unsafe한 이유는 무엇이며, debug 모드와 release 모드에서 잘못된 포인터를 전달했을 때 동작이 어떻게 다른가요? 이를 C++ template 기반 inline 함수로 개선하면 어떤 이점이 생기나요?

18. `volatile int flags`가 x86에서는 실질적으로 문제가 없지만 ARM/RISC-V에서는 memory ordering 보장이 불충분한 이유를 TSO(Total Store Ordering)와 약한 메모리 모델의 차이를 들어 설명해 주세요. `std::atomic<int>`으로 교체할 때 각 읽기/쓰기 연산에 어떤 `memory_order`를 지정해야 할까요?

19. `count_fix_and_avoid_dealloc` 필드가 하나의 `volatile int`에 fix count(상위 16비트)와 dealloc 방지 카운터(하위 16비트)를 함께 담는 이유를 설명하고, C++11의 `std::atomic<uint16_t>`로 분리하는 것이 실질적으로 더 나은지 논하세요.

20. BCB mutex monitor가 동시에 최대 2개의 BCB mutex만 허용하는 이유는 무엇이며, 이 제한을 초과하면 왜 deadlock 위험이 급증하나요? 또한 `pgbuf_Pool` 전역 상태 의존성이 page buffer 모듈의 단위 테스트를 어렵게 만드는 구체적인 이유를 설명하고, 테스트 가능성을 높이기 위한 현실적인 개선 방향을 제시해 주세요.
