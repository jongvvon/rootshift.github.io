---
title: "Claude Code 소스코드 해부 — 프로덕션 AI 에이전트는 어떻게 설계됐나"
date: 2026-04-07 11:00:00 +0900
categories: [DevOps 전환기]
tags: [claude-code, agent, llm, typescript, architecture]
---

뜯어볼 생각은 없었다. 근데 그게 눈앞에 펼쳐져 있으면 안 볼 수가 없잖아.

## 1. 배경 — 왜 이걸 들여다봤나

2026년 3월 31일, Anthropic이 npm에 Claude Code를 올렸다. 그런데 `.map` 파일을 `.gitignore`에 추가하는 걸 깜빡했다.

소스맵(`.map`)이 뭔지 모르는 분들을 위해: JavaScript를 빌드하면 압축/난독화된 `.js` 파일이 나온다. 디버깅을 위해 원본 소스 위치를 알려주는 게 `.map` 파일인데, 이 안에 `sourcesContent` 필드로 **원본 TypeScript 소스가 통째로 박혀있다**.

즉, npm 패키지 안에 소스가 다 들어있었던 거다.

법적으로는 문제없다. 공개된 패키지니까. Anthropic 입장에서는 꽤 쪽팔린 실수였겠지만, 덕분에 프로덕션 AI 에이전트의 내부를 들여다볼 기회가 생겼다.

뜯어봤더니 생각보다 훨씬 배울 게 많았다. 특히 "에이전트 설계를 어떻게 해야 하나" 고민하던 시점에 딱 맞는 레퍼런스였다.

---

## 2. 전체 아키텍처 — src/ 디렉토리 구조

```
src/
├── Tool.ts          ← 도구 추상화 핵심 타입 정의
├── Task.ts          ← 비동기 작업 관리
├── context.ts       ← 시스템 프롬프트, git 상태, 컨텍스트 구성
├── assistant/       ← Claude API 호출 루프
├── coordinator/     ← 멀티에이전트 조율
├── tools/           ← 실제 도구 구현 (Bash, FileRead, FileEdit, Grep 등)
├── permissions/     ← 권한 체크 시스템
├── bootstrap/       ← 세션 초기화
└── components/      ← React + Ink UI
```

구조 자체는 단순하다. 핵심 흐름:

```
사용자 입력
  → context.ts (시스템 프롬프트 구성 + git 상태 수집)
  → assistant/ (Claude API 호출)
  → Tool.ts (도구 선택 + 실행)
  → coordinator/ (서브에이전트 조율)
  → 결과 렌더링
```

처음 보면 "이게 다야?" 싶다. 근데 각 레이어에서 뭘 하는지 보면 얘기가 달라진다.

---

## 3. Tool.ts — 도구 추상화의 핵심

제일 먼저 봐야 할 파일이다. `Tool` 타입 정의:

```typescript
export type Tool<Input, Output, P> = {
  name: string
  call(args, context, canUseTool, parentMessage, onProgress): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  inputSchema: Input          // Zod 스키마
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>
  // ... UI 렌더링 메서드들
}
```

주목할 포인트들:

**Zod 스키마로 입력 타입 정의**
Claude가 도구를 JSON으로 호출할 때 Zod가 검증한다. 타입 안전성이 LLM ↔ 코드 경계에서 보장되는 구조.

**isReadOnly / isDestructive 분리**
읽기 전용인지, 파괴적인지를 별도로 선언한다. 단순히 "위험/안전" 이분법이 아니라 세분화된 권한 체크가 가능하다.

**isConcurrencySafe**
병렬 실행 가능 여부를 도구가 직접 선언한다. 오케스트레이터가 판단하는 게 아니라 도구 스스로가 "나 병렬로 써도 돼/안 돼"를 말하는 것.

**checkPermissions**
도구별 권한 로직이 캡슐화된다. 중앙 권한 관리자가 모든 걸 알 필요가 없다.

### buildTool() 패턴

도구를 만들 때 기본값을 어떻게 설정했는지가 인상적이다:

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,   // 기본값: 안전하지 않음 (fail-closed)
  isReadOnly: () => false,           // 기본값: 쓰기로 간주
  checkPermissions: () => Promise.resolve({ behavior: 'allow' }),
}

export function buildTool<D extends ToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, userFacingName: () => def.name, ...def }
}
```

`isConcurrencySafe`의 기본값이 `false`다. "모르면 안전하지 않다고 가정"하는 fail-closed 설계. 새 도구를 만들 때 병렬 실행 가능하다고 명시적으로 선언해야만 허용된다.

반대로 `isReadOnly` 기본값도 `false`. "모르면 쓰기로 간주". 더 엄격한 권한 체크를 유발한다.

### 실제 도구 목록

`src/tools/`에 구현된 도구들:

- `BashTool` — 셸 명령 실행
- `FileReadTool` — 파일 읽기
- `FileEditTool` — 파일 수정 (diff 기반)
- `FileWriteTool` — 파일 쓰기
- `GlobTool` — 파일 패턴 검색
- `GrepTool` — 텍스트 검색
- `AgentTool` — 서브에이전트 실행
- `LSPTool` — Language Server Protocol 연동
- `MCPTool` — MCP 프로토콜 도구
- `NotebookEditTool` — Jupyter 노트북 편집
- 그 외 10개 이상

20개가 넘는 도구가 전부 같은 `Tool` 인터페이스를 구현한다. 새 도구 추가가 기계적이 될 수밖에 없는 구조.

---

## 4. Task.ts — 비동기 작업 관리

에이전트가 멀티태스킹을 어떻게 추상화하는지 볼 수 있다:

```typescript
export type TaskType =
  | 'local_bash'          // 로컬 bash 실행
  | 'local_agent'         // 로컬 서브에이전트
  | 'remote_agent'        // 원격 에이전트
  | 'in_process_teammate' // 인프로세스 동료 에이전트
  | 'local_workflow'      // 로컬 워크플로우
  | 'monitor_mcp'         // MCP 모니터링
  | 'dream'               // (내부용)

export type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'
```

`in_process_teammate`가 흥미롭다. 같은 프로세스 안에서 동료처럼 동작하는 에이전트 타입인데, 컨텍스트를 공유할 수 있다. 반면 `local_agent`는 별도 프로세스로 격리된다.

**TaskID 설계**도 실용적이다:
- `b` 접두사 = bash 태스크
- `a` 접두사 = agent 태스크  
- `r` 접두사 = remote 태스크
- `t` 접두사 = teammate 태스크

접두사 + 랜덤 문자열로 ID를 만들어서, ID만 봐도 어떤 종류의 태스크인지 알 수 있다. 36^8 ≈ 2.8조 조합이라 충돌도 사실상 없다.

---

## 5. context.ts — 시스템 프롬프트 구성

이 파일을 보기 전까지는 "Claude가 알아서 하는 거 아닌가" 생각했다. 아니었다.

```typescript
export const getGitStatus = memoize(async (): Promise<string | null> => {
  // 동시에 branch, mainBranch, status, log, userName 가져옴
  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short']),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5']),
    execFileNoThrow(gitExe(), ['config', 'user.name']),
  ])
  // ...
})
```

주목할 포인트들:

**memoize 래핑**
`getGitStatus`가 memoize로 감싸져 있다. 같은 세션에서 git 상태를 여러 번 조회해도 실제 git 명령은 한 번만 실행된다. 프롬프트 캐시와 연동해서 토큰 낭비를 줄이는 구조.

**Promise.all로 병렬 수집**
브랜치, 메인 브랜치, 상태, 로그, 유저명을 동시에 가져온다. 순차적으로 실행하면 느리니까.

**`--no-optional-locks`**
git이 내부적으로 쓰는 lock 파일을 만들지 않고 읽기만 한다. 백그라운드에서 자주 호출해도 다른 git 명령과 충돌하지 않도록.

**CLAUDE.md 자동 주입**
프로젝트 루트의 `CLAUDE.md` 파일을 자동으로 컨텍스트에 포함한다. 이게 Claude Code에서 `CLAUDE.md`가 특별한 파일로 취급되는 이유다.

---

## 6. PermissionMode — 권한 시스템

```typescript
type PermissionMode = 'default' | 'bypass' | 'auto' | 'plan'
```

- **default**: 위험한 작업은 사용자 확인 필요. 일반적인 대화형 사용.
- **bypass**: 모든 권한 자동 허용. `--dangerously-skip-permissions` 플래그로 활성화. CI/자동화용.
- **auto**: 자동화 모드. UI 없이 동작하지만 bypass보다 덜 위험.
- **plan**: 계획 모드. 실행 없이 계획 수립만. 권한 체크가 다르게 동작.

권한 체크는 단층이 아니라 여러 레이어로 구성된다:

1. **`validateInput()`** — 입력 자체가 스키마에 맞는지
2. **`checkPermissions()`** — 도구별 커스텀 권한 로직
3. **`alwaysAllowRules / alwaysDenyRules`** — 패턴 기반 규칙 (예: 특정 파일은 항상 허용/거부)
4. **auto-classifier** — AI 기반 보안 위험도 자동 분류

단순히 "이 도구 허용?" 이 아니라, 같은 도구라도 어떤 입력이냐에 따라 다른 권한 레벨이 적용된다.

---

## 7. 멀티에이전트 구조 (coordinator/)

```
Main Agent (coordinator)
├── Architect (계획)
├── Executor (실행)
└── Reviewer (검토)
```

Claude Code의 `--multi-agent` 모드가 이 구조로 동작한다.

에이전트 타입별 격리 수준:
- **in_process_teammate**: 같은 프로세스, 컨텍스트 공유 가능
- **local_agent**: 별도 프로세스, 격리됨
- **remote_agent**: 원격 실행, 완전 격리

**서브에이전트 상태 격리** 설계가 특히 인상적이다:

```typescript
// 서브에이전트에서 setAppState 호출 시 → no-op
// setAppStateForTasks 호출 시 → 세션 수준 공유
```

서브에이전트가 부모의 UI 상태나 앱 상태를 건드리지 못한다. `setAppState`가 no-op으로 막혀있다. 반면 `setAppStateForTasks`는 공유된다. 세션 수준의 인프라(실행 중인 태스크 목록 등)만 공유하고, 나머지는 격리하는 것.

---

## 8. 내가 배운 것

직접 뜯어보면서 메모해둔 패턴들.

### 1. 도구 설계는 fail-closed가 원칙

`isConcurrencySafe: () => false` — 기본값이 "안전하지 않음". 위험할 수 있는 것은 허용을 선언해야 한다. 허용이 기본값인 시스템은 새 도구가 추가될 때마다 구멍이 생긴다.

### 2. 권한 체크는 레이어드

단순 허용/거부 Boolean이 아니다. 입력 검증 → 도구별 로직 → 패턴 규칙 → AI 분류기. 각 레이어가 독립적으로 막을 수 있다. 한 레이어가 뚫려도 다음 레이어가 잡는다.

### 3. 상태 관리는 불변

```typescript
setAppState(f: prev => nextState)
```

Redux 패턴. 상태를 직접 수정하지 않고, 변환 함수를 통해서만 변경한다. 서브에이전트에서 이게 no-op이면 — 서브에이전트가 아무리 날뛰어도 부모 상태는 건드릴 수 없다.

### 4. 컨텍스트 구성이 핵심

Claude API에 뭘 넣느냐가 품질의 90%다. `context.ts`가 하는 일이 바로 그거다. git 상태, CLAUDE.md, 관련 파일 내용을 어떻게 선별해서, 어떤 순서로 넣는지. 이게 진짜 노하우.

모델이 좋아져도 컨텍스트를 못 만들면 결과가 나쁘다. 반대로 컨텍스트를 잘 만들면 작은 모델도 잘 동작한다.

### 5. memoize를 적극 활용

같은 세션에서 반복 조회 방지. git 상태를 50번 물어봐도 실제로는 한 번만 실행. 프롬프트 캐시와 연동하면 토큰도 절약.

---

## 9. 이걸 내 프로젝트에 어떻게 적용할까

트레이딩봇 `run.py`가 지금 모놀리식으로 돼있다. 주문, 조회, 리스크 체크, 포지션 관리가 다 섞여있는 구조.

Tool 패턴으로 분리하면:

```python
class AlpacaOrderTool:
    name = "alpaca_order"
    is_read_only = False
    is_destructive = True  # 돈이 나가는 작업

    def check_permissions(self, input):
        if input['qty'] > MAX_QTY:
            return {'behavior': 'deny', 'reason': '수량 한도 초과'}
        if input['symbol'] in BLACKLIST:
            return {'behavior': 'deny', 'reason': '블랙리스트 종목'}
        return {'behavior': 'allow'}

    def validate_input(self, input):
        # Pydantic으로 타입 검증
        return OrderInput(**input)

    def call(self, args):
        # 실제 주문 로직
        validated = self.validate_input(args)
        return alpaca_client.submit_order(validated)
```

이렇게 도구를 분리하면:
- 각 도구가 자신의 권한과 검증 로직을 관리
- Claude(또는 다른 LLM)가 어떤 도구를 쓸지 선택
- 새 도구 추가가 기존 코드 수정 없이 가능
- 권한 로직이 한 곳에 모임 → 감사(audit)가 쉬움

`is_destructive = True`를 선언한 도구는 자동으로 더 엄격한 확인 절차를 거치도록 만들 수 있다. 주문 도구가 실수로 병렬 실행되는 일도 `is_concurrency_safe = False`로 막을 수 있다.

---

## 10. 마무리

Claude Code 소스를 보면서 가장 인상적이었던 건 **단순함**이었다.

복잡한 ML 파이프라인이 있을 거라 생각했는데 — 없었다. 잘 설계된 타입 시스템, 레이어드 권한 체크, 명확한 도구 추상화. 그게 전부다.

에이전트 개발이 어렵게 느껴지는 건 AI 때문이 아니다. **"Claude에게 뭘 어떻게 전달하느냐"**의 설계 문제다.

어떤 컨텍스트를 어떻게 구성할지, 도구를 어떻게 정의할지, 권한을 어떻게 레이어링할지 — 이 질문들에 대한 답이 이 코드베이스에 있다.

Anthropic이 실수로 오픈소스를 해버린 셈인데, 공부 자료로는 최고다.
