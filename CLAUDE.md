# React Native Project Context

## Project Stack

- Framework: React Native CLI (bare) or Expo (auto-detect from package.json)
- Language: TypeScript
- Target: Android (Google Play Store) + iOS (App Store)

## Key Commands

```bash
# 의존성 설치
npm install

# iOS 빌드
npx react-native run-ios

# Android 빌드
npx react-native run-android

# TypeScript 타입 체크
npx tsc --noEmit

# 린트
npx eslint . --ext .ts,.tsx

# 번들 분석
npx react-native bundle --platform android --dev false --entry-file index.js --bundle-output /tmp/bundle.js --sourcemap-output /tmp/bundle.js.map
```

## Code Conventions

- 컴포넌트는 함수형 컴포넌트 + React.FC 타입 사용
- 상태관리는 프로젝트 내 실제 사용 라이브러리 우선 따름
- 스타일은 StyleSheet.create() 또는 NativeWind 중 프로젝트 설정 따름
- 파일명은 PascalCase (컴포넌트) 또는 camelCase (유틸)

## Store Submission Checklist (항상 참고)

### Android (Google Play)

- targetSdkVersion: 34 이상 (2024년 기준 필수)
- minSdkVersion: 23 이상 권장
- android:exported 속성 명시 필수
- 64비트 지원 필수 (arm64-v8a)
- ProGuard / R8 난독화 활성화 권장

### iOS (App Store)

- iOS 최소 배포 버전: 16.0 이상 권장
- Privacy manifest (PrivacyInfo.xcprivacy) 포함 필수
- NSUserTrackingUsageDescription 등 권한 설명 Info.plist 포함
- Bitcode: Xcode 14+ 에서 비활성화가 기본값

## Performance Guidelines

- FlatList keyExtractor 필수 지정
- useCallback / useMemo 과도한 사용 지양 (측정 후 적용)
- 이미지는 react-native-fast-image 또는 Expo Image 사용 권장
- Hermes 엔진 활성화 여부 확인 (android/app/build.gradle)
- InteractionManager.runAfterInteractions() 활용으로 초기 로딩 분산

## Skills Available

- `/rn-audit` : 앱 전체 감사 (오류 / 스토어 심사 / 성능 / 메모리)
