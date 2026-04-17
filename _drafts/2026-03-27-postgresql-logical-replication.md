---
title: "PostgreSQL 논리 복제가 전체 서버를 멈출 수 있다 — pg_logical 슬롯 누적 문제"
date: 2026-03-27 21:00:00 +0900
categories: [인프라 실무]
tags: [postgresql, replication, pg_logical, wal, 장애처리, db, 트러블슈팅]
---

## 신고는 이렇게 왔다

"DB 서버 디스크가 빠르게 차고 있어요. 어젯밤부터 계속 늘어나는데..."

평소엔 안정적이던 서버였다. 데이터가 갑자기 폭발적으로 늘어날 리 없었다. 디스크를 파고드니 범인은 `pg_wal` 디렉터리였다.

---

## 시스템 구조 먼저

운영 중인 시스템에는 PostgreSQL 기반의 데이터 수집 서버가 있다. 현장 곳곳의 단말기들이 이 서버에 연결되어 실시간으로 데이터를 받아간다.

```
[PostgreSQL Publisher 서버]
    │
    ├── 단말기 A ←── 논리 복제 구독
    ├── 단말기 B ←── 논리 복제 구독
    ├── 단말기 C ←── 논리 복제 구독
    │   ...
    └── 단말기 N ←── 논리 복제 구독 (총 30개 이상)
```

각 단말기는 PostgreSQL **논리 복제(Logical Replication)**를 통해 서버의 데이터 변경사항을 실시간으로 수신한다.

---

## PostgreSQL 논리 복제란

### 물리 복제 vs 논리 복제

PostgreSQL 복제에는 두 가지 방식이 있다.

**물리 복제(Physical Replication)**
- WAL(Write-Ahead Log) 바이트를 그대로 전송
- 전체 데이터베이스 클러스터를 동일하게 복제
- Primary → Standby 구조 (HA 목적)

**논리 복제(Logical Replication)**
- 변경된 데이터를 SQL 수준으로 해석해서 전송
- 특정 테이블만 선택적으로 복제 가능
- 서로 다른 PostgreSQL 버전 간에도 동작
- Publisher → 다수의 Subscriber 구조

현재 환경은 **논리 복제**다. 서버 하나(Publisher)에서 다수의 단말기(Subscriber)로 변경사항을 내보내는 구조.

### WAL이란

PostgreSQL은 모든 변경사항을 먼저 WAL에 기록한다. 트랜잭션이 커밋되기 전에 WAL에 쓰는 것이 보장되어야 장애 시 복구가 가능하다.

```
INSERT/UPDATE/DELETE
    ↓
WAL 파일에 기록 (pg_wal 디렉터리)
    ↓
실제 데이터 파일 반영
    ↓
Checkpoint 이후 불필요한 WAL 삭제
```

논리 복제에서는 이 WAL을 구독자들이 읽어가는 방식으로 데이터를 전달한다.

### 복제 슬롯(Replication Slot)

핵심 개념이다.

각 구독자마다 **복제 슬롯**이 하나씩 생성된다. 슬롯은 "이 구독자가 어디까지 읽었는지"를 추적하는 포인터 역할을 한다.

```sql
SELECT slot_name, plugin, active, restart_lsn
FROM pg_replication_slots;

 slot_name │  plugin   │ active │ restart_lsn
-----------+-----------+--------+-------------
 svc_1    │ pgoutput  │ t      │ 0/B4A3F210
 svc_2    │ pgoutput  │ t      │ 0/B4A3F210
 svc_3    │ pgoutput  │ f      │ 0/A1230000  ← 문제
```

- `active: t` → 현재 연결 중, 정상적으로 WAL을 읽어가고 있음
- `active: f` → 연결 끊김, 슬롯이 멈춰있음
- `restart_lsn` → 이 슬롯이 필요로 하는 가장 오래된 WAL 위치

---

## 문제 발생 메커니즘

### 정상 상태

```
새 WAL 생성 → 모든 슬롯이 읽어감 → 오래된 WAL 삭제 → pg_wal 안정적 유지
```

### 구독자 하나가 끊겼을 때

```
단말기 C, 네트워크 단절
    ↓
svc_C 슬롯: active → false
svc_C 슬롯의 restart_lsn이 단절 시점에서 STOP
    ↓
DB에는 계속 데이터가 들어오고 WAL이 계속 생성됨
    ↓
PostgreSQL: "svc_C가 아직 이 WAL 안 읽었음 → 삭제 불가"
    ↓
WAL 파일이 삭제되지 못하고 pg_wal에 누적 시작
    ↓
나머지 단말기들은 정상인데 서버 디스크가 빠르게 참
    ↓
디스크 풀 → PostgreSQL 쓰기 불가 → 전체 서비스 영향
```

단말기 하나의 연결 문제가 전체 시스템을 멈추는 구조다.

### 실제 측정값 예시

슬롯 lag를 확인하는 쿼리:

```sql
SELECT
    slot_name,
    active,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS wal_lag
FROM pg_replication_slots
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;

 slot_name │ active │ wal_lag
-----------+--------+----------
 svc_1    │ t      │ 18 kB     ← 정상 (실시간 동기화)
 svc_2    │ t      │ 18 kB     ← 정상
 svc_C    │ f      │ 2400 MB   ← 4시간 후 이 상태
 svc_C    │ f      │ 38 GB     ← 2일 후
```

정상 구독자는 18KB 수준을 유지한다. 끊긴 구독자는 시간이 지날수록 WAL lag가 폭증한다.

---

## 왜 더 심각한가 — txid Wraparound

디스크 문제보다 더 치명적인 게 있다.

PostgreSQL은 트랜잭션 ID(txid)를 32비트 순환 방식으로 사용한다. 약 21억 개 트랜잭션마다 한 바퀴를 돈다.

정상적인 상황에서는 `VACUUM`이 주기적으로 실행되어 오래된 트랜잭션 정보를 정리하고 새 ID를 사용할 수 있게 한다. 그런데 슬롯이 오래된 스냅샷을 잡고 있으면:

```
끊긴 슬롯이 과거 특정 시점의 스냅샷을 잡고 있음
    ↓
VACUUM이 "이 스냅샷보다 오래된 건 건드리면 안 돼"라고 판단
    ↓
VACUUM이 제대로 동작 안 함
    ↓
txid 소진 → PostgreSQL 강제 shutdown 
    ↓
DB에 접근 불가 (txid Wraparound 방지를 위한 보호 모드)
```

이 상태가 되면 `pg_resetwal`로 강제 복구하거나 pg_dump/restore를 해야 한다. 운영 환경에서 최악의 시나리오다.

---

## 현재 환경 분석

### 환경 정보

| 항목 | 값 |
|---|---|
| PostgreSQL 버전 | 13.2 |
| 복제 플러그인 | pgoutput (내장) |
| 구독자 수 | 30개+ |
| `max_replication_slots` | 100 |
| `max_wal_size` | 미설정 (기본값 1GB) |
| `max_slot_wal_keep_size` | 미설정 (기본값 -1, 무제한) |

**`max_wal_size` 미설정의 함정**

많은 분들이 `max_wal_size`를 WAL 최대 보존량으로 오해한다. 실제로는 **checkpoint 빈도 조절 파라미터**에 가깝다. 슬롯이 잡고 있는 WAL은 이 설정과 무관하게 삭제되지 않는다.

```
max_wal_size = 1GB 설정해도
슬롯이 잡고 있는 WAL은 삭제 불가
→ pg_wal 디렉터리는 무제한 증가 가능
```

**`max_slot_wal_keep_size = -1` (무제한)**

이게 현재 환경의 핵심 문제다. 슬롯이 보유할 수 있는 WAL에 아무런 제한이 없다.

---

## 대응 방안

### 1단계: 즉각 대응 (현재 운영 중)

끊긴 구독자를 감지하면 해당 슬롯을 수동으로 정리한다.

```sql
-- lag가 큰 슬롯 확인
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots
WHERE NOT active
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;

-- 해당 슬롯 삭제 (구독자 재연결 시 재생성 필요)
SELECT pg_drop_replication_slot('svc_C');
```

슬롯을 삭제하면 WAL이 즉시 정리된다. 구독자가 다시 연결하면 슬롯이 재생성되고, 이때 누락된 데이터는 full resync가 발생한다.

### 2단계: 파인튜닝 (즉시 적용 가능)

**`max_slot_wal_keep_size` 설정** — PG 13에서 추가된 파라미터

```sql
-- postgresql.conf
max_slot_wal_keep_size = 10GB

-- 또는 ALTER SYSTEM으로
ALTER SYSTEM SET max_slot_wal_keep_size = '10GB';
SELECT pg_reload_conf();
```

슬롯의 WAL 보유량이 이 값을 초과하면 슬롯이 자동으로 무효화(invalidated)된다.

```sql
-- 무효화된 슬롯 확인
SELECT slot_name, invalidation_reason
FROM pg_replication_slots;

 slot_name │ invalidation_reason
-----------+--------------------
 svc_C    │ wal_removed
```

무효화된 슬롯은 더 이상 WAL을 잡지 않는다. 구독자가 재연결하면 full resync 후 정상 복구된다.

**`wal_sender_timeout` 단축**

```sql
wal_sender_timeout = 30s  -- 기본값 60s
```

끊긴 구독자를 더 빠르게 감지한다. 슬롯은 남지만 감지 속도가 빨라진다.

### 3단계: 모니터링 자동화

주기적으로 슬롯 상태를 확인하고 임계값 초과 시 알림을 보내는 스크립트:

```sql
-- lag가 5GB 이상인 비활성 슬롯 조회
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots
WHERE NOT active
  AND pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 5 * 1024^3;
```

이 쿼리 결과가 나오면 즉시 대응이 필요하다.

---

## PG 버전의 한계

| 파라미터 | 지원 버전 | 역할 |
|---|---|---|
| `max_slot_wal_keep_size` | PG 13+ ✅ | WAL 누적 한도 |
| `idle_replication_slot_timeout` | PG 17+ ❌ | 비활성 슬롯 자동 비활성화 |

현재 PG 13.2 환경에서는 `idle_replication_slot_timeout`을 쓸 수 없다. PG 17로 업그레이드하면 일정 시간 비활성 슬롯을 자동으로 비활성화할 수 있어서 운영 편의성이 크게 개선된다.

---

## 근본적인 구조 문제

지금까지 설명한 해결책들은 모두 **사후 대응**이다.

진짜 문제는 구조에 있다. 구독자 하나의 장애가 Publisher 전체에 영향을 주는 tight coupling 구조. 단말기 수가 늘어날수록 리스크가 커진다.

```
단말기 30개 → 장애 확률 = 단말기 1개보다 30배 높음
max_replication_slots = 100 → 언젠가는 소진
```

다음 글에서는 이 문제를 컨테이너 환경에서 어떻게 다르게 접근할 수 있는지 다룬다.

---

## 한 줄 요약

> 구독자 하나의 연결 문제가 Publisher 전체를 멈출 수 있다. `max_slot_wal_keep_size` 설정이 가장 빠른 안전망이다.
