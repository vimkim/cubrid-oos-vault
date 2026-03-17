OOS Milestone 2 이슈들을 정리한 상위 에픽 이슈입니다.
h3. Description
h4. 배경

OOS Milestone 1에서 기본 CRUD 동작(insert/read/update/delete), WAL 로깅, recovery, replication을 구현하여 OOS의 핵심 기능이 동작하는 상태이다. 그러나 M1은 "먼저 정확하게 동작하는 것"에 집중했기 때문에, 운영 환경에서 필수적인 여러 기능이 빠져 있다.
 Milestone 2의 목표는 *M1에서 구현하지 못한 필수 보완 사항을 해결* 하고, *develop 브랜치에 머지 가능한 품질* 로 만드는 것이다.
h4. 목표
 # OOS 파일/페이지 공간 회수 경로 확보 (drop table, vacuum 연동)
 # Best Page 정책 개선으로 hotspot/fragmentation 완화
 # In-page compaction 적용으로 페이지 내 공간 재활용
 # develop 브랜치 머지 및 CI 통과

h4. M1 현황 (AS-IS)
|영역|현재 상태|문제점|
|—|—|—|
|Best Page|마지막 삽입 페이지 1개만 기억|hotspot 집중, 다른 페이지 빈 공간 낭비|
|drop table|OOS 페이지 미회수|DROP TABLE 해도 OOS 파일 공간 반환 안 됨|
|Vacuum 연동|미구현|DELETE된 record의 OOS 레코드가 영구히 잔존|
|Compaction|미적용|Vacuum 연동하여 회수하더라도 재사용 안함 , continuous free < total free|
----
h3. 하위 Story/Task 구성
h4. Story 1: Best Page 정책 개선

*설명:* 현재 VFID별 "마지막 삽입 페이지" 하나만 기억하는 방식에서, 여러 후보 페이지를 관리하는 방식으로 개선한다.
 *상세 요구사항:*
 * bestspace 배열 도입 (heap file의 기존 bestspace 구조 참고)
 * 여러 후보 페이지 중 OOS 레코드 크기에 맞는 페이지 선택
 * 옵션: locality 최적화 검토: 같은 record의 OOS를 인접 페이지에 배치

*인수 조건:*
 * [ ] 동일 워크로드에서 OOS 페이지 할당 수가 M1 대비 감소 확인
 * [ ] hotspot 발생 없이 여러 페이지에 분산 삽입 확인
 * [ ] 기존 OOS insert/read/delete 기능 정상 동작 (regression 없음)
 *참고 코드:* ' {{src/storage/oos_file.cpp}} ' (oos_insert 내 페이지 선택 로직)

----
h4. Story 2: In-page Compaction 적용

*설명:* OOS 페이지에서 ' {{spage_delete}} ' 후 흩어진 free space를 ' {{spage_compact}} '로 합치는 정책을 수립하고 적용한다.
 *상세 요구사항:*
 * compaction 수행 시점 결정 (UPDATE 시마다 / 페이지 여유 공간 부족 시 / vacuum 시)
 * 기존 슬롯 페이지 인프라(' {{spage_compact}} ') 활용
 * compaction 수행 시 WAL 로깅 정합성 확인

*인수 조건:*
 * [ ] 반복 UPDATE 후 OOS 페이지 내 free space가 재활용되는 것 확인
 * [ ] compaction 중 crash → recovery 후 OOS 페이지 정합성 유지
 * [ ] compaction 전후 OOS read 결과 동일
 *참고 코드:* ' {{spage_compact}} ', ' {{spage_delete}} ' (기존 슬롯 페이지 API)

----
h4. Story 3: DROP TABLE 시 OOS 파일 회수

*설명:* ' {{oos_file_destroy}} ' 구현. DROP TABLE 시 해당 테이블의 OOS 파일을 함께 삭제하여 디스크 공간을 회수한다.
 *상세 요구사항:*
 * ' {{heap_file_destroy}} ' 경로에서 OOS VFID 확인 후 ' {{oos_file_destroy}} ' 호출
 * OOS 파일 삭제 시 WAL 로깅
 * heap header의 OOS VFID를 NULL로 초기화

*인수 조건:*
 * [ ] DROP TABLE 후 OOS 파일이 삭제되고 디스크 공간 회수 확인
 * [ ] DROP TABLE 중 crash → recovery 후 정합성 유지 (orphan 파일 없음)
 * [ ] OOS가 없는 테이블 DROP 시 영향 없음
 *참고 코드:* ' {{src/storage/heap_file.c}} ' (heap_file_destroy), ' {{src/storage/oos_file.cpp}} '

----
h4. Story 4: Vacuum ↔ OOS 연동

*설명:* DELETE된 record의 OOS 레코드를 vacuum이 정리하도록 구현한다. M1에서는 DELETE 시 OOS를 건드리지 않기 때문에, vacuum이 heap record 정리 시 OOS도 함께 삭제해야 한다.
 *상세 요구사항:*
 * vacuum_heap에서 heap record 정리 시 ' {{oos_delete}} ' 호출 경로 추가
 * 방식 결정 필요:
 ** 방식 A: heap record 정리 시 동기적으로 ' {{oos_delete}} ' (단순, 구현 비용 낮음)
 ** 방식 B: OOS 전용 vacuum job 생성 (비동기, 확장성 높음)
 * vacuum 중 crash 시 recovery 정합성 보장
 ** WAL에 ' {{oos_delete}} '가 로깅된 경우 → REDO로 삭제 완료
 ** 로깅 전 crash → vacuum 재실행 시 재시도

*인수 조건:*
 * [ ] DELETE → vacuum 후 orphan OOS 레코드가 남지 않음 확인
 * [ ] vacuum 중 crash → recovery 후 OOS 정합성 유지
 * [ ] vacuum 성능 regression 측정 (OOS delete 추가 비용 확인)
 *참고 코드:* ' {{src/storage/vacuum.c}} ' (vacuum_heap), ' {{src/storage/oos_file.cpp}} ' (oos_delete)

----
h4. Story 5: develop 브랜치 머지 및 통합 검증

*설명:* M2 구현 완료 후 develop 브랜치에 머지한다. CI 파이프라인 통과 및 기존 테스트 regression 검증을 수행한다.
 *상세 요구사항:*
 * OOS user scenario 테스트 통과
 * test_sql, test_medium, HA, shell 등 전수 통과
 * 기존 heap/overflow 관련 테스트 regression 없음
 * replication 시나리오 재검증 (vacuum 연동 후 변경사항 반영)

*인수 조건:*
 * [ ] CI 파이프라인 (check, test, build) 전체 통과
 * [ ] OOS 전용 테스트 시나리오 통과
 * [ ] 기존 test_sql regression 없음
 * [ ] develop 브랜치에 머지 완료

----
h3. Scope 외 (Milestone 3에서 진행)

아래 항목은 M2 범위에 포함하지 않는다. 혼동 방지를 위해 명시한다.
|항목|사유|
|—|—|
|Across-page compaction|성능 최적화 영역, M3에서 진행|
|UPDATE 시 OOS OID 재사용|값 비교 로직 + 불변식 변경 필요, M3에서 진행|
|PEEK 모드 지원|성능 최적화, 우선순위 낮음|
|OOS 값 압축 (compression)|M1/M2 범위 외|
----
h3. 리스크 및 고려사항
|리스크|대응|
|—|—|
|Vacuum 연동 방식 검증|
|Best Page 정책 변경 시 기존 OOS 파일 호환성|기존 OOS 파일에 대해 하위 호환 유지 (bestspace 정보는 런타임 메모리에서 관리)|
|In-page compaction 시점에 따른 lock 경합|slotted page 특성상 같은 페이지 내 다른 record의 OOS에 영향 — 벤치마크로 시점 결정|
|한 레코드 읽기에 여러 OOS page fix/unfix 시 데드락|ordered fix 정책 검토 필요 (pageid 순서로 fix)|
----
h3. 선행 조건
 * OOS Milestone 1 구현 완료 및 기능 검증 통과
