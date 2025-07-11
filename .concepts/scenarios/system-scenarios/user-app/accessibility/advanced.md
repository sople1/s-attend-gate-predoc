# User App 고급 접근성 기능 개요

## 개요

User App의 고급 접근성 기능들을 주제별로 구성한 개요입니다. 운동 능력, 인지적 접근성, 테스트 검증 등의 상세 내용은 각 전문 파일을 참조하세요.

## 📋 고급 접근성 영역

### 🖐️ 운동 능력 접근성
**파일**: [accessibility/motor.md](accessibility/motor.md)

운동 능력 제약이 있는 사용자를 위한 접근성 기능:
- 음성 명령 인터페이스 구현
- 큰 터치 영역 및 터치 제스처 단순화
- 스위치 입력 및 시선 고정 클릭(Dwell Click) 지원
- 대체 입력 방법 및 하드웨어 연동

### 🧠 인지적 접근성
**파일**: [accessibility/cognitive.md](accessibility/cognitive.md)

인지적 제약이 있는 사용자를 위한 접근성 기능:
- 단순화된 인터페이스 및 언어
- 단계별 가이드 시스템
- 명확한 피드백 및 상태 표시
- 반복 작업 지원 및 자주 사용하는 기능 바로가기

### 🧪 테스트 및 검증
**파일**: [accessibility/testing.md](accessibility/testing.md)

접근성 기능의 체계적인 검증 방법론:
- 자동화된 접근성 테스트 (axe-core 활용)
- 수동 검사 체크리스트 (키보드, 스크린 리더, 시각적, 모바일)
- 지속적인 접근성 모니터링 시스템
- 사용자 피드백 수집 및 개선 프로세스

## 📚 연관 문서

### 기본 접근성
- [accessibility-basic.md](accessibility-basic.md) - 기본 접근성 기능 (스크린 리더, 시각적, 청각적)
- [accessibility-implementation.md](accessibility-implementation.md) - 전체 접근성 시스템 개요

### 사용자 앱 시나리오
- [technical-performance-optimization.md](technical-performance-optimization.md) - 성능 최적화
- [technical-constraints-solutions.md](technical-constraints-solutions.md) - 기술적 제약 해결방안
- [technical-performance-monitoring.md](technical-performance-monitoring.md) - 성능 모니터링

### 시스템 전체
- [../../common/technical-patterns.md](../../common/technical-patterns.md) - 기술 패턴 개요
- [../../event-management/](../../event-management/) - 이벤트 관리 시나리오
