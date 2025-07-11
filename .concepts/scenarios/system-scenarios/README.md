# 시스템 시나리오 (Technical API/ABI Focus)

## 🎯 개요

s-attend-gate 시스템의 **기술적 구현과 API/ABI 인터페이스**에 집중한 시나리오 모음입니다.
사용자 경험은 [user-scenarios](../user-scenarios/)에서 다루며, 여기서는 순수한 기술적 구현과 서비스 간 통신에 초점을 맞춥니다.

## 🏗️ 시스템 아키텍처 (Technical Focus)

4개 독립 서비스의 기술적 구조와 API 인터페이스:

```mermaid
graph TB
    subgraph "API Layer"
        UAAPI[User App APIs<br/>REST + WebSocket]
        GMAPI[Gate Management APIs<br/>REST + Local Cache]
        EMAPI[Event Management APIs<br/>REST + GraphQL]
        IPAPI[Integrated Platform APIs<br/>REST + Analytics]
    end
    
    subgraph "Data Layer"
        UADB[(User App LocalDB<br/>SQLite + Sync)]
        GMDB[(Gate LocalDB<br/>SQLite + Cache)]
        EMDB[(Event Database<br/>PostgreSQL)]
        IPDB[(Analytics DB<br/>ClickHouse)]
    end
    
    subgraph "Communication Protocols"
        BLE[BLE Protocol<br/>Beacon/Scan]
        HTTP[HTTP/HTTPS<br/>REST APIs]
        WS[WebSocket<br/>Real-time Updates]
        QUEUE[Message Queue<br/>Async Processing]
    end
    
    UAAPI <--> BLE
    GMAPI <--> BLE
    UAAPI <--> HTTP
    GMAPI <--> HTTP
    EMAPI <--> HTTP
    IPAPI <--> HTTP
    
    EMAPI <--> WS
    IPAPI <--> WS
    
    EMAPI <--> QUEUE
    IPAPI <--> QUEUE
    
    UAAPI <--> UADB
    GMAPI <--> GMDB
    EMAPI <--> EMDB
    IPAPI <--> IPDB
```

---

## 📋 시나리오 카테고리

### 🔧 [Core APIs & Services](./core-apis/)
- Event Management Service API 명세
- Gate Management Service API 명세  
- User App Client API 명세
- Integrated Platform API 명세

### 🔗 [Inter-Service Communication](./communication/)
- API Gateway 패턴
- Message Queue 구조
- Real-time Sync Protocols
- Error Handling & Retry Logic

### 💾 [Data Architecture](./data-architecture/)
- Database Schema Design
- Data Synchronization Patterns
- Cache Strategies
- Backup & Recovery

### 🔒 [Security & Authentication](./security/)
- JWT Token Management
- API Rate Limiting
- Data Encryption
- Security Audit Logs

### 📊 [Performance & Monitoring](./performance/)
- System Metrics Collection
- Load Testing Scenarios
- Performance Optimization
- Alert & Monitoring

### 🚀 [Deployment & DevOps](./deployment/)
- CI/CD Pipeline Design
- Container Orchestration
- Environment Management
- Rolling Update Strategies

### 데이터 구조
```json
{
  "user": { "userId": "...", "profile": {...} },
  "events": [
    {
      "eventId": "tech-conference-2024",
      "token": "TCF24-ABCD-1234",
      "serverEndpoint": "https://tc24.events.com/api",
      "attendance": { "checkedIn": true, "time": "..." }
    }
  ]
}
```

## 🚪 Gate Management (단일 행사 특화)

### 핵심 특징
- **특정 행사만을 위한 전용 시스템**
- **현장 최적화된 태블릿 UI**
- **실시간 출석 처리 및 검증**
- **오프라인 모드 필수 지원**

### 주요 구성 요소
```
현장 도구:
├── Gate Admin Tablet App
│   ├── 터치 최적화 UI
│   ├── 참가자 지원
│   └── 실시간 모니터링
├── QR Scanner Device
│   ├── 고성능 스캔
│   ├── 자동 처리
│   └── 대량 처리 지원
└── BLE Beacon System
    ├── User App 연동
    ├── 자동 감지
    └── 근접 기반 체크인
```

### 배포 특징

```mermaid
graph TB
    subgraph "행사별 독립 배포 예시"
        subgraph "Tech Conference 2024"
            TC_EM[Event Management<br/>tc24.events.com]
            TC_GM[Gate Management<br/>태블릿 5대 + 스캐너 3대]
            TC_BLE[BLE 비콘<br/>입구 2곳]
        end
        
        subgraph "Startup Meetup 2024"
            SM_EM[Event Management<br/>sm24.events.com]
            SM_GM[Gate Management<br/>태블릿 2대 + 스캐너 1대]
            SM_BLE[BLE 비콘<br/>입구 1곳]
        end
    end
    
    subgraph "공통 서비스"
        UA[User App<br/>모든 행사 지원]
        IP[Integrated Platform<br/>통합 분석]
    end
    
    UA -.-> TC_EM
    UA -.-> SM_EM
    TC_EM -.-> IP
    SM_EM -.-> IP
    
    style TC_EM fill:#e3f2fd
    style SM_EM fill:#e8f5e8
    style UA fill:#fff3e0
    style IP fill:#fce4ec
```

## 📊 Event Management (단일 행사 백엔드)

### 핵심 특징
- **행사별 독립 서버 배포**
- **참가자 데이터 중앙 관리**
- **토큰 생성 및 검증 시스템**
- **실시간 출석 데이터 수집**

### 주요 기능
```
데이터 관리:
├── CSV/API 참가자 업로드
├── 외부 시스템 동기화
├── 참가자 정보 관리
└── 데이터 검증 및 정제

토큰 시스템:
├── 참가자별 고유 토큰 생성
├── QR 코드 자동 생성
├── 실시간 토큰 검증
└── 권한 관리

실시간 추적:
├── Gate Management 연동
├── 출석 데이터 수집
├── 중복 방지 및 검증
└── 실시간 통계 업데이트
```

### API 엔드포인트 예시
```
POST /api/participants/bulk-upload    # 참가자 대량 등록
POST /api/tokens/verify              # 토큰 검증
POST /api/attendance/checkin         # 출석 체크
GET  /api/analytics/realtime         # 실시간 현황
```

## 🌐 Integrated Platform (다중 행사 통합)

### 핵심 특징
- **여러 Event Management Service 통합**
- **크로스 이벤트 분석 및 인사이트**
- **외부 시스템 통합 API 허브**
- **경영진용 통합 대시보드**

### 주요 기능
```
통합 관리:
├── 다중 행사 실시간 모니터링
├── 통합 대시보드 (웹/모바일)
├── 행사별 성과 비교
└── 전체 현황 한눈에 보기

고급 분석:
├── 크로스 이벤트 패턴 분석
├── 참가자 행동 분석 (익명화)
├── 예측 모델링
└── ROI 분석 및 최적화

API 허브:
├── 외부 시스템 통합 API
├── 써드파티 연동 (Zapier 등)
├── 웹훅 기반 실시간 이벤트
└── 개발자 API 플랫폼
```

### 플랫폼 구조
```
외부 연동:
├── CRM 시스템 ◄────┐
├── 마케팅 도구 ◄───┤
├── 회계 시스템 ◄───┼── API Gateway
├── BI 도구 ◄───────┤
└── 자동화 도구 ◄───┘

데이터 분석:
├── 실시간 스트리밍 처리
├── 배치 분석 (야간)
├── 머신러닝 모델
└── 예측 분석 엔진
```

## 🔄 서비스 간 연동 흐름

### 참가자 온보딩 프로세스

```mermaid
sequenceDiagram
    participant Admin as 관리자
    participant EM as Event Management
    participant Email as 이메일 시스템
    participant User as 참가자
    participant UA as User App
    
    Admin->>EM: 1. 참가자 데이터 업로드 (CSV/API)
    EM->>EM: 2. 토큰 생성 및 QR 코드 생성
    EM->>Email: 3. 토큰 배포 요청
    Email->>User: 4. 앱 설치 안내 + 토큰
    User->>UA: 5. 앱 설치 및 토큰 입력
    UA->>EM: 6. 토큰 검증 요청
    EM->>UA: 7. 행사 정보 및 권한 전송
    UA->>UA: 8. 행사 추가 및 설정 완료
```

### 실시간 출석 체크 프로세스

```mermaid
sequenceDiagram
    participant UA as User App
    participant GM as Gate Management
    participant EM as Event Management
    participant IP as Integrated Platform
    
    UA->>GM: 1. BLE 통신으로 자동 감지
    GM->>EM: 2. 토큰 검증 요청
    EM->>GM: 3. 검증 결과 응답
    GM->>UA: 4. 출석 확인 알림
    EM->>EM: 5. 출석 데이터 저장
    EM->>IP: 6. 실시간 데이터 전송
    IP->>IP: 7. 통합 분석 및 대시보드 업데이트
```

### 데이터 동기화 흐름

```mermaid
graph LR
    subgraph "실시간 동기화"
        EM1[Event Management] -->|5분 주기| IP[Integrated Platform]
        EM2[Event Management 2] -->|5분 주기| IP
    end
    
    subgraph "오프라인 동기화"
        GM[Gate Management] -->|재연결시| EM1
    end
    
    subgraph "크로스 서비스"
        UA[User App] <-->|토큰 검증| EM1
        GM <-->|출석 검증| EM1
    end
```

## 🎯 독립성과 연동의 균형

### 완전한 독립성
- **Event Management**: 행사별 완전 분리
- **Gate Management**: 행사별 독립 배포
- **장애 격리**: 한 행사 문제가 다른 행사에 영향 없음

### 유연한 연동
- **User App**: 다중 행사 통합 경험
- **Integrated Platform**: 전사적 인사이트
- **API 기반**: 느슨한 결합으로 유연성 확보

### 확장 전략

```mermaid
graph TD
    A[소규모 시작] --> B[MVP 단계]
    B --> C[점진적 확장]
    C --> D[글로벌 확장]
    
    B --> B1[Event Management + Gate Management<br/>단일 행사 MVP]
    
    C --> C1[User App 추가<br/>다중 행사 지원]
    C --> C2[추가 행사별 서비스 배포]
    C --> C3[Integrated Platform<br/>통합 분석]
    
    D --> D1[지역별 Event Management 배포]
    D --> D2[다국가 Integrated Platform]
    D --> D3[글로벌 User App 서비스]
    
    style B1 fill:#e3f2fd
    style C1 fill:#e8f5e8
    style C2 fill:#e8f5e8
    style C3 fill:#fff3e0
    style D1 fill:#fce4ec
    style D2 fill:#fce4ec
    style D3 fill:#fce4ec
```

## 🚀 구현 우선순위

```mermaid
gantt
    title s-attend-gate 개발 로드맵
    dateFormat  YYYY-MM-DD
    section Phase 1: MVP (3개월)
    Event Management Service    :active, phase1-1, 2024-01-01, 60d
    Gate Management Service     :phase1-2, after phase1-1, 30d
    기본 QR 체크인 시스템        :phase1-3, after phase1-1, 45d
    section Phase 2: 사용자 경험 (2개월)
    User App 개발               :phase2-1, after phase1-2, 45d
    BLE 비콘 연동               :phase2-2, after phase2-1, 30d
    오프라인 모드 구현          :phase2-3, after phase2-1, 30d
    section Phase 3: 통합 플랫폼 (3개월)
    Integrated Platform 개발    :phase3-1, after phase2-2, 60d
    통합 대시보드 구축          :phase3-2, after phase3-1, 30d
    고급 분석 기능              :phase3-3, after phase3-1, 45d
    section Phase 4: 최적화 (지속)
    성능 최적화                 :phase4-1, after phase3-2, 30d
    사용자 경험 개선            :phase4-2, after phase4-1, 30d
    새로운 기능 추가            :phase4-3, after phase4-1, 60d
```

### Phase 1: MVP (3개월)
- Event Management Service 개발
- Gate Management Service (태블릿 앱)
- 기본 QR 체크인 시스템

### Phase 2: 사용자 경험 (2개월)
- User App 개발 (다중 행사 지원)
- BLE 비콘 연동
- 오프라인 모드 구현

### Phase 3: 통합 플랫폼 (3개월)
- Integrated Platform 개발
- 통합 대시보드 구축
- 고급 분석 기능

### Phase 4: 최적화 (지속)
- 성능 최적화
- 사용자 경험 개선
- 새로운 기능 추가

이 아키텍처는 각 서비스의 독립성을 보장하면서도 전체 시스템으로서의 시너지를 만들어내는 균형잡힌 설계입니다.
