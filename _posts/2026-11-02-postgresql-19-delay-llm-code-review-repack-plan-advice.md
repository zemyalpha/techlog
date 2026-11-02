---
title: "PostgreSQL 19은 왜 11월로 미뤄졌나 — LLM 코드 리뷰가 찾아낸 버그와 살아남은 기능(REPACK, pg_plan_advice) 실전 가이드"
date: "2026-11-02"
keywords: ["PostgreSQL 19", "LLM 코드 리뷰", "REPACK", "pg_plan_advice", "WAIT 명령", "업그레이드", "데이터베이스"]
lang: "ko"
description: "LLM 기반 코드 리뷰가 PostgreSQL 19의 출시를 11월로 밀어냈다. 리버트된 기능과 살아남은 REPACK·pg_plan_advice·WAIT 명령 실전 사용법, 그리고 업그레이드 전 반드시 확인해야 할 비호환 변경점을 정리한다."
---

# PostgreSQL 19은 왜 11월로 미뤄졌나 — LLM 코드 리뷰가 찾아낸 버그와 살아남은 기능 실전 가이드

매년 가을 칼 같이 메이저 버전을 내놓던 PostgreSQL이 올해 처음으로 일정을 놓쳤다. PostgreSQL 19는 원래 9월 말~10월 초 출시 예정이었으나 10월 15일 RC(Release Candidate) 1 단계에 머물러 있고, 정식 출시(GA)는 11월 19일 전후로 미뤄진 상태다. 이유는 재미있게도 "AI 때문"이다. 정확히는, 개발자들이 도입한 LLM 기반 코드 리뷰가 인간 리뷰어가 놓쳤을 버그들을 대량으로 찾아냈고, 커뮤니티는 그 버그들을 다 고치고 출시하기보다는 준비 안 된 기능을 과감히 되돌리는(revert) 쪽을 택했다.

이 글에서는 (1) 지연이 일어난 경위와 숫자, (2) 살아남아 11월에 만나게 될 핵심 기능의 실전 사용법, (3) 업그레이드 전 반드시 점검해야 할 비호환 변경점을 다룬다.

## 무슨 일이 있었나: 53번의 리버트와 "40개 결함 리포트"

PostgreSQL의 개발 프로세스는 투명하기로 유명하다. 커밋페스트라는 공개 리뷰 절차를 거쳐 약 30명의 커미터가 패치를 반영하고, 베타가 나오면 수개월간 커뮤니티가 테스트하며 문제가 있는 기능은 출시 전에 되돌린다. 리버트 자체는 매 버전마다 있었던 일이다. 다만 규모가 달랐다. Snowflake의 개발자 어드보케이트 Elizabeth Garrett Christensen이 정리했을 때 PostgreSQL 18은 약 44건을 되돌렸고, PostgreSQL 19는 베타 시작 이후 누적 53건에 달했다. 이번엔 그중 SQL/PGQ, `GROUP BY ALL`처럼 헤드라인급 대형 기능이 다수 포함돼 있었다는 점이 달랐다.

The Register 보도(2026년 10월 16일)에 따르면 PostgreSQL 19에서는 8~10개의 유의미한 기능이 빠졌고, PostgreSQL 코어 커미터이자 EDB 부사장인 Peter Eisentraut는 그 원인의 상당 부분을 LLM 보조 코드 리뷰에서 찾았다. 그의 설명을 요약하면 이렇다.

> 과거에는 출시 후 5년에 걸쳐 월별 패치로 잡혔을 "비교적 무해한 엣지 케이스"들이 LLM 리뷰로 훨씬 빨리 발견된다. 그러면 어느 순간 결함 리포트 40개짜리 목록과 마주하게 되고, 하나당 평균 3줄 수정이라도 건당 처리 비용이 있으니 시간이 부족해진다.

즉 "AI가 망쳐서"가 아니라 "AI가 퀄리티 바를 올려서" 출시가 늦어진 것이다. Eisentraut는 리버트가 '바이브 코딩' 탓이 아니라고도 못 박았다 — 문제가 된 기능들이 작성될 당시에는 지금 수준의 LLM 리뷰 도구가 없었다는 얘기다. 실제로 기능 구현 자체에 결함이 있었던 사례도 있다. SQL:2023 표준에 들어온 그래프 쿼리(SQL/PGQ)가 대표적이다. 커미터 Tom Lane은 "v19에 싣고 가면 출시 후 v20 전까지 못 고치는 버그가 발견될 것에 저녁 한 끼를 걸겠다"고 경고했고, SQL/PGQ는 결국 v20으로 이월됐다. 코어 기여자 Robert Haas도 10월 말 "Postgres 19 is late, and it's AI's fault"라는 제목의 포스팅으로 같은 맥락을 설명했다. 되돌려진 기능 대부분은 다음 버전(v20)에서 다시 시도될 예정이다.

참고로 이런 흐름에는 전조가 있었다. Snowflake 블로그에 따르면 2026년 8월 PostgreSQL 18 패치에는 28개의 CVE가 포함됐는데, 과거 한 릴리스 평균 1~2개였던 것과 비교하면 AI 보조 리뷰·테스트가 잠재 결함의 가시성을 크게 높인 셈이다.

## 살아남은 기능 1: REPACK — VACUUM FULL의 잠금 지옥을 대체한다

지연과 리버트 소식에 가려서 그렇지, 살아남은 PostgreSQL 19는 알차다. 첫 번째는 테이블 재작성 명령 `REPACK`이다. 기존에 디스크 공간을 OS로 돌려주려면 `VACUUM FULL`을 써야 했는데, 이건 실행 내내 `ACCESS EXCLUSIVE` 잠금을 잡아 서비스 중인 테이블에서는 사실상 쓸 수 없었다. pg_repack 확장이 그 갈증을 채워왔는데, 이제 코어에 동일 기능이 들어온 것이다.

```sql
-- 특정 테이블을 읽기/쓰기 허용하면서 재패킹
REPACK (VERBOSE, ANALYZE) orders;

-- 인덱스 순서로 물리 재배치(클러스터링)까지 수행
REPACK orders USING INDEX idx_orders_customer_id;

-- DB 전체(잠금 있음, 트랜잭션 블록 안에서 실행 불가)
REPACK;
```

공식 문서 기준 주의점은 다음과 같다.

- `CONCURRENTLY`는 잠금 없이 재패킹하지만 MVCC-safe하지 않다. 문서에 명시된 경고이므로 적용 전 [공식 문서](https://www.postgresql.org/docs/19/sql-repack.html)의 동시 실행 절을 반드시 읽을 것.
- 테이블+인덱스 크기만큼의 여유 디스크가 필요하다(임시 복사본 생성).
- `REPACK`은 긴 트랜잭션 하나로 취급되어 다른 테이블의 VACUUM을 막을 수 있다. 대형 테이블에서는 `pg_stat_activity.backend_xid`를 모니터링하라.
- 트랜잭션 ID 랩어라운드 방지 목적이라면 REPACK이 아니라 `VACUUM`이 답이다.

## 살아남은 기능 2: pg_plan_advice — 쿼리 플랜 고정의 표준 화법

두 번째는 확장 모듈 `pg_plan_advice`다. 그간 PostgreSQL에는 오라클식 힌트가 없어서 플랜이 튀는 문제를 pg_hint_plan으로 때워야 했는데, PG19는 "어드바이스 문자열(advice string)"이라는 미니 언어로 플래너 결정을 기술·재현·변경하는 표준 접근을 제공한다. 자동 적용을 담당하는 `pg_stash_advice` 확장도 함께 들어왔다.

동작 방식은 단순하다. `EXPLAIN`에 `PLAN_ADVICE` 옵션을 붙이면 플래너가 고른 결정을 어드바이스 문자열로 출력해준다.

```sql
LOAD 'pg_plan_advice';

EXPLAIN (COSTS OFF, PLAN_ADVICE)
SELECT * FROM join_fact f JOIN join_dim d ON f.dim_id = d.id;
-- Generated Plan Advice:
-- JOIN_ORDER(f d) HASH_JOIN(d) SEQ_SCAN(f d) NO_GATHER(f d)

-- 그중 고정하고 싶은 결정만 골라 세팅
SET pg_plan_advice.advice = 'JOIN_ORDER(f d)';

EXPLAIN (COSTS OFF)
SELECT * FROM join_fact f JOIN join_dim d ON f.dim_id = d.id;
-- Supplied Plan Advice: JOIN_ORDER(f d) /* matched */
```

시스템이 생성한 문자열에서 필요한 태그만 뽑아 쓰는 게 권장 워크플로우다. 다만 문서는 "플래너 판단을 덮어쓰는 건 쉽게 역효과가 난다"고 경고한다. 데이터 분포가 바뀌어도 플래너가 적응하지 못하게 되기 때문이다. 리그레션 테스트로 플랜 변화를 잡아내는 용도로 먼저 도입하고, 강제는 신중하게 하는 것이 좋겠다.

## 살아남은 기능 3: WAIT 명령과 논리 복제 개선

복제 영역의 변화도 실용적이다. 특히 눈에 띄는 것이 읽기 복제본에서 쓰기-후-읽기(read-your-writes) 일관성을 구현할 수 있는 `WAIT` 명령이다.

```sql
-- 애플리케이션에서: 쓰기 LSN을 기억했다가
SELECT pg_current_wal_insert_lsn();   -- 예: '0/3A91C48'

-- 복제본 접속 후 해당 LSN이 반영(replay)될 때까지 대기
WAIT FOR LSN '0/3A91C48'
  WITH (MODE 'standby_replay', TIMEOUT '5s', NO_THROW);
```

모드는 `standby_replay`/`standby_write`/`standby_flush`/`primary_flush` 4가지고, `NO_THROW`를 주면 타임아웃이나 프로모션 발생 시 에러 대신 상태 문자열(`success`, `timeout`, `not in recovery`)을 반환한다. "복제본에서 방금 쓴 데이터가 안 보여요" 류의 문제를 커넥션 풀 수준에서 해결할 수 있게 된 셈이다.

여기에 더해: (1) `wal_level`이 `replica`면 서버 재시작 없이 논리 복제를 켤 수 있게 됐고, (2) 시퀀스 값의 논리 복제가 지원되며, (3) autovacuum이 인덱스를 병렬로 정리하고 새로운 스코어링 시스템으로 우선순위를 정한다. I/O 워커 수도 `io_min_workers`/`io_max_workers`/`io_worker_idle_timeout`/`io_worker_launch_interval` 변수로 자동 조절된다. PG18에서 도입된 async I/O가 실제 운영 환경에 맞춰 성숙해지는 릴리스라는 평가다.

## 업그레이드 전 체크리스트: 소리 없이 바뀌는 기본값들

PG19는 기능 추가만큼이나 기본값 변경이 많다. 공식 릴리스 노트 기준으로 꼭 확인할 것들이다.

- **JIT가 기본 비활성화** — 이전까지 비용 기반으로 켜지던 JIT가, 그 비용 산정이 신뢰할 수 없다는 판단으로 기본 off됐다. 대형 분석 쿼리가 많은 워크로드는 직접 `jit = on`을 확인해야 한다.
- **`max_locks_per_transaction` 기본값 64 → 128** — 잠금 할당 구조가 바뀌어 같은 용량을 내려면 값을 두 배로 올려야 한다. 이전 값을 명시적으로 운영 중이었다면 재계산이 필요하다.
- **RADIUS 인증 제거** — UDP 기반 RADIUS는 "고칠 수 없이 불안전"하다는 이유로 완전히 삭제됐다. RADIUS를 쓰던 환경은 다른 인증 수단으로 미리 전환하라.
- **MD5 비밀번호 경고** — PG18에서 폐기(deprecated)된 MD5 인증이 성공 시 경고를 출력한다(기본 7일 전 만료 경고 `password_expiration_warning_threshold`도 추가).
- **`standard_conforming_strings` 강제 on** — PG19 이전 버전의 pg_dump로 만든 덤프 중 해당 설정이 off인 덤프는 PG19에 로드되지 않는다. 덤프는 반드시 PG19 도구로 뜰 것.
- **`MULE_INTERNAL` 인코딩 제거, btree_gist의 inet/cidr 인덱스 → GiST 기본 변경** — 해당 기능 사용자는 마이그레이션 계획 필요.

마이그레이션 자체는 관례대로 `pg_upgrade`·논리 복제·dump/restore 셋 중 하나다. 단, 위 비호환 항목 때문에 "덤프는 반드시 PG19 버전 도구로"라는 원칙만은 새로 생겼다.

그리고 이번 달과 직결된 일정 하나. **PostgreSQL 14는 2026년 11월 12일에 지원이 종료된다.** PG14 운영 환경이라면 PG19 출시를 기다릴 것이 아니라 지금 당장 PG18로의 업그레이드를 진행하는 게 정석이다 — Snowflake 블로그도 "오늘의 안전한 업그레이드 대상은 PG18"이라고 권하고 있다.

## 결론

- PostgreSQL 19 정식 출시는 11월 19일 전후로 지연됐다(10월 15일 RC1). 원인은 LLM 코드 리뷰가 엣지 케이스 결함을 조기에 대량 발견하면서 퀄리티 바가 올라갔기 때문이다.
- SQL/PGQ, `GROUP BY ALL`, `MERGE/SPLIT PARTITION` 등 8~10개 기능은 v20으로 이월됐다. "출시 후 5년에 걸쳐 고치는" 대신 "출시 전에 되돌리는" 선택은 프로세스가 작동한다는 증거다.
- 살아남은 기능은 실용적이다 — 잠금 없는 테이블 재패킹(REPACK), 표준화된 플랜 고정(pg_plan_advice), 복제본 읽기 일관성(WAIT), 재시작 없는 논리 복제.
- 업그레이드 담당자는 JIT 기본 off, `max_locks_per_transaction` 두 배, RADIUS 제거, 덤프 도구 버전 원칙을 먼저 점검하라.
- 당장 할 일: RC1을 받아서 운영 워크로드를 돌려보는 것이다. 커뮤니티가 명시적으로 요청하는 것도 이것이고, LLM이 찾지 못한 버그는 여전히 여러분의 실데이터에서만 나온다.
