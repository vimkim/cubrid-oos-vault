# byte_order.md 학습 질문

아래 질문들은 CUBRID `byte_order.c/h` 모듈의 아키텍처와 이식성 설계를 깊이 이해하기 위한 학습 질문입니다.

1. `OR_LITTLE_ENDIAN = 1234`, `OR_BIG_ENDIAN = 4321`이라는 매직 넘버 값을 선택한 이유는 무엇인가? 이 값들이 `<endian.h>`의 `__LITTLE_ENDIAN`, `__BIG_ENDIAN`과 어떻게 대응되는가?

2. `byte_order.h`에서 엔디언 감지를 런타임이 아닌 컴파일 타임에 수행하는 방식의 장점과 한계는 무엇인가? 런타임 감지가 필요한 상황이 있다면 어떤 경우인가?

3. 빅 엔디언 플랫폼(HPUX, AIX, SPARC)에서 `ntohs`, `ntohl`, `ntohi64` 등이 모두 항등 매크로(identity macro)로 정의되는 이유는 무엇인가?

4. `OR_HAVE_NTOHF`, `OR_HAVE_NTOHD` 등의 가용성 매크로가 Linux에서 정의되지 않아 `byte_order.c`에서 폴백(fallback) 구현이 컴파일되는 이유는 무엇인가?

5. `MOVING_VAN` union이 `double`을 `unsigned int buf[2]`로 재해석하는 방식으로 설계된 이유는 무엇인가? 직접 포인터 캐스트를 사용하지 않는 이유(C 표준 aliasing rules)를 설명하라.

6. IA-64(Itanium) 아키텍처에서 `OR_MOVE_DOUBLE`이 struct 복사 대신 `memcpy`를 사용해야 하는 이유는 무엇인가? strict alignment 규칙이란 무엇인가?

7. `ntohi64`와 `htoni64`가 OS에서 제공되지 않아 항상 `byte_order.c`에서 구현되는 이유는 무엇인가? 이 함수들이 구현하는 64비트 바이트 스왑 알고리즘을 설명하라.

8. `htonf`/`ntohf`와 `htond`/`ntohd`가 값(value) 방식이 아닌 포인터(ptr/value 쌍) 방식의 함수 시그니처를 갖는 이유는 무엇인가?

9. `byte_order` 모듈이 완전히 무상태(stateless)로 설계된 의미는 무엇이며, 이것이 멀티스레드 환경에서 갖는 이점은 무엇인가?

10. `object_representation.h`의 `OR_PUT_*`/`OR_GET_*` 매크로들이 `byte_order.h`의 함수들을 광범위하게 사용하는 이유는 무엇인가? 디스크 직렬화에서 엔디언 변환이 항상 필요한 이유를 설명하라.

11. `swap64` 매크로가 직렬화 매크로에서 사용되는 구체적인 역할은 무엇이며, `ntohi64`와의 차이점은 무엇인가?

12. `byte_order.c`가 `memory_wrapper.hpp`를 마지막으로 포함해야 한다는 프로젝트 규칙의 목적은 무엇이며, 순서가 바뀌면 어떤 문제가 발생할 수 있는가?

13. CUBRID가 네트워크 바이트 순서(big-endian)를 디스크 직렬화의 표준으로 사용하는 설계 결정의 이유는 무엇인가? 플랫폼 네이티브 바이트 순서를 사용했을 때의 문제점은?

14. `cas_net_buf.c`에서 `ntohi64`를 직접 호출하여 BIGINT를 역직렬화하는 이유는 무엇인가? CS_MODE 클라이언트에서 엔디언 변환이 필요한 상황은?

15. Windows MSVC >= 1700(VS 2012+)에서 `OR_HAVE_NTOHF`, `OR_HAVE_NTOHD` 등이 정의되는 반면 Linux에서는 정의되지 않는 이유는 무엇인가?

16. `UINT32`가 `float` 변환에서 비트 패턴 전달체(carrier)로 사용되는 방식을 설명하라. `float`를 직접 네트워크 전송하면 안 되는 이유는 무엇인가?

17. `OR_BYTE_ORDER` 매크로가 `sparc` 식별자를 빅 엔디언으로 분류하는 이유는 무엇인가? SPARC 아키텍처의 엔디언 특성을 설명하라.

18. `byte_order` 모듈의 리버스 의존성(reverse dependency)이 `object_representation.h`를 포함하는 거의 모든 파일로 확장되는 구조적 이유는 무엇인가?

19. `ntohl`/`htonl`이 OS에서 제공되는 경우(Linux, Windows 등) `byte_order.c`의 폴백 구현이 컴파일되지 않도록 하는 메커니즘을 설명하라.

20. 미래에 CUBRID가 네이티브 빅 엔디언 플랫폼 지원을 완전히 제거한다면, `byte_order` 모듈의 어떤 부분을 어떻게 단순화할 수 있겠는가? 그럼에도 유지해야 할 부분은 무엇인가?
