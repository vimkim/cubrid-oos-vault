큐브리드에 OOS를 도입하는 것에 대한 설계 발표를 진행하려 한다.

대상은 개발 2팀, 같은 개발자들이지만 OOS 에 대해서는 잘 모른다.

목차는 대략 다음과 같다.

1. OOS란 무엇인가?
2. 왜 필요한가?
3. 다른 DB들은 어떻게 이 문제를 해결하는가? 즉, select * 대신 select id 만 할 경우 어떻게 최적화되는가?
- pg의 toast, mysql 의 off page column storage...

4. 큐브리드는 어떻게 해결하려고 하는가? OOS 도입
5. OOS는 어떤 구조인가?
6. select, insert, update, delete 연산 시 어떻게 동작하는가?
7. logging 은 어떻게 되는가? 복구는 어떻게 되는가?
만약 OOS 값이 delete marked 되었다가 unreachable 되면 누가 어떻게 이걸 지워주는가? 배큠이 어떻게?
만약 vacuum이 지우다가 서버가 crash 되면 리커버리에서 이걸 어떻게 지워주는가?
8. replication 은 어떻게 되는가?
9. OOS 의 고민거리는 무엇인가?
10. best page를 찾는 정책은 무엇인가? 현재 heap 정책을 그냥 따bestpage를 찾는 정책은 무엇이 있는가?
11. compaction 하는 정책은 무엇인가? 우선 across page compaction은 AS-IS도 안되기 때문에, in-page compaction 부터 따져보자. 무엇이 있는가?
