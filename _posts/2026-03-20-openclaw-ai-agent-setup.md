---
title: "AI 에이전트를 집사로 쓰는 법 — OpenClaw + Claude 세팅기"
date: 2026-03-20 18:00:00 +0900
categories: [테크 탐험]
tags: [ai, openclaw, claude, 자동화, llm, 텔레그램, n8n, api]
---

## 시작은 단순한 호기심이었어

"AI가 내 대신 블로그 포스팅 해주면 편하지 않을까?"

그냥 그 생각 하나로 시작했는데, 어느 순간 AI가 n8n 워크플로우 직접 수정하고, 블로그 배포하고, 방화벽 룰까지 만지고 있었다.

이게 어떻게 된 건지 기술적으로 정리해봤다.

---

## OpenClaw가 뭔데

간단히 말하면 **AI 에이전트 플랫폼**이다. Claude나 GPT 같은 LLM 위에서 실제로 "일을 시킬 수 있는" 레이어를 추가해준다.

일반 ChatGPT 쓰는 것과 결정적인 차이:

| | ChatGPT | OpenClaw |
|---|---|---|
| 파일 접근 | ❌ | ✅ 읽기/쓰기/수정 |
| 기억 | ❌ 세션 끊기면 리셋 | ✅ 파일 기반 영구 메모리 |
| 도구 | ❌ | ✅ 쉘, API, 웹 검색 |
| 채널 | 웹 UI만 | 텔레그램, Signal, Discord |
| 스케줄 | ❌ | ✅ cron/heartbeat |

쉽게 말하면, "n8n 워크플로우 가져다가 장 시간에만 돌게 고쳐줘" 하면 진짜로 API 호출해서 수정해버린다.

---

## 전체 아키텍처

내가 구성한 스택 전체 그림:

```
[텔레그램 / 폰]
      │
      ▼
[OpenClaw Gateway] ← 맥북 M2에서 데몬으로 실행
      │
      ├── Claude Sonnet 4.6 (Anthropic API)
      │       └── Tool Use 기능으로 도구 호출
      │
      ├── 로컬 도구들
      │       ├── exec (zsh 쉘 실행)
      │       ├── read / write / edit (파일 시스템)
      │       ├── web_fetch / web_search
      │       └── memory_search / memory_get
      │
      └── 외부 연동
              ├── GitHub CLI (gh) → 블로그 배포
              ├── n8n API (192.168.123.102:5678) → 워크플로우 제어
              └── Alpaca API → 트레이딩봇 (예정)

[윈도우 데스크탑 / 같은 네트워크]
      └── Docker n8n
              ├── Trading Bot (Alpaca Paper Trading)
              └── Miles Claude Bot (텔레그램 봇)
```

OpenClaw는 macOS 서비스로 등록되어 부팅 시 자동 시작된다. 텔레그램 메시지가 오면 → Gateway가 수신 → Claude에게 컨텍스트와 함께 전달 → Claude가 도구 호출 판단 → 실행 → 결과 반환하는 흐름이다.

---

## 핵심: Tool Use 기반 에이전트 루프

OpenClaw의 실제 동작 원리는 Anthropic의 **Tool Use API**다.

Claude한테 단순히 텍스트 답변만 받는 게 아니라, "이런 도구들이 있고 필요하면 써도 돼" 하고 도구 목록을 넘겨준다. Claude가 판단해서 도구를 호출하면, 그 결과를 다시 Claude에게 넣어주는 루프 구조다.

```
사용자 메시지
    ↓
Claude 판단: "이걸 하려면 exec 도구가 필요하겠다"
    ↓
도구 호출: exec("curl -sv https://blog.rootshift.dev")
    ↓
결과 반환: SSL 인증서 정보, HTTP 상태코드 등
    ↓
Claude가 결과 해석 → 사용자에게 답변
```

실제로 오늘 SSL 상태 확인할 때 이 흐름이 돌아간 거다. 내가 "SSL 확인해줘" 했더니 Claude가 `curl -sv` 명령어를 직접 실행하고, 출력 결과에서 인증서 subject, 만료일, SAN 매칭 여부를 파싱해서 분석해줬다.

---

## 레이어드 컨텍스트 시스템

일반 ChatGPT랑 가장 다른 점이 이거다.

세션이 시작될 때마다 에이전트가 워크스페이스 파일들을 읽어온다:

```
~/.openclaw/workspace/
├── SOUL.md        # 성격, 말투, 가치관 정의
├── USER.md        # Miles에 대한 정보 (이름, 타임존, 언어 선호)
├── IDENTITY.md    # 에이전트 정체성 (이름: 밀리, 이모지: 🐶)
├── MEMORY.md      # 장기 기억 (프로젝트 현황, 중요 결정 사항)
├── AGENTS.md      # 행동 지침 (메모리 관리 방법, 그룹챗 규칙 등)
├── TOOLS.md       # 로컬 환경 정보 (SSH 호스트, 카메라 이름 등)
└── memory/
    ├── 2026-03-19.md   # 어제 있었던 일
    └── 2026-03-20.md   # 오늘 있었던 일
```

덕분에 세션이 새로 시작돼도:
- 블로그 도메인이 `blog.rootshift.dev`인 거 알고
- n8n이 `192.168.123.102:5678`에 있는 거 알고
- 내가 한국어 선호하는 거 알고
- 이전에 뭘 했는지 흐름을 파악하고 있다

일반 ChatGPT한테 매번 "나는 인프라 엔지니어고, 블로그는 Jekyll이고, GitHub 계정은..." 설명하던 거랑 비교하면 차원이 다르다.

---

## SOUL.md — AI에게 성격을 주는 방법

이게 좀 흥미로운 개념인데, `SOUL.md` 파일로 AI의 말투와 성격을 정의한다.

내 설정 일부:

```markdown
# SOUL.md
Be genuinely helpful, not performatively helpful.
Skip the "Great question!" — just help.
Have opinions. You're allowed to disagree.
Be resourceful before asking.
```

그래서 밀리가 "Great question!" 같은 쓸데없는 말 안 하고 바로 일 처리하는 거다. 프롬프트 엔지니어링을 파일로 관리하는 셈이다.

`IDENTITY.md`로는 이름이랑 이모지까지 정의할 수 있다:

```markdown
- 이름: 밀리 (Milli)
- Emoji: 🐶
- Language Default: 한국어
```

---

## 실제로 오늘 한 작업들

오늘 하루 밀리가 직접 처리한 것들:

**① SSL 인증서 상태 확인**
```bash
curl -sv https://blog.rootshift.dev
# → subjectAltName does not match 확인
# → DNS Check in Progress 상태 파악
```

**② n8n 접근 권한 확보**
- 윈도우 방화벽 룰 추가 명령어 생성
- `curl http://192.168.123.102:5678` 접근 테스트
- n8n API Key로 워크플로우 목록 조회

**③ Trading Bot 스케줄 수정**
```python
# n8n API PUT으로 워크플로우 업데이트
# Schedule Trigger 파라미터 변경:
# 기존: */15 * * * *  (24시간)
# 변경: */15 22,23,0,1,2,3,4 * * 1-5  (미국 장 시간만)
```

**④ 블로그 세팅**
```bash
# _tabs/about.md 생성
# index.html 추가 (홈 레이아웃)
# _config.yml 수정 (description, avatar, github 링크)
git add -A && git commit -m "feat: ..." && git push
```

이걸 내가 직접 다 했으면 솔직히 귀찮아서 반은 미뤘을 거다.

---

## 텔레그램 채널이 핵심인 이유

폰에서 텔레그램으로 메시지 보내면 맥북에서 실행된다. 이게 생각보다 강력하다.

야근 중에 "n8n 트레이딩봇 장외 시간에 왜 돌아?" 생각나면 텔레그램으로 물어보면 된다. 밀리가 API 호출해서 확인하고, 문제 있으면 직접 수정한다. 나는 그냥 텔레그램만 보면 된다.

이미지도 된다. 스크린샷 찍어서 보내면 내용 파악하고 답해준다. 오늘 Cloudflare DNS 설정 확인할 때 이미지 보내줬더니 "회색 구름 맞아, DNS only 설정 정상" 이라고 바로 답해줬다.

---

## 어디까지 할 수 있냐

직접 써본 것들:

**시스템 레벨**
- 맥북 로컬 쉘 명령어 실행 (zsh)
- 파일 읽기/쓰기/수정/삭제
- 프로세스 관리 (백그라운드 실행, 상태 확인)

**네트워크/API**
- HTTP 요청 (curl 수준)
- REST API 호출 (n8n API, Alpaca API, GitHub API)
- 웹 페이지 fetch 및 파싱

**개발 도구**
- Git (add, commit, push, log, status)
- GitHub CLI (gh run list, gh run watch)
- Docker 명령어 (원격으로도 가능)

**분석**
- 이미지 분석 (스크린샷, 사진)
- 웹 검색 결과 요약
- 로그 파일 분석

**메모리**
- 중요 정보 파일에 저장
- 다음 세션에서 recall
- 날짜별 작업 기록

못 하는 것도 있다. 트리거 없이 스스로 깨어나진 못한다. 주기적 작업은 n8n이나 cron이 호출해줘야 한다. 브라우저를 직접 조작하거나 GUI 앱 제어도 안 된다.

---

## Rate Limit 이슈

솔직히 오늘 한 가지 문제가 있었다.

n8n Trading Bot이 Claude API를 15분마다 호출하는데, 내가 텔레그램으로 대화하는 타이밍이 겹치면 rate limit 에러가 났다.

Anthropic API는 Tier별로 분당 요청 한도가 있다:

| Tier | 조건 | RPM (claude-sonnet) |
|---|---|---|
| Tier 1 | 최초 | 50 |
| Tier 2 | $40 결제 후 7일 | 1,000 |
| Tier 3 | $500 사용 | 2,000 |

$40 충전하고 Tier 2로 올리면 해결된다. n8n 호출이랑 대화가 겹쳐도 여유 있게 처리된다.

---

## 써보니까

**좋은 점**
- 폰으로 텔레그램 메시지 하나로 실제 작업이 실행됨
- 컨텍스트 기억해서 매번 설명 안 해도 됨
- 자연어로 복잡한 작업 위임 가능 ("장 시간에만 돌게 고쳐줘")

**아쉬운 점**
- API 비용 발생 (하루 적극적으로 쓰면 $1~3 수준)
- 스스로 트리거 못 함 (n8n/cron 필요)
- 가끔 rate limit (Tier 올리면 해결)

**결론:** 인프라 엔지니어 입장에서 "자연어로 서버 작업 위임"이 가능해진다는 게 생각보다 실용적이다. 귀찮음을 비용으로 해결하는 건데, 이 정도면 충분히 값어치 한다.

다음엔 Alpaca Live 계좌 승인나면 실제 주식 자동매매 연동 후기도 써볼 예정이다.

---

*이 포스팅도 밀리(AI)랑 같이 썼다. 초안 잡고 기술적인 내용 채우는 거 도움받음.*
