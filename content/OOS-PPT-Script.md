# OOS 설계 발표 스크립트

> 대상: 개발 2팀 (OOS에 대해 잘 모르는 개발자)
> 형식: 슬라이드별 발표 스크립트
> 기준: OOS-Presentation.md 최종본

---

## 슬라이드 1: 타이틀

> 안녕하세요. 오늘은 OOS, Out-of-row Overflow Storage라는 이름으로 진행 중인 프로젝트에 대해 소개드리겠습니다. 큐브리드에서 큰 가변 컬럼을 힙 레코드로부터 분리 저장하는 구조인데, 왜 필요한지부터 시작해서 어떻게 설계했는지까지 쭉 이야기하겠습니다.

---

## 슬라이드 2: 목차

> 전체 흐름은 이렇습니다. OOS가 뭔지, 왜 필요한지부터 시작해서, 다른 DB들은 어떻게 하고 있는지 비교하고, 큐브리드에서 어떤 구조를 선택했는지 설명드립니다. 그 다음 CRUD 연산별 동작, 로깅, 복구, 복제를 다루고, 마지막으로 현재 고민거리와 향후 계획까지 이야기하겠습니다.

---

## 슬라이드 3: 섹션 — 1. OOS란 무엇인가?

> (넘기면서) 먼저 OOS가 뭔지부터 보겠습니다.

---

## 슬라이드 4: RDBMS의 기본 저장 방식

> 배경부터 간단히 짚겠습니다. RDBMS는 기본적으로 하나의 row를 디스크에 연속적으로 저장합니다. 저장 단위가 DB page인데, 보통 8K에서 16K 사이입니다.
>
> `INSERT INTO ... VALUES (...)`를 하면, 괄호 안에 들어가는 값들이 통째로 하나의 레코드에 저장됩니다. 많은 OLTP 데이터베이스들이 이런 구조를 취하고 있는데, 이유는 OLTP 워크로드에서는 특정 row의 값을 전부 가져오는 경우가 많다고 가정하기 때문입니다.

---

## 슬라이드 5: OOS의 핵심 아이디어

> OOS는 Out-of-row Overflow Storage의 약자입니다. 핵심 아이디어는 단순합니다.
>
> 하나의 레코드에서 특정 컬럼의 크기가 너무 크다면, 그 컬럼만 별도의 파일에 분리 저장하고, 원래 위치에는 "여기 가면 진짜 값이 있다"는 8바이트짜리 포인터만 남겨놓는 겁니다.
>
> 그림을 보시면, AS-IS에서는 big_text 1.7KB, big_blob 2KB가 전부 하나의 heap record에 들어가 있습니다. TO-BE에서는 이 자리에 8바이트 OOS OID만 남기고, 실제 값은 OOS 파일에 따로 저장합니다.

---

## 슬라이드 6: 섹션 — 2. 왜 필요한가?

> (넘기면서) 그러면 왜 이게 필요한지 보겠습니다.

---

## 슬라이드 7: Disk I/O가 성능의 핵심

> DB에서 Disk I/O는 성능에 가장 큰 영향을 주는 요인들 중 하나입니다. 옵티마이저, 쿼리 프로세서도 기본적으로 Disk I/O를 줄이는 방향으로 설계되어 있고, 온갖 최적화를 해봐도 Disk I/O 수 자체가 크면 느립니다.
>
> OOS는 특정 상황에서 이 Disk I/O를 줄이는 근본적인 레벨의 최적화입니다. 예를 들어 `SELECT id FROM tbl`을 보시면, id는 4바이트인데 AS-IS에서는 big_text 1.7KB, big_blob 2KB까지 전부 읽어야 합니다. TO-BE에서는 작아진 heap record만 읽으면 되니까, OOS 파일 접근 자체가 불필요해집니다.

---

## 슬라이드 8: 이 프로젝트의 가치

> 이 프로젝트의 가치는 꽤 높다고 생각합니다. 우리 팀이 안 하면 결국 다른 팀이 다른 방식으로든 시도할 만한 가치가 있는 프로젝트이고, 실제로 다른 팀에서도 관심이 많습니다.
>
> 핵심은 이겁니다. ACID를 지키기 위해서 사용자가 "이 데이터는 저장되어야 한다"고 선언하면 반드시 디스크로 내려가야 하지만, 읽을 때까지 그 전부를 다 읽어올 필요는 없습니다. OOS는 이 "읽기 비효율"을 해결하는 겁니다.

---

## 슬라이드 9: 섹션 — 3. 다른 DB들은 어떻게 해결하는가?

> (넘기면서) 이 문제는 큐브리드만의 고민이 아닙니다. 다른 DB들이 어떻게 하고 있는지 보겠습니다.

---

## 슬라이드 10: 같은 문제, 다른 이름

> 모든 row-store 데이터베이스가 마주하는 문제입니다. PostgreSQL은 TOAST, MySQL InnoDB는 Off-page Column Storage라는 이름으로 이미 해결하고 있습니다. 이름은 다르지만 핵심 아이디어는 동일합니다. "큰 컬럼을 row 밖으로 빼서 따로 저장한다."

---

## 슬라이드 11: PostgreSQL — TOAST

> PostgreSQL의 TOAST를 먼저 보겠습니다. TOAST는 "The Oversized-Attribute Storage Technique"의 약자입니다. Row 크기가 약 2KB를 넘으면 자동 발동됩니다.
>
> 동작 순서는, 먼저 pglz나 lz4로 압축을 시도하고, 그래도 크면 별도의 TOAST 테이블로 빼냅니다. 원래 tuple에는 18바이트짜리 TOAST 포인터만 남깁니다.
>
> 특이한 점은 컬럼별로 저장 전략을 지정할 수 있다는 겁니다. 기본은 EXTENDED로 압축 먼저, 그래도 크면 분리합니다. 그리고 한 가지 주목할 점은, TOAST 안 된 컬럼만 UPDATE하면 TOAST 포인터가 그대로 유지된다는 겁니다. 이건 우리 OOS Milestone 1과 다른 부분인데, 뒤에서 다시 비교하겠습니다.

---

## 슬라이드 12: MySQL InnoDB — Off-page Column Storage

> MySQL InnoDB를 보겠습니다. InnoDB는 row가 page 크기의 절반, 보통 8KB를 초과하면 큰 컬럼을 off-page로 보냅니다.
>
> ROW_FORMAT에 따라 동작이 좀 다릅니다. 예전 COMPACT 포맷은 큰 컬럼의 앞 768바이트를 inline에 남기고 나머지만 overflow page로 보냈는데, inline에 768B나 남아서 공간 효율이 좋지 않았습니다.
>
> MySQL 5.7부터 기본인 DYNAMIC 포맷은 큰 컬럼을 통째로 off-page로 보내고, inline에는 20바이트 포인터만 남깁니다.
>
> 핵심은, SELECT 시 off-page 컬럼을 요청하지 않으면 overflow page에 접근하지 않는다는 점입니다. 그리고 MySQL은 internal fragmentation을 punch hole system call로 최적화한다는 것도 참고할 만합니다.

---

## 슬라이드 13: 비교 요약

> 세 DB를 표로 정리한 겁니다. 공통점은 모두 컬럼 단위로 큰 값을 분리 저장한다는 것이고, 포인터로 가리킨다는 점도 같습니다.
>
> 차이점을 보면, PG는 압축을 적극적으로 사용하고, MySQL은 ROW_FORMAT에 따라 동작이 달라집니다. 우리 OOS는 Milestone 1에서는 압축 없이 순수하게 분리 저장만 합니다.
>
> OOS OID가 8바이트인 것은 기존 큐브리드 OID 구조를 그대로 활용하기 때문입니다. PG의 18바이트, MySQL의 20바이트보다 작습니다.
>
> 마지막 행이 중요한데, PG와 MySQL은 변경되지 않은 컬럼의 포인터를 재사용합니다. 우리 OOS Milestone 1은 UPDATE 시 항상 새 OOS OID를 발급하는 방식입니다. 이건 향후 마일스톤에서 개선할 부분입니다.

---

## 슬라이드 14: 섹션 — 4. 큐브리드는 어떻게 해결하려 하는가?

> (넘기면서) 이제 큐브리드의 접근 방법을 보겠습니다.

---

## 슬라이드 15: 현재 큐브리드 — Overflow Page

> 현재 큐브리드는 overflow page 구조를 사용하고 있습니다. 레코드 크기가 일정 수준 이상을 넘어서면, 레코드를 통으로 Overflow file에 저장하고 해당 위치 정보만 남겨둡니다.
>
> 문제점이 세 가지 있습니다. 첫째, 컬럼 단위 분리가 아니라 레코드 전체를 빼버립니다. 둘째, 하나의 page에 1개의 record만 저장하니까 내부 단편화가 심합니다. 셋째, MySQL처럼 punch hole 최적화도 없습니다.

---

## 슬라이드 16: OOS — Overflow Page의 진화

> OOS는 기존 Overflow 구조를 발전시킨 것으로 이해하시면 됩니다. CUBRID 개발자분들에게는 "Overflow File + Overflow Page와 기본적으로 동일하다"고 말씀드리는 게 가장 빠릅니다.
>
> 핵심 차이는 두 가지입니다. 분리 단위가 레코드 전체에서 컬럼 단위로 바뀌었고, 페이지 구조가 전용 Overflow page에서 slotted page로 바뀌었습니다. 파일 매핑은 동일하게 테이블 1개당 OOS file 1개입니다.

---

## 슬라이드 17: 왜 Slotted Page인가?

> 왜 Overflow page 형식 대신 slotted page를 선택했는지 설명드리겠습니다.
>
> Overflow page는 1 record / 1 page라서 다루기가 쉽고 MySQL도 이 형식을 쓰지만, internal fragmentation이 발생합니다. MySQL은 이걸 punch hole syscall로 최적화하는데, CUBRID는 그런 최적화가 없습니다.
>
> Slotted page로 바꾸면 하나의 page에 여러 OOS record를 저장할 수 있어서 내부 단편화가 줄어듭니다. 대신 단점이 있는데, 같은 page에 서로 다른 record의 OOS 값이 있으면 UPDATE 시 page lock 경합이 발생할 수 있습니다. 두 트랜잭션이 다른 record를 수정하는데도 같은 page면 기다려야 하는 상황이 생길 수 있어서, update 벤치마크에서 우려가 있습니다.

---

## 슬라이드 18: 왜 우리 팀이 해야 하는가?

> 여기서 좀 더 근본적인 이야기를 하겠습니다. 큐브리드는 RECDES를 가장 낮은 단위의 디스크 데이터 단위로 사용하고 있습니다. 값의 형태를 세 가지로 나눠보면, 메모리에서 쓰는 DB Value, 네트워크 전송용 OR, 디스크 저장용 OR이 있는데, 큐브리드는 2번과 3번을 혼용합니다. RECDES를 그대로 네트워크에 보내고, 그대로 디스크에 저장해왔습니다.
>
> 지금까지는 그래도 문제가 없었는데, OOS를 도입하면 상황이 달라집니다. 각 레이어에서 OOS locator를 전부 고려해줘야 합니다. 복구 로그, 복제 로그, heap과 OOS page의 일관성, network serialization, unloaddb, createdb, CDC까지. 이런 걸 다 하려면 결국 스토리지 레이어를 다루고 있는 우리 팀만이 할 수 있는 업무입니다.

---

## 슬라이드 19: 대안은 없었는가?

> 현재 구조만 고민한 건 아닙니다. 대안 두 가지를 검토했습니다.
>
> 첫째, 4팀에서 제안한 방법인데, Overflow page 형식을 배껴서 큰 컬럼만 Overflow로 보내는 hacky한 방법입니다. 많이 변경하지 않아도 될 수 있지만, 네트워크 레이어나 unloaddb, 로깅에서는 여전히 큰 RECDES를 통으로 보내니까, 어차피 그 부분들을 고쳐야 합니다. 그래서 결국 손이 많이 듭니다.
>
> 둘째, PG처럼 내부 TOAST 테이블을 사용하는 방법입니다. 내부 테이블 API를 활용하면 될 것 같지만, 락을 여러 번 걸어야 해서 성능 저하가 우려되고, 두 테이블의 트랜잭션 sync를 맞춰야 합니다. 그리고 PG는 2001년부터 TOAST에 최적화해온 건데 그 내공이 우리 아키텍처에 안 맞을 위험이 있습니다.

---

## 슬라이드 20: 왜 현재 구조를 선택했는가?

> 결정적인 이유를 정리하면 이렇습니다. 우리 팀이 heap, slotted page API에 가장 익숙합니다. low-level API라서 동작 파악이 쉽고, 여차하면 바꿔버릴 수 있습니다. 상위 API로 올라갈수록 추상화가 많아서 내공이 필요하고 리스크가 커집니다. 그리고 Overflow file 구현이 이미 존재하니까 패턴을 참고할 수 있었습니다.
>
> 한 줄로 요약하면, Overflow file을 배껴서 기본 동일한 구조인데, Overflow page가 OOS page로 바뀌고, 1 record/page가 slotted page로 바뀐 것이 핵심 차이입니다.

---

## 슬라이드 21: 섹션 — 5. OOS 구조

> (넘기면서) 이제 OOS의 구체적인 구조를 보겠습니다.

---

## 슬라이드 22: OOS 구성 요소

> OOS 파일은 heap file과 1:1로 대응됩니다. 테이블 하나에 OOS 파일 하나입니다. FILE_OOS라는 새로운 파일 타입을 도입했고, heap header page에 OOS 파일의 VFID를 저장합니다. OOS가 필요 없는 테이블은 VFID가 NULL이니까 오버헤드가 없습니다.
>
> OOS 페이지는 기존 slotted page와 동일한 구조입니다. OOS 레코드가 실제 컬럼 데이터이고, OOS OID가 이 레코드를 가리키는 8바이트 포인터입니다.
>
> 빠른 판별을 위해 두 가지 플래그를 사용합니다. MVCC 헤더의 HAS_OOS 비트를 보면 이 레코드에 OOS가 있는지 즉시 알 수 있고, VOT의 IS_OOS 비트를 보면 각 컬럼이 실제 값인지 OOS OID인지 구분할 수 있습니다.

---

## 슬라이드 23: OOS 발동 조건

> OOS가 발동하려면 두 가지 조건을 모두 만족해야 합니다.
>
> 첫째, 전체 레코드 크기가 DB_PAGESIZE의 8분의 1을 넘어야 합니다. 16KB 기준으로 약 2KB입니다. 이 조건은 레코드 전체에 대해 한 번만 판단합니다.
>
> 둘째, 개별 컬럼이 가변 타입이면서 512바이트를 넘어야 합니다. 고정 길이 타입은 OOS 대상이 아닙니다.
>
> 예시를 보시면, 첫 번째는 레코드 자체가 2KB를 안 넘으니까 OOS가 안 됩니다. 두 번째는 레코드가 2KB를 넘으니까, 512B를 초과하는 vc1만 OOS로 가고 vc2는 heap에 남습니다. 세 번째는 vc1, vc2 모두 512B를 넘으니까 둘 다 OOS로 갑니다.

---

## 슬라이드 24: Record 바이너리 레이아웃 변화

> 실제 레코드의 바이너리 레이아웃이 어떻게 바뀌는지 보겠습니다.
>
> AS-IS에서는 Variable 영역에 vc1의 1700바이트가 통째로 들어가 있습니다. TO-BE에서는 이 자리에 8바이트 OOS OID만 들어가니까, 레코드가 확 줄어듭니다.
>
> VOT, Variable Offset Table의 각 entry 하위 1비트를 IS_OOS 플래그로 재활용합니다. 이 비트가 1이면 해당 오프셋 위치에 실제 값이 아니라 OOS OID가 있다는 뜻입니다.
>
> MVCC 헤더의 4번째 비트, bit 3이 HAS_OOS 플래그입니다. 주의할 점은, 기존 큐브리드는 MVCC 헤더 크기 lookup에 3비트만 사용해왔습니다. HAS_OOS는 4번째 비트이므로, lookup할 때 `idx & 0x07`로 마스킹해서 크기 계산에 영향을 주지 않도록 했습니다.

---

## 슬라이드 25: 섹션 — 6. CRUD 연산별 동작

> (넘기면서) 이제 각 연산별로 어떻게 동작하는지 보겠습니다.

---

## 슬라이드 26: INSERT 동작

> INSERT 경로를 따라가 보겠습니다.
>
> 먼저 `heap_attrinfo_determine_disk_layout`에서 레코드 크기를 계산합니다. 임계치를 넘으면 큰 가변 컬럼들을 OOS 후보로 마킹합니다.
>
> 그 다음 OOS 파일의 VFID를 확보합니다. 이 테이블에 OOS 파일이 아직 없으면 새로 생성합니다. OOS 파일 생성은 system operation으로 감싸서 crash-safe하게 처리합니다.
>
> 그리고 OOS 후보로 마킹된 각 컬럼에 대해 `oos_insert`를 호출해서 OOS 레코드를 삽입하고, OOS OID를 돌려받습니다. 이 OOS OID를 heap record의 variable 영역에 실제 값 대신 기록하고, IS_OOS 플래그를 켭니다.
>
> 마지막으로 WAL에 heap insert와 OOS insert를 모두 로깅합니다.

---

## 슬라이드 27: SELECT 동작

> SELECT 경로입니다. heap page에서 레코드를 읽은 후, MVCC 헤더의 HAS_OOS 비트를 확인합니다.
>
> 0이면 일반 레코드이니 그대로 반환합니다. 1이면 OOS resolve가 필요합니다. `heap_record_replace_oos_oids_with_values_if_exists`라는 함수에서 VOT를 순회하면서 IS_OOS가 1인 컬럼마다 `oos_read`로 실제 값을 가져와서 레코드를 재구성합니다.
>
> 재구성 후 레코드 크기가 크게 늘어날 수 있습니다. 원래 8바이트 OOS OID 자리에 수백에서 수천 바이트의 실제 값이 들어가니까요.

---

## 슬라이드 28: UPDATE 동작 (Milestone 1)

> UPDATE가 가장 복잡한 부분입니다. 4단계로 동작합니다.
>
> 첫 번째, 기존 레코드에 있는 모든 OOS OID를 실제 값으로 resolve합니다. 그리고 이 resolve된 전체 레코드를 undo log에 기록합니다. 이게 핵심입니다. undo log에는 OOS OID가 절대로 남아있으면 안 됩니다.
>
> 두 번째, 기존 OOS 레코드의 physical delete인데, 이 부분은 마일스톤 2에서 구현 예정입니다. 현재는 기존 OOS 레코드를 바로 삭제하지 않습니다.
>
> 세 번째, 새 레코드에 대해 다시 OOS 후보를 결정하고 `oos_insert`로 새 OOS OID를 받습니다.
>
> 네 번째, 새 heap record를 작성합니다.
>
> Milestone 1에서는 OOS 값이 바뀌지 않아도 항상 새 OOS OID를 발급합니다. 이건 의도적인 단순화입니다. 덕분에 하나의 OOS OID는 오직 하나의 record에서만 참조되는 것이 보장됩니다.

---

## 슬라이드 29: UPDATE — 그림으로 보기

> 그림으로 한번 보겠습니다. 원래 heap record에 OOS OID (1|1|33)이 있었습니다. vc2만 'hello'로 바꾸는 UPDATE입니다.
>
> Step 1에서 OOS OID를 따라가서 vc1의 실제 값을 읽어온 다음, 이 전체 레코드를 undo log에 기록합니다. undo log에는 OOS OID가 아니라 진짜 값이 들어갑니다.
>
> Step 2에서 기존 OOS slot 33을 삭제합니다. 다만 앞 슬라이드에서 말씀드린 대로 이 부분은 마일스톤 2 예정입니다.
>
> Step 3에서 새로 oos_insert를 해서 slot 44에 같은 값을 다시 넣습니다. 값은 같지만 OOS OID는 새로 발급된 (1|2|44)입니다.
>
> 최종 heap record에는 새 OOS OID와 변경된 vc2 'hello'가 들어갑니다.

---

## 슬라이드 30: DELETE 동작 (Milestone 1)

> DELETE는 비교적 단순합니다. 큐브리드의 삭제는 MVCC Delete ID를 레코드에 추가하는 방식이라서, 레코드 자체는 heap page에 남아있습니다.
>
> 여기서 핵심은, OOS OID를 건드리지 않는다는 겁니다. OOS 레코드를 삭제하지도 않고, OOS OID를 resolve하지도 않습니다.
>
> 왜냐하면, 삭제된 레코드가 여전히 heap page에 있는데, OOS OID를 실제 값으로 바꾸면 레코드 크기가 늘어나서 16KB 페이지에 안 들어가기 때문입니다. resolve가 불가능하니까 oos_delete도 할 수 없습니다.
>
> 결론적으로 OOS 정리는 vacuum에게 위임합니다. 추후 마일스톤에서 구현할 부분입니다.

---

## 슬라이드 31: 섹션 — 7. 로깅과 복구

> (넘기면서) 로깅과 복구 파트입니다.

---

## 슬라이드 32: 생략

> 로깅의 세부 구조는 이 자리에서는 생략하겠습니다. 핵심만 말씀드리면, 모든 OOS insert와 delete는 WAL에 기록되고, 가장 중요한 불변식은 "undo log에는 OOS OID가 절대 존재하지 않는다"는 것입니다. 이 원칙 덕분에 복구가 깔끔하게 동작합니다.
>
> 궁금하신 분은 별도로 질문 주시면 상세하게 설명드리겠습니다.

---

## 슬라이드 33: Delete + Vacuum + Crash 시나리오

> 자주 나오는 질문을 하나 다루겠습니다. "DELETE한 레코드의 OOS는 누가 지우는가?"
>
> DELETE 시점에는 건드리지 않습니다. 대신 vacuum이 heap record를 정리할 때 OOS도 함께 삭제합니다. `vacuum_heap`에서 `oos_delete`를 호출하는 방식인데, 이건 Milestone 1 이후에 구현됩니다.
>
> 구현 방식은 두 가지를 고려하고 있습니다. heap record 정리할 때 동기적으로 바로 지우는 방법과, OOS 전용 vacuum job을 만드는 방법입니다.
>
> "그러면 vacuum이 OOS 삭제하다가 crash하면?" — WAL에 oos_delete가 로깅되어 있다면 recovery REDO로 삭제가 완료됩니다. 아직 로깅 전이었다면 vacuum이 재실행되어 처음부터 다시 시도합니다. 기존 vacuum의 crash recovery 메커니즘과 동일합니다.

---

## 슬라이드 34: 섹션 — 8. Replication

> (넘기면서) 복제 파트입니다.

---

## 슬라이드 35: Replication — 생략 + 주의사항

> Replication의 세부 동작도 이 자리에서는 생략하겠습니다. 원칙은 단순합니다. replication log에 OOS 연산을 재현할 수 있는 충분한 정보가 들어가면 됩니다.
>
> 한 가지 알아두실 점은, Slave의 OOS OID가 Master와 반드시 같지는 않을 수 있다는 것입니다. 중요한 건 OOS OID의 동등성이 아니라 실제 값의 동등성입니다.

---

## 슬라이드 36: 섹션 — 9. OOS의 고민거리

> (넘기면서) 솔직하게 고민거리를 이야기하겠습니다.

---

## 슬라이드 37: Milestone 1 이후 해결 과제

> 현재 알고 있는 과제가 다섯 가지 있습니다.
>
> 첫째, OOS 파일이 계속 커집니다. `oos_file_destroy`가 미구현이고, DELETE해도 OOS가 안 지워지니까 파일이 줄어들 일이 없습니다.
>
> 둘째, UPDATE 시 값이 안 바뀌어도 OOS를 새로 만듭니다. 불필요한 I/O와 공간 낭비입니다.
>
> 셋째, `heap_get_record_data_when_all_ready` 같은 함수를 호출하는 모든 caller가 OOS resolve를 해야 하는지 직접 판단해야 합니다. 이걸 빠뜨리면 유저에게 OOS OID 값이 그대로 노출될 수 있습니다. 이건 꽤 위험합니다.
>
> 넷째, 한 레코드를 완전히 읽으려면 여러 OOS 페이지를 fix해야 할 수 있는데, page fix 순서에 따른 데드락 가능성을 고려해야 합니다.
>
> 다섯째, OOS 레코드가 여러 페이지에 산발적으로 저장되어 random I/O가 많이 발생합니다.

---

## 슬라이드 38: Multi-chunk OOS (대형 값)

> 컬럼 값이 OOS 페이지 하나에 다 안 들어갈 만큼 크면, 여러 chunk로 쪼개서 linked list로 연결합니다.
>
> 삽입은 역순으로 합니다. 맨 뒤 chunk부터 넣고, 각 chunk에 다음 chunk의 OOS OID를 기록합니다. 마지막에 삽입된 첫 번째 chunk의 OOS OID가 heap record에 저장됩니다.
>
> 읽기는 정순으로 합니다. 첫 chunk부터 next OID를 따라가면서 데이터를 모아 값을 재조립합니다.
>
> 역순 삽입을 하는 이유는, 삽입 시점에 다음 chunk의 OID를 이미 알고 있어야 기록할 수 있기 때문입니다. 뒤에서부터 넣으면 앞 chunk를 넣을 때 뒤 chunk의 OID가 이미 확정되어 있겠죠.

---

## 슬라이드 39: 섹션 — 10. Best Page 정책

> (넘기면서) Best Page 정책을 보겠습니다.

---

## 슬라이드 40: OOS 레코드를 어느 페이지에 넣을 것인가?

> OOS 레코드를 어느 페이지에 삽입할 것인가의 문제입니다.
>
> 현재 Milestone 1에서는 가장 단순한 방법을 사용합니다. OOS 파일 VFID별로 마지막에 삽입한 페이지 하나만 기억하고 있다가, 공간이 있으면 거기에 넣고, 없으면 새 페이지를 할당합니다.
>
> 장점은 단순하고 빠르다는 것이지만, 특정 페이지에만 삽입이 집중되는 hotspot 문제가 있고, 다른 페이지에 빈 공간이 있어도 활용하지 못합니다.
>
> 향후 개선 방향으로는 전역 bestspace 배열로 여러 후보 페이지를 관리하거나, OOS 크기별로 페이지를 분류하거나, 같은 record의 OOS를 인접 페이지에 배치해서 locality를 높이는 방법이 있습니다.

---

## 슬라이드 41: 섹션 — 11. Compaction 정책

> (넘기면서) 마지막으로 Compaction 정책입니다.

---

## 슬라이드 42: OOS 페이지의 빈 공간 정리

> 먼저 in-page compaction, 페이지 내부 정리부터 보겠습니다. OOS 페이지는 slotted page이므로 기존 `spage_compact` 함수를 그대로 사용할 수 있습니다. `oos_delete`로 레코드를 삭제하면 total_free가 올라가고, `spage_compact`으로 흩어진 빈 공간을 하나로 합칠 수 있습니다. 이 부분은 기존 인프라를 활용하는 것이고, 시점은 아직 미정입니다. UPDATE 시마다 할지, vacuum에서 할지는 추후 결정해야 합니다.
>
> 문제는 across-page compaction입니다. 반쯤 빈 페이지 두 개를 하나로 합치는 건데, 이건 기존 heap file에서도 안 됩니다. OOS에서도 마찬가지로 미지원입니다.
>
> 장기적으로 보면, 반복적인 UPDATE와 DELETE로 OOS 페이지마다 빈 공간이 쌓이면, in-page compact만으로는 전체 파일 크기를 줄일 수 없습니다. 결국 across-page compaction이나 파일 재구성 같은 기능이 필요하게 될 겁니다.

---

## 슬라이드 43: Milestone 1 요약

> Milestone 1의 범위를 정리한 겁니다. 전략은 "먼저 정확하게 동작하는 것을 만들고, 최적화는 이후에 진행한다"입니다. 목표는 OOS user scenario와 test_sql 통과 검증입니다.
>
> 왼쪽이 구현 완료, 오른쪽이 향후 마일스톤으로 미룬 것입니다. FILE_OOS/PAGE_OOS 타입 도입, OOS insert/read, heap과 OOS 연동, 플래그, WAL 로깅, Replication, Recovery까지 전체 파이프라인이 동작합니다. 성능 최적화와 공간 관리는 의도적으로 제외했습니다.

---

## 슬라이드 44: Milestone 2 이후 고려 사항

> Milestone 2의 목표는 Milestone 1에서 구현하지 못한 부분을 보완하고 develop branch에 머지하는 것입니다.
>
> Best Page 정책 개선으로 여러 페이지를 관리하고 크기별 분류나 locality 최적화를 도입할 수 있습니다. In-page compaction도 이 시점에 구현합니다. 그리고 현재 drop table 시 OOS page가 회수되지 않는 문제도 해결해야 합니다.

---

## 슬라이드 45: Milestone 3

> Milestone 3의 목표는 성능 개선입니다.
>
> Across-page compaction은 여러 페이지에 흩어진 OOS 레코드를 한 페이지로 모으는 기능이고, Update 시 OOS OID 재사용은 값이 안 바뀌었을 때 기존 OID를 그대로 쓰는 최적화입니다. 앞서 비교표에서 PG와 MySQL은 이걸 이미 하고 있다고 했는데, 우리도 이 시점에서 구현하게 됩니다.

---

## 슬라이드 46: Q&A

> 이상으로 OOS 설계 발표를 마치겠습니다. 질문 있으시면 편하게 해주세요.

---

## 슬라이드 47: 부록 — 예상 Q&A

> (질문이 나오면 활용)
>
> **Q. OOS OID가 8바이트인 이유는?**
> 기존 CUBRID OID 구조, volid 2바이트, pageid 4바이트, slotid 2바이트, 합해서 8바이트를 그대로 활용합니다. 별도 구조를 만들 필요 없이 기존 OID 유틸리티 함수들을 재사용할 수 있습니다.
>
> **Q. 고정 길이 컬럼은 왜 OOS 대상이 아닌가?**
> INT는 4바이트, BIGINT는 8바이트 등 크기가 작고 예측 가능합니다. 512B 임계치를 넘을 수 없습니다. 다만 CHAR는 예외적으로 큰 경우가 있을 수 있습니다.
>
> **Q. SELECT \* 할 때 성능이 오히려 나빠지지 않나?**
> 맞습니다. 여러 OOS resolve와 추가 page 접근이 필요합니다. OOS는 "필요한 컬럼만 읽는" 쿼리 패턴에서 이득을 보는 구조입니다.
>
> **Q. UPDATE 시 OOS 값이 안 바뀌었는데 왜 새로 만드는가?**
> Milestone 1의 의도적 단순화입니다. "하나의 OOS OID = 하나의 record"라는 불변식을 유지하기 위해서입니다. 이후 마일스톤에서 개선합니다.
