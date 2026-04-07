---
title: "트레이딩봇 개선 — 뉴스는 이미 늦다, 테마로 먼저 잡아라"
date: 2026-04-07 20:49:00 +0900
categories: [테크 탐험]
tags: [trading-bot, python, alpaca, claude, ai, macro, theme-detection]
---

## 발단: 봇이 항상 같은 종목만 산다

운영하다 보니 패턴이 보이기 시작했다.

글로벌 이슈가 터져도 봇은 SPY, QQQ, NVDA, AAPL, MSFT, XOM, USO 이 7개만 들여다봤다. 고정 워치리스트였다. 그러다 보니 장이 어떻게 흘러도 거래 종목이 항상 비슷했고, 수익률도 그냥 평범했다.

예를 들어 중동 긴장이 고조되면 에너지주, 방산주, 금이 움직인다. 근데 봇은 그 뉴스를 받아도 연결을 못 했다. LMT, RTX, GLD 같은 종목은 아예 눈에 없었으니까.

---

## 문제 구조 분석

크게 두 가지 문제였다.

**① 종목 직접 언급 의존**

기존 뉴스 수집은 이런 식이었다:

```
?symbols=SPY,QQQ,NVDA,AAPL,MSFT,XOM,USO
```

종목 필터를 걸어두면 해당 종목이 직접 언급된 뉴스만 온다. "이란 제재 강화"라는 기사가 방산주 호재여도, LMT나 RTX 언급이 없으면 봇에게 오지 않는다.

**② 매크로 컨텍스트 미주입**

Claude에게 넘기는 프롬프트에 "현재 이런 글로벌 이슈가 있다"는 컨텍스트가 없었다. 뉴스 텍스트 나열에서 LLM이 알아서 연결해야 했는데, 정보 자체가 잘려 있으니 판단이 좁을 수밖에 없었다.

---

## 해결 방향

두 가지를 동시에 고쳤다.

### 1. 매크로 뉴스 별도 수집

종목 필터 없는 광범위한 뉴스를 추가로 수집한다.

```python
# 종목 특화 뉴스 (기존)
symbol_news = alpaca_get(
    "https://data.alpaca.markets/v1beta1/news"
    "?symbols=SPY,QQQ,NVDA,AAPL,MSFT,XOM,USO,LMT,RTX,GLD,TLT&limit=30&sort=desc"
)
# 매크로 뉴스 (신규) — 종목 필터 없음
macro_news = alpaca_get(
    "https://data.alpaca.markets/v1beta1/news"
    "?limit=20&sort=desc"
)
```

두 결과를 합치고 중복 제거해서 최근 30개 헤드라인을 완성한다.

### 2. 테마 감지 + 동적 워치리스트

헤드라인 전체를 텍스트로 합쳐서 키워드 스캔을 돌린다.

```python
THEME_MAP = {
    "geopolitical": {
        "keywords": ["iran", "war", "military", "attack", "missile", "sanction",
                     "conflict", "escalat", "tension", "strike", "russia", "ukraine"],
        "tickers": ["LMT", "RTX", "NOC", "GD", "GLD"],
        "bias": "BEARISH_MARKET",
        "label": "지정학적 리스크",
    },
    "energy": {
        "keywords": ["oil", "opec", "crude", "gasoline", "petroleum", "pipeline"],
        "tickers": ["XOM", "CVX", "USO", "XLE", "UNG"],
        "bias": None,
        "label": "에너지",
    },
    "fed_hawkish": {
        "keywords": ["rate hike", "inflation", "cpi", "hawkish", "tighten"],
        "tickers": ["TLT", "GLD", "SLV"],
        "bias": "BEARISH_MARKET",
        "label": "긴축/인플레이션",
    },
    "fed_dovish": {
        "keywords": ["rate cut", "dovish", "easing", "pivot"],
        "tickers": ["QQQ", "NVDA", "AAPL", "MSFT"],
        "bias": "BULLISH_MARKET",
        "label": "금리 인하 기대",
    },
    # ... 테크, 경기침체 등 추가
}
```

감지된 테마에 맞는 종목이 워치리스트에 자동으로 붙는다.

```
[THEME] ['지정학적 리스크', '에너지'] | 워치리스트: ['SPY', 'QQQ', 'NVDA', 'AAPL', 'MSFT', 'XOM', 'USO', 'LMT', 'RTX', 'NOC', 'GD', 'GLD', 'CVX', 'XLE']
```

평상시 7개짜리 워치리스트가 이슈에 따라 14개, 15개로 동적으로 확장된다.

### 3. Claude 프롬프트에 테마 컨텍스트 주입

```python
theme_context = f"감지된 시장 테마: {', '.join(theme_labels)}\n"
theme_context += f"관련 종목 추가: {', '.join(extra_tickers)}\n"
if bias_notes:
    theme_context += f"시장 편향 시그널: {', '.join(set(bias_notes))}\n"
```

프롬프트에 이게 들어가면 Claude가 "이란 긴장 → 방산주 유리"라는 연결을 스스로 할 필요가 줄어든다. 이미 힌트를 줬으니까.

---

## 텔레그램 알림에도 반영

기존엔 BUY 신호가 와도 왜 그 종목인지 맥락이 없었다.

```
🟢 [TradingBot] BUY
  종목: RTX
  수량: 8
  신뢰도: 71%
  근거: 중동 긴장 고조로 방산 수요 증가, 이란 관련 리스크 헤지
  테마: 지정학적 리스크       ← 추가됨
  시장: BEARISH
  리스크: 협상 타결 시 급락 가능성
```

`테마` 필드가 붙으니까 "아 오늘 이 이슈로 이 종목 잡은 거구나"가 바로 보인다.

---

## 부가 수정: datetime 경고 제거

매번 실행마다 뜨던 경고:

```
DeprecationWarning: datetime.datetime.utcnow() is deprecated
```

```python
# Before
now = datetime.datetime.utcnow()

# After
now = datetime.datetime.now(datetime.UTC)
```

사소하지만 로그 읽을 때 거슬렸다.

---

## 한계 / 남은 과제

키워드 매칭이라 오탐이 있을 수 있다. "oil"이 들어간 뉴스가 에너지 테마로 잡히는데, 실제로는 무관한 기사일 수도 있다. 지금은 Claude가 최종 판단하니까 완충이 되지만, 테마 감지 자체를 더 정교하게 만들 여지는 있다.

또 커뮤니티 데이터(종토방 반응 같은 것)는 아직 미반영이다. 뉴스 API도 발행 시점 기준이라 시장 반응보다 항상 조금 늦다. 선행 지표를 추가하려면 옵션 플로우나 소셜 데이터가 필요한데, 그건 다음 단계로.

---

*페이퍼 트레이딩 기록입니다. 투자 권유 아닙니다.*
