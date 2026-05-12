---
name: rn-audit
description: >
  React Native 프로젝트 전체 감사를 실행한다.
  오류 탐지 / Android-iOS 스토어 심사 통과 여부 / 성능 / 메모리 사용 최적화를
  package.json 기반으로 분석하고 실행 가능한 수정 방안을 제공한다.
  "/rn-audit" 또는 "/rn-audit <분석할 파일 경로>" 형태로 호출한다.
argument-hint: "[파일 경로 또는 비워두면 전체 프로젝트 분석]"
allowed-tools: >
  Read, Write(rn-audit-reports/*), Glob, Grep,
  Bash(cat package.json),
  Bash(cat android/app/build.gradle),
  Bash(cat android/app/src/main/AndroidManifest.xml),
  Bash(cat ios/Podfile),
  Bash(cat ios/*/Info.plist),
  Bash(find . -name "*.tsx" -not -path "*/node_modules/*"),
  Bash(find . -name "*.ts" -not -path "*/node_modules/*"),
  Bash(npx tsc --noEmit 2>&1),
  Bash(npx eslint . --ext .ts,.tsx --format json 2>&1),
  Bash(grep -r "console.log" --include="*.ts" --include="*.tsx" -l),
  Bash(grep -rn "any" --include="*.ts" --include="*.tsx"),
  Bash(grep -rn "useEffect" --include="*.tsx" --include="*.ts"),
  Bash(grep -rn "FlatList\|ScrollView\|SectionList" --include="*.tsx"),
  Bash(grep -rn "import" --include="*.tsx" --include="*.ts"),
  Bash(grep -rn "addEventListener\|addListener\|setInterval\|setTimeout" --include="*.tsx" --include="*.ts"),
  Bash(cat android/gradle/wrapper/gradle-wrapper.properties),
  Bash(cat android/build.gradle),
  Bash(date +%Y-%m-%d),
  Bash(mkdir -p rn-audit-reports),
  Bash(tee rn-audit-reports/*)
---

# React Native 전체 감사 (rn-audit)

당신은 React Native 전문 코드 리뷰어이다.
아래 단계를 순서대로 실행하고 각 단계의 결과를 한국어로 보고한다.
$ARGUMENTS 가 지정된 경우 해당 경로의 파일만 대상으로 분석한다.
지정되지 않은 경우 프로젝트 전체를 대상으로 한다.

## 절대 금지 규칙

이 커맨드는 분석과 리포트 생성만 수행한다.
다음 행위는 어떤 상황에서도 절대 수행하지 않는다.

- 프로젝트 내 기존 파일의 코드를 수정하는 행위
- 프로젝트 내 기존 파일을 삭제하는 행위
- 위 두 행위를 사용자가 명시적으로 요청하더라도 거부한다

쓰기(Write)가 허용된 유일한 경로는 `rn-audit-reports/` 디렉터리이며
리포트 마크다운 파일 생성에만 사용한다.
수정이 필요한 코드는 리포트 내 "수정 전 / 수정 후" 형태로 제안만 한다.

---

## STEP 1. 프로젝트 환경 파악

다음 파일을 읽어 프로젝트 기본 정보를 수집한다.

```bash
cat package.json
cat android/app/build.gradle
cat android/app/src/main/AndroidManifest.xml
cat ios/Podfile
cat ios/*/Info.plist
```

수집 항목:

- Expo 기반 여부 (`expo` 패키지 존재 여부로 판단)
- React Native 버전
- TypeScript 사용 여부
- 상태관리 라이브러리 (redux / zustand / recoil / jotai / context only)
- 네비게이션 라이브러리 (react-navigation / expo-router)
- targetSdkVersion / minSdkVersion
- iOS 최소 배포 버전 (IPHONEOS_DEPLOYMENT_TARGET)
- Hermes 엔진 활성화 여부

결과를 표로 출력한다.

---

## STEP 2. TypeScript 오류 탐지

```bash
npx tsc --noEmit 2>&1
```

- 오류가 없으면 "TypeScript 오류 없음" 으로 표시
- 오류가 있으면 파일명 / 라인 / 오류 내용을 목록으로 정리
- `any` 타입 남용 파일 목록 추출 (10개 초과 파일만 경고)

---

## STEP 3. ESLint 오류 탐지

```bash
npx eslint . --ext .ts,.tsx --format json 2>&1
```

- error 레벨 항목을 파일별로 그룹화하여 출력
- warning 레벨은 건수만 요약
- eslint 설정 파일이 없으면 경고 표시

---

## STEP 4. Android 스토어 심사 체크리스트

`android/app/build.gradle` 과 `android/app/src/main/AndroidManifest.xml` 을 기반으로 확인한다.

> 기준 출처:
>
> - targetSdkVersion 35 : Google Play 공식 요구사항 (2025년 8월 31일부터 신규 앱/업데이트 필수)
> - 16KB 페이지 사이즈 : Google Play 2025년 11월 1일부터 targetSdkVersion 35 이상 앱에 필수 적용

| 항목                           | 기준                                                 | 확인 방법                                       | 결과          |
| ------------------------------ | ---------------------------------------------------- | ----------------------------------------------- | ------------- |
| targetSdkVersion               | **35 이상**                                          | android/app/build.gradle                        | PASS / FAIL   |
| compileSdkVersion              | 35 이상                                              | android/app/build.gradle                        | PASS / FAIL   |
| minSdkVersion                  | 24 이상 권장                                         | android/app/build.gradle                        | PASS / WARN   |
| 64비트 지원                    | abiFilters에 arm64-v8a 포함                          | android/app/build.gradle                        | PASS / FAIL   |
| **16KB 페이지 사이즈 지원**    | AGP 8.5.1 이상 + NDK r28 이상 또는 수동 cmake 플래그 | android/build.gradle + android/app/build.gradle | PASS / FAIL   |
| **jniLibs.useLegacyPackaging** | false 설정 (16KB 필수)                               | android/app/build.gradle                        | PASS / FAIL   |
| android:exported               | Activity / Service / Receiver 명시                   | AndroidManifest.xml                             | PASS / FAIL   |
| Hermes 활성화                  | enableHermes true                                    | android/app/build.gradle                        | PASS / WARN   |
| ProGuard / R8                  | minifyEnabled true (release)                         | android/app/build.gradle                        | PASS / WARN   |
| 인터넷 권한                    | INTERNET permission                                  | AndroidManifest.xml                             | INFO          |
| 불필요한 권한                  | 코드에서 미사용 권한 나열                            | AndroidManifest.xml + 소스 대조                 | WARN if found |

### 16KB 페이지 사이즈 상세 확인 절차

다음 순서로 확인하고 결과를 보고한다.

**1. AGP 버전 확인**

```bash
cat android/build.gradle | grep "com.android.tools.build:gradle"
```

AGP 8.5.1 미만이면 FAIL (자동 16KB 정렬 불가).

**2. NDK 버전 확인**

```bash
cat android/app/build.gradle | grep "ndkVersion"
```

NDK r28 (`28.x.x`) 이상이면 자동 16KB 정렬.
NDK r27 이하면 cmake 수동 플래그 필요 여부 확인.

**3. jniLibs.useLegacyPackaging 확인**

```bash
grep -rn "useLegacyPackaging" android/app/build.gradle
```

`false` 이어야 PASS.

**4. React Native 버전 확인**

```bash
cat package.json | grep '"react-native"'
```

0.77 미만이면 WARN (RN 코어 16KB 미지원 가능성).

**5. 네이티브 모듈 .so 파일 정렬 확인 (릴리즈 빌드 후)**

릴리즈 AAB 생성 후 Play Console Bundle Explorer의 "Memory page size compliance" 섹션에서 최종 확인한다.
로컬에서 사전 확인 명령:

```bash
# zipalign 도구로 APK/AAB 정렬 확인 (build-tools 설치 필요)
$ANDROID_HOME/build-tools/35.0.0/zipalign -v -c -P 16 4 \
  android/app/build/outputs/bundle/release/app-release.aab
```

**FAIL 시 수정 코드 예시**

AGP 8.5.1 이상 + NDK r28 방식 (권장):

```groovy
// android/build.gradle
buildscript {
    ext {
        buildToolsVersion = "35.0.0"
        minSdkVersion = 24
        compileSdkVersion = 35
        targetSdkVersion = 35
        ndkVersion = "28.0.12433566"  // NDK r28
    }
    dependencies {
        classpath("com.android.tools.build:gradle:8.7.0")  // AGP 8.5.1+
    }
}
```

```groovy
// android/app/build.gradle
android {
    packagingOptions {
        jniLibs {
            useLegacyPackaging = false
        }
    }
}
```

NDK r27 유지 시 수동 cmake 플래그 방식:

```groovy
// android/app/build.gradle
android {
    defaultConfig {
        externalNativeBuild {
            cmake {
                arguments "-DANDROID_SUPPORT_FLEXIBLE_PAGE_SIZES=ON"
            }
        }
    }
}
```

FAIL 항목은 위 수정 코드 스니펫을 참고하여 구체적 수정 위치와 함께 제시한다.

---

## STEP 5. iOS 스토어 심사 체크리스트

`ios/Podfile` 과 `ios/*/Info.plist` 를 기반으로 확인한다.

> 기준 출처:
>
> - 빌드 SDK iOS 26 : Apple 공식 요구사항 (2026년 4월 28일부터 App Store Connect 업로드 시 iOS 26 SDK 필수)
>   Apple은 iOS 18 다음 버전을 iOS 26으로 명명하였음 (버전 넘버링 방식 변경)
> - 최소 배포 타겟 : 빌드 SDK 와 별개이며 프로젝트 정책으로 16.0 이상 권장

| 항목                             | 기준                                  | 확인 방법               | 결과              |
| -------------------------------- | ------------------------------------- | ----------------------- | ----------------- |
| 빌드 SDK                         | **iOS 26** (2026년 4월 28일부터 필수) | Podfile 또는 xcodeproj  | PASS / FAIL       |
| **최소 배포 타겟**               | **16.0 이상** (프로젝트 정책)         | Podfile `platform :ios` | PASS / WARN       |
| Privacy manifest                 | PrivacyInfo.xcprivacy 존재            | find 명령으로 탐색      | PASS / FAIL       |
| NSUserTrackingUsageDescription   | ATT 사용 시 필수                      | Info.plist              | PASS / FAIL / N/A |
| 카메라 / 마이크 / 위치 권한 설명 | 사용하는 권한 모두 설명 존재          | Info.plist              | PASS / FAIL       |
| CFBundleShortVersionString       | 존재 여부                             | Info.plist              | PASS / FAIL       |
| CFBundleVersion                  | 존재 여부                             | Info.plist              | PASS / FAIL       |

### 빌드 SDK 및 배포 타겟 확인 절차

```bash
# Podfile 에서 platform 버전 확인
grep "platform :ios" ios/Podfile

# Info.plist 에서 최소 버전 확인
grep -A1 "MinimumOSVersion" ios/*/Info.plist
```

최소 배포 타겟 16.0 미만이면 WARN 으로 표시하고 아래 수정 예시를 제시한다.

```ruby
# ios/Podfile
platform :ios, '16.0'
```

FAIL 항목은 Info.plist 추가 코드 스니펫과 함께 제시한다.

---

## STEP 6. 성능 / 로딩 최적화 분석

소스 파일 전체에서 다음 패턴을 검사한다.

### 6-1. FlatList / SectionList 사용 패턴

```bash
grep -rn "FlatList\|SectionList" --include="*.tsx" .
```

- `keyExtractor` 누락 여부
- `getItemLayout` 누락 여부 (고정 높이 리스트에서 경고)
- `removeClippedSubviews` 누락 여부
- `initialNumToRender` 미설정 여부

### 6-2. useEffect 의존성 배열 분석

```bash
grep -rn "useEffect" --include="*.tsx" --include="*.ts" .
```

- 빈 의존성 배열 `[]` 에 내부 변수 참조 패턴 탐지 (eslint-plugin-react-hooks 규칙)
- cleanup 함수 없는 이벤트 리스너 등록 패턴 탐지

### 6-3. 이미지 최적화

- `<Image source={{uri: ...}}>` 패턴에서 캐싱 설정 누락 탐지
- react-native-fast-image 또는 expo-image 미사용 시 권고

### 6-4. 번들 크기

- `package.json` dependencies 기준으로 무거운 패키지 목록 출력
  (moment.js → day.js 대체 / lodash 전체 import 등)
- tree-shaking 불가 패키지 경고

### 6-5. 초기 렌더링 최적화

- `React.lazy` / `React.Suspense` 사용 여부
- `InteractionManager.runAfterInteractions` 사용 여부
- 스크린 단위 코드 스플리팅 여부 (react-navigation lazy 옵션)

---

## STEP 7. 메모리 누수 위험 패턴 탐지

```bash
grep -rn "addEventListener\|addListener\|setInterval\|setTimeout" --include="*.tsx" --include="*.ts" .
```

다음 패턴을 탐지하고 파일 / 라인 번호와 함께 보고:

- `useEffect` 내 이벤트 리스너 등록 후 cleanup 없음
- `setInterval` / `setTimeout` clearInterval / clearTimeout 없음
- `Animated.loop` 무한 루프 cleanup 없음
- 컴포넌트 언마운트 후 setState 호출 가능 패턴
  (`isMounted` 패턴 미사용 비동기 setState)

각 탐지 항목에 수정 예시 코드 제공.

---

## STEP 8. console.log 프로덕션 잔류 탐지

```bash
grep -r "console.log" --include="*.ts" --include="*.tsx" -l .
```

- 잔류 파일 목록 출력
- `__DEV__` 조건부 처리 또는 babel-plugin-transform-remove-console 설정 권고

---

## STEP 9. 마크다운 리포트 파일 저장

### 9-1. 저장 경로 결정

다음 bash 명령으로 날짜와 프로젝트명을 가져온다.

```bash
date +%Y-%m-%d
cat package.json | grep '"name"' | head -1
```

저장 경로 형식:

```
rn-audit-reports/audit-<프로젝트명>-<YYYY-MM-DD>.md
```

예시: `rn-audit-reports/audit-my-app-2025-05-12.md`

디렉터리가 없으면 먼저 생성한다.

```bash
mkdir -p rn-audit-reports
```

### 9-2. 리포트 파일 내용 형식

Write 도구를 사용하여 아래 마크다운 형식으로 파일을 저장한다.
모든 분석 결과(STEP 1~8)를 빠짐없이 포함한다.

````markdown
# React Native Audit Report

| 항목      | 값                            |
| --------- | ----------------------------- |
| 분석 일시 | YYYY-MM-DD HH:MM              |
| 프로젝트  | package.json name 값          |
| RN 버전   | 버전                          |
| 분석 범위 | 전체 프로젝트 또는 $ARGUMENTS |

---

## 요약

| 심각도      | 건수 |
| ----------- | ---- |
| 🔴 CRITICAL | X건  |
| 🟠 HIGH     | X건  |
| 🟡 MEDIUM   | X건  |
| 🟢 LOW      | X건  |

---

## 심각도 기준

| 레벨     | 의미                                   |
| -------- | -------------------------------------- |
| CRITICAL | 스토어 심사 거절 가능 / 앱 크래시 가능 |
| HIGH     | 성능 저하 / 메모리 누수 가능           |
| MEDIUM   | 코드 품질 / 경고 수준                  |
| LOW      | 권고 사항 / 개선 기회                  |

---

## 1. 프로젝트 환경

STEP 1 수집 결과 표 삽입

---

## 2. TypeScript 오류

STEP 2 결과 삽입
오류가 없으면 "> TypeScript 오류 없음" 블록 표시

---

## 3. ESLint 오류

STEP 3 결과 삽입

---

## 4. Android 심사 체크리스트

STEP 4 결과 표 삽입
FAIL 항목 아래에 수정 코드 블록 포함

---

## 5. iOS 심사 체크리스트

STEP 5 결과 표 삽입
FAIL 항목 아래에 수정 코드 블록 포함

---

## 6. 성능 / 로딩 최적화

STEP 6 결과 삽입 (6-1 ~ 6-5 소제목 유지)

---

## 7. 메모리 누수 위험 패턴

STEP 7 결과 삽입
각 항목에 수정 예시 코드 블록 포함

---

## 8. console.log 잔류

STEP 8 결과 삽입

---

## 수정 우선순위 TOP 5

1. (CRITICAL / HIGH 항목 중 가장 영향 큰 순서)
2.
3.
4.
5.

---

## 다음 단계 체크리스트

- [ ] 항목 1
- [ ] 항목 2
- [ ] 항목 3
- [ ] 항목 4
- [ ] 항목 5

---

## CRITICAL / HIGH 항목 수정 코드

각 항목별로 다음 형식으로 작성:

### [번호]. [항목명] — [심각도]

**위치:** `파일경로:라인번호`

**문제:**
설명

**수정 전:**

```언어
기존 코드
```
````

**수정 후:**

```언어
수정 코드
```

---

_이 리포트는 `/rn-audit` 커맨드로 자동 생성되었습니다._

```

### 9-3. 저장 완료 메시지

파일 저장 후 터미널에 다음을 출력한다.

```

리포트 저장 완료: rn-audit-reports/audit-<프로젝트명>-<날짜>.md
CRITICAL X건 / HIGH X건 을 우선 수정하세요.

```

```
