---
title: "[K8s ELK 시리즈 #1] minikube에 ELK 스택 올리기 — 삽질 포함 전 과정"
date: 2026-03-22 15:47:00 +0900
categories: [DevOps 전환기]
tags: [kubernetes, elk, elasticsearch, logstash, kibana, minikube, k8s, arm64]
---

## 목표

macOS M2(arm64) 환경에서 minikube로 로컬 K8s 클러스터를 띄우고, ELK 스택(Elasticsearch + Logstash + Kibana)을 배포해서 실제 데이터가 Kibana에 보이는 것까지 확인한다.

---

## 환경

- MacBook M2 (arm64), RAM 16GB
- macOS 15.5
- minikube v1.31.2
- kubectl v1.28.2
- Docker Desktop

---

## Step 1. 리소스 확인 먼저

ELK는 메모리를 많이 먹는다. 시작 전에 여유 리소스 확인이 필수다.

```bash
# 메모리 확인
top -l 1 | grep PhysMem

# 디스크 확인
df -h /
```

**내 상황:** 시작 전 메모리 여유가 100MB도 안 남아서 minikube가 응답 불능 상태가 됐다. 백그라운드 프로세스(`mediaanalysis-access` 등)를 정리하고 나서야 4GB 확보됐다.

```bash
# 백그라운드 무거운 프로세스 강제 종료
pkill -f mediaanalysis
```

**minikube 메모리 권장:** 16GB 맥북 기준 **6GB** 할당이 적당했다.

---

## Step 2. minikube 시작

```bash
minikube start --memory=6144 --cpus=4 --disk-size=30g
```

**주의:** 처음에 4096MB로 시작했다가 ELK 3개 올리니까 QEMU VM이 7GB까지 늘어나면서 맥북 전체가 죽었다. 6144MB로 시작하는 게 맞다.

기존 프로파일 있으면 메모리 변경이 안 된다:
```bash
# 완전 삭제 후 재시작
minikube delete
minikube start --memory=6144 --cpus=4 --disk-size=30g
```

정상 기동 확인:
```bash
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   8s    v1.27.4
```

---

## Step 3. 네임스페이스 생성

```bash
kubectl create namespace elk
```

K8s에서 네임스페이스는 논리적 격리 단위다. `elk`라는 공간 안에 ES/Logstash/Kibana를 모두 넣는다.

---

## Step 4. Elasticsearch 배포

### arm64 이슈 먼저

처음에 `elasticsearch:6.8.23`으로 시도했다가 에러:

```
Failed to pull image: no matching manifest for linux/arm64/v8
```

공식 Docker Hub의 6.x 이미지는 arm64 미지원이다. **8.12.0으로 변경**했다. 8.x는 arm64 지원.

### elasticsearch.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: elk
spec:
  serviceName: elasticsearch
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "sysctl -w vm.max_map_count=262144"]
        securityContext:
          privileged: true
      containers:
      - name: elasticsearch
        image: elasticsearch:8.12.0
        ports:
        - containerPort: 9200
        - containerPort: 9300
        env:
        - name: discovery.type
          value: single-node
        - name: ES_JAVA_OPTS
          value: "-Xms1g -Xmx1g"
        - name: xpack.security.enabled
          value: "false"       # 테스트용 보안 비활성
        - name: xpack.security.http.ssl.enabled
          value: "false"
        resources:
          requests:
            memory: "1Gi"
          limits:
            memory: "2Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elk
spec:
  selector:
    app: elasticsearch
  ports:
  - name: http
    port: 9200
    targetPort: 9200
  - name: transport
    port: 9300
    targetPort: 9300
```

**포인트:**
- `StatefulSet` 사용 — DB처럼 상태 있는 앱은 Deployment 대신 StatefulSet
- `initContainers`로 `vm.max_map_count` 설정 — ES 필수 커널 파라미터
- `xpack.security.enabled: false` — 테스트 환경에서 인증 비활성

```bash
kubectl apply -f elasticsearch.yaml
kubectl get pods -n elk
# elasticsearch-0   1/1   Running   0   93s
```

---

## Step 5. Logstash 배포

### logstash.yaml

ConfigMap으로 파이프라인 설정 분리:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.conf: |
    input {
      tcp {
        port => 6001
        codec => json
      }
    }

    filter {
      if [type] == "syslog" {
        mutate {
          add_field => { "[@metadata][index]" => "log-syslog" }
        }
      } else if [type] == "snmp" {
        mutate {
          add_field => { "[@metadata][index]" => "log-snmptrap" }
        }
      } else {
        mutate {
          add_field => { "[@metadata][index]" => "log-general" }
        }
      }
    }

    output {
      elasticsearch {
        hosts => ["http://elasticsearch:9200"]
        index => "%{[@metadata][index]}-%{+YYYY.MM.dd}"
      }
      stdout { codec => rubydebug }
    }
```

**포인트:**
- `type` 필드로 분기 → 인덱스명 동적 생성 (`log-syslog-2026.03.22`)
- K8s 내부 DNS로 ES 접근: `elasticsearch:9200` (서비스명으로 통신)

Logstash도 `logstash:6.8.23` arm64 미지원. `docker.elastic.co/logstash/logstash:8.12.0` 공식 Elastic 레지스트리 이미지 사용.

```bash
kubectl apply -f logstash.yaml
```

---

## Step 6. Kibana 배포

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: kibana:8.12.0
        ports:
        - containerPort: 5601
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch:9200"
        resources:
          requests:
            memory: "512Mi"
          limits:
            memory: "1Gi"
```

```bash
kubectl apply -f kibana.yaml
kubectl get pods -n elk

# NAME                        READY   STATUS    RESTARTS   AGE
# elasticsearch-0             1/1     Running   0          93s
# kibana-789bd4f849-cmx2q     1/1     Running   0          93s
# logstash-6b8b9c4669-sx6kp   1/1     Running   0          93s
```

---

## Step 7. 포트 포워딩

minikube QEMU 드라이버는 `minikube service` 명령이 안 된다. 대신 port-forward 사용:

```bash
kubectl port-forward svc/kibana 5601:5601 -n elk &
kubectl port-forward svc/elasticsearch 9200:9200 -n elk &
kubectl port-forward svc/logstash 6001:6001 -n elk &
```

ES 동작 확인:
```bash
curl http://localhost:9200
# {"name":"elasticsearch-0","cluster_name":"docker-cluster","version":{"number":"8.12.0",...}}
```

---

## Step 8. 테스트 데이터 전송

Python으로 Logstash TCP 6001에 직접 JSON 데이터 전송:

```python
import socket, json, time

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(('localhost', 6001))

test_logs = [
    {"type": "syslog", "hostname": "server-01", "message": "disk usage 85%", "@timestamp": "2026-03-22T06:00:00Z"},
    {"type": "snmp",   "hostname": "switch-01", "message": "interface down",  "@timestamp": "2026-03-22T06:01:00Z"},
    {"type": "syslog", "hostname": "server-02", "message": "CPU load high: 95%", "@timestamp": "2026-03-22T06:02:00Z"},
]

for log in test_logs:
    sock.send((json.dumps(log) + '\n').encode())
    time.sleep(0.5)

sock.close()
```

ES 인덱스 생성 확인:
```bash
curl http://localhost:9200/_cat/indices

# yellow open log-snmptrap-2026.03.22  1doc
# yellow open log-syslog-2026.03.22    2docs
```

---

## Step 9. Kibana에서 확인

1. `http://localhost:5601` 접속
2. **Analytics → Discover → Create a data view**
3. Index pattern: `log-*` / Timestamp: `@timestamp`
4. **Save** → Discover에서 데이터 확인

3개 문서 모두 표시됨 ✅

---

## 삽질 요약

| 문제 | 원인 | 해결 |
|---|---|---|
| `ErrImagePull` | ES/Logstash 6.x arm64 미지원 | 8.12.0으로 버전 업 |
| minikube 응답 불능 | 메모리 부족 (VM이 7GB까지 팽창) | 6144MB로 재시작 |
| Logstash CrashLoop | Docker Hub 이미지 arm64 미지원 | Elastic 공식 레지스트리 사용 |
| `minikube service` 안 됨 | QEMU builtin network 제한 | port-forward로 대체 |

---

## 다음 편

6.x vs 8.x 버전 차이 정리 + 현업 ELK와 구조 비교
