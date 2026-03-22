---
title: "[Leafday #1] React Native 개발 환경 세팅 — Expo로 첫 화면까지"
date: 2026-03-22 17:08:00 +0900
categories: [사이드 프로젝트]
tags: [leafday, react-native, expo, jsx, 개발환경, 앱개발]
---

## "그냥 JS 코드 같은데 React랑 뭔 상관이야?"

첫 코드 보고 드는 생각이 딱 이거다.

```tsx
export default function App() {
  return (
    <View style={styles.container}>
      <Text>Leafday</Text>
    </View>
  );
}
```

JavaScript인데 `<View>`, `<Text>` 같은 HTML처럼 생긴 태그가 섞여있다.

이걸 **JSX**라고 한다. JavaScript + XML의 합성어. React가 만든 문법인데, JavaScript 안에서 UI를 직접 표현할 수 있게 해준다.

```jsx
// 이게 JSX
const element = <Text>안녕</Text>;

// 컴파일하면 사실 이거
const element = React.createElement(Text, null, "안녕");
```

두 번째 줄처럼 매번 쓰면 불편하니까 첫 번째처럼 쓸 수 있게 만든 것. 실제로 실행될 때는 JavaScript로 변환된다.

---

## React Native가 하는 일

```
JSX 코드 (내가 작성)
    ↓
JavaScript 엔진
    ↓
iOS: UIKit (Swift/ObjC 네이티브)
Android: Android SDK (Kotlin/Java 네이티브)
```

`<Text>` 하나 써도 iOS에서는 `UILabel`, Android에서는 `TextView`로 각각 변환된다. 그래서 코드 하나로 두 플랫폼이 동작한다.

웹의 `<div>`, `<p>` 대신 `<View>`, `<Text>`를 쓰는 이유가 여기에 있다. 모바일 네이티브 컴포넌트로 변환하기 위해서.

---

## 개발 환경 세팅

### 필요한 것들

**필수:**
- Node.js (JavaScript 실행 환경)
- npm (패키지 관리자)
- Watchman (파일 변경 감지)
- Xcode (iOS 빌드용, macOS 필수)
- CocoaPods (iOS 의존성 관리)

**확인 방법:**
```bash
node --version    # v25.8.1
npm --version     # 11.11.0
xcode-select --version
pod --version     # 1.16.2
```

**Watchman 설치:**
```bash
brew install watchman
```

---

## Expo Managed Workflow 선택 이유

React Native로 앱을 만드는 방법은 세 가지다:

```
Expo Managed  ← 이걸 선택
Expo Bare
React Native CLI
```

**Expo Managed를 고른 이유:**

1. 카메라, 사진, 알림 같은 기능이 이미 구현돼 있다
2. iOS/Android 네이티브 코드 건드릴 필요 없음
3. 나중에 Bare로 전환 가능

Leafday의 핵심이 사진 촬영인데, Managed에서 `expo-camera`만 import하면 바로 된다. 처음부터 Bare로 가면 Swift/Kotlin까지 공부해야 한다.

---

## 프로젝트 생성

```bash
npx create-expo-app@latest leafday --template blank-typescript
```

**TypeScript를 선택한 이유:**

JavaScript랑 거의 같은데, 타입이 있어서 실수를 컴파일 시점에 잡아준다.

```typescript
// JavaScript
function add(a, b) { return a + b; }
add("hello", 5); // 실행 시 이상한 결과

// TypeScript
function add(a: number, b: number): number { return a + b; }
add("hello", 5); // 컴파일 에러로 미리 잡힘
```

DevOps 관점에서도 TypeScript가 맞다. 코드 규모가 커지면 타입이 없으면 유지보수가 힘들어진다.

---

## 프로젝트 구조

```
leafday/
├── App.tsx          ← 앱 시작점, 여기서 코딩
├── assets/          ← 이미지, 폰트
├── app.json         ← 앱 이름, 아이콘 등 설정
├── package.json     ← 의존성 목록
├── tsconfig.json    ← TypeScript 설정
└── node_modules/    ← 설치된 라이브러리
```

---

## 실행하기

```bash
cd leafday
npx expo start
```

터미널에 QR 코드와 옵션이 뜬다:
- `i` → iOS 시뮬레이터 실행
- `a` → Android 에뮬레이터 실행
- QR 스캔 → 실제 폰에서 바로 확인 (Expo Go 앱 필요)

---

## 첫 수정 — Hot Reload 확인

`App.tsx`를 열면 기본 코드가 있다:

```tsx
import { StatusBar } from 'expo-status-bar';
import { StyleSheet, Text, View } from 'react-native';

export default function App() {
  return (
    <View style={styles.container}>
      <Text>Open up App.tsx to start working on your app!</Text>
      <StatusBar style="auto" />
    </View>
  );
}
```

텍스트를 `Leafday`로 바꾸고 저장하면 — 빌드 없이 시뮬레이터에 즉시 반영된다.

이게 **Hot Reload**다. 코드 → 저장 → 화면 반영이 1~2초 안에 된다. 개발 속도가 빨라지는 이유다.

---

## JSX 기본 문법

```tsx
// 컴포넌트 = 함수 (대문자로 시작)
function MyComponent() {
  return (
    <View>           {/* 박스 (div 같은 것) */}
      <Text>텍스트</Text>          {/* 글자 */}
      <Image source={...} />      {/* 이미지 */}
      <TouchableOpacity>          {/* 버튼 (터치 가능) */}
        <Text>클릭</Text>
      </TouchableOpacity>
    </View>
  );
}
```

웹 HTML과 다른 점:
| HTML | React Native |
|---|---|
| `<div>` | `<View>` |
| `<p>`, `<span>` | `<Text>` |
| `<img>` | `<Image>` |
| `<button>` | `<TouchableOpacity>` |
| CSS 파일 | StyleSheet.create({}) |

---

## 오늘의 결과

```
✅ Watchman 설치
✅ Expo 프로젝트 생성 (TypeScript)
✅ iOS 시뮬레이터 실행
✅ 첫 텍스트 수정 + Hot Reload 확인
```

다음 편에서 Leafday의 첫 화면 UI를 설계하고 구현한다.

---

*"그냥 JS 같은데" → JSX다. React Native가 이걸 iOS/Android 네이티브로 바꿔준다.*
