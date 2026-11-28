---
title: "PostgreSQL 19 업그레이드 가이드 — REPACK, ON CONFLICT DO SELECT, WAIT FOR LSN 실전 정리"
date: "2026-11-28"
keywords: ["PostgreSQL 19", "REPACK", "pg_plan_advice", "ON CONFLICT DO SELECT", "WAIT FOR LSN", "pg_upgrade", "SQL/PGQ"]
lang: "ko"
description: "2026년 11월 19일 정식 출시된 PostgreSQL 19의 핵심 기능과 브레이킹 체인지를 운영자·개발자 관점으로 정리하고, SQL/PGQ가 사라진 경위까지 확인한 업그레이드 실전 가이드"
---

# PostgreSQL 19 업그레이드 가이드 — REPACK, ON CONFLICT DO SELECT, WAIT FOR LSN 실전 정리

PostgreSQL 19가 2026년 11월 19일 정식 출시되었습니다. 공식 지원 기간은 5년이라 최종 지원 종료는 2031년 11월 19일입니다. 눈여겨볼 타이밍이 하나 더 있는데, PostgreSQL 14의 지원이 바로 이달 11월 12일에 끝났다는 점입니다. 아직 PG14를 운영 중이라면 이번 방학(연말 배포 동결)이 지나가기 전에 업그레이드 경로를 정리해야 합니다.

PG19의 방향성을 한 문장으로 요약하면 "운영 중단 없이 큰 테이블을 정리하고, 실행계획 회귀를 통제하고, 애플리케이션의 잦은 왕복을 줄인다"입니다. 이 글에서는 공식 릴리스 발표문을 기준으로 실제 배포판에 들어간 기능만 다루고, 마지막에 베타에서 사라진 기능(SQL/PGQ)의 경위까지 짚어봅니다. 출처는 모두 PostgreSQL 공식 발표문과 공식 버저닝 문서, 그리고 Neon의 기능 가이드입니다.

## 운영자 관점 3종 세트: REPACK, 실행계획 고정, 병렬 autovacuum

### REPACK — VACUUM FULL의 퇴장

업데이트와 삭제가 많은 테이블은 "블로트(bloat)"라 불리는 사용하지 않는 공간이 쌓이고, 이것이 쿼리 성능을 갉아먹습니다. 기존의 해결책은 VACUUM FULL이나 CLUSTER였는데, 둘 다 작업이 끝날 때까지 테이블 접근을 막아 운영 서비스에서는 사실상 쓰기 어려운 명령이었습니다. 그래서 실무에서는 pg_repack 확장을 따로 설치해 쓰는 게 일반적이었죠.

PostgreSQL 19는 이 기능을 코어로 가져왔습니다.

```sql
-- 읽기·쓰기를 막지 않고 테이블을 재구성
REPACK (CONCURRENTLY) my_table;

-- 인덱스 순서에 맞춰 물리적 재정렬까지
REPACK (CONCURRENTLY) my_table USING INDEX my_table_pkey;
```

VACUUM FULL과 CLUSTER는 호환성을 위해 남아 있지만, 공식 발표문은 REPACK이 이 둘을 대체한다고 명시합니다. pg_repack 확장을 별도로 관리하던 조직이라면 PG19로 올라오는 순간 의존성 하나를 줄일 수 있습니다.

### pg_plan_advice — 실행계획 회귀를 잡는 공식 도구

데이터가 변하면 플래너가 고르는 실행계획도 변합니다. 그런데 그 새 계획이 갑자기 느려지는 "계획 회귀(plan regression)"는 전통적으로 DBA의 머리를 아프게 하는 문제였습니다. 애플리케이션 코드는 그대로인데 특정 쿼리만 하루아침에 10배 느려지는 상황 말이죠.

PG19는 `pg_plan_advice` 확장으로 계획 선택을 안정화하고, "plan advice"라는 미니 언어로 다른 계획을 실험할 수 있게 합니다. 짝이 되는 `pg_stash_advice` 확장은 쿼리 식별자(query identifier) 기준으로 조언(advice)을 저장해 자동 적용합니다. 즉, 애플리케이션을 건드리지 않고 특정 쿼리의 계획을 고정할 수 있는 셈입니다. PG19 배포 전에 실행계획 문제로 pg_hint_plan이나 plan management 도구를 쓰고 있었다면 이제 코어 기능으로 이관할지 검토할 시점입니다.

### 병렬 autovacuum과 새로운 관측 지점

autovacuum이 인덱스를 병렬 워커로 청소할 수 있게 되었고, 워커 수는 새 설정 `autovacuum_max_parallel_workers`로 조절합니다. 어떤 테이블부터 처리할지도 새 스코어링 시스템으로 결정합니다. 또 일반 쿼리가 테이블을 읽는 도중 visibility map의 all-visible 비트를 자동으로 켜서, 이후 vacuum의 일감 자체를 줄입니다.

관측 측면에서는 `pg_stat_lock`(락 타입별 통계), `pg_stat_recovery`(복구 상태), `pg_stat_autovacuum_scores`(autovacuum 우선순위) 뷰가 새로 들어왔고, `EXPLAIN ANALYZE`의 `IO` 옵션이 비동기 I/O 활동을 보고합니다. PG18에서 도입된 비동기 I/O 하위시스템 위에서 `io_workers` 설정은 `io_min_workers`/`io_max_workers` 두 설정으로 갈라지며 자동 조절로 바뀌었습니다.

## 개발자 관점: 원자적 get-or-create와 시간 데이터

### INSERT ... ON CONFLICT DO SELECT

"있으면 가져오고 없으면 만들기"는 애플리케이션에서 가장 흔한 패턴 중 하나입니다. 기존에는 충돌 시 더미 UPDATE를 날리는 `DO UPDATE SET col = EXCLUDED.col` 꼼수를 썼는데, 이 방식은 충돌할 때마다 죽은 튜플(dead tuple)을 양산합니다. PG19는 세 번째 액션으로 DO SELECT를 추가해 이 문제를 원천 제거합니다.

```sql
INSERT INTO users (email, name)
VALUES ('alice@example.com', 'Alice')
ON CONFLICT (email) DO SELECT
RETURNING *;
```

이미 있으면 기존 행을, 없으면 새로 만든 행을 RETURNING으로 돌려줍니다. 네트워크 왕복도 한 번 줄어듭니다. Neon의 가이드는 벤치마크에서 no-op UPDATE 대비 거의 4배 빠르다고 소개하는데, 이 수치는 공식 발표가 아닌 Neon 측 측정값이므로 참고치로 봐야 합니다.

### FOR PORTION OF — 시간 구간 데이터의 자동 분할

PG18의 `WITHOUT OVERLAPS` 시간 제약에 이어, PG19는 `UPDATE ... FOR PORTION OF`와 `DELETE ... FOR PORTION OF`를 추가해 SQL:2011 시간 데이터 기능을 사실상 완성했습니다.

```sql
-- Q3 구간만 가격을 바꾸면 나머지 기간은 자동으로 보존·분할된다
UPDATE product_prices
FOR PORTION OF valid_range
FROM '2025-07-01' TO '2025-10-01'
SET price = 34.99
WHERE product_id = 1;
```

이 명령을 실행하면 원래 한 행이 세 행(1~6월, 7~9월, 10~12월)으로 분할되며 수정되지 않은 구간의 데이터는 자동 보존됩니다. 예약 시스템, 보험 증권, 직원 이력처럼 유효기간이 있는 데이터를 다루는 팀이라면 애플리케이션 레벨의 분할 로직을 DB로 내릴 수 있습니다.

### 그 밖의 개선

- **COPY의 JSON 출력**: `COPY (SELECT ...) TO STDOUT` 형태로 COPY가 JSON, 단일 JSON 배열 출력을 지원합니다.
- **SQL/JSON 경로 언어**: `lower()`, `upper()`, `replace()`, `split_part()`, `btrim()` 같은 문자열 메서드 추가.
- **윈도우 함수 IGNORE NULLS**: `lead()`, `lag()`, `first_value()` 등에서 NULL을 건너뛸 수 있습니다.
- **외래 키 체크 개선**: 공식 발표문은 테스트에서 대량 INSERT가 거의 2배 빨라졌다고 밝힙니다.
- **플래너 최적화**: NULL이 없으면 `NOT IN`을 안티 조인으로 변환, 조인 전 적극적 집계(eager aggregation) 수행.
- **postgres_fdw import_stats**: 원격 서버에서 통계를 직접 가져와 샘플링 대신 플래너 정확도를 높입니다.

## 복제·분산: WAIT FOR LSN과 논리 복제의 성숙

읽기를 스탠바이로 보내는 아키텍처의 고질병은 "방금 쓴 데이터를 스탠바이에서 못 읽는" 문제(read-your-writes 위반)입니다. PG19의 `WAIT FOR LSN`은 스탠바이 세션이 특정 WAL 위치까지 재생을 기다린 후 쿼리를 실행하게 만들어, 프라이머리에 쓴 직후 스탠바이에서 일관성 있게 읽는 패턴을 네이티브로 지원합니다.

논리 복제 쪽 변화도 큽니다.

- 첫 논리 복제 슬롯 생성 시 WAL 레벨이 자동으로 logical로 올라가 **서버 재시작이 불필요**해졌습니다(새 설정 `effective_wal_level`으로 확인).
- `CREATE PUBLICATION ... FOR ALL SEQUENCES`로 **시퀀스 값의 논리 복제**가 가능해졌습니다. 기존에는 시퀀스가 복제되지 않아 구독 측 nextval이 꼬이는 문제를 애플리케이션에서 우회해야 했습니다.
- `FOR ALL TABLES`에서 `EXCEPT` 절로 테이블 제외 가능, 구독이 `SERVER` 절로 postgres_fdw 외부 서버 정의에서 접속 정보를 상속.
- `retain_dead_tuples` 구독 옵션으로 동시 삭제와 충돌하는 업데이트를 감지하고 `pg_stat_subscription_stats`에서 카운트.

## 브레이킹 체인지 체크리스트 — 업그레이드 전 반드시 점검

PG19는 기능 추가만큼이나 기본값 변경이 많은 릴리스입니다. 다음 항목은 pg_upgrade 실행 전에 체크해야 할 목록입니다.

1. **JIT가 기본 비활성화** — 비용 추정이 신뢰할 수 없다고 판단되어 기본값이 바뀌었습니다. 대형 분석 쿼리가 많은 워크로드라면 수동으로 JIT를 켜야 이전 성능이 유지됩니다.
2. **TOAST 기본 압축이 LZ4** — LZ4 지원으로 빌드된 경우 기본 압축이 lz4로 바뀝니다. pglz와의 상호 운용은 유지되지만, 압축률/속도 특성이 달라지므로 대용량 TEXT 저장이 있는 시스템은 재측정이 필요합니다.
3. **RADIUS 인증 제거** — RADIUS 기반 인증을 쓰고 있다면 SCRAM 등 다른 방식으로 미리 이관해야 합니다.
4. **md5 경고 시작** — PG18에서 폐기(deprecated)된 md5 인증이 PG19부터 성공 시에도 경고를 출력하며, 향후 릴리스에서는 제거됩니다. 공식 권장은 SCRAM입니다. 관련 설정 `md5_password_warnings`, 비밀번호 만료 사전 경고용 `password_expiration_warning_threshold`도 새로 들어왔습니다.
5. **I/O 워커 설정 분리** — `io_workers`가 `io_min_workers`/`io_max_workers`로 대체되었습니다. PG18에서 io_method를 조정해 쓰던 환경은 설정 파일을 수정해야 합니다.
6. **서버 측 SNI 도입** — 클라이언트가 접속한 호스트명에 따라 다른 TLS 인증서를 제공할 수 있고, 매핑은 새 `pg_hosts.conf` 파일에 합니다. 다중 인증서 운영에 유용하지만 인증서 배포 스크립트가 있다면 이 구조를 반영할 시점입니다.

업그레이드 자체는 전통과 같습니다. 메이저 버전 간에는 pg_upgrade 또는 dump/restore가 필요하고, 건너뛰기는 가능하지만 중간 버전의 릴리스 노트를 읽는 것이 권장됩니다. PG14 사용자는 지원이 이미 끝난 상태이므로, 보안 수정이 더 이상 제공되지 않는다는 점을 감안해 이번 창에 17이나 19로 직행하는 계획을 세우는 것이 합리적입니다.

## SQL/PGQ는 어디로 갔나 — 베타 기능 문서를 읽을 때의 교훈

PG19 이야기에서 빼놓을 수 없는 것이 하나 있습니다. 개발 초기부터 이 릴리스의 헤드라인으로 소개되던 SQL/PGQ(속성 그래프 쿼리, SQL:2023 표준)는 2026년 9월 7일, 최종 릴리스를 앞두고 되돌려졌습니다. `CREATE PROPERTY GRAPH`와 `GRAPH_TABLE`은 PG19에 들어가지 않았고, Neon의 가이드에 따르면 조기에 돌아온다면 2027년 9월 예상인 PostgreSQL 20입니다.

흥미로운 건 많은 2026년 상반년 글과 베타 리뷰가 아직 "PostgreSQL 19에 그래프 쿼리가 온다"고 소개하고 있다는 점입니다. SQL/PGQ를 기대하고 PG19 업그레이드를 계획하고 있었다면 계획을 수정해야 합니다. 이 사건이 주는 교훈은 명확합니다 — **베타 기능은 최종 릴리스 직전까지도 사라질 수 있으므로, 업그레이드 근거는 공식 릴리스 발표문 기준으로 세울 것.** 다행히 PG19에 실제로 들어간 기능만으로도 이번 릴리스는 충분히 무겁습니다.

## 결론: 핵심 요약과 다음 단계

- PostgreSQL 19는 2026년 11월 19일 정식 출시, 지원은 2031년 11월 19일까지. PostgreSQL 14는 2026년 11월 12일부로 지원 종료.
- 운영자 핵심은 REPACK(중단 없는 테이블 재구성), pg_plan_advice(계획 회귀 통제), 병렬 autovacuum.
- 개발자 핵심은 ON CONFLICT DO SELECT(원자적 get-or-create), FOR PORTION OF(시간 구간 분할), COPY JSON 출력.
- 읽기 분산 아키텍처라면 WAIT FOR LSN과 시퀀스 논리 복제가 기다림 없는 read-your-writes를 가능하게 합니다.
- 업그레이드 전 체크리스트의 최우선순위는 JIT 기본 비활성화, LZ4 TOAST 기본값, RADIUS 제거, SCRAM 이관입니다.
- SQL/PGQ 그래프 쿼리는 최종 릴리스에서 제외되었습니다. 이를 근거로 한 업그레이드 계획은 수정이 필요합니다.

당장 취할 첫 단계는 간단합니다. 운영 중인 주요 쿼리의 실행계획과 설정(pg의 `SHOW jit;`, 인증 설정, io_method)을 덤프해두고, PG19 패키지를 스테이징에 띄워 애플리케이션 회귀 테스트를 돌려보는 것입니다. 특히 대형 분석 쿼리가 있는 시스템이라면 JIT 비활성화의 성능 영향을 실측부터 하시기 바랍니다.

참고 자료:

- PostgreSQL 19 공식 발표문: https://www.postgresql.org/about/news/postgresql-19-released-3449/
- PostgreSQL 버저닝/지원 정책: https://www.postgresql.org/support/versioning/
- PostgreSQL 19 릴리스 노트: https://www.postgresql.org/docs/release/19.0/
- Neon PostgreSQL 19 신기능 가이드: https://neon.com/postgresql/postgresql-19-new-features
