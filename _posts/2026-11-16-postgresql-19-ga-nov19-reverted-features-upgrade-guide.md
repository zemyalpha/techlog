---
title: "PostgreSQL 19가 3일 뒤 나온다 — 베타에서 되돌려진 기능과 살아남은 기능의 최종 목록"
date: "2026-11-16"
keywords: ["PostgreSQL 19", "PostgreSQL", "REPACK", "pg_plan_advice", "ON CONFLICT DO SELECT", "데이터베이스", "업그레이드 가이드"]
lang: "ko"
description: "PostgreSQL 19는 2026년 11월 19일 GA 예정이다. SQL/PGQ와 FOR PORTION OF가 베타 4에서 되돌려진 이유와, REPACK·pg_plan_advice·ON CONFLICT DO SELECT 등 살아남은 기능의 실전 사용법, 업그레이드 전 반드시 점검할 호환성 변경점을 정리한다."
---

# PostgreSQL 19가 3일 뒤 나온다 — 베타에서 되돌려진 기능과 살아남은 기능의 최종 목록

PostgreSQL 19의 정식 출시(GA)가 2026년 11월 19일로 예정되어 있다. RC 2가 이미 11월 12일에 나왔고, PostgreSQL 글로벌 개발 그룹은 RC 1 공지에서 "치명적인 문제가 발견되지 않는 한" 이 일정을 유지한다고 밝혔다. 이번 릴리스는 새 기능 목록만큼이나 **무엇이 빠졌는지**가 중요하다. 여름 내내 베타에 포함됐던 헤드라인 기능 여럿이 출시 직전에 되돌려졌기 때문이다. 이 글에서는 공식 릴리스 노트와 베타 발표를 기준으로 되돌려진 기능과 살아남은 기능을 구분하고, 업그레이드를 계획 중인 팀이 지금 확인해야 할 항목을 정리한다.

## 되돌려진 기능: 출시 일정이 우선이었다

PostgreSQL 19의 베타 주기는 순탄하지 않았다. Beta 1이 6월 4일에 나온 뒤 Beta 2(7월 16일), Beta 3(8월 13일), Beta 4(9월 24일)를 거쳤고, Beta 4 발표에서 커뮤니티는 여러 기능을 "이후 메이저 릴리스에 포함할 수 있도록" 공식적으로 철회했다. 신뢰성과 예측 가능한 출시 일정을 기능 확대보다 우선한 선택이었다.

Beta 4에서 제외된 항목은 다음과 같다.

- **SQL/PGQ 프로퍼티 그래프 쿼리** — `CREATE PROPERTY GRAPH`와 `GRAPH_TABLE()`로 기존 테이블 위에서 그래프 패턴 매칭을 하려던 SQL:2023 표준 기능. 베타 초반 가장 주목받은 기능이었으나 되돌려졌고, 재도입은 PostgreSQL 20(2027년 9월 예상)이 되어야 가능하다.
- **`UPDATE/DELETE ... FOR PORTION OF`** — 시간 범위를 지정해 수정하면 나머지 구간을 자동 분할 보존하던 SQL:2011 템포럴 연산. PostgreSQL 18의 `WITHOUT OVERLAPS`에 이은 완성작이었으나 빠졌다.
- **파티션 `MERGE PARTITIONS` / `SPLIT PARTITION`**
- **`pg_get_role_ddl()` 등 DDL 추출 함수들**
- **온라인 체크섬 활성화/비활성화**

또한 Beta 3에서는 `GROUP BY ALL`이 되돌려진 것으로 알려져 있다(출처: Jam SQL의 베타 발표 정리). 이 목록은 공식 릴리스 노트에서 해당 항목이 완전히 사라진 것으로 교차 확인할 수 있다. 릴리스 노트 전문에서 FOR PORTION OF, PGQ, GROUP BY ALL을 검색하면 더 이상 나오지 않는다.

**교훈은 명확하다.** 여름에 "PostgreSQL 19에 그래프 쿼리가 들어온다"는 글을 읽고 도입을 계획했다면, 그 계획은 지금 다시 짜야 한다. 베타 기능 목록을 근거로 한 1년 로드맵은 출시 3개월 전까지도 뒤집힐 수 있다.

## 살아남은 기능: 실전에서 바로 쓸 수 있는 것들

### REPACK — VACUUM FULL의 잠금 문제를 푸는 정식 명령

PG19의 대표 기능이다. 테이블을 재작성해 블로트(bloat)를 회수하고 인덱스 순서로 재배열까지 하는 명령으로, 기존 `VACUUM FULL`과 `CLUSTER`의 기능을 흡수한다. 핵심은 `CONCURRENTLY` 옵션이다.

```sql
-- ACCESS EXCLUSIVE 잠금 없이 온라인 재구성
REPACK (CONCURRENTLY) orders USING INDEX orders_created_at_idx;
```

일반 모드는 `VACUUM FULL`처럼 배타적 잠금을 잡지만, `CONCURRENTLY` 모드에서는 최종 파일 스왑 순간에만 잠금을 잡고 재구성 내내 읽기·쓰기가 가능하다. 지금까지 이 작업을 하려면 pg_repack 확장을 따로 깔아야 했는데, 이제 코어 명령으로 들어왔다.

### pg_plan_advice + pg_stash_advice — PostgreSQL식 플랜 힌트

PostgreSQL은 오랫동안 "힌트는 주지 않는다"는 철학을 유지했는데, PG19에서 contrib 확장 형태로 공식적으로 방향을 틀었다. `pg_plan_advice`는 알려진 좋은 실행 계획을 캡처해 플래너에 되먹이고, `pg_stash_advice`는 쿼리에 따라 플랜 어드바이스를 자동 적용한다.

```sql
-- 현재 플랜에서 어드바이스 생성
EXPLAIN (COSTS OFF, PLAN_ADVICE)
SELECT * FROM orders o JOIN customers c ON o.cust_id = c.id;
-- 출력: JOIN_ORDER(o c) HASH_JOIN(c) SEQ_SCAN(o c)

-- 해당 플랜으로 고정
SET pg_plan_advice.advice = 'JOIN_ORDER(o c) HASH_JOIN(c) SEQ_SCAN(o c)';
```

오라클/MySQL처럼 SQL 주석에 힌트를 박는 방식과 달리, 어드바이스는 GUC 설정으로 관리되고 각 힌트가 실제로 반영됐는지 피드백을 준다는 점이 다르다.

### ON CONFLICT DO SELECT — 아토믹 get-or-create

`INSERT ... ON CONFLICT DO SELECT ... RETURNING`가 공식 릴리스 노트에 명시됐다. "있으면 가져오고 없으면 만들기"를 한 문장으로 처리할 수 있다.

```sql
INSERT INTO users (email, name)
VALUES ('alice@example.com', 'Alice')
ON CONFLICT (email) DO SELECT
RETURNING *;
```

기존의 꼼수인 `DO UPDATE SET col = EXCLUDED.col`(no-op 업데이트)는 충돌할 때마다 죽은 튜플(dead tuple)을 양산했다. Neon이 베타 구현에서 측정한 바에 따르면 DO SELECT는 이 no-op 회피법 대비 약 4배 빠르다고 한다(베타 기준 수치이므로 GA 후 재측정이 필요하다).

### 병렬 autovacuum + 스코어링 — 이제 왜 그 테이블이 먼저인지 보인다

autovacuum이 인덱스별로 병렬 워커를 쓸 수 있게 됐다. 워커 수는 `autovacuum_max_parallel_workers`(전역)와 `autovacuum_parallel_workers`(테이블 단위 스토리지 파라미터)로 제어한다. Neon 문서에 따르면 전역 기본값은 0, 즉 비활성이므로 혜택을 보려면 명시적으로 켜야 한다.

```sql
ALTER SYSTEM SET autovacuum_max_parallel_workers = 4;
ALTER TABLE events SET (autovacuum_parallel_workers = 6);
```

더 흥미로운 건 `pg_stat_autovacuum_scores` 뷰다. autovacuum이 각 테이블을 언제, 왜 먼저 처리하는지 점수(xid_score, vacuum_score, analyze_score 등)로 공개한다. "왜 이 테이블만 자꾸 vacuum 되지?"라는 질문에 처음으로 데이터로 답할 수 있다.

### 그 밖의 개선

- **`WAIT` 명령** — 스탠바이에서 WAL이 특정 LSN에 도달할 때까지 기다려 read-your-writes 일관성을 제공한다.
- **논리 복제 시퀀스 동기화** — 페일오버 뒤 중복 키 에러의 흔한 원인이었던 시퀀스 값 미동기가 해결됐다. `CREATE PUBLICATION ... FOR ALL TABLES, ALL SEQUENCES`.
- **`wal_level` 재시작 없이 전환** — replica 레벨이면 서버 재시작 없이 논리 복제 활성화가 가능해졌다.
- **`COPY TO` JSON 출력** — `FORMAT JSON`으로 NDJSON이 기본이고 `FORCE_ARRAY` 옵션으로 배열 출력도 된다. `COPY FROM`도 text/CSV 입력에 SIMD 명령어를 써서 빨라졌다.
- **`EXPLAIN (ANALYZE, IO)`** — 실행 계획 노드별 비동기 I/O 통계를 보여준다.
- **`IGNORE NULLS`** — `lead()`, `lag()`, `first_value()` 등 5개 윈도우 함수에서 NULL을 건너뛸 수 있다. 시계열 결측 처리에 유용하다.
- **jsonpath 문자열 메서드** — 경로 안에서 `.lower()`, `.replace()` 등을 쓸 수 있어 JSON 값의 대소문자 무시 매칭이 가능해졌다.
- **`pg_stat_lock` 뷰** — 락 타입별 대기 통계가 누적된다.

## 업그레이드 전 체크리스트

릴리스 노트의 호환성 섹션에서 실무 영향이 큰 항목만 추리면:

1. **RADIUS 인증 제거** — PostgreSQL은 RADIUS를 UDP로만 지원했고, 이는 "고칠 수 없을 만큼 불안전"하다는 이유로 완전히 제거됐다. RADIUS를 쓰는 조직은 마이그레이션 계획이 필요하다.
2. **`standard_conforming_strings` 강제 on** — PG19 이전 버전의 pg_dump로 만든 덤프 중 이 설정이 off인 상태에서 만든 것은 PG19에 로드되지 않는다. 반드시 PG19 이상의 pg_dump로 덤프를 다시 만들 것.
3. **MD5 비밀번호 경고** — 성공적인 MD5 인증 뒤 경고를 출력한다(PG18에서 폐기 예고됨). `md5_password_warnings`로 끌 수 있다.
4. **inet/cidr 기본 opclass 변경** — btree_gist의 inet/cidr opclass가 결과에서 제외되는 버그가 있어 기본이 GiST로 바뀌었다. btree_gist inet/cidr 인덱스가 있으면 pg_upgrade가 차단한다.
5. **이름에 CR/LF 금지** — 데이터베이스·롤·테이블스페이스 이름에 캐리지 리턴/라인 피드가 있으면 업그레이드가 막힌다.
6. **PostgreSQL 14 지원 종료** — 11월 12일 업데이트(14.25)가 PG14의 마지막 릴리스다. 아직 14를 쓴다면 이번 주기를 EOL 대응 기한으로 잡아야 한다.

메이저 버전 업그레이드는 pg_upgrade, pg_dumpall, 논리 복제 세 가지 경로 모두 풀 덤프/리스토어 없이는 불가능하다는 기본 전제는 그대로다.

## 결론

- PostgreSQL 19는 **11월 19일 GA 예정**이며 RC 2까지 나온 상태다. 일정은 프로젝트 계획이지 확약은 아니므로 공식 발표를 확인하라.
- **SQL/PGQ, FOR PORTION OF, 파티션 MERGE/SPLIT은 Beta 4에서 공식 철회**됐다. 베타 리뷰 글을 근거로 한 도입 계획은 지금 다시 검증해야 한다.
- 살아남은 것 중 실전 가치가 큰 것은 **REPACK (CONCURRENTLY), pg_plan_advice, ON CONFLICT DO SELECT, 병렬 autovacuum + pg_stat_autovacuum_scores, 시퀀스 논리 복제**다.
- 업그레이드 전에는 **RADIUS 제거, standard_conforming_strings 강제, inet/cidr opclass 변경** 세 가지를 먼저 점검하라.
- 당장 할 일: 운영 중인 클러스터에서 `SELECT version();`으로 현재 버전을 확인하고, PG14 이하라면 EOL 대응을, 그 이상이라면 위 체크리스트로 호환성 점검을 시작하면 된다.

모든 기능 명세와 호환성 변경점은 공식 릴리스 노트(postgresql.org/docs/release/19.0/, 2026-10-09 기준 초안)를 출처로 하며, 출시 일정 세부 사항은 PostgreSQL 공식 뉴스와 Jam SQL의 베타 발표 정리를 교차 확인했다.
