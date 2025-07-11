# 시스템 흐름 스토리보드

이 디렉토리는 s-attend-gate 시스템의 내부 작동 방식과 데이터 흐름을 정의하는 스토리보드들을 포함합니다.

## 구조

- `data-processing/`: 데이터 처리 관련 시스템 흐름
- `integrations/`: 외부 시스템 및 확장 기능 통합 흐름
- `error-handling/`: 오류 처리 및 복구 흐름

## 파일 명명 규칙

- 모든 스토리보드 파일은 `-system.md` 또는 `-flow.md`로 끝나야 합니다.
- 파일명은 소문자와 하이픈을 사용합니다.
- 예: `data-processing-system.md`, `error-recovery-flow.md`

## 메타데이터 요구사항

각 스토리보드 파일은 다음 메타데이터를 포함해야 합니다:

```yaml
---
title: "스토리보드 제목"
flow-type: "system"
component: "관련 컴포넌트"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
---
```
