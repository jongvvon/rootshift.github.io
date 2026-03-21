---
title: "[K8s ELK 시리즈 #0] 왜 시작했나 — 물리 인프라를 클라우드로 옮기기"
date: 2026-03-22 00:00:00 +0900
categories: [DevOps 전환기]
tags: [kubernetes, elk, elasticsearch, logstash, kibana, k8s, devops, 클라우드]
---

## 시작은 현업에서 쓰는 시스템이었다

나는 대형 운영 시설의 IT 인프라팀에서 일한다.

담당 시스템 중에 내부적으로 **모니터링 플랫폼**이라고 부르는 게 있다. 이름은 거창한데 구조를 뜯어보면 딱 ELK 스택이다.

```
현장 장비/서버 (SNMP, 시스템 로그)
    ↓
Logstash AP서버 (수집/파싱)
    ↓
Elasticsearch (저장/검색)
    ↓
대시보드 서버 (시각화)
```

수십 대 서버, 물리 장비. 인덱스 수천 개, 데이터 TB 단위.

근데 이게 전부 물리 서버다. 서버 한 대 죽으면 사람이 가서 살려야 하고, 용량 늘리려면 장비 구매 품의 올려야 한다. 스케일 아웃? 몇 달 걸린다.

**"이걸 K8s 위에 올리면 어떻게 될까?"**

그 질문에서 이 시리즈가 시작됐다.

---

## ELK 스택이 뭔데

**E**lasticsearch + **L**ogstash + **K**ibana

```
[데이터 소스]  →  [Logstash]  →  [Elasticsearch]  →  [Kibana]
장비/서버 로그     수집·파싱·변환    저장·검색·인덱싱     시각화·대시보드
```

### Logstash — 수집과 변환

데이터를 받아서 가공하는 파이프라인이다. 구조는 단순하다:

```ruby
input {
  tcp { port => 6001 codec => json }  # TCP 6001로 JSON 수신
}

filter {
  grok {
    # 텍스트 로그를 필드로 분리
    match => { "message" => "%{TIMESTAMP_ISO8601:time} %{HOSTNAME:host} %{GREEDYDATA:msg}" }
  }
  mutate {
    add_field => { "[@metadata][index]" => "log-syslog" }
  }
}

output {
  elasticsearch {
    hosts => ["es:9200"]
    index => "%{[@metadata][index]}-%{+YYYY.MM.dd}"
  }
}
```

**Input → Filter → Output** 3단계. 어디서 받고, 어떻게 가공하고, 어디로 보낼지.

### Elasticsearch — 검색 엔진이자 저장소

일반 DB(MySQL, MSSQL)랑 근본적으로 다르다:

| 일반 RDB | Elasticsearch |
|---|---|
| 스키마 고정 | 유연한 매핑 |
| Row 단위 | Document(JSON) 단위 |
| SQL | REST API + JSON 쿼리 |
| 정확한 값 검색 | 전문 검색(Full-text) |
| 테이블 | Index |

로그 데이터에 특화된 이유:
- 구조가 제각각인 로그 → 스키마 고정 불필요
- "disk full" 같은 텍스트 검색 → Full-text search
- 날짜별 인덱스 자동 분리 → `log-syslog-2026.03.22`
- 대용량 → 샤드로 분산 저장

### Kibana — 눈으로 보는 대시보드

브라우저에서 접속하는 웹 UI. 코딩 없이 ES 데이터를 시각화한다. SFMS의 AppleMango 솔루션이 이 역할이다.

---

## 왜 K8s인가

**현재 물리 환경의 한계:**

```
서버 죽음 → 수동 복구 (사람이 가야 함)
용량 부족 → 장비 구매 (몇 달 걸림)
설정 변경 → SSH 접속해서 직접 수정
환경 재현 → 문서 보고 처음부터 다시
```

**K8s로 바꾸면:**

```
Pod 죽음 → 자동 재시작 (Self-healing)
용량 부족 → 명령어 한 줄 (Scale out)
설정 변경 → YAML 파일 수정 후 apply
환경 재현 → YAML 파일 그대로 다른 곳에 올리기
```

인프라를 코드로 관리한다(IaC). 이게 DevOps/SRE의 핵심이다.

---

## 시리즈 계획

| 편 | 내용 |
|---|---|
| **#0 (이 글)** | 개론, 기본 개념, 왜 시작했나 |
| #1 | minikube 환경 세팅, K8s 기본 개념 실습 |
| #2 | Elasticsearch StatefulSet 배포 |
| #3 | Logstash 배포 + 파이프라인 설정 |
| #4 | Kibana 배포 + 대시보드 구성 |
| #5 | 테스트 데이터 유입 + 검증 |
| #6 | 현업 구조와 비교, 한계와 개선점 |

물리 서버 10대짜리를 K8s YAML 몇 개로 재현하는 과정이다. 잘 될지 모른다. 삽질하면 그것도 그대로 쓴다.

---

## 환경

- MacBook M2 (minikube)
- Docker Desktop
- minikube v1.31.2
- kubectl v1.28.2
- 목표 스택: Elasticsearch 6.8.x + Logstash 6.8.x + Kibana 6.8.x

현업 ES 버전이 6.8.23이라 맞추기로 했다. 최신 8.x랑 많이 다르다는 거 알고 시작한다.

---

다음 편에서 minikube 띄우는 것부터 시작한다.
