CUBRID OOS (Out-of-row Overflow Storage) 프로젝트에 대해 다룬 문서이다.

- 목적
	- 현재 (AS-IS)
	- 목표 (TO-BE)

---

## 1. 목표

AS-IS
현재 큐브리드는 테이블의 row(하나의 heap 레코드)를 무조건 연속적으로 slotted page, 혹은 overflow page에 저장함.
이는 테이블의 일부 컬럼만 조회하더라도 전체 레코드를 다 읽어와야 해서 불필요한 disk IO가 발생함.

TO-BE
큰 가변 컬럼을 heap 레코드에서 분리 저장(OOS)하여 저장 효율과 읽기 성능 개선한다.


26년 2월 현황:
- Milestone 1
	- 가장 기초적인 POC 수준 구현
	- overflow file과 같이, heap page 수준에서 OOS 처리
		- overflow file을 oos file로 변경
		- overflow page를 oos page로 변경
	- 유저 레벨 test_sql 통과 목표
	- Replication 지원
	- Recovery 지원

---

## 용어/구성 요소

- OOS 레코드: heap 레코드에서 분리되어 OOS 파일에 저장되는 컬럼 단위 데이터
- OOS 파일: heap 파일과 1:1로 연결된 FILE_OOS 타입 파일. 테이블 1개 당 1개의 OOS 파일 보유
- OOS 페이지: 1개의 OOS파일은 여러 개의 OOS 페이지로 구성되어 있으며 제한은 없음. 크기는 DB_PAGESIZE 이며, 내부 구현은 slotted page로 되어 있음. 
- OOS OID: heap 레코드의 variable 영역에 저장되는 OID (OOS 레코드 위치). 기존 OID와 동일하게 volid, pageid, slotid로 구성되어 있으며 8바이트.
- HAS_OOS 플래그:  레코드 내 OOS OID가 있는지 여부 표시. 레코드 포인터만 있어도 빠르게 OOS OID 여부를 판단하기 위해 존재한다.
- IS_OOS 플래그: variable offset table의 entry에 기록되어 있으며, 해당 컬럼 값이 OOS OID인지, 실제 값인지 여부 표시
	- IS_OOS 플래그가 적어도 한개가 켜져있다면, HAS_OOS flag가 1이다.
	- HAS_OOS 플래그가 1이라면 적어도 한개의 IS_OOS 플래그가 1이다.
- OOS Resolve: 레코드 내 OOS OID들을 실제 OOS value로 바꿔서 채워넣는 작업. 기존 8바이트만 차지하던 OID를 빼내고, 적어도 512 byte 이상인 OOS value들로 바꾸기 때문에 레코드의 크기가 무조건 늘어난다.

---

## Milestone 1

- POC
- heap record 레벨 OOS 처리
- slotted page 형식의 OOS 저장소
- OOS OID (8 byte)를 통한 OOS 값 직접 접근

다루지 않는 내용
- compaction
- best space 최적화
- update 시 OOS value deduplication (재사용)
- vacuum 성능 최적화
- 기타 TODO http://jira.cubrid.org/browse/CBRD-26517

---

## Milestone 1 상세 스펙

### OOS 대상 컬럼 선택 정책

원칙
- 대상은 가변 타입 (Variable-length type) 만 가능
- 우선 전체 레코드 하나의 크기를 기준으로 결정
- 그 이후 각 컬럼 별 크기를 기준으로 최종 OOS 결정

- 임계치: `header + payload + mvcc_extra > DB_PAGESIZE / 8`
- OOS 조건: `is_variable && column_size > 512B`

#### 예시

DB_PAGESIZE가 16K일 경우,
- record 크기가 2K를 넘고
- column 크기가 512 Byte를 넘으면
OOS로 보내진다.

참고:

```sql
create table tbl (id int, vc1 varchar, vc2 varchar);

-- 둘다 heap page에 저장 (oos로 가지 않음)
insert into tbl (1, 'hello', 'world');

-- record 크기가 2K를 넘지 않으므로 둘 다 oos로 가지 않음
insert into tbl (1, REPEAT('a', 900), REPEAT('b', 600));

-- record 크기가 2K를 넘으므로 vc1은 oos로 간다. vc2는 512B를 넘지 않으므로 heap에 저장.
insert into tbl (1, REPEAT('a', 1700), REPEAT('b', 400));

-- vc1, vc2 둘 다 oos
insert into tbl (1, REPEAT('a', 1700), REPEAT('b', 600));
```


코드 레퍼런스
- `src/storage/heap_file.c`

---

## User-level Operations: SQL Insert & Select


```sql
create table tbl (id int, vc1 varchar, vc2 varchar);
insert into tbl (1, REPEAT('a', 1700), REPEAT('b', 400));
```

위와 같은 sql이 수행되면 

```
[ MVCC header | VOT | Fixed | Variable ]
```

위와 같이 Record가 디스크 형태로 변환되어 heap file에 저장된다. 즉, 아래와 같은 형태이다.

AS-IS

```
# inside heap page
[ MVCC header | VOT | 1 | 'aaaaa ... (length: 1700)', 'bbbb ... (length: 400)' ]
```

TO-BE

```
# inside heap page
[ MVCC header | VOT | 1 | OOS OID to 'aaa...' (8 byte), 'bbbb ... (length: 400)' ]

...

# inside OOS page
[ OOS Header | 'aaaa ... (length: 1700) ]
```

vc1은 heap page에서 벗어나 따로 저장되고, heap에는 vc1으로 접근할 수 있는 8 바이트 OID만 남는다.

이로써 만약 유저가 vc1이 필요 없고 id와 vc2만 원할 경우, 즉,

```sql
select id, vc2 from tbl; 
```

위와 같은 sql 쿼리를 수행할 경우,
디스크로부터 'aaaa...'를 꺼내오는 불필요한 비용을 방지할 수 있다.

---

## User-level Operations: SQL Update

업데이트는 크게 두 분류로 나눌 수 있다.
1. OOS 컬럼을 포함해서 update하는 경우
2. OOS 컬럼을 포함하지 않고 update하는 경우

이론 상 OOS 컬럼을 포함하지 않고 update 하는 경우, OOS 값에는 전혀 변화가 없으므로, 기존 OOS OID를 재사용하여 수정된 Record 에 기입하는 방식으로 불필요한 중복 연산을 제거할 수 있다.

그러나 이 경우 orphan 된 OOS 값들을 vacuum 에서 지워줘야 하므로, Milestone 1에서는 제외한다.

따라서, Milestone 1에서는 OOS 값의 변화에 상관 없이, OOS에 삽입하여 항상 새로운 OID를 발급받는 형태로 동작한다.

따라서, OOS OID는 오직 하나의 record에 의해 참조되는 것이 보장되는 단순한 구현이다. 하나의 OOS OID는 (record가 heap page에 존재하는지, LSA에 로그 형태로 존재하는지의 여부와 관계 없이) 오로지 단 하나의 record에 의해 참조된다.

```sql
create table tbl (id int, vc1 varchar, vc2 varchar);
insert into tbl (1, REPEAT('a', 1700), REPEAT('b', 400));
```

위와 같이 vc1을 OOS로 보낸 단순한 예를 보자.

대략 아래와 같은 형태로 record는 디스크에 저장될 것이다.

```
[ mvcc header | VOT | 1 | (OOS OID to vc1 (1|1|33)) | 'bbbbb.....' ]
```

 > 예시에서 OOS OID는 volid = 1, fileid = 1, slotid = 33 를 가짐

위 recdes를 아래와 같이 update 하면

```sql
update tbl set vc2 = 'hello' where id = 1;
```

아래와 같은 형태로 디스크에 저장된다.

```
[ mvcc header | VOT | 1 | (OOS OID to vc1 (1|2|44)) | 'hello' ]
```

vc1이 변경되지 않았음에도 신규 OOS 값이 삽입되고 새로운 OOS OID (1|2|44) 가 vc1의 자리에 저장된 것을 볼 수 있다.

---

#### Q. OOS OID (1|1|33) 은 어떻게 physical delete 되는가?

oos_delete라는 physical delete API가 있다. oos_delete는 내부적으로 spage_delete로 구현되어 있으므로, total_space 가 free된 양 만큼 올라가고, 추후 spage_compact를 통해서 공간을 확보할 수 있다.

#### Q. OOS OID (1|1|33) 은 언제 physical delete 되는가?

1. update 이전 버전 heap record가 undo log에 남아서, MVCC를 통해 다시 접근될 수 있으니 남겨놓는다. 
	추후 heap record의 MVCCID가 min active snapshot 보다 작아서 더 이상 접근 불가능해질 경우, 언젠가 vacuum이 해당 record를 vacuum_heap 함수에서 physical spage_delete, spage_vacuum을 통해 정리할 것이다. 이때 oos_delete 를 같이 수행한다.

---

## User-level Operations: SQL Delete

큐브리드의 Delete는 기존 Record에 MVCC Delete ID를 추가하고, 해당 Record를 다시 spage에 삽입하는 방식으로 이루어진다.

Undo log를 생성하는 대신, 단순히 Record를 Heap에 남겨두고 Del id만을 남겨서, 볼 수 있는 트랜잭션들은 Heap에서 즉시 보게 하고, 신규 트랜잭션들은 Del id에 막혀 레코드를 볼 수 없게 하는 기본적인 MVCC 구현이다. 

update 연산에서는 이전 OOS 레코드를 즉시 oos_delete하고 undo log에는 이전 heap record(OOS OID 포함)를 그대로 기록하는 반면, 삭제된 레코드들은 여전히 heap file에 남아있으므로 OOS OID도 그대로 남아있다.

따라서 OOS OID들을 oos_delete 하지 못하고 남겨둬야 한다. 이 때문에 결국 vacuum에게 oos_delete를 언젠가 지워야 한다는 사실을 알려 주어야 한다. 이는 추후 Milestone에서 다룬다.

---

## 상세 구현 (Implementation Details)

---

## Record Format 변화

### Variable offset table 플래그

- offset table 하위 2비트를 플래그로 재활용
- 실제 길이는 `OR_GET_VAR_LENGTH`로 복원

```cpp
// object_representation.h
#define OR_VAR_BIT_OOS 0x1
#define OR_VAR_BIT_RESERVED 0x2
#define OR_VAR_FALG_MASK 0x3

#define OR_SET_VAR_OOS(length) ((int) (length) | OR_VAR_BIT_OOS)
#define OR_GET_VAR_FLAG(length) ((int) (length) & OR_VAR_FALG_MASK)
#define OR_GET_VAR_LENGTH(length) ((int) (length) & (~OR_VAR_FALG_MASK))
#define OR_IS_OOS(length) (OR_GET_VAR_FLAG (length) & OR_VAR_BIT_OOS)
```

주의
- `OR_VAR_TABLE_ELEMENT_OFFSET_INTERNAL`는 반드시 `OR_GET_VAR_LENGTH`를 통해 길이를 복원해야 한다.

### MVCC 헤더 플래그 확장

mvcc_flag는 총 5비트이기 때문에, mvcc header size를 lookup할 때, 이론 상 2^5 = 32번째 인덱스까지 지원한다.
그러나 관행 상 큐브리드는 최근 몇년간 3비트만을 사용해왔으므로, lookup table이 8번째 인덱스까지만 구현되어 있다.

즉, mvcc_header_size_lookup 배열은 8번째 원소만을 가지고, 그 이상을 접근할 경우 오버플로다.

OOS를 지원함에 따라 4번째 (LSB+3) 비트 또한 사용하게 되므로, idx 를 최대 3비트로 제한해야 한다. 기존 mvcc header size lookup 코드에 idx를 0x07와 & 연산시켜주어야 한다.

```c
// object_representation_constants.h
#define OR_MVCC_FLAG_HAS_OOS 0x08 // LSB + 3
```

- MVCC 헤더 크기 lookup은 하위 3비트만 사용

```c
int mvcc_flag = OR_GET_MVCC_FLAG (ptr);
int idx = mvcc_flag & 0x07; // last 3 bits
return mvcc_header_size_lookup[idx];
```


---

## Heap Insert → OOS Insert 경로

### OOS 대상 컬럼 결정

- `heap_attrinfo_determine_disk_layout()`에서 payload 크기 계산
- 기준 초과 시 큰 가변 컬럼을 OOS로 분리

핵심 흐름

```c
payload_size = heap_attrinfo_get_record_payload_size(attr_info, &column_size);
header_size = heap_attrinfo_get_record_header_size(...);

if (header_size + payload_size + mvcc_extra > DB_PAGESIZE / 8)
  {
    for each column:
      if variable && column_size[i] > 512
        mark oos_columns[i] = true
        payload_size -= column_size[i]
        payload_size += OR_OID_SIZE
        has_oos = true
    header_size = heap_attrinfo_get_record_header_size(...)
  }
```

### OOS 파일 VFID 획득

- Heap header에 OOS VFID 저장
- 없으면 생성 (`oos_file_create` + WAL undo/redo)

```c
bool heap_oos_find_vfid(..., bool docreate)
{
  if (VFID_ISNULL(&heap_hdr->oos_vfid))
    {
      if (docreate)
        {
          log_sysop_start(thread_p);
          if (oos_file_create(thread_p, *oos_vfid) != NO_ERROR)
            log_sysop_abort(thread_p);
          log_append_undo_data(...);
          VFID_COPY(&heap_hdr->oos_vfid, oos_vfid);
          log_append_redo_data(...);
          log_sysop_commit(thread_p);
        }
      else return false;
    }
}
```

### OOS 레코드 삽입

- 각 OOS 대상 컬럼을 RECDES로 변환
- `oos_insert()`로 삽입 후 OID 기록

```c
if (heap_attrinfo_dbvalue_to_recdes(...) != S_SUCCESS) return S_ERROR;
if (oos_insert(thread_p, oos_vfid, recdes, oos_oid) != NO_ERROR) return S_ERROR;
oos_oids[i] = oos_oid;
```

### Heap 레코드 쓰기

- offset 테이블에 OOS 플래그 설정
- payload에는 OID 기록

```c
if (is_oos)
  {
    length = OR_SET_VAR_OOS(length);
    or_put_offset_internal(buf, length, offset_size);
    or_put_oid(buf, oos_oid);
  }
```

코드 레퍼런스
- Heap insert 경로: `src/storage/heap_file.c`
- OOS insert: `src/storage/oos_file.cpp`

---

## Heap Read → OOS Read 경로

### OOS 여부 판단

```c
if (OR_IS_OOS(offset))
  {
    or_get_oid(&buf, &oos_oid);
    oos_read(thread_p, oos_oid, *raw);
  }
```

### Heap record 복원 루틴

- `heap_record_replace_oos_oids_with_values_if_exists()`에서 OOS 값을 실제 값으로 확장

핵심 단계
1) `heap_attrinfo_start()`로 attr_info 준비
2) `heap_attrinfo_read_dbvalues()`로 DBVALUE 로딩
3) `heap_attrinfo_transform_to_disk_develop_ver()`로 레코드 재구성
4) 재구성된 레코드를 caller에 반환

주의
- `PEEK` 모드 미지원
- `S_DOESNT_FIT` 반환 시 상위 호출자 처리 필요

코드 레퍼런스
- Heap read 확장: `src/storage/heap_file.c`
- OOS read: `src/storage/oos_file.cpp`

---

## 대형 OOS 값 삽입 방식

### Chunk 분할

- `oos_get_max_chunk_size_within_page()`로 최대 chunk 크기 산출
	- 대략적으로 DB_PAGESIZE
- 큰 값은 여러 chunk로 분할

### 다중 청크 삽입

- 역순 삽입
- 각 chunk 헤더에 다음 chunk OID 기록

### 읽기

- 첫 chunk 읽기 → next OID 확인 → 체인 따라 재구성

코드 레퍼런스
- OOS 체인/분할: `src/storage/oos_file.cpp`

---


## Bestspace 정책 (현행)

- 전역 탐색 없음
- VFID별 “최근 삽입 페이지”만 후보
- 충분한 공간이면 재사용, 아니면 새 페이지 할당

장점
- 단순, 빠름

리스크
- 핫스팟
- 파편화

코드 레퍼런스
- bestspace 로직: `src/storage/oos_file.cpp`

---

## 마일스톤 1 구현 상태 정리

구현 완료
- FILE_OOS/PAGE_OOS 타입 도입 및 OOS 파일 생성
- OOS insert/read (single/multi chunk)
- heap ↔ OOS 연동 (OID 치환, read 시 OOS 확장)

제한 사항
- `oos_file_destroy` 미구현
- OOS 확장 후 `recdes.area_size` 초과 처리 미완
- bestspace가 “최근 삽입 페이지”만 사용
- across-page GC/compaction 미구현

---


## Develop branch Merge 도전 과제

1. OOS 페이지가 계속 증가함
2. heap_get_record_data_when_all_ready () 함수에서 OOS resolve 여부
	1. 해당 함수를 쓰는 caller 들이 상황에 따라 OOS resolve 결정해야 함

---

## Release 도전 과제

1. 한 레코드 전체를 읽기 위해 여러 oos 페이지를 fix / unfix 해서 읽어야하는 문제
	1. OOS ordered fix 구현?
2. OOS 레코드가 산발적으로 저장됨 
	1. Bestspace 최적화?

---

## 개선 과제

1. across-page compaction (페이지 간 컴팩션)
2. best space 개선 (전역 구조체 제거)

---

