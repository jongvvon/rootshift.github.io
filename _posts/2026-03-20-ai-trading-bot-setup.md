---
title: "AI가 내 돈을 굴린다 — 자동 트레이딩봇 구축기"
date: 2026-03-20 23:30:00 +0900
categories: [테크 탐험]
tags: [trading, n8n, alpaca, claude, ai, 자동화, docker, telegram]
---

## 시작은 이런 질문이었다

"AI가 주식을 판단하면 어떨까?"

단순한 호기심이었다. 근데 하다 보니 어느 순간 8개 서비스가 유기적으로 연결되어 실제로 페이퍼 트레이딩을 돌리고 있었다.

이 글은 그 구축 과정과, AI에게 투자 판단을 맡기면 어떻게 되는지에 대한 기록이다.

---

## 전체 아키텍처

```
NewsAPI (뉴스 30개)
    │
    ▼
Alpaca API ──── 계좌/포지션/SPY가격
    │
    ▼
n8n (Docker, 15분 스케줄)
    │
    ▼
Claude (Anthropic) ← OpenClaw 통해 판단 위임
    │
    ├── BUY → Alpaca 주문 (bracket order + stop-loss)
    └── HOLD → 패스
    │
    ▼
Telegram 보고
    │
    ▼
Cloudflare → blog.rootshift.dev 결과 업데이트
```

총 연계 서비스: **n8n, Docker, Alpaca, NewsAPI, Anthropic(Claude), Telegram, Cloudflare, OpenClaw**

---

## 각 레이어 역할

### 1. n8n (Docker) — 자동화 엔진

윈도우 데스크탑에서 Docker로 실행 중인 n8n이 전체 오케스트레이션을 담당한다.

```bash
docker run -d \
  --name n8n \
  --restart always \
  -p 5678:5678 \
  -e N8N_HOST="n8n.example.com" \
  -e N8N_PROTOCOL=https \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

15분마다 스케줄이 트리거되면 아래 순서로 실행된다:

```
계좌 조회 → 포지션 조회 → SPY 현재가 → 뉴스 수집 → Claude 판단 → 주문/보고
```

스케줄은 미국 장 시간(KST 22:00~05:00, 월~금)에만 동작하도록 제한했다:

```
*/15 22,23,0,1,2,3,4 * * 1-5
```

### 2. Alpaca — 브로커 API

페이퍼 트레이딩 계좌로 실제 시장 환경에서 가상 매매를 한다. REST API로 주문/포지션/계좌 정보를 주고받는다.

손절 자동화를 위해 **bracket order**를 사용한다. 매수와 동시에 stop-loss가 설정된다:

```json
{
  "symbol": "AAPL",
  "qty": "3",
  "side": "buy",
  "type": "market",
  "time_in_force": "day",
  "order_class": "bracket",
  "stop_loss": {
    "stop_price": "190.00"
  }
}
```

진입가 대비 **-5%** 에 stop-loss가 걸린다. 손절은 자동이다.

### 3. NewsAPI — 정보 수집

미국 주식 시장 관련 뉴스 30개를 실시간으로 수집한다.

```
q=stock+market+OR+S&P500+OR+nasdaq+OR+trading
language=en&sortBy=publishedAt&pageSize=30
```

### 4. Claude (Anthropic) — 판단 주체

여기가 핵심이다. Claude한테 단순히 뉴스 감성 분석을 시키는 게 아니라, **전체 컨텍스트를 넘기고 종합 판단**을 맡긴다.

Claude에게 넘기는 정보:

```
=== 계좌 현황 ===
총 자산: $100,236
현금: $88,807
총 손익: +$236 (+0.24%)
일일 손실 한도 도달: 아니오

=== 현재 포지션 ===
MU 26주(long) | 매입$437.50 현재$435.01 | 손익$-64.85(-0.57%)

=== 시장 지표 ===
SPY 현재가: $652.13

=== 최근 뉴스 (30개) ===
- 중동 긴장 고조로 유가 급등...
- 나스닥 1% 하락...
(30개)

=== 투자 규칙 ===
- 1회 최대 투자: 총 자산의 5%
- 손절선: -5% (bracket order)
- 일일 손실 한도: -20%
- 숏 포지션 금지
- confidence 70% 미만이면 HOLD
```

Claude가 반환하는 형식:

```json
{
  "signal": "HOLD",
  "ticker": "MU",
  "confidence": 42,
  "reason": "중동 긴장 고조로 유가 $119 급등, 인플레이션 재점화 우려. 나스닥 1% 하락 등 기술주 하방 압력 강함.",
  "market_sentiment": "부정",
  "risk_note": "지정학적 불확실성 해소 전 추가 매수 부적절"
}
```

### 5. OpenClaw — AI 레이어 통합

Claude를 단순 API 호출로 쓰는 게 아니라, OpenClaw를 통해 더 풍부한 컨텍스트와 판단 능력을 활용한다. 워크플로우 디버깅, 로직 수정, 포지션 분석까지 실시간으로 개입할 수 있다.

오늘도 숏 포지션이 잘못 잡혔을 때 즉시 청산하고, 워크플로우를 전면 개편했다. 모두 텔레그램으로 지시하고 맥북에서 실행됐다.

### 6. Telegram — 인터페이스

15분마다 판단 결과가 텔레그램으로 온다:

```
🤖 밀리 트레이딩 판단

📊 시장: 부정
⏸ 신호: HOLD
🎯 신뢰도: 42%

📝 판단 근거:
중동 긴장 고조로 유가 $119 급등...

━━━━━━━━━━━━━━━
💼 계좌 현황
총 자산: $100,116
총 손익: +$116 (+0.12%)

📈 보유 포지션
MU 26주 | 매입$437.50 현재$435.01 | 손익$-64.85
```

---

## 투자 규칙 설계

감정 없는 AI도 규칙이 없으면 위험하다. 아래 파라미터를 명시적으로 시스템 프롬프트에 넣었다.

| 규칙 | 초기값 | 현재값 |
|---|---|---|
| 원금 | $1,000 (페이퍼 $100,000) | 동일 |
| 1회 최대 투자 | 총 자산의 5% | **12%** |
| 투자 목표 비중 | 60% | **70~80%** |
| 손절선 | -5% | **-4%** |
| 일일 손실 한도 | -20% | 동일 |
| 숏 포지션 | 금지 | **허용** |
| 최소 신뢰도 | 70% | **40%** |
| 주문 방식 | bracket order | **market order** |

> **2026-03-27 업데이트:** 일주일 운영 결과 투자금의 10%도 활용 못 하는 문제가 있었다. confidence 기준을 70%→40%로 낮추고, SELL(숏) 신호를 추가했다. bracket order가 take_profit 누락으로 계속 실패해서 단순 market order로 전환했다.

초반에 숏 포지션이 잡혀버리는 문제가 있었다. SELL 신호를 받았을 때 보유 포지션 없이 숏을 쳐버린 것. 규칙에 "숏 금지"를 명시하고 로직도 수정해서 해결했다.

---

## 실제 첫 판단 결과

**2026-03-20 23:19 (KST)**

```
시장: 부정
신호: HOLD (confidence 42%)
근거: 중동 긴장 고조, 유가 급등, 나스닥 하락
```

42%는 기준치 70%에 한참 못 미쳐서 HOLD. 당연한 판단이다.

---

## 앞으로

이 섹션은 매일 미국 장 마감(KST 05:00~06:00) 후 업데이트 예정이다.

---

## 📊 트레이딩 결과 로그

### 2026년 3월

| 날짜 | 시작 자산 | 종료 자산 | 손익 | 거래 종목 | 비고 |
|---|---|---|---|---|---|
| 03-20 | $100,000 | $99,766 | -$234 (-0.23%) | SPY 숏커버, QQQ 숏커버, MU 26주 매수 | 페이퍼 시작. SPY/QQQ 숏 소폭 수익, MU 미실현손실 보유중 |
| 03-23 | $99,803 | $99,299 | -$504 (-0.50%) | MU 26주 청산(매도), QQQ 8주 숏, NVDA 28주 숏, XOM 31주 매수 | MU 포지션 전량 청산. NVDA/QQQ 신규 숏, XOM 신규 롱 진입 |
| 03-24 | $99,302 | $99,366 | +$64 (+0.06%) | 없음 | 포지션 유지. XOM 롱 이익 확대(+$118.52), NVDA/QQQ 숏 소폭 손실(-$57.12) |
| 03-25 | $99,478 | $99,290 | -$189 (-0.19%) | 없음 | 포지션 유지. XOM 롱 하락(-$72.85), NVDA 숏 손실 확대(-$91.28), QQQ 숏 보합 |
| 03-26 | $99,285 | $99,100 | -$185 (-0.19%) | QQQ 숏 7회 체결(avg $575.76), USO 매수 4회 체결(avg $118.43) | confidence 기준 40%로 하향, SELL 신호 추가로 매매 대폭 활성화. QQQ 숏 126주, USO 롱 289주, XOM 롱 31주 보유. 주문 오류(bracket order) 수정 후 market 주문으로 전환 |
| 03-27 | $99,568 | $105,182 | +$5,614 (+5.64%) | QQQ 숏 10회 체결(avg $568.0, 150주 추가) | QQQ 숏 276주(미실현 +$3,157), USO 롱 289주(미실현 +$2,158), NVDA 숏 28주(미실현 +$235), XOM 롱 31주(미실현 +$328) 보유. 전략 전환 후 첫 큰 수익일 |

---

## 📈 자산 추이

<canvas id="equityChart" style="max-height:300px"></canvas>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
new Chart(document.getElementById('equityChart'), {
  type: 'line',
  data: {
    labels: ['03-20', '03-23', '03-24', '03-25', '03-26', '03-27'],
    datasets: [{
      label: '총 자산 ($)',
      data: [99766, 99299, 99366, 99290, 99100, 105182],
      borderColor: '#4CAF50',
      backgroundColor: 'rgba(76,175,80,0.08)',
      borderWidth: 2,
      pointRadius: 4,
      pointBackgroundColor: '#4CAF50',
      fill: true,
      tension: 0.3
    }]
  },
  options: {
    responsive: true,
    plugins: {
      legend: { display: false },
      tooltip: {
        callbacks: {
          label: ctx => '$' + ctx.parsed.y.toLocaleString()
        }
      }
    },
    scales: {
      y: {
        min: 98000,
        ticks: {
          callback: val => '$' + val.toLocaleString()
        }
      }
    }
  }
});
</script>

---

*페이퍼 트레이딩입니다. 실제 투자 권유가 아닙니다.*
*AI 판단에는 오류가 있을 수 있으며, 모든 투자 결정의 책임은 본인에게 있습니다.*
