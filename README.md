# react-native-audit-prompt

React Native 프로젝트 전체 감사를 자동화하는 Claude Code 커스텀 슬래시 커맨드.

`/rn-audit` 한 번 실행으로 다음 항목을 점검하고 마크다운 리포트로 저장합니다.

- TypeScript / ESLint 오류 탐지
- Android (Google Play) 스토어 심사 체크리스트 — targetSdkVersion 35, 16KB 페이지 사이즈 등 2025~2026 기준 반영
- iOS (App Store) 스토어 심사 체크리스트 — iOS 26 SDK, Privacy manifest 등
- 성능 / 로딩 최적화 분석 (FlatList, useEffect, 이미지, 번들)
- 메모리 누수 위험 패턴 탐지
- console.log 프로덕션 잔류 탐지

---

## 설치 방법

이 저장소의 `SKILL.md` 파일을 Claude Code 가 인식하는 커맨드 디렉터리에 배치하면 됩니다.

### 1. 프로젝트 전용으로 사용 (해당 RN 프로젝트에서만)

분석 대상 React Native 프로젝트의 루트에서 실행합니다.

```bash
mkdir -p .claude/commands
cp SKILL.md .claude/commands/rn-audit.md
```

`.claude/commands/` 디렉터리는 git 으로 커밋해 팀원과 공유할 수 있습니다.

### 2. 전역으로 사용 (내 모든 프로젝트에서)

```bash
mkdir -p ~/.claude/commands
cp SKILL.md ~/.claude/commands/rn-audit.md
```

> 파일명이 곧 커맨드명이 됩니다. `rn-audit.md` 로 저장하면 `/rn-audit` 으로 호출됩니다.
> SKILL.md 상단 frontmatter 의 `name: rn-audit` 값과 파일명을 맞춰 두세요.

---

## 사용 방법

분석할 React Native 프로젝트 루트에서 Claude Code 를 실행합니다.

```bash
cd <your-rn-project>
claude
```

Claude Code 프롬프트에서 슬래시 커맨드를 입력합니다.

### 전체 프로젝트 분석

```
/rn-audit
```

### 특정 파일 / 디렉터리만 분석

```
/rn-audit src/screens/HomeScreen.tsx
/rn-audit src/components
```

`SKILL.md` 의 `$ARGUMENTS` 자리에 위 인자가 그대로 전달되어 해당 경로만 분석합니다.

---

## 실행 결과

분석이 끝나면 프로젝트 루트에 다음 경로로 리포트가 저장됩니다.

```
rn-audit-reports/audit-<프로젝트명>-<YYYY-MM-DD>.md
```

리포트에는 심각도별(🔴 CRITICAL / 🟠 HIGH / 🟡 MEDIUM / 🟢 LOW) 요약, 항목별 PASS/FAIL 표, 수정 전 / 수정 후 코드 스니펫이 포함됩니다.

---

## 안전 규칙

이 커맨드는 **분석과 리포트 생성만** 수행합니다.

- 프로젝트 내 기존 파일은 절대 수정/삭제하지 않습니다.
- 쓰기가 허용된 유일한 경로는 `rn-audit-reports/` 디렉터리입니다.
- 수정이 필요한 코드는 리포트 안에 "수정 전 / 수정 후" 형태로 제안만 합니다.

`SKILL.md` frontmatter 의 `allowed-tools` 로 허용 명령이 화이트리스트화 되어 있어, 다른 destructive 명령은 실행되지 않습니다.
