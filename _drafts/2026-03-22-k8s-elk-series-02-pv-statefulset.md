---
title: "[K8s ELK 시리즈 #2] PersistentVolume과 StatefulSet — 물리 디스크랑 뭐가 달라?"
date: 2026-03-22 16:00:00 +0900
categories: [DevOps 전환기]
tags: [kubernetes, elk, elasticsearch, persistentvolume, statefulset, k8s, devops]
---

## "그냥 디스크에 저장하는 거랑 뭐가 달라?"

맞는 질문이다.

물리 서버에 Elasticsearch 설치하면 `/var/lib/elasticsearch` 디렉터리에 데이터 쌓인다. 서버 재시작해도 디스크가 날아가지 않으니까 데이터는 유지된다.

K8s PersistentVolume도 결국 어딘가의 디스크에 저장하는 건데, 왜 굳이 PV라는 개념을 만들었을까?

---

## 물리 환경의 한계

물리 서버 3대에 ES를 설치했다고 가정하자.

```
server-01: ES 데이터 → /var/lib/elasticsearch (디스크 A)
server-02: ES 데이터 → /var/lib/elasticsearch (디스크 B)
server-03: ES 데이터 → /var/lib/elasticsearch (디스크 C)
```

**문제 1: 서버와 데이터가 결합됨**

server-01이 죽으면 디스크 A의 데이터도 같이 접근 불가. 새 서버(server-04) 띄워도 디스크 A 데이터를 자동으로 붙일 방법이 없다. 사람이 개입해야 한다.

**문제 2: 서버 증설이 수동**

ES 노드 추가하려면:
1. 서버 구매/준비
2. OS 설치
3. ES 설치 + 설정
4. 클러스터에 수동 합류

→ 몇 주~몇 달 걸린다.

**문제 3: 환경 불일치**

개발 서버, 스테이징 서버, 운영 서버 설정이 제각각. "내 로컬에서는 됐는데 운영에서 안 돼"가 일상.

---

## K8s PersistentVolume이 다른 이유

PV는 **스토리지를 추상화**한다.

```
[Pod] ← [PVC] ← [PV] ← [실제 스토리지]
                         (로컬 디스크, NFS, AWS EBS, GCP PD...)
```

Pod 입장에서는 "5GB 스토리지가 필요하다"고 요청(PVC)만 하면 된다. 실제로 어느 서버의 어느 디스크에 저장되는지 몰라도 된다.

**핵심 장점:**

**① Pod와 스토리지 분리**

Pod가 죽어도 PV는 살아있다. 새 Pod가 뜨면 같은 PV에 자동으로 연결된다.

```
elasticsearch-0 Pod 죽음
    ↓
StatefulSet이 elasticsearch-0 재생성
    ↓
PVC(es-data-elasticsearch-0)를 자동으로 다시 마운트
    ↓
데이터 그대로
```

**② 스토리지 백엔드 교체 가능**

개발: `hostPath` (로컬 디스크)
운영: `AWS EBS` or `GCP PD` (클라우드 블록 스토리지)

YAML에서 StorageClass만 바꾸면 된다. ES 설정은 그대로.

**③ 용량 관리 선언적**

```yaml
resources:
  requests:
    storage: 5Gi  # "5GB 주세요"
```

필요하면 늘리기도 된다 (일부 StorageClass 지원).

---

## StatefulSet이 Deployment랑 다른 이유

K8s에서 Pod 관리자는 두 가지다.

**Deployment:** 웹서버, API 서버 같은 무상태(Stateless) 앱

```
pod-abc123 죽음 → pod-xyz789 새로 생성 (이름 달라도 상관없음)
```

**StatefulSet:** DB, ES 같은 상태 있는(Stateful) 앱

```
elasticsearch-0 죽음 → elasticsearch-0 그대로 재생성 (이름 고정)
```

### StatefulSet의 3가지 보장

**① 안정적인 네트워크 ID**

```
Deployment Pod:  app-7d9f4b-xkp2q (매번 랜덤)
StatefulSet Pod: elasticsearch-0   (항상 고정)
                 elasticsearch-1
                 elasticsearch-2
```

ES 클러스터링할 때 노드들이 서로를 `elasticsearch-0.elasticsearch.elk.svc.cluster.local` 같은 고정 DNS로 찾는다. 이름이 매번 바뀌면 클러스터 구성이 깨진다.

**② 안정적인 스토리지**

각 Pod마다 자기 전용 PVC가 자동 생성된다.

```yaml
volumeClaimTemplates:
- metadata:
    name: es-data
  spec:
    storage: 5Gi
```

이렇게 선언하면:
- elasticsearch-0 → PVC: `es-data-elasticsearch-0`
- elasticsearch-1 → PVC: `es-data-elasticsearch-1`
- elasticsearch-2 → PVC: `es-data-elasticsearch-2`

각 노드가 자기 데이터를 따로 관리. Pod 재생성돼도 항상 자기 PVC에 연결.

**③ 순서 보장**

```
elasticsearch-0 먼저 시작
    ↓ Ready 확인 후
elasticsearch-1 시작
    ↓ Ready 확인 후
elasticsearch-2 시작
```

ES는 마스터 노드가 먼저 떠야 다른 노드가 조인할 수 있다. Deployment는 동시에 다 띄워버리니까 클러스터링이 꼬인다.

---

## 실제로 검증해봤다

**Before (PV 없을 때):**

```bash
# 데이터 있는 상태
curl http://localhost:9200/_cat/indices
# log-syslog-2026.03.22   2docs
# log-snmptrap-2026.03.22 1doc

# Pod 강제 재시작
kubectl delete pod elasticsearch-0 -n elk

# 재시작 후
curl http://localhost:9200/_cat/indices
# (빈 결과) ← 데이터 사라짐
```

**After (PV 붙인 후):**

```yaml
# elasticsearch StatefulSet에 추가
volumeClaimTemplates:
- metadata:
    name: es-data
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 5Gi
```

```bash
# Pod 재시작 후
kubectl get pvc -n elk
# es-data-elasticsearch-0   Bound   5Gi  ← PVC 유지

curl http://localhost:9200/_cat/indices
# log-syslog-2026.03.22   1doc   ← 데이터 살아있음!
# log-snmptrap-2026.03.22 1doc   ← 데이터 살아있음!
```

---

## 그럼 물리 환경이랑 결국 같은 거 아닌가?

아니다. 핵심 차이는 **"선언"으로 관리한다**는 점이다.

| | 물리 환경 | K8s PV |
|---|---|---|
| 스토리지 추가 | 디스크 구매, 마운트, 설정 | YAML 한 줄 수정 |
| 서버 죽으면 | 수동 복구 | 자동 재생성 + PV 재연결 |
| 환경 복제 | 문서 보고 처음부터 | YAML 파일 그대로 apply |
| 스토리지 위치 | 서버에 종속 | 추상화 (어디든 가능) |
| 이사 (클라우드 전환) | 데이터 이전 작업 | StorageClass만 교체 |

물리 환경도 디스크에 저장하는 건 같다. 다른 건 **운영 방식**이다.

물리 환경은 "서버 A의 `/dev/sdb`에 마운트했다"는 지식이 사람 머릿속에 있다. 담당자 바뀌면 아무도 모른다.

K8s는 YAML 파일이 전부 알고 있다. 사람이 없어도 시스템이 스스로 복구한다.

---

## 한 줄 요약

> StatefulSet은 Pod에 이름과 스토리지를 보장한다. PV는 스토리지를 Pod로부터 분리한다. 이 둘이 합쳐져야 DB/ES 같은 상태 있는 앱을 K8s에서 안전하게 돌릴 수 있다.
