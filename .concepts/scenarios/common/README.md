# 공통 패턴 및 비즈니스 규칙

## 🎯 개요

s-attend-gate 시스템 전반에서 **공유되는 비즈니스 규칙**, **기술적 패턴**, **재사용 가능한 컴포넌트**들을 정의합니다.
각 서비스가 독립적으로 개발되더라도 일관된 동작과 품질을 보장하기 위한 표준을 제공합니다.

## 📋 구성 요소

### 🔧 [Business Rules](./business-rules.md)
- 출석 체크 비즈니스 로직
- 토큰 생성 및 검증 규칙
- 참가자 권한 관리 정책
- 데이터 보존 및 삭제 정책

### 🛠️ [Technical Patterns](./technical-patterns.md)
- BLE 통신 표준 패턴
- 오프라인 동기화 메커니즘
- API 통신 및 재시도 로직
- 캐싱 전략

### 🧩 [Shared Components](./shared-components.md)
- 공통 UI 컴포넌트 라이브러리
- 유틸리티 함수 모음
- 검증 로직 모듈
- 암호화 및 보안 모듈

### 📊 [Data Standards](./data-standards.md)
- 공통 데이터 모델
- API 응답 형식 표준
- 에러 코드 체계
- 로깅 표준

### 🌐 [Integration Patterns](./integration-patterns.md)
- 서비스 간 통신 패턴
- 이벤트 기반 아키텍처
- Message Queue 활용
- 분산 트랜잭션 처리
- **CQRS 패턴**: 읽기/쓰기 분리를 통한 성능 최적화
- **캐싱 전략**: 다층 캐싱을 통한 응답 속도 개선

---

## 🔄 서비스별 활용

### User App Service
- ✅ BLE 통합 패턴 (자동 출석 체크)
- ✅ 오프라인 동기화 (네트워크 없이도 작동)
- ✅ 보안 모듈 (사용자 인증)
- ✅ API 통신 (Event Management 연동)
- ✅ 메트릭 수집 (사용자 행동 분석)

### Gate Management Service
- ✅ BLE 통합 패턴 (비콘 및 스캐닝)
- ✅ 오프라인 동기화 (현장 네트워크 불안정 대응)
- ✅ 보안 모듈 (게이트 인증)
- ✅ API 통신 (Event Management 연동)
- ✅ 메트릭 수집 (게이트 성능 모니터링)

### Event Management Service
- ❌ BLE 통합 패턴 (사용 안함)
- ✅ 오프라인 동기화 (데이터 백업 및 복구)
- ✅ 보안 모듈 (관리자 인증 및 데이터 암호화)
- ✅ API 통신 (User App, Gate Management, Integrated Platform 연동)
- ✅ 메트릭 수집 (이벤트 성능 분석)

### Integrated Platform Service
- ❌ BLE 통합 패턴 (사용 안함)
- ✅ 오프라인 동기화 (분산 데이터 수집)
- ✅ 보안 모듈 (엔터프라이즈 보안)
- ✅ API 통신 (모든 서비스 및 외부 시스템 연동)
- ✅ 메트릭 수집 (통합 분석 및 BI)

---

## 💡 공통 패턴 적용 예시

### 1. BLE 출석 체크 플로우
```
User App (BLE 스캔) ↔ Gate Management (BLE 비콘)
     ↓ 공통 BLE 패턴 사용
Event Management (출석 데이터 저장) ← API 통신 패턴
     ↓ 메트릭 수집 패턴
Integrated Platform (실시간 분석)
```

### 2. 오프라인 모드 시나리오
```
네트워크 단절 감지 → 로컬 저장 (공통 오프라인 패턴)
     ↓
네트워크 복구 감지 → 자동 동기화 (공통 동기화 패턴)
     ↓
데이터 검증 및 병합 → 정상 동작 복구
```

### 3. 보안 토큰 전파
```
사용자 로그인 → JWT 생성 (공통 보안 패턴)
     ↓
모든 API 호출에 토큰 포함 (공통 API 패턴)
     ↓
서비스별 토큰 검증 (공통 보안 패턴)
```

---

## 🔧 기술 표준화

### 개발 언어 및 프레임워크
- **Mobile**: React Native / Flutter + TypeScript
- **Backend**: Node.js / Express.js + TypeScript
- **Database**: SQLite (로컬), PostgreSQL (서버)
- **Communication**: REST API, WebSocket, BLE

### 코드 품질 표준
- **Linting**: ESLint + Prettier
- **Testing**: Jest + React Native Testing Library
- **Documentation**: JSDoc + TypeScript
- **Version Control**: Conventional Commits

### 배포 및 운영
- **Containerization**: Docker
- **Orchestration**: Kubernetes (서버 서비스)
- **CI/CD**: GitHub Actions
- **Monitoring**: Prometheus + Grafana

이러한 공통 패턴을 통해 4개 서비스 그룹 간의 **기술적 일관성**과 **코드 재사용성**을 확보하고, **개발 효율성**과 **유지보수성**을 크게 향상시킬 수 있습니다.
