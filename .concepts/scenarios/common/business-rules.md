# 비즈니스 규칙 (Business Rules)

## 🎯 개요

s-attend-gate 시스템의 **핵심 비즈니스 로직과 규칙**을 정의합니다.
모든 서비스와 컴포넌트에서 일관되게 적용되어야 하는 도메인 로직들을 명시합니다.

---

## 📋 출석 체크 비즈니스 규칙

### 1. 출석 인정 기준

#### BLE 자동 감지 출석
```typescript
interface BLEAttendanceRule {
  signalStrength: {
    minimum: -70, // dBm, 최소 신호 강도
    optimal: -50,  // dBm, 최적 신호 강도
    timeout: 30    // seconds, 신호 유지 시간
  };
  
  distance: {
    maximum: 3,    // meters, 최대 거리
    accuracy: 0.8  // confidence level
  };
  
  location: {
    gpsAccuracy: 50,  // meters, GPS 오차 허용 범위
    required: true    // GPS 필수 여부
  };
  
  timing: {
    eventBuffer: 30,  // minutes, 행사 시작/종료 전후 버퍼
    duplicateWindow: 15 // minutes, 중복 체크 방지 간격
  };
}
```

#### QR 코드 스캔 출석
```typescript
interface QRAttendanceRule {
  validation: {
    tokenExpiry: 300,    // seconds, QR 토큰 유효 시간
    locationMatch: true, // 위치 검증 필수
    oneTimeUse: true     // 일회용 사용
  };
  
  gateManagement: {
    tempQRExpiry: 300,   // seconds, 임시 QR 유효 시간
    manualOverride: true // 관리자 수동 승인 가능
  };
}
```

#### 수동 등록 출석
```typescript
interface ManualAttendanceRule {
  authorization: {
    requiredRole: 'GATE_MANAGER',
    reasonRequired: true,
    supervisorApproval: false // 현장에서는 즉시 승인
  };
  
  documentation: {
    reasonCodes: [
      'APP_NOT_INSTALLED',
      'BLE_MALFUNCTION', 
      'PHONE_UNAVAILABLE',
      'TECHNICAL_ISSUE',
      'ACCESSIBILITY_NEED',
      'OTHER'
    ],
    photoEvidence: false, // 선택사항
    witnessRequired: false
  };
}
```

### 2. 중복 출석 방지 규칙

```typescript
interface DuplicatePreventionRule {
  timeWindow: {
    sameMethod: 15,    // minutes, 동일 방법 재시도 간격
    differentMethod: 5, // minutes, 다른 방법 간 간격
    exitReentry: 30    // minutes, 퇴장 후 재입장 간격
  };
  
  locationCheck: {
    sameGate: 10,      // minutes, 동일 게이트 재체크
    differentGate: 0   // minutes, 다른 게이트는 즉시 가능
  };
  
  exceptionHandling: {
    managerOverride: true,    // 관리자 재정의 가능
    systemMalfunction: true,  // 시스템 오류 시 예외
    emergencyMode: true       // 비상 모드 시 예외
  };
}
```

---

## 🔐 토큰 및 인증 규칙

### 1. 참가자 토큰 생성

```typescript
interface ParticipantTokenRule {
  generation: {
    algorithm: 'RANDOM_ALPHANUMERIC',
    length: 12,
    caseStyle: 'UPPERCASE',
    excludeAmbiguous: true, // 0, O, I, L 등 제외
    uniquenessCheck: true
  };
  
  format: {
    pattern: 'EVENT2024-ABC123',
    eventPrefix: true,
    hyphenSeparated: true
  };
  
  lifecycle: {
    validFrom: 'EVENT_START_MINUS_24H',
    validUntil: 'EVENT_END_PLUS_2H',
    renewalAllowed: false,
    revokeOnSuspicious: true
  };
}
```

### 2. 토큰 검증 규칙

```typescript
interface TokenValidationRule {
  checks: {
    format: true,        // 형식 검증
    existence: true,     // DB 존재 확인
    status: true,        // 활성 상태 확인
    timeWindow: true,    // 유효 시간 확인
    rateLimit: true      // 사용 빈도 제한
  };
  
  security: {
    maxRetries: 3,       // 최대 재시도 횟수
    lockoutDuration: 300, // seconds, 잠금 시간
    suspiciousThreshold: 10, // 의심스러운 활동 임계값
    alertOnFailure: true     // 실패 시 알림
  };
  
  caching: {
    cacheDuration: 300,   // seconds, 캐시 보관 시간
    cacheValidTokens: true,
    cacheFailures: false
  };
}
```

---

## 👥 참가자 권한 관리

### 1. 참가자 상태 관리

```typescript
enum ParticipantStatus {
  ACTIVE = 'ACTIVE',           // 정상 활성
  INACTIVE = 'INACTIVE',       // 비활성 (출석 불가)
  SUSPENDED = 'SUSPENDED',     // 일시 정지
  CANCELLED = 'CANCELLED',     // 참가 취소
  PENDING = 'PENDING'          // 승인 대기
}

interface ParticipantStatusRule {
  transitions: {
    PENDING: ['ACTIVE', 'CANCELLED'],
    ACTIVE: ['INACTIVE', 'SUSPENDED', 'CANCELLED'],
    INACTIVE: ['ACTIVE', 'CANCELLED'],
    SUSPENDED: ['ACTIVE', 'CANCELLED'],
    CANCELLED: [] // 최종 상태
  };
  
  permissions: {
    ACTIVE: ['ATTEND', 'VIEW_SCHEDULE', 'UPDATE_PROFILE'],
    INACTIVE: ['VIEW_SCHEDULE'],
    SUSPENDED: [],
    CANCELLED: [],
    PENDING: ['VIEW_SCHEDULE']
  };
}
```

### 2. 개인정보 처리 규칙

```typescript
interface PrivacyRule {
  dataCollection: {
    minimumRequired: ['name', 'email', 'phone'],
    optionalFields: ['company', 'position', 'bio'],
    sensitiveData: ['location', 'device_id', 'ble_mac'],
    consentRequired: true
  };
  
  dataRetention: {
    personalData: {
      duration: '2_YEARS',
      eventData: '5_YEARS',
      anonymizedAnalytics: 'INDEFINITE'
    },
    
    deletionRules: {
      userRequest: '30_DAYS',      // 사용자 요청 시
      accountClosure: 'IMMEDIATE',  // 계정 폐쇄 시
      inactivity: '3_YEARS'        // 비활성 상태 지속
    }
  };
  
  dataSharing: {
    withinSystem: true,          // 시스템 내 서비스 간
    thirdParty: false,           // 외부 공유 금지
    anonymizedAnalytics: true,   // 익명화된 분석 데이터
    eventOrganizer: 'MINIMAL'    // 행사 주최자에게 최소한만
  };
}
```

---

## 📊 데이터 품질 관리

### 1. 데이터 검증 규칙

```typescript
interface DataValidationRule {
  participantData: {
    name: {
      minLength: 2,
      maxLength: 50,
      allowedChars: /^[a-zA-Z가-힣\s\-'\.]+$/,
      required: true
    },
    
    email: {
      format: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
      maxLength: 254,
      uniqueness: 'PER_EVENT',
      required: true
    },
    
    phone: {
      format: /^\+?[\d\s\-\(\)]+$/,
      minLength: 10,
      maxLength: 20,
      uniqueness: 'PER_EVENT',
      required: true
    }
  };
  
  eventData: {
    eventId: {
      format: /^[A-Z0-9]{4,20}$/,
      uniqueness: 'GLOBAL',
      required: true
    },
    
    coordinates: {
      latitude: { min: -90, max: 90 },
      longitude: { min: -180, max: 180 },
      required: true
    }
  };
}
```

### 2. 데이터 정합성 규칙

```typescript
interface DataIntegrityRule {
  referentialIntegrity: {
    participantEvent: true,     // 참가자-행사 연결 필수
    attendanceParticipant: true, // 출석-참가자 연결 필수
    cascadeDelete: false,       // 연쇄 삭제 금지
    orphanCleanup: true         // 고아 레코드 정리
  };
  
  temporalIntegrity: {
    attendanceTime: 'WITHIN_EVENT_PERIOD',
    registrationTime: 'BEFORE_EVENT_END',
    updateTime: 'NOT_FUTURE',
    deletionTime: 'AFTER_CREATION'
  };
  
  businessIntegrity: {
    duplicateAttendance: false,  // 중복 출석 방지
    futureEvents: true,         // 미래 행사 등록 가능
    pastEvents: false,          // 과거 행사 신규 등록 불가
    statusTransition: true      // 상태 전환 규칙 준수
  };
}
```

---

## ⚠️ 예외 처리 및 비상 상황

### 1. 시스템 장애 대응

```typescript
interface EmergencyRule {
  systemFailure: {
    offlineMode: {
      enabled: true,
      maxDuration: '4_HOURS',
      localStorage: true,
      syncOnRecovery: true
    },
    
    manualOverride: {
      gateManagerAuth: true,
      reasonRequired: true,
      auditTrail: true
    },
    
    dataRecovery: {
      backupFrequency: '15_MINUTES',
      rollbackCapability: true,
      conflictResolution: 'MANUAL_REVIEW'
    }
  };
  
  networkIssues: {
    retryStrategy: 'EXPONENTIAL_BACKOFF',
    maxRetries: 5,
    fallbackToLocal: true,
    queuePendingRequests: true
  };
}
```

### 2. 보안 사고 대응

```typescript
interface SecurityIncidentRule {
  detection: {
    multipleFailedAttempts: 5,   // 연속 실패 임계값
    suspiciousLocation: true,     // 위치 이상 감지
    timePatternAnomaly: true,     // 시간 패턴 이상
    deviceFingerprinting: true    // 디바이스 특성 추적
  };
  
  response: {
    automaticLockout: true,       // 자동 잠금
    alertSecurity: true,          // 보안팀 알림
    auditLogEntry: true,          // 감사 로그 기록
    managerNotification: true     // 관리자 알림
  };
  
  recovery: {
    manualUnlock: true,           // 수동 잠금 해제
    identityVerification: true,   // 신원 확인
    reasonDocumentation: true,    // 사유 문서화
    monitoringPeriod: '24_HOURS'  // 모니터링 기간
  };
}
```

---

## 📈 성능 및 확장성 규칙

### 1. 성능 기준

```typescript
interface PerformanceRule {
  responseTime: {
    attendanceCheck: { max: 500, unit: 'ms' },
    participantLookup: { max: 300, unit: 'ms' },
    tokenValidation: { max: 200, unit: 'ms' },
    reportGeneration: { max: 5000, unit: 'ms' }
  };
  
  throughput: {
    concurrentUsers: 1000,        // 동시 사용자
    attendancePerSecond: 50,      // 초당 출석 처리
    apiCallsPerMinute: 10000,     // 분당 API 호출
    dataExportSize: '100MB'       // 최대 내보내기 크기
  };
  
  resource: {
    memoryPerUser: '2MB',         // 사용자당 메모리
    databaseConnections: 20,      // DB 연결 수
    diskSpaceGrowth: '1GB/1000_users', // 디스크 사용량
    networkBandwidth: '10Mbps'    // 네트워크 대역폭
  };
}
```

### 2. 확장성 기준

```typescript
interface ScalabilityRule {
  horizontal: {
    autoScaling: true,            // 자동 확장
    maxInstances: 10,             // 최대 인스턴스
    scaleUpThreshold: 70,         // 확장 임계값 (CPU %)
    scaleDownThreshold: 30,       // 축소 임계값 (CPU %)
    cooldownPeriod: 300           // 대기 시간 (seconds)
  };
  
  vertical: {
    memoryLimit: '8GB',           // 최대 메모리
    cpuLimit: '4_CORES',          // 최대 CPU
    diskLimit: '100GB',           // 최대 디스크
    upgradeWindow: 'MAINTENANCE'  // 업그레이드 시간
  };
  
  database: {
    readReplicas: 3,              // 읽기 복제본
    shardingStrategy: 'BY_EVENT', // 샤딩 전략
    connectionPooling: true,      // 연결 풀링
    queryOptimization: true       // 쿼리 최적화
  };
}
```

---

## 💡 비즈니스 규칙 적용 가이드

### 개발 시 고려사항
1. **규칙 우선순위**: 보안 > 개인정보 > 비즈니스 로직 > 성능
2. **예외 처리**: 모든 규칙에 대한 적절한 예외 상황 고려
3. **감사 추적**: 중요한 비즈니스 규칙 실행 시 로그 기록
4. **설정 가능성**: 가능한 한 설정을 통해 조정 가능하도록 구현

### 테스트 전략
1. **단위 테스트**: 각 규칙별 개별 테스트 케이스
2. **통합 테스트**: 규칙 간 상호작용 테스트
3. **성능 테스트**: 성능 기준 준수 여부 검증
4. **보안 테스트**: 보안 규칙 우회 가능성 검토

### 모니터링
1. **규칙 위반 감지**: 실시간 모니터링 및 알림
2. **성능 지표**: 각 규칙의 성능 영향도 측정
3. **비즈니스 메트릭**: 규칙 적용 효과 측정
4. **개선 기회**: 규칙 최적화 가능성 식별
