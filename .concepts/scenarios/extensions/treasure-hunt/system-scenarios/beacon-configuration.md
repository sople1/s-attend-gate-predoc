# BLE 비콘 설정 및 배포

이 문서는 보물찾기 게임을 위한 BLE 비콘의 기술적 설정과 행사장 내 효과적인 배포 전략을 설명합니다.

## 기술적 개요

BLE 비콘 시스템은 행사장 전체에 전략적으로 배치된 저전력 Bluetooth 비콘을 통해 정교한 실내 위치 기반 경험을 제공합니다. 각 비콘은 고유 식별자를 브로드캐스트하며, 모바일 앱이 이를 감지하여 참가자의 위치와 게임 진행 상황을 추적합니다.

## 구현 세부사항

### 비콘 하드웨어 사양

```json
{
  "beacon_types": [
    {
      "name": "Standard Beacon",
      "protocol": "iBeacon",
      "battery_life": "12 months",
      "range": "10-15 meters",
      "broadcast_interval": "350ms",
      "tx_power": "-12 to -4 dBm",
      "waterproof": true,
      "use_case": "General location markers"
    },
    {
      "name": "Micro Beacon",
      "protocol": "Eddystone",
      "battery_life": "6 months",
      "range": "5-8 meters",
      "broadcast_interval": "500ms",
      "tx_power": "-20 to -12 dBm",
      "waterproof": false,
      "use_case": "Hidden treasure points"
    },
    {
      "name": "Long-range Beacon",
      "protocol": "iBeacon + Eddystone",
      "battery_life": "8 months",
      "range": "25-50 meters",
      "broadcast_interval": "300ms",
      "tx_power": "0 to +4 dBm",
      "waterproof": true,
      "use_case": "Entry/exit points, large halls"
    }
  ]
}
```

### 비콘 구성 매개변수

```javascript
// 비콘 구성 코드
function configureBeacon(beaconId, type, location, questData) {
  // 기본 설정
  const config = {
    uuid: generateUUID(beaconId, EVENT_ID),
    major: calculateMajorValue(location.zone),
    minor: calculateMinorValue(location.subZone, beaconId),
    txPower: getTxPowerForLocation(location),
    advertisingInterval: getOptimalInterval(type, location),
    
    // 게임 데이터
    questId: questData.id,
    pointValue: questData.points,
    difficultyLevel: questData.difficulty,
    
    // 성능 최적화
    batteryOptimizationMode: shouldOptimizeBattery(location, type),
    encryptPayload: questData.isHidden,
    securityToken: generateSecurityToken(beaconId, EVENT_ID)
  };
  
  // 위치 기반 특수 설정
  if (location.isHighTraffic) {
    config.advertisingInterval = 200; // 더 빠른 감지를 위해
    config.txPower = increaseTxPowerBySteps(config.txPower, 2);
  }
  
  if (questData.isTimeRestricted) {
    config.activationSchedule = {
      start: questData.activeFrom,
      end: questData.activeUntil,
      timezone: EVENT_TIMEZONE
    };
  }
  
  return config;
}
```

### 비콘 데이터 구조

```typescript
interface BeaconData {
  // 식별 정보
  id: string;
  name: string;
  type: BeaconType;
  
  // 위치 정보
  location: {
    zone: string;        // 예: "expo-hall", "conference-room-a"
    subZone: string;     // 예: "entrance", "sponsor-area-north"
    coordinates: {       // 상대적 실내 좌표
      x: number;         // 미터 단위
      y: number;
      floor: number;
    };
    description: string; // 사람이 읽을 수 있는 위치 설명
  };
  
  // 게임 관련 정보
  gameData: {
    questId: string;
    pointValue: number;
    hintText: string;
    discoveryMessage: string;
    unlocksChallengeId?: string;
    requiredPrecedingBeacons: string[];
    timeRestriction?: {
      start: string;     // ISO 시간 형식
      end: string;
      recurrence?: string; // RRULE 형식 (예: "FREQ=DAILY")
    };
  };
  
  // 기술 설정
  technical: {
    uuid: string;
    major: number;
    minor: number;
    txPower: number;
    advertisingInterval: number;
    batteryLevel: number; // 퍼센트
    lastChecked: string;  // ISO 날짜
    firmwareVersion: string;
  };
}
```

## 배포 전략

### 행사장 매핑 및 비콘 배치

```typescript
function createBeaconDeploymentPlan(venueMap, gameDesign) {
  const deploymentPlan = [];
  
  // 1. 핵심 이동 경로에 기본 비콘 배치
  venueMap.mainPaths.forEach(path => {
    const optimalPositions = calculatePathBeacons(
      path.start, path.end, path.width, MAX_BEACON_DISTANCE
    );
    
    optimalPositions.forEach(position => {
      deploymentPlan.push({
        type: "Standard",
        location: position,
        purpose: "Navigation",
        questTier: "Basic",
        installationHeight: "2.1m",
        fixationType: "3M Command Strip"
      });
    });
  });
  
  // 2. 주요 관심 지점에 특별 비콘 배치
  venueMap.pointsOfInterest.forEach(poi => {
    if (poi.type === "SponsorBooth" && poi.tier === "Gold") {
      deploymentPlan.push({
        type: "Standard",
        location: poi.location,
        purpose: "Sponsor Engagement",
        questTier: "Sponsor",
        installationHeight: "1.2m",
        fixationType: "Desk Mount"
      });
    } else if (poi.type === "Refreshment") {
      deploymentPlan.push({
        type: "Micro",
        location: poi.location,
        purpose: "Engagement",
        questTier: "Social",
        installationHeight: "1.8m",
        fixationType: "Ceiling Mount"
      });
    }
  });
  
  // 3. 숨겨진 보물 비콘 배치
  gameDesign.hiddenTreasures.forEach(treasure => {
    deploymentPlan.push({
      type: "Micro",
      location: findOptimalHidingSpot(venueMap, treasure.difficulty),
      purpose: "Hidden Treasure",
      questTier: "Advanced",
      installationHeight: calculateVariableHeight(0.6, 2.4),
      fixationType: "Concealed Mount"
    });
  });
  
  // 4. 넓은 공간에 장거리 비콘 배치
  venueMap.largeAreas.forEach(area => {
    const center = calculateCentroid(area.boundaries);
    deploymentPlan.push({
      type: "Long-range",
      location: center,
      purpose: "Area Coverage",
      questTier: "Basic",
      installationHeight: "3.0m",
      fixationType: "Ceiling Mount"
    });
  });
  
  // 5. 신호 충돌 확인 및 최적화
  return optimizeForSignalConflicts(deploymentPlan, venueMap);
}
```

### 신호 강도 및 범위 최적화

비콘 간 최적의 거리와 신호 강도는 다음 요소에 따라 결정됩니다:

- **물리적 장애물**: 벽, 대형 구조물, 전자 장비
- **인구 밀도**: 사람이 많을수록 신호 감쇠 증가
- **주변 RF 환경**: WiFi, 다른 블루투스 기기의 간섭
- **배치 높이**: 이상적으로 지상 2-2.5m 높이
- **전송 전력**: 배터리 수명과 범위 사이의 균형

다음 표는 환경별 최적 설정을 나타냅니다:

| 환경 유형 | 권장 간격 | TX 전력 | 광고 간격 |
|----------|---------|---------|---------|
| 개방 홀 | 15-20m | -8 dBm | 350ms |
| 부스 지역 | 8-12m | -12 dBm | 400ms |
| 좁은 복도 | 10-15m | -16 dBm | 450ms |
| 식사/휴식 공간 | 5-8m | -12 dBm | 300ms |
| 입구/출구 | 3-5m | -4 dBm | 250ms |

## 구성

### 배터리 수명 최적화

```yaml
battery_optimization:
  # 동적 전송 전력 조정
  dynamic_tx_power: true
  high_traffic_hours: ["10:00-12:00", "15:00-17:00"]
  low_power_hours: ["00:00-08:00"]  # 행사장 폐쇄 시간
  
  # 광고 간격 설정
  default_advertising_interval_ms: 400
  high_traffic_interval_ms: 300
  low_power_interval_ms: 1000
  
  # 예약 모드
  scheduled_activation: true
  event_days: ["2025-03-15", "2025-03-16", "2025-03-17"]
  daily_hours: "08:00-19:00"
```

### 비콘 모니터링 및 유지보수

- **실시간 상태 모니터링**: 배터리 수준, 연결성, 올바른 브로드캐스팅
- **자동 경고**: 배터리 부족, 오작동, 위치 변경 시 알림
- **현장 지원 도구**: 기술 담당자를 위한 모바일 관리 앱
- **빠른 교체 프로토콜**: 5분 이내 문제 비콘 교체 절차

## 통합 지점

- **모바일 앱**: Core Bluetooth/BLE 라이브러리를 통한 감지
- **이벤트 관리 시스템**: 행사장 맵 및 게임 로직 연동
- **관리자 대시보드**: 비콘 상태 및 발견 패턴 모니터링
- **게임 서버**: 비콘 발견 이벤트 처리 및 보상 발급

## 모니터링 및 지표

### 주요 성능 지표

- **감지 성공률**: 비콘 근처에 있을 때 앱이 감지하는 비율 (목표: > 95%)
- **감지 지연 시간**: 범위 내 진입 후 감지까지 소요 시간 (목표: < 3초)
- **배터리 수명**: 실제 사용 중 배터리 소모율 (목표: > 6개월)
- **위치 정확도**: 실제 위치와 추정 위치 간의 오차 (목표: < 2m)

### 게임 효과성 지표

- **발견 분포**: 각 비콘별 발견 비율 및 패턴
- **완료율**: 전체 비콘 대비 평균 발견된 비콘 비율
- **참여 시간**: 게임으로 인해 증가한 행사장 탐색 시간
- **스폰서 참여**: 스폰서 관련 비콘 발견으로 인한 부스 방문 증가율

## 관련 문서

- [비콘 감지 알고리즘](./beacon-detection.md)
- [게임 진행 추적 시스템](./progress-tracking.md)
- [보물찾기 게임 흐름](../mermaid-diagrams.md)
