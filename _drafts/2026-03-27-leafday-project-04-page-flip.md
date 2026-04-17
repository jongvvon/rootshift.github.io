---
title: "Leafday 외전 #1 — 책 페이지 넘기기 애니메이션 13번의 시도"
date: 2026-03-27 16:00:00 +0900
categories: [사이드 프로젝트]
tags: [react-native, animation, leafday, ios, expo, eas, flipbook]
---

> **Leafday 시리즈 외전 #1**  
> 이 글은 완성된 구현이 아닙니다. 아직 작업 중인 기능의 삽질 기록입니다.  
> 최종 해결까지의 과정을 시리즈로 이어갑니다.

---

## 목표

Leafday 앱에서 책 페이지를 넘기는 느낌을 구현하고 싶었다.  
단순히 좌우로 슬라이드하는 게 아니라, **오른쪽 끝을 축으로 접히는** 실제 책 넘기기 느낌.  
그리고 세게 스와이프하면 여러 장이 `촤르르륵` 빠르게 넘어가는 flipbook 효과.

---

## 삽질 과정 요약

### 시도 1: `pagingEnabled` ScrollView

```tsx
<ScrollView horizontal pagingEnabled decelerationRate="fast">
  {pages.map(...)}
</ScrollView>
```

결과: 카드가 옆에 나란히 배치되는 구조라 페이지 넘기기 느낌이 전혀 안 남.  
매끄럽게 슬라이드만 될 뿐.

---

### 시도 2: `snapToInterval` + `decelerationRate="normal"`

```tsx
<Animated.ScrollView
  snapToInterval={W}
  decelerationRate="normal"
  onScroll={Animated.event([{ nativeEvent: { contentOffset: { x: scrollX } } }], ...)}
>
```

`scrollX` 값으로 각 페이지의 `rotateY`를 interpolate해서 입체감을 줬다.

```tsx
const rotateY = scrollX.interpolate({
  inputRange: [(index-1)*W, index*W, (index+1)*W],
  outputRange: ['80deg', '0deg', '-80deg'],
});
```

결과: 가속도는 생겼지만 여전히 카드 슬라이드 느낌. 회전 축이 중앙이라 책처럼 안 보임.

---

### 시도 3: 오른쪽 끝 축 회전 (translateX trick)

CSS에서 `transform-origin: 100%`를 React Native에서 구현하는 방법:

```tsx
transform: [
  { perspective: 900 },
  { translateX: W / 2 },     // 오른쪽 끝으로 이동
  { rotateY: '80deg' },
  { translateX: -(W / 2) },  // 원위치
]
```

이렇게 하면 오른쪽 끝이 고정 축이 되어 실제 책 넘기듯 접힌다.

결과: 방향은 맞는데 ScrollView 기반이라 z-index 관리가 안 됨. 페이지가 겹쳐서 이상하게 보임.

---

### 시도 4: 스택 구조 (페이지를 absolute로 쌓기)

```tsx
{pages.map((page, index) => (
  <Animated.View
    key={page.id}
    style={[styles.pageStack, { zIndex: zIndexAnim }]}
  >
    ...
  </Animated.View>
))}
```

`Animated.Value`로 zIndex를 애니메이션하려 했지만 React Native에서 z-index는 Animated로 제어가 안 됨.  
현재 페이지가 다른 페이지들 뒤에 숨어버림.

결과: 애니메이션이 아예 보이지 않음 💀

---

### 최종 해결: 페이지 하나만 렌더 + 2단계 flip

핵심 아이디어: **화면에는 현재 페이지 하나만 표시하고**, 스와이프 시 그 페이지가 접혔다가 새 페이지가 펼쳐지는 2단계 애니메이션을 구현한다.

```tsx
const flipOnePage = (direction: 'next' | 'prev') => {
  return new Promise((resolve) => {
    const sign = direction === 'next' ? 1 : -1;

    // Phase 1: 현재 페이지 접힘 (0 → sign)
    slideAnim.setValue(0);
    Animated.timing(slideAnim, {
      toValue: sign,
      duration: FLIP_DURATION, // 200ms
      useNativeDriver: true,
    }).start(() => {

      // 인덱스 변경
      currentIndexRef.current = newIdx;
      slideAnim.setValue(-sign); // 반대편에서 시작
      setCurrentIndex(newIdx);

      // Phase 2: 새 페이지 펼쳐짐 (-sign → 0)
      Animated.timing(slideAnim, {
        toValue: 0,
        duration: FLIP_DURATION,
        useNativeDriver: true,
      }).start(() => resolve(true));
    });
  });
};
```

rotateY 계산:

```tsx
const rotateY = slideAnim.interpolate({
  inputRange: [-1, 0, 1],
  outputRange: ['60deg', '0deg', '-60deg'],
});

// 오른쪽 끝 축
const translateX = slideAnim.interpolate({
  inputRange: [-1, 0, 1],
  outputRange: [W * 0.4, 0, -W * 0.4],
});
```

실제 transform에 적용:

```tsx
transform: [
  { perspective: 900 },
  { translateX },
  { rotateY },
  { translateX: Animated.multiply(translateX, -1) },
]
```

---

### flipbook 효과 (촤르륵)

스와이프 속도(`vx`)를 감지해서 여러 장을 연속으로 뒤집는다.

```tsx
PanResponder.create({
  onPanResponderRelease: (_, g) => {
    const speed = Math.abs(g.vx);
    // 속도 비례 페이지 수 계산
    const count = Math.max(1, Math.min(20, Math.round(speed * 7 + 1)));
    flipPages(g.dx < 0 ? 'next' : 'prev', count);
  },
});
```

`flipPages`는 `flipOnePage`를 count만큼 순차 호출:

```tsx
const flipPages = async (direction, count) => {
  for (let i = 0; i < count; i++) {
    const ok = await flipOnePage(direction);
    if (!ok) break; // 끝에 도달하면 중단
  }
};
```

Promise 체이닝으로 각 flip이 완료된 후 다음 flip이 시작되므로 자연스러운 연속 동작이 나온다.  
살살 스와이프 → 1장, 세게 스와이프 → 20장까지 한 번에.

---

## 최종 구조

```
BookViewScreen
├── PanResponder (스와이프 감지)
├── slideAnim (Animated.Value: 페이지 전환)
├── Animated.View (오른쪽 끝 축 rotateY)
│   ├── 앞면 (사진)
│   └── 뒷면 (노트) ← 탭으로 flip
└── 하단 버튼 (이전/다음)
```

---

## 결론

React Native Animated로 책 페이지 넘기기를 구현할 때 핵심은:

1. **ScrollView 기반은 포기** — z-index 제어가 안 되고 스택 구조에 안 맞음
2. **한 번에 하나의 페이지만 렌더** — 복잡한 z-index 관리 불필요
3. **translateX trick으로 오른쪽 축** — `transform-origin: 100%` 없이 구현
4. **Promise 체이닝으로 연속 flip** — async/await로 순차 실행

총 빌드 횟수: 21번 (...삽질의 역사)

---

## 현재 상태

현재 빌드 21 기준으로 2단계 flip 애니메이션이 구현되어 있지만, 아직 원하는 느낌은 아니다.  
진짜 책처럼 넘어가는 느낌을 위해 계속 작업 중.

다음 외전에서 최종 구현을 다룰 예정.

---

*→ 다음: Leafday 외전 #2 — 최종 page flip 구현 (작업 중)*  
*← 이전: [Leafday #3 — TestFlight 배포까지](/posts/leafday-project-03-testflight)*
