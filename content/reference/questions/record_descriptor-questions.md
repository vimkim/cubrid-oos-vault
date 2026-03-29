# record_descriptor.md 학습 질문

아래 질문들은 CUBRID `record_descriptor` C++ 래퍼 클래스의 아키텍처를 깊이 이해하기 위한 학습 질문입니다.

1. `record_descriptor` 클래스가 기존 C 구조체 `recdes`를 래핑하는 이유는 무엇인가? 원본 `recdes`를 직접 사용했을 때 발생하는 버퍼 관리 문제를 설명하라.

2. `recdes.area_size < 0`이 "페이지 슬롯 내부를 peek하고 있는 상태"를 나타내는 관례가 된 이유는 무엇인가? 이 음수 규약이 타입 안전성 측면에서 갖는 위험성은?

3. `data_source` enum의 네 가지 상태(`INVALID`, `PEEKED`, `COPIED`, `ALLOCED`)가 각각 어떤 메모리 소유권 의미를 갖는가? `PEEKED` 상태에서 데이터를 수정하면 안 되는 이유는?

4. `record_descriptor`가 `cubpacking::packable_object`를 상속하는 이유는 무엇인가? 레코드를 네트워크 전송이나 직렬화 컨텍스트에서 pack/unpack해야 하는 유스케이스는 무엇인가?

5. `cubmem::extensible_block`이 단순 `malloc/free` 대신 사용되는 이유는 무엇인가? 레코드 데이터 버퍼가 여러 번 재할당(grow) 될 수 있는 상황은 언제인가?

6. `spage_get_record()`에서 `S_DOESNT_FIT` 반환 시 버퍼를 자동으로 확장하고 재시도하는 패턴이 `record_descriptor::get()`에 내장된 이유는 무엇인가? 이 패턴이 없으면 호출자가 해야 할 작업은?

7. `record_get_mode` enum이 기존 `bool PEEK/COPY` 상수를 scoped enum으로 감싸는 이유는 무엇인가? `PEEK = true`, `COPY = false`라는 기존 값이 반직관적인 이유는?

8. `record_descriptor`의 크기가 64바이트, 정렬이 8바이트인 이유는 무엇인가? 이 크기에 vtable 포인터가 포함되는 이유는?

9. `record_descriptor`가 모든 빌드 모드(`SERVER_MODE`, `SA_MODE`, `CS_MODE`)에서 컴파일되는 이유는 무엇인가? 클라이언트 코드에서 `record_descriptor`를 사용하는 유스케이스는 무엇인가?

10. `m_own_data` 필드가 `cubmem::extensible_block` 타입이고 `m_recdes.data`와 별도로 존재하는 이유는 무엇인가? 두 포인터(`m_own_data` vs `m_recdes.data`)가 다른 주소를 가리킬 수 있는 상황은?

11. `locator_sr.c`에서 `std::vector<record_descriptor>`를 멀티 레코드 삽입에 사용하는 이유는 무엇인가? C 스타일 `recdes` 배열 대비 `vector<record_descriptor>`의 이점은?

12. `record_descriptor`가 copy constructor와 move constructor를 어떻게 처리해야 하는가? `PEEKED` 상태의 `record_descriptor`를 복사할 때 발생하는 소유권 문제는?

13. `heap_attrinfo_transform_to_disk*()`에서 `record_descriptor`를 출력 파라미터로 받는 이유는 무엇인가? 함수 내부에서 버퍼 크기를 알 수 없을 때 `extensible_block`이 어떻게 도움이 되는가?

14. `memory_wrapper.hpp`가 `record_descriptor.cpp`의 마지막 include로 강제되는 CUBRID 프로젝트 규칙이 `record_descriptor` 클래스의 메모리 추적에 미치는 영향은 무엇인가?

15. `recdes.type` 필드의 `REC_HOME`, `REC_BIGONE`, `REC_RELOCATION` 등 레코드 타입이 `record_descriptor` 래퍼를 통해 접근할 때 어떻게 노출되는가?

16. `record_descriptor::peek()`과 `record_descriptor::copy()`의 차이가 이후 `set_data_after_play()` 같은 변경 작업에서 어떤 제약을 만드는가?

17. `load_server_loader.cpp`에서 bulk loading 시 `record_descriptor`를 local 변수로 선언하는 이유는 무엇인가? 루프마다 새로 할당하지 않고 재사용하는 최적화가 가능한 이유는?

18. C 코드(`heap_file.c` 등)에서 `record_descriptor *`를 파라미터로 받는 함수 시그니처가 사용되는 이유는 무엇인가? C 파일이 C++ 클래스 포인터를 사용할 수 있는 컴파일 조건은?

19. `record_descriptor`의 `data_source`가 `INVALID`인 상태에서 데이터 접근을 시도할 때 어떤 안전장치가 있는가? assert 기반 검사와 런타임 에러 코드 반환 방식 중 어느 것이 사용되는가?

20. `recdes` C 구조체와 `record_descriptor` C++ 클래스가 장기적으로 공존하는 상황에서 신규 코드는 어떤 타입을 선택해야 하며, 기존 C 인터페이스와의 호환성을 유지하는 방법은 무엇인가?
