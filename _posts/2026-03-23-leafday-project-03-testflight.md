---
title: "[Leafday #3] TestFlight에 앱 올리기 — EAS Build로 삽질하며 배운 것들"
date: 2026-03-23 16:14:00 +0900
categories: [사이드 프로젝트]
tags: [leafday, expo, eas, testflight, ios, apple-developer, 앱배포]
---

## 오늘 목표

어제 MVP를 만들었으니 이제 실제 폰에서 써봐야 한다.

목표: **TestFlight에 Leafday 올리기**

결론부터 말하면, 5번 빌드 실패했다. 기록해둔다.

---

## EAS Build가 뭔데

처음 들어보는 이름이었다.

**EAS = Expo Application Services**

Expo가 제공하는 클라우드 빌드 서비스다. 내 맥북에서 Xcode 안 켜도 Apple 서버에서 .ipa 파일(iOS 앱 패키지)을 만들어준다.

```
내 코드 → EAS 서버 (macOS 클라우드) → .ipa 파일 → App Store Connect
```

전통적인 iOS 개발이라면:
1. Xcode 설치
2. 인증서 발급
3. Provisioning Profile 설정
4. Archive → Export
5. Transporter로 업로드

이게 전부 수동이었는데, EAS가 전부 자동화해준다.

```bash
eas build --platform ios --profile production
eas submit --platform ios --latest
```

두 줄이면 빌드부터 TestFlight 업로드까지 끝난다. (물론 처음엔 안 된다.)

---

## Apple Developer 계정부터

TestFlight를 쓰려면 Apple Developer Program 가입이 필수다.

- 연간 ₩129,000 (약 $99)
- 가입 → 심사 → 승인 (보통 24~48시간)
- 승인되면 App Store Connect 접근 가능

승인 메일 두 개가 동시에 왔다:
```
"Apple Developer Program 시작하기"
"Welcome to App Store Connect"
```

---

## App Store Connect에서 앱 등록

1. `appstoreconnect.apple.com` 접속
2. 새 앱 → iOS
3. 번들 ID 등록 필요 → `developer.apple.com` → Identifiers → + → App IDs
   - Bundle ID: `dev.rootshift.leafday` (Explicit)
4. App Store Connect 돌아와서 번들 ID 선택
5. SKU: `leafday-001`
6. 생성 완료

---

## EAS 설정

```bash
npm install -g eas-cli
eas login  # Expo 계정으로 로그인
cd leafday && eas build:configure  # EAS 프로젝트 연결
```

`eas.json` 설정:

```json
{
  "build": {
    "production": {
      "ios": {
        "buildConfiguration": "Release",
        "image": "latest"
      }
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "my@email.com",
        "ascAppId": "앱ID",
        "appleTeamId": "팀ID"
      }
    }
  }
}
```

---

## 삽질 기록 — 빌드 5번 실패

### 실패 1: CocoaPods 에러

```
[Reanimated] Your installed version of Worklets (0.8.1) is not compatible
with installed version of Reanimated (4.2.1)
```

`react-native-reanimated`가 설치돼 있었는데 소스에서 실제로 안 쓰고 있었다. 그냥 제거했다.

```bash
npm uninstall react-native-reanimated react-native-gesture-handler
```

### 실패 2: Folly 헤더 에러

```
'folly/coro/Coroutine.h' file not found
```

React Native 0.83.2 + Expo SDK 55 조합에서 Folly 헤더 충돌. EAS 빌드 이미지를 최신으로 바꿔서 해결.

```json
"image": "latest"
```

### 실패 3: lock 파일 불일치

```
npm error npm ci can only install packages when your package.json
and package-lock.json are in sync
```

패키지 여러 번 설치/제거하다 보니 `package-lock.json`이 꼬였다. 삭제 후 재생성.

```bash
rm -f package-lock.json && npm install
```

### 실패 4~5: GitHub Actions 충돌

어제 만들어둔 GitHub Actions 워크플로우가 push 때마다 자동으로 빌드 시도하면서 실패 메일이 계속 왔다. 워크플로우 삭제로 해결.

---

## 드디어 성공

6번째 시도:

```
✅ Build finished
🍎 iOS app: leafday.ipa
```

그리고 바로 제출:

```bash
eas submit --platform ios --latest
```

```
✅ Submitted your app to Apple App Store Connect!
✅ Your binary has been successfully uploaded to App Store Connect!
```

---

## TestFlight가 뭔데

이것도 처음 들어봤다.

**TestFlight = Apple의 베타 테스트 플랫폼**

App Store 정식 출시 전에 실제 기기에서 테스트할 수 있다.

| | TestFlight | App Store |
|---|---|---|
| 심사 | 간단 (1~2일) | 엄격 (3~7일) |
| 설치 방법 | 초대 링크 | 앱스토어 검색 |
| 빌드 유효기간 | 90일 | 무기한 |

흐름:
```
EAS Build → .ipa 파일 → App Store Connect 업로드
    ↓
TestFlight 처리 완료 (10~30분)
    ↓
내부 테스터 추가 → 초대 이메일
    ↓
TestFlight 앱 설치 → Leafday 설치
```

---

## 오늘의 결과

```
✅ Apple Developer 계정 가입
✅ App Store Connect 앱 등록
✅ EAS Build 설정
✅ 빌드 5번 실패, 6번째 성공
✅ TestFlight 업로드 완료
```

코드 짜는 게 아니라 **배포 파이프라인 세팅**이 이렇게 복잡할 줄 몰랐다.

EAS가 없었으면 Xcode 설정만 하루 종일 했을 거다. 실제로 iOS 개발자들 말 들어보면 인증서/프로비저닝 설정이 제일 고통스러운 단계라고 하는데, EAS가 그걸 거의 다 해결해줬다.

---

## 다음

- TestFlight 내부 테스트
- 버그 수정
- 개인정보처리방침 페이지 생성
- App Store 정식 제출 준비
