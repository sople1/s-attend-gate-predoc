# 출석 관리 API 명세

## 📌 개요

출석 관리 시스템의 핵심 API들을 정의합니다. 이 API들은 BLE 비콘 기반 자동 출석, QR 코드 출석, 수동 출석 등 다양한 출석 방식을 지원합니다.

## 🔄 API 엔드포인트

### 1. 자동 출석 체크 API

```typescript
POST /api/v1/attendance/auto

// Request
interface AutoAttendanceRequest {
  userId: string;
  eventId: string;
  timestamp: number;
  location: {
    lat: number;
    lng: number;
  };
  beaconData: {
    uuid: string;
    major: number;
    minor: number;
    rssi: number;
  };
}

// Response
interface AutoAttendanceResponse {
  status: 'success' | 'duplicate' | 'error';
  message: string;
  attendanceTime?: string;
  direction?: 'in' | 'out';
}
```

### 2. QR 코드 출석 API

```typescript
POST /api/v1/attendance/qr

// Request
interface QRAttendanceRequest {
  userId: string;
  eventId: string;
  qrToken: string;
  timestamp: number;
  location?: {
    lat: number;
    lng: number;
  };
}

// Response
interface QRAttendanceResponse {
  status: 'success' | 'duplicate' | 'error';
  message: string;
  attendanceTime?: string;
}
```

### 3. 게이트 상태 모니터링 API

```typescript
GET /api/v1/gate/{gateId}/status

// Response
interface GateStatusResponse {
  gateId: string;
  status: 'active' | 'inactive';
  lastHeartbeat: string;
  connectedBeacons: Array<{
    uuid: string;
    major: number;
    minor: number;
    batteryLevel: number;
    lastSeen: string;
  }>;
}
```

### 4. 출석 이벤트 스트림 API

```typescript
GET /api/v1/attendance/stream

// Server-Sent Events
interface AttendanceEvent {
  type: 'attendance' | 'leave' | 'error';
  userId: string;
  eventId: string;
  timestamp: string;
  location?: {
    lat: number;
    lng: number;
  };
  source: 'auto' | 'qr' | 'manual';
}
```

## 🔌 내부 시스템 간 ABI

### 1. 게이트 관리자 인터페이스

```typescript
interface IGateManager {
  // 게이트 초기화
  initialize(config: GateConfig): Promise<void>;
  
  // 비콘 신호 처리
  handleBeaconSignal(signal: BeaconSignal): void;
  
  // 입/출장 판별
  determineDirection(signals: BeaconSignal[]): 'in' | 'out';
  
  // 상태 보고
  reportStatus(): GateStatus;
}

interface BeaconSignal {
  uuid: string;
  major: number;
  minor: number;
  rssi: number;
  timestamp: number;
}
```

### 2. 이벤트 관리자 인터페이스

```typescript
interface IEventManager {
  // 이벤트 상태 확인
  checkEventStatus(eventId: string): Promise<EventStatus>;
  
  // 출석 유효성 검증
  validateAttendance(request: AttendanceRequest): Promise<ValidationResult>;
  
  // 출석 기록 저장
  recordAttendance(data: AttendanceData): Promise<void>;
}

interface EventStatus {
  active: boolean;
  gateIds: string[];
  attendanceRules: AttendanceRule[];
}
```

### 3. 통합 플랫폼 인터페이스

```typescript
interface IPlatformIntegration {
  // 데이터 동기화
  syncAttendanceData(data: AttendanceData[]): Promise<SyncResult>;
  
  // 실시간 알림 전송
  sendNotification(notification: AttendanceNotification): Promise<void>;
  
  // 분석 데이터 제공
  getAnalytics(filter: AnalyticsFilter): Promise<AnalyticsData>;
}
```

## ⚡ 성능 요구사항

### 응답 시간
- 자동 출석 API: < 500ms
- QR 코드 출석 API: < 300ms
- 실시간 스트림: < 100ms 지연

### 처리량
- 단일 게이트: 초당 최대 50건
- 전체 시스템: 초당 최대 1000건

### 가용성
- API 가용성: 99.9%
- 데이터 정확도: 99.99%

## 🔒 보안 요구사항

1. 모든 API는 SSL/TLS 1.3 사용
2. JWT 기반 인증
3. API 키 로테이션 (24시간)
4. 요청 제한 (Rate Limiting)
5. 데이터 암호화 (전송 및 저장)

## 📡 통신 프로토콜

### REST API
- HTTPS 기반
- JSON 페이로드
- HTTP/2 지원

### WebSocket
- 실시간 업데이트
- 양방향 통신
- 자동 재연결

### Server-Sent Events
- 단방향 실시간 알림
- 자동 재연결
- 이벤트 필터링
