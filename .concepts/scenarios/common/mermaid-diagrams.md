# s-attend-gate Mermaid 다이어그램 소스 모음

이 파일은 s-attend-gate 프로젝트의 모든 문서에서 사용된 mermaid 다이어그램의 소스 코드를 모아둔 참고 자료입니다.

## 📁 시스템 아키텍처 다이어그램 (system-scenarios/README.md)

### 전체 시스템 구조
```mermaid
graph TB
    subgraph "s-attend-gate 생태계"
        subgraph "User App (다중 행사)"
            UA[모바일 앱<br/>- BLE 자동감지<br/>- 토큰 관리<br/>- 다중 행사 지원]
        end
        
        subgraph "Gate Management (단일 행사)"
            GM[태블릿 앱<br/>- QR 스캐너<br/>- 현장 관리<br/>- 오프라인 지원]
        end
        
        subgraph "Event Management (단일 행사)"
            EM[백엔드 서비스<br/>- 참가자 DB<br/>- 토큰 발급<br/>- 실시간 추적]
        end
        
        subgraph "Integrated Platform (다중 행사)"
            IP[통합 플랫폼<br/>- 통합 분석<br/>- API 허브<br/>- 대시보드]
        end
    end
    
    UA <-->|BLE 통신| GM
    UA -.->|토큰 인증| EM
    GM <-->|API 연동| EM
    EM -.->|데이터 제공| IP
    
    style UA fill:#e1f5fe
    style GM fill:#f3e5f5
    style EM fill:#e8f5e8
    style IP fill:#fff3e0
```

### 행사별 독립 배포
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
    EM->>GM: 3. 참가자 정보 반환
    GM->>GM: 4. 출석 기록 저장
    GM->>UA: 5. 출석 확인 알림
    GM->>EM: 6. 출석 데이터 동기화
    EM->>IP: 7. 통계 데이터 전송
```

## 📊 기술 아키텍처 다이어그램

### BLE 통신 모델
```mermaid
graph LR
    subgraph "모바일 앱"
        BLEScanner[BLE 스캐너]
        ProximityManager[근접 감지 관리자]
        AttendanceController[출석 컨트롤러]
    end
    
    subgraph "게이트"
        BLEBeacon[BLE 비콘]
        GateController[게이트 컨트롤러]
    end
    
    BLEScanner -->|"1. 스캔"| BLEBeacon
    BLEBeacon -->|"2. UUID/신호강도"| BLEScanner
    BLEScanner -->|"3. 비콘 데이터"| ProximityManager
    ProximityManager -->|"4. 근접 이벤트"| AttendanceController
    AttendanceController -->|"5. 출석 요청"| GateController
```

### 토큰 인증 흐름
```mermaid
sequenceDiagram
    participant User as 사용자
    participant App as 앱
    participant Auth as 인증 서비스
    participant API as API 서비스
    
    User->>App: 1. 토큰 입력
    App->>Auth: 2. 토큰 검증 요청
    Auth->>Auth: 3. 토큰 검증
    Auth->>App: 4. JWT 발급
    App->>API: 5. API 요청 + JWT
    API->>API: 6. JWT 검증
    API->>App: 7. 리소스 접근 허용
```

## 🧩 시스템 구성요소 다이어그램

### Event Management 구성요소
```mermaid
graph TD
    subgraph "Event Management"
        API[API Gateway]
        Auth[인증 서비스]
        Event[이벤트 관리]
        Participant[참가자 관리]
        Token[토큰 서비스]
        Analytics[분석 엔진]
    end
    
    API --> Auth
    API --> Event
    API --> Participant
    API --> Token
    Auth --> Token
    Participant --> Token
    Event --> Analytics
    Participant --> Analytics
```

### User App 아키텍처
```mermaid
graph TD
    subgraph "User App Architecture"
        UI[UI Layer]
        BL[Business Logic]
        Data[Data Layer]
        
        subgraph "Core Modules"
            Auth[인증 모듈]
            Events[이벤트 모듈]
            BLE[BLE 모듈]
            Offline[오프라인 모듈]
        end
    end
    
    UI --> BL
    BL --> Data
    BL --> Auth
    BL --> Events
    BL --> BLE
    BL --> Offline
    Auth --> Data
    Events --> Data
    Offline --> Data
```

## 🔄 데이터 흐름 다이어그램

### 참가자 데이터 흐름
```mermaid
graph LR
    CSV[CSV 파일] --> Importer[데이터 임포터]
    API[외부 API] --> Importer
    Importer --> DB[(데이터베이스)]
    DB --> TokenGen[토큰 생성기]
    TokenGen --> Notifier[알림 서비스]
    Notifier --> Email[이메일]
    Notifier --> SMS[SMS]
    DB --> Analytics[분석 엔진]
```

### 오프라인 동기화 흐름
```mermaid
sequenceDiagram
    participant GM as 게이트 관리
    participant LC as 로컬 캐시
    participant ES as 이벤트 서비스
    participant DB as 중앙 DB
    
    GM->>LC: 1. 오프라인 데이터 저장
    Note over LC: 네트워크 연결 없음
    LC->>LC: 2. 주기적 동기화 시도
    Note over LC,ES: 네트워크 복구됨
    LC->>ES: 3. 누적 데이터 전송
    ES->>DB: 4. 데이터 동기화
    DB->>ES: 5. 확인 응답
    ES->>LC: 6. 동기화 완료 알림
    LC->>GM: 7. 상태 업데이트
```
