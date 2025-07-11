# s-attend-gate 시나리오 문서

## 🎯 시스템 아키텍처 개요

s-attend-gate는 **4개의 독립적인 서비스 그룹**으로 구성된 BLE 기반 출석 체크 시스템입니다:

1. **User App** (다중 행사 지원) - 참가자용 모바일 앱
2. **Gate Management** (단일 행사) - 현장 게이트 관리 시스템  
3. **Event Management** (단일 행사) - 행사별 데이터 관리 서비스
4. **Integrated Platform** (다중 행사) - 통합 분석 및 API 허브

---

## 📁 폴더 구성

### 👥 [user-scenarios/](./user-scenarios/) - **순수 사용자 경험** ⭐
**사용자 행동과 감정에 집중한 시나리오** (기술 구현 배제)

**서비스별 사용자 시나리오**
- `common/` - 공통 사용자 시나리오
- `user-app/` - 사용자 앱 관련 행동 패턴
- `gate-management/` - 게이트 관리 사용자 행동
- `event-management/` - 이벤트 관리 사용자 시나리오
- `integrated-platform/` - 통합 플랫폼 사용자 시나리오
- `_archive/` - 이전 버전 시나리오 백업

**✅ 포함**: 실제 사용자 행동, 감정 반응, 의사결정 과정, 문제 해결 방식
**❌ 제외**: 기술적 세부사항, API 호출, 시스템 성능, 개발자 제약사항

### 🔧 [system-scenarios/](./system-scenarios/) - **기술적 구현** ⭐
**API/ABI와 서비스 간 통신에 집중한 기술 문서**
- `core-apis/` - 각 서비스의 API 명세와 인터페이스
- `user-app/` - 사용자 앱 기술적 구현 사양
- `event-management/` - 이벤트 관리 시스템 기술 사양
- `gate-management/` - 게이트 관리 시스템 기술 사양
- `integrated-platform/` - 플랫폼 통합 기술 사양

### 🧩 [common/](./common/) - **공유 컴포넌트와 비즈니스 규칙** ⭐
**모든 서비스에서 공통으로 사용하는 표준과 패턴**
- `business-rules.md` - 출석 체크, 토큰 관리, 권한 정책 등 핵심 비즈니스 로직
- `technical-patterns.md` - BLE 통신, 오프라인 동기화, API 통신 패턴
- `patterns/` - 세부 기술 패턴들
  - `ble-communication.md` - BLE 비콘 통신 패턴
  - `offline-sync.md` - 오프라인 동기화 패턴
  - `data-sync.md` - 데이터 동기화 패턴
  - `security-auth.md` - 보안 및 인증 패턴
  - `backup.md` - 백업 및 복구 패턴
  - `metrics.md` - 메트릭 수집 패턴

### 🔌 [extensions/](./extensions/) - **확장 기능 (플러그인)** ⭐
**핵심 출석 체크와 독립적인 부가 기능들**
- `treasure-hunt/` - BLE 비콘 기반 보물찾기 게임
- `networking-game/` - 참가자 간 네트워킹 유도 게임
- `analytics-dashboard/` - 고급 분석 및 인사이트
- `integrations/` - 외부 시스템 연동 (CRM, 마케팅 도구 등)

**핵심 원칙**: 확장 기능이 비활성화되어도 출석 체크는 완벽히 동작

---

## 🏗️ 시스템 아키텍처 특징

### 독립적 서비스 구조
각 서비스 그룹은 완전히 독립적으로 배포 및 운영됩니다:

```
User App (다중 행사)          Event Management (단일 행사)
     ↓ 토큰 인증                    ↑ 참가자 데이터
Gate Management (단일 행사) ←→ API 연동
     ↓ 출석 데이터
Integrated Platform (다중 행사) ←→ 통합 분석
```

### 확장성과 안정성
- **수평적 확장**: 행사 증가 시 독립 서비스 추가 배포
- **장애 격리**: 한 행사의 문제가 다른 행사에 영향 없음  
- **기술 선택의 자유**: 각 서비스별 최적 기술 스택 선택 가능

### 데이터 흐름
1. **참가자 등록**: Event Management로 CSV/API 업로드
2. **토큰 발급**: 참가자별 고유 토큰 생성 및 배포
3. **앱 설정**: User App에서 토큰으로 행사 추가
4. **출석 체크**: Gate Management에서 실시간 검증
5. **데이터 분석**: Integrated Platform에서 통합 분석

---

## 📖 시나리오 활용 가이드
- **시스템 설계**: `user-scenarios/` 각 서비스별 아키텍처 이해
- **API 연동**: 서비스 간 연동 방법 및 데이터 흐름 파악
- **기술 구현**: `technical-scenarios/` 참조하여 핵심 기술 구현

### 기획자를 위한 가이드  
- **사용자 경험**: 각 서비스별 세분화된 시나리오로 UX/UI 설계
- **비즈니스 로직**: 서비스별 핵심 기능 및 워크플로우 이해
- **운영 시나리오**: 실제 행사 운영 시 고려사항 파악

### 운영자를 위한 가이드
- **배포 전략**: 독립 서비스별 배포 계획 수립
- **모니터링**: 실시간 모니터링 및 장애 대응 방안
- **확장 계획**: 행사 규모 증가에 따른 인프라 확장 전략

---

## 📊 최신 구조 최적화 (2025.07.05)

### ✅ 완성된 시나리오 구조
- **32개 세분화된 시나리오 파일** (README 포함 33개)
- **4개 서비스 × 평균 8개 파일** = 체계적 구조화
- **각 파일 100-400줄** 적정 크기로 관리 효율성 극대화

### 🎯 주요 개선사항
1. **기능별 세분화**: 거대한 단일 파일을 기능별로 분할
2. **팀별 독립 작업**: 각 시스템별 명확한 담당 영역 구분
3. **레거시 통합**: _legacy 폴더 내용 완전 통합 후 정리
4. **구조 최적화**: by-system 중간 단계 제거로 직관적 접근

### 📁 최종 파일 통계
```
user-scenarios/
├── event-management/     (9개 파일) - 하이브리드, 교육, 준비, Base.md 단계
├── gate-management/      (8개 파일) - 데스크톱 관리, 보안, 분석, Base.md 하드웨어
├── integrated-platform/  (8개 파일) - AI/ML, 거버넌스, 경영진 포함
├── user-app/            (8개 파일) - 접근성, 다중행사, 수동출석, Base.md 제약사항
└── README.md            (1개 파일)
```

---

## 🎯 다음 단계

1. **구현 우선순위**
   - Phase 1: Event Management + Gate Management (단일 행사 MVP)
   - Phase 2: User App (다중 행사 지원)  
   - Phase 3: Integrated Platform (통합 분석)

2. **기술 검증**
   - BLE 비콘 프로토타입 개발
   - 오프라인 동기화 메커니즘 검증
   - 대규모 동시 접속 성능 테스트

3. **파일럿 테스트**
   - 소규모 행사에서 MVP 테스트
   - 사용자 피드백 수집 및 개선
   - 운영 프로세스 최적화

---

## 📄 추가 리소스

### 관련 문서
- **기술 스펙**: 각 서비스별 상세 기술 명세서
- **API 문서**: 서비스 간 연동 API 레퍼런스  
- **배포 가이드**: 독립 서비스 배포 및 운영 매뉴얼
- **보안 정책**: 데이터 보호 및 보안 가이드라인

### 외부 참조
- **BLE 기술 표준**: Bluetooth SIG 규격 문서
- **출석 관리 베스트 프랙티스**: 업계 표준 및 규정
- **개인정보보호**: GDPR, 개인정보보호법 준수 가이드

---

> **📝 문서 업데이트 이력**  
> - 2025-07-05: 32개 세분화 시나리오 구조 완성 및 _legacy 통합
> - 2024-01-20: 시스템 기반 구조로 전면 개편
> - 2024-01-15: 4개 독립 서비스 그룹 구조 확정
> - 2024-01-10: 다중 행사 지원 아키텍처 추가
> - 2024-01-05: 초기 사용자 중심 구조 생성

## 🎯 사용 가이드

### 프로젝트 초기 설계 시
1. 전체 시스템 개요 파악 (이 README 파일)
2. `user-scenarios/` - 각 서비스별 사용자 경험 설계
3. `technical-scenarios/` - 기술 아키텍처 설계

### 개발 단계별 참조
- **요구사항 분석**: 각 서비스별 시나리오 폴더
- **시스템 설계**: technical-scenarios 폴더
- **구현 및 테스트**: 해당 기능별 세분화된 시나리오 문서
- **운영 및 장애 대응**: preparation-scenarios.md, desktop-admin-scenarios.md

### 팀 협업 활용
- **개발팀**: 각 서비스 폴더에서 독립적 작업
- **기획팀**: personas.md와 각 시나리오로 UX 설계
- **운영팀**: operations-scenarios.md와 monitoring-scenarios.md 활용
- **교육팀**: admin-training-scenarios.md로 사용자 교육

---

## ✅ 재구성 작업 완료 사항 (2025.07.07)

### 🎭 사용자 경험 시나리오 분리
- **user-scenarios** 정비: 순수 사용자 행동과 감정에 집중
- **personas.md**: 4가지 주요 사용자 유형 정의 (Sarah, Alex, Robert, Emma)
- **onboarding.md**: 첫 행사 참가부터 앱 숙련까지의 사용자 여정
- **attendance.md**: 다양한 출석 상황에서의 실제 사용자 경험
- **edge-cases.md**: 예외 상황에서의 사용자 대응과 감정 반응

### 🔧 기술 구현 시나리오 분리  
- **system-scenarios** 재구성: API/ABI와 서비스 간 통신에 집중
- **core-apis** 구조 설계: 각 서비스별 API 명세 프레임워크
- 기술적 세부사항과 사용자 경험 완전 분리

### 🧩 공통 패턴 정리
- **common** 폴더 재정의: 비즈니스 규칙과 공유 컴포넌트 중심으로 변경
- **business-rules.md** 생성: 출석 체크, 토큰 관리, 권한 정책 등 핵심 비즈니스 로직
- 모든 서비스에서 일관되게 적용될 표준 정의

### 🔌 확장 기능 플러그인화
- **extensions** 폴더 생성: 핵심 출석 체크와 독립적인 부가 기능들
- **treasure-hunt** 플러그인 상세 설계: BLE 비콘 기반 보물찾기 게임
- 확장 기능 아키텍처와 개발 가이드라인 수립

---

## 🚀 다음 단계 (우선순위)

### 1. 시스템 시나리오 완성 (High Priority)
- [ ] Event Management Service API 명세 작성
- [ ] Gate Management Service API 명세 작성  
- [ ] User App Client API 명세 작성
- [ ] 서비스 간 통신 프로토콜 정의

### 2. 공통 패턴 구체화 (High Priority)
- [ ] BLE 통신 표준 패턴 구현 가이드
- [ ] 오프라인 동기화 메커니즘 상세 설계
- [ ] 공통 UI 컴포넌트 라이브러리 설계
- [ ] 데이터 표준 및 검증 규칙 완성

### 3. 레거시 마이그레이션 (Medium Priority)
- [ ] 기존 user-scenarios의 유용한 내용을 v2로 마이그레이션
- [ ] 중복 제거 및 일관성 검토
- [ ] 문서 간 상호 참조 정리

### 4. 확장 기능 개발 (Low Priority)
- [ ] 네트워킹 게임 플러그인 설계
- [ ] 고급 분석 대시보드 플러그인 설계
- [ ] 외부 시스템 연동 플러그인 프레임워크

---

## 📋 새로운 문서 작성 가이드라인

### 사용자 시나리오 작성 시 (user-scenarios)
- ✅ **포함할 내용**: 실제 사용자 행동, 감정 반응, 의사결정 과정, 문제 해결 방식
- ❌ **제외할 내용**: API 호출, 시스템 성능 지표, 데이터베이스 설계, 개발자 제약사항

### 시스템 시나리오 작성 시 (system-scenarios)
- ✅ **포함할 내용**: API 명세, 프로토콜 정의, 데이터 구조, 성능 요구사항, 보안 정책
- ❌ **제외할 내용**: 사용자 감정, 개인적 경험, 주관적 판단, 사용성 평가

### 공통 패턴 작성 시 (common)
- ✅ **포함할 내용**: 모든 서비스에 적용되는 규칙, 재사용 가능한 컴포넌트, 표준 정의
- ❌ **제외할 내용**: 특정 서비스에만 적용되는 로직, 임시적 해결책, 실험적 기능

---

## 💡 핵심 철학

이번 재구성의 핵심은 **관심사의 명확한 분리**입니다:

- **사용자 경험**: 사람이 시스템을 어떻게 사용하고 느끼는가?
- **기술 구현**: 시스템이 어떻게 동작하고 통신하는가?
- **비즈니스 규칙**: 무엇이 허용되고 금지되는가?
- **확장 기능**: 핵심 기능 외에 어떤 부가 가치를 제공할 것인가?

각 영역이 독립적으로 발전할 수 있으면서도, 필요할 때 서로 참조할 수 있는 구조를 만들었습니다.
