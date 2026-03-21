---
title: "자동화 스택 구축 후 뒤늦게 챙긴 보안 — n8n, 텔레그램 봇, Cloudflare"
date: 2026-03-21 10:00:00 +0900
categories: [테크 탐험]
tags: [보안, cloudflare, n8n, telegram, zero-trust, devops]
---

## 먼저 구축하고, 보안은 나중에?

자동화 스택을 빠르게 올리다 보면 보안이 뒤로 밀린다.

n8n 도커 올리고, 텔레그램 봇 연결하고, 도메인 붙이고 — 다 됐다 싶었는데 돌아보니 구멍이 몇 개 있었다.

- `n8n.rootshift.dev` → 인터넷에 로그인 페이지가 그냥 열려있음
- 텔레그램 봇 → 아무나 메시지 보낼 수 있음
- 블로그 글에 내부 도메인 노출

데브옵스/SRE 전환을 준비하는 입장에서 보안은 결국 기본기다. 뒤늦게라도 챙겼다.

---

## 문제 1: n8n 로그인 페이지 외부 노출

### 상황

n8n을 `n8n.rootshift.dev` 도메인으로 외부에서 접근 가능하게 설정해뒀다. 문제는 n8n 로그인 페이지가 그냥 인터넷에 열려있다는 것.

```bash
curl -o /dev/null -w "%{http_code}" https://n8n.rootshift.dev
# 200 → 누구나 접근 가능
```

브루트포스 공격 대상이 될 수 있고, n8n에 취약점이 있으면 악용 가능성도 있다.

### 해결: Cloudflare Zero Trust Access

Cloudflare Zero Trust를 이용해 n8n 앞에 인증 레이어를 추가했다.

**설정 방법:**

1. Cloudflare 대시보드 → **Zero Trust**
2. **Access** → **Applications** → **Add an application**
3. **Self-hosted** 선택
4. Application domain: `n8n.rootshift.dev`
5. Policy 추가:
   - Action: `Allow`
   - Selector: `Emails`
   - Value: 허용할 이메일 주소

이제 `n8n.rootshift.dev` 접속 시 Cloudflare 인증 페이지가 먼저 뜬다. 등록된 이메일로 구글 로그인해야만 n8n에 접근할 수 있다.

```
외부 접속
    ↓
Cloudflare Access (이메일 인증)
    ↓ 인증 성공
n8n 대시보드
```

**왜 IP 제한 대신 이메일 인증인가:**
- IP 제한: 폰 LTE 사용 시 IP가 바뀌어 매번 막힘
- 이메일 인증: 어느 기기, 어느 IP에서든 이메일만 맞으면 통과. 24시간 세션

**중요:** Cloudflare Access는 도메인 기반이라 로컬 IP 직접 호출은 영향 없다.

```bash
# 이건 Cloudflare 거치지 않아서 그대로 동작
curl http://192.168.1.100:5678/api/v1/workflows
```

---

## 문제 2: 텔레그램 봇 무단 접근

### 상황

텔레그램 봇 토큰만 알면 누구든 봇에 메시지를 보낼 수 있다. n8n 워크플로우가 트리거되고, Claude API가 호출된다. 비용도 나가고, 의도치 않은 동작도 생길 수 있다.

내 자동화 스택에는 봇이 두 개 있었다:
- **Claude 연동 봇** — 텔레그램 메시지로 Claude API 트리거
- **트레이딩 알림 봇** — 자동 매매 결과 수신

둘 다 나만 써야 하는데 열려있었다.

### 해결: chat_id 화이트리스트

텔레그램에서 사용자 고유 ID(`chat_id`)는 변하지 않는다. n8n 워크플로우 맨 앞에 chat_id 체크 노드를 추가했다.

**n8n 구성:**

```
Telegram Trigger
    ↓
Is Miles? (If 노드)
├── True (chat_id === 내 ID) → 워크플로우 실행
└── False (다른 사람) → 종료, 아무것도 안 함
```

**If 노드 조건:**

```
leftValue:  {{ $json.message.chat.id }}
operator:   equals
rightValue: [내 chat_id]
```

이렇게 하면 내 chat_id가 아닌 메시지는 워크플로우 실행 자체가 안 된다. Claude API 호출도 없고, 응답도 없다.

---

## 문제 3: 블로그에 내부 도메인 노출

자동화 구축기를 블로그에 올리면서 실제 n8n 도메인을 그대로 썼다.

```bash
docker run -e N8N_HOST="n8n.rootshift.dev" ...
```

도메인이 노출되면 공격 대상이 특정된다. `n8n.example.com` 같은 플레이스홀더로 교체했다.

```bash
docker run -e N8N_HOST="n8n.example.com" ...
```

블로그에 인프라 관련 글 쓸 때 주의할 것들:
- 실제 도메인/IP → 플레이스홀더로
- API Key, 토큰 → 절대 노출 금지
- 시스템 구조가 특정되는 정보 → 추상화

---

## 정리

| 문제 | 해결 방법 | 도구 |
|---|---|---|
| n8n 외부 노출 | 인증 레이어 추가 | Cloudflare Zero Trust |
| 봇 무단 접근 | chat_id 화이트리스트 | n8n If 노드 |
| 도메인 노출 | 플레이스홀더 교체 | 수동 검토 |

---

## 교훈

빠르게 구축하는 것도 중요하지만, 외부에 노출되는 엔드포인트는 반드시 인증 레이어가 필요하다.

특히 자동화 스택은:
1. **API Key** → 환경변수로 관리, 코드/블로그에 노출 금지
2. **외부 접근 포인트** → Zero Trust 또는 VPN
3. **봇/웹훅** → 발신자 검증 필수

DevOps/SRE 관점에서 보안은 기능 구현 후 추가하는 게 아니라, 처음부터 설계에 포함되어야 한다. 이번엔 뒤늦게 챙겼지만, 다음 프로젝트에서는 처음부터.

---

*이 글은 실제 구축 과정에서 발견한 보안 이슈와 해결 방법을 기록한 것입니다.*
