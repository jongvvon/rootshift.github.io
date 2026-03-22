---
title: "[Leafday #2] 화면 설계부터 데이터 저장까지 — 하루 만에 MVP 뼈대 완성"
date: 2026-03-22 19:37:00 +0900
categories: [사이드 프로젝트]
tags: [leafday, react-native, expo, navigation, asyncstorage, animation, 앱개발]
---

## 오늘 목표

지난 포스팅에서 Expo 환경 세팅 + Hello World까지 했다. 오늘은 실제 Leafday 앱의 뼈대를 만드는 날.

목표:
1. 화면 4개 설계 및 구현
2. 화면 간 네비게이션 연결
3. 핵심 애니메이션 (북플립 + 카드플립)
4. AsyncStorage로 실제 데이터 저장

---

## 화면 구조 설계

먼저 앱에 필요한 화면을 정리했다.

```
📚 책장 화면 (BookshelfScreen)
    └── 내 책들이 꽂혀있는 책장
🌿 오늘 화면 (TodayScreen)
    └── 오늘/어제 사진 + 느낀점 작성
📖 새 책 화면 (NewBookScreen)
    └── 기간, 색상, 제목 설정
📄 책 보기 화면 (BookViewScreen)
    └── 페이지 넘기기 + 카드 뒤집기
```

---

## Step 1. 네비게이션 설치

화면 간 이동을 위해 React Navigation을 설치했다.

```bash
npx expo install @react-navigation/native @react-navigation/bottom-tabs
npx expo install @react-navigation/native-stack
npx expo install react-native-screens react-native-safe-area-context
```

**네비게이션 구조:**

```
Tab Navigator (하단 탭)
├── 책장 탭 → Stack Navigator
│   ├── BookshelfScreen (메인)
│   ├── NewBookScreen (모달)
│   └── BookViewScreen (책 열기)
└── 오늘 탭 → TodayScreen
```

`App.tsx`에서 전체 구조를 잡았다:

```tsx
function BookshelfStack() {
  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      <Stack.Screen name="BookshelfMain" component={BookshelfScreen} />
      <Stack.Screen name="NewBook" component={NewBookScreen}
        options={{ presentation: 'modal' }} />
      <Stack.Screen name="BookView" component={BookViewScreen} />
    </Stack.Navigator>
  );
}
```

`presentation: 'modal'`을 주면 새 책 화면이 아래에서 올라오는 느낌으로 뜬다.

---

## Step 2. 책장 UI

책장 화면의 핵심은 **책이 4권씩 선반에 꽂히고, 꽉 차면 새 선반이 추가**되는 구조다.

```tsx
const BOOKS_PER_SHELF = 4;

// 책을 4개씩 나눠서 선반 배열 생성
const shelves: BookItem[][] = [];
for (let i = 0; i < books.length; i += BOOKS_PER_SHELF) {
  shelves.push(books.slice(i, i + BOOKS_PER_SHELF));
}
```

책등(Spine) 컴포넌트:

```tsx
function BookSpine({ book, onPress }) {
  return (
    <TouchableOpacity
      style={[styles.spine, { backgroundColor: book.color }]}
      onPress={onPress}
    >
      <View style={styles.spineGloss} />  {/* 광택 효과 */}
      <Text style={styles.spineYear}>{book.year}</Text>
      <View style={styles.spineProgressTrack}>
        <View style={[styles.spineProgressFill, {
          height: `${(book.pages / book.total) * 100}%`
        }]} />
      </View>
    </TouchableOpacity>
  );
}
```

책등 아래쪽에 작은 진행 바를 넣어서 책이 얼마나 채워졌는지 한눈에 보이게 했다.

---

## Step 3. 오늘 화면

사진 선택/촬영 기능은 `expo-image-picker`를 썼다:

```bash
npx expo install expo-image-picker
```

```tsx
const pickImage = async () => {
  const { granted } = await ImagePicker.requestMediaLibraryPermissionsAsync();
  if (!granted) return;

  const result = await ImagePicker.launchImageLibraryAsync({
    allowsEditing: true,
    aspect: [3, 4],
    quality: 0.8,
  });

  if (!result.canceled) setPhoto(result.assets[0].uri);
};
```

**날짜 정책 결정:**
오늘만 쓸 수 있게 할지, 과거도 허용할지 고민했다. 결론은 **오늘 + 어제(하루 전)까지만** 허용. 야간 근무 끝나고 다음날 아침에 어제 것 올리는 상황을 고려했다.

```tsx
function getAvailableDates() {
  const today = new Date();
  const yesterday = new Date(today);
  yesterday.setDate(today.getDate() - 1);
  return [
    { label: '오늘', date: formatDate(today) },
    { label: '어제', date: formatDate(yesterday) },
  ];
}
```

---

## Step 4. 책 보기 — 핵심 애니메이션

두 가지 애니메이션을 분리해서 구현했다.

### 북플립 (페이지 넘기기)

스와이프 제스처로 페이지를 넘긴다. `PanResponder`로 손가락 속도를 추적하고, 빠르게 스와이프하면 여러 장이 연속으로 넘어가는 모멘텀 효과를 구현했다.

```tsx
const panResponder = PanResponder.create({
  onMoveShouldSetPanResponder: (_, g) => Math.abs(g.dx) > 10,
  onPanResponderMove: (e) => {
    // 드래그 중 실시간 미리보기
    const ratio = Math.min(Math.abs(dx) / (W * 0.7), 0.95);
    bookFlip.setValue(dx < 0 ? ratio : -ratio);
  },
  onPanResponderRelease: (_, g) => {
    if (Math.abs(vel) > 0.25 || Math.abs(g.dx) > W * 0.25) {
      momentumFlip(vel * 1000);  // 빠른 스와이프 → 여러 장
    } else {
      // 취소 → 원위치
      Animated.spring(bookFlip, { toValue: 0, useNativeDriver: true }).start();
    }
  },
});
```

모멘텀 플립 — 속도에 비례해서 넘어가는 페이지 수를 계산한다:

```tsx
const momentumFlip = (vel: number) => {
  const speed = Math.abs(vel);
  const pages = speed > 1.5 ? Math.min(Math.floor(speed * 5), 25)
              : speed > 0.6 ? Math.min(Math.floor(speed * 3), 10) : 1;

  let count = 0;
  const next = () => {
    if (count >= pages) return;
    count++;
    doBookFlip(dir, speed);
    setTimeout(next, Math.max(60, 220 / speed));  // 속도에 따라 딜레이 조절
  };
  next();
};
```

페이지 전환 애니메이션은 `perspective + rotateY + translateX` 조합:

```tsx
const bookRotate = bookFlip.interpolate({
  inputRange: [-1, 0, 1],
  outputRange: ['180deg', '0deg', '-180deg'],
});
const bookTranslate = bookFlip.interpolate({
  inputRange: [-1, 0, 1],
  outputRange: [W, 0, -W],
});

<Animated.View style={{
  transform: [
    { perspective: 1400 },
    { translateX: bookTranslate },
    { rotateY: bookRotate },
  ],
}}>
```

### 카드플립 (앞/뒷면 전환)

페이지 사진을 탭하면 카드가 뒤집혀서 느낀점이 나온다. `Animated.spring`으로 자연스러운 탄성을 줬다.

```tsx
const flipCard = () => {
  Animated.spring(cardFlip, {
    toValue: isCardFlipped ? 0 : 1,
    friction: 7,
    tension: 45,
    useNativeDriver: true,
  }).start(() => setIsCardFlipped(!isCardFlipped));
};

// 앞면이 사라지고 뒷면이 나타나는 타이밍
const frontOpacity = cardFlip.interpolate({
  inputRange: [0, 0.499, 0.5, 1],
  outputRange: [1, 1, 0, 0],  // 절반 지점에서 전환
});
const backOpacity = cardFlip.interpolate({
  inputRange: [0, 0.499, 0.5, 1],
  outputRange: [0, 0, 1, 1],
});
```

`backfaceVisibility: 'hidden'`을 양쪽 면에 설정해야 앞/뒤가 동시에 보이는 현상이 안 생긴다.

---

## Step 5. 데이터 저장 (AsyncStorage)

```bash
npx expo install @react-native-async-storage/async-storage
```

데이터 구조를 먼저 설계했다:

```typescript
type Page = {
  id: string;
  bookId: string;
  date: string;        // 'YYYY.MM.DD'
  photoUri: string | null;
  note: string;
  createdAt: number;
};

type Book = {
  id: string;
  title: string;
  color: string;
  startDate: string;
  endDate: string;
  totalDays: number;
  createdAt: number;
};
```

저장/조회 함수를 `src/storage/storage.ts`에 분리해서 관리했다:

```typescript
const KEYS = {
  BOOKS: 'leafday:books',
  PAGES: (bookId: string) => `leafday:pages:${bookId}`,
};

export async function savePage(page: Page): Promise<void> {
  const pages = await getPages(page.bookId);
  const existing = pages.findIndex(p => p.id === page.id);
  if (existing >= 0) {
    pages[existing] = page;  // 기존 페이지 수정
  } else {
    pages.push(page);
  }
  pages.sort((a, b) => a.date.localeCompare(b.date));  // 날짜순 정렬
  await AsyncStorage.setItem(KEYS.PAGES(page.bookId), JSON.stringify(pages));
}
```

화면 포커스될 때마다 최신 데이터를 불러오기 위해 `useFocusEffect`를 썼다:

```tsx
useFocusEffect(
  useCallback(() => {
    loadBooks();
  }, [])
);
```

---

## 오늘의 결과

```
✅ 화면 4개 구현
✅ 스택 + 탭 네비게이션 연결
✅ 책장 → 새 책 → 책 보기 플로우
✅ 북플립 (모멘텀 + 드래그 미리보기)
✅ 카드플립 (사진 ↔ 느낀점)
✅ 오늘/어제 날짜 선택
✅ AsyncStorage 데이터 저장
✅ GitHub 레포 생성 (jongvvon/leafday)
```

---

## 코드 히스토리

오늘 가장 많이 바뀐 것들:

**네비게이션 구조 변경**
처음엔 탭에 "새 책" 탭을 따로 만들었다가 → 책장 내부 스택으로 이동. 책 만들기는 흐름상 책장에서 시작해야 자연스럽다.

**책장 데이터**
처음엔 하드코딩된 `SAMPLE_BOOKS` 배열 → AsyncStorage에서 실제 저장된 책 불러오는 방식으로 교체. `useFocusEffect`로 화면 돌아올 때마다 갱신.

**BookViewScreen 3번 갈아엎음**
- 1차: FlatList 방식 (촤르르 스크롤) → 페이지 넘기는 느낌 없음
- 2차: `rotateY` 슬라이드 → 여전히 북플립 느낌 약함
- 3차: `perspective + PanResponder + 모멘텀` 조합으로 완성

실제로 써봐야 뭐가 어색한지 알고, 써봐야 뭘 고쳐야 할지 안다.

---

## 다음 편

- 사진 실제 표시 완성 ✅ (이미 반영)
- Git CI/CD — TestFlight 자동 배포
- 테마 선택 기능
- App Store 제출 준비
