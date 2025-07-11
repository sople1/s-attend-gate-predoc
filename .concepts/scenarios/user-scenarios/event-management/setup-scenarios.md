# Setup Scenarios - Overview

**분할된 상세 파일들:**
- [Setup Scenarios - Basic](./setup/basic.md) - 표준 5단계 위저드 설정 프로세스
- [Setup Scenarios - Advanced](./setup/advanced.md) - 복합 이벤트 및 다중 위치 설정
- [Setup Scenarios - Templates](./setup/templates.md) - 템플릿 기반 설정 및 모범 사례

---

## 개요

S-Attend Gate 시스템의 행사 설정은 다음 3가지 주요 방식으로 구성됩니다:

### 🚀 [기본 설정 (Basic Setup)](./setup/basic.md)
**표준 5단계 위저드 프로세스**
- 단계별 안내를 통한 직관적 설정
- 소규모~중규모 이벤트 (50-500명)
- 기본적인 출입 관리 및 출석 추적

**주요 단계:**
1. 기본 정보 입력 (행사명, 기간, 장소)
2. 참가자 관리 설정 (CSV 업로드, 토큰 생성)
3. 출입 게이트 구성 (BLE/QR 설정)
4. 추가 기능 설정 (네트워킹, 게임화)
5. 최종 검토 및 활성화

### 🔧 [고급 설정 (Advanced Setup)](./setup/advanced.md)
**복합 이벤트 및 다중 위치 관리**
- 하이브리드 이벤트 (온라인+오프라인)
- 다중 위치 동시 진행
- 복잡한 일정 및 세션 관리
- 대규모 이벤트 (500명+) 최적화

**핵심 기능:**
- 중앙 집중식 관리 시스템
- 위치별 독립적 출입 관리  
- 온오프라인 참가자 상호작용
- 동적 일정 변경 지원

### 📋 [템플릿 설정 (Template-based)](./setup/templates.md)
**사전 정의된 템플릿 활용**
- 컨퍼런스, 네트워킹, 워크샵 템플릿
- 게임화 및 인센티브 설계
- 예외 상황 대응 체계
- 성공 사례 및 모범 사례

**제공 템플릿:**
- 표준 컨퍼런스 템플릿
- 네트워킹 이벤트 템플릿
- 집중 워크샵 템플릿
- 커스텀 하이브리드 템플릿

## 설정 방식 선택 가이드

### 이벤트 규모별 권장사항
- **50명 이하**: Basic Setup → 단일 게이트 + QR 코드
- **51-200명**: Basic Setup → 이중 게이트 + BLE/QR 조합
- **201-500명**: Advanced Setup → 다중 게이트 + 자동화 우선
- **500명 이상**: Advanced Setup → 전용 서버 + 로드 밸런싱

### 이벤트 유형별 권장사항
- **일반 컨퍼런스**: Template → conference-standard
- **네트워킹 중심**: Template → networking-event
- **교육/워크샵**: Template → workshop-intensive  
- **하이브리드**: Advanced → hybrid-conference

### 복잡도별 권장사항
- **단순 설정**: Basic Setup (5단계 위저드)
- **중간 복잡도**: Template (커스터마이징)
- **높은 복잡도**: Advanced (완전 맞춤 설정)

## 공통 성공 요인

### ✅ 사전 준비 체크리스트
- [ ] 참가자 데이터 검증 및 정제
- [ ] 시스템 연동 테스트 완료
- [ ] 백업 계획 수립 및 테스트
- [ ] 현장 지원팀 교육 완료

### 🎯 핵심 성과 지표
- **참가자 토큰 발급률**: 목표 100%
- **시스템 간 동기화 성공률**: 목표 99.9%
- **출석 처리 시간**: 목표 평균 2초 이내
- **사용자 만족도**: 목표 4.5/5.0 이상

### 🛠️ 공통 기술 요구사항
- Event Management Service 배포
- Gate Management 시스템 연동
- User App 환경 구성
- Analytics 대시보드 설정

## 빠른 시작 가이드

### 첫 번째 이벤트 설정 (추천)
1. **[Basic Setup](./setup/basic.md)** 문서 참조
2. 5단계 위저드로 기본 설정 완료
3. 테스트 참가자로 검증
4. 실제 참가자 등록 및 알림 발송

### 복잡한 요구사항이 있는 경우
1. **[Advanced Setup](./setup/advanced.md)** 문서 검토
2. 요구사항에 맞는 아키텍처 설계
3. 단계별 구현 및 테스트
4. 점진적 기능 확장

### 검증된 패턴 사용
1. **[Templates](./setup/templates.md)** 문서에서 유사 사례 확인
2. 적합한 템플릿 선택 및 커스터마이징
3. 모범 사례 적용
4. 성공 지표 모니터링

---

**📚 관련 문서:**
- [Gate Management Core Scenarios](../../system-scenarios/gate-management/core-scenarios.md)
- [Event Management Overview](./event-management-overview.md)
- [Technical Implementation Patterns](../../common/technical-patterns.md)
