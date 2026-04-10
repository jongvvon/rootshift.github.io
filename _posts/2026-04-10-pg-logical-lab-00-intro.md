---
title: "PostgreSQL 논리 복제 랩 #0 — 현업 장애를 컨테이너로 재현한다"
date: 2026-04-10 11:48:00 +0900
categories: [인프라 실무]
tags: [postgresql, logical-replication, docker, docker-compose, pg_logical, wal, lab]
---

## 시작하게 된 이유

[지난 글](https://blog.rootshift.dev/posts/postgresql-logical-replication/)에서 썼던 장애 — WAL이 수십 GB 쌓이고, `pg_logical/snapshots` 디렉터리에 파일이 수천 개까지 불어나던 그 문제.

그때 대응은 어떻게 했냐면, 솔직히 말하면 **슬롯 삭제 + 파라미터 조정**이 전부였다. 급한 불은 껐다. 그런데 왜 그런지, 정확히 어떤 메커니즘인지는 로그 보면서 추측하는 수준이었다.

운영 환경에서는 함부로 건드릴 수가 없다. 확인하고 싶어도 "이 설정 바꾸면 어떻게 돼요?"를 실험해볼 수가 없다. 그래서 직접 재현해보기로 했다.

목표는 단순하다. **현업과 동일한 환경을 만들고, 문제를 다시 터뜨리고, 단계별로 해결해본다.**

---

## 현업 환경 (익명 처리)

대형 운영 시설 내 데이터 수집 서버 구조다.

```
[PostgreSQL Publisher 서버]
    │
    ├── 단말기 A ←── 논리 복제 구독
    ├── 단말기 B ←── 논리 복제 구독
    ├── 단말기 C ←── 논리 복제 구독
    │   ...
    └── 단말기 N ←── 논리 복제 구독 (30개+)
```

| 항목 | 값 |
|---|---|
| PostgreSQL 버전 | 13.2 |
| 복제 플러그인 | pgoutput (내장) |
| 구독자 수 | 30개+ |
| `max_replication_slots` | 100 |
| `max_slot_wal_keep_size` | 미설정 (-1, 무제한) |

각 단말기는 Publisher에 논리 복제로 연결되어 실시간으로 데이터를 받아간다. 단말기가 끊기면 해당 슬롯이 `active: false` 상태가 되고, WAL이 쌓이기 시작하는 구조.

---

## 왜 VM이 아니라 Docker Compose인가

처음엔 클라우드 VM 2대로 구성하려 했다. 근데 생각해보면:

- VM은 24시간 켜두면 비용이 나온다
- 실험 후 정리가 귀찮다
- 환경 망가지면 처음부터 다시 세팅해야 한다
- PG 버전 바꿔가며 비교 테스트가 불편하다

Docker Compose로 하면 이 문제가 다 사라진다.

```yaml
# 이것만으로 publisher/subscriber 분리 환경이 생긴다
services:
  publisher:
    image: postgres:13.2

  subscriber:
    image: postgres:13.2
```

로컬에서 무료로 돌아가고, 문제가 생기면 `docker compose down && docker compose up`으로 리셋. 시리즈가 끝나면 코드로 환경 전체를 재현할 수 있다.

클라우드는 소스 수정 후 장기 운영 검증이 필요한 시점에 올려도 늦지 않다.

---

## 이 시리즈에서 할 것

단계별로 진행한다.

**#1 — 환경 구축**
Docker Compose로 Publisher/Subscriber 논리 복제 환경 세팅. 현업과 동일한 PG 13.2, pgoutput 기준.

**#2 — 문제 재현**
슬롯 inactive 상황을 강제로 만들어 WAL 누적과 스냅샷 파일 폭증을 직접 확인. 수치로 보는 메커니즘.

**#3 — Config 레벨 해결**
`max_slot_wal_keep_size`, `wal_sender_timeout` 등 파라미터 튜닝. 효과 측정.

**#4 — 모니터링/자동화**
슬롯 상태 감지, 임계값 초과 시 알림, 자동 정리 로직. Extension 또는 외부 스크립트로 구현.

**#5 — 소스 수정**
Apply worker 재시도 로직, Sync worker cleanup. PostgreSQL License(BSD급)이라 포크해서 수정 가능하다. 이게 마지막 단계.

---

## 한 줄 요약

> 운영에서 터진 문제를 컨테이너로 재현하고, Config → Extension → 소스 수정까지 단계별로 파고든다.

다음 글: Docker Compose로 PG 13.2 논리 복제 환경 구축
