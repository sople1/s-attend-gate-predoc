# BLE 스캐닝 구현

모바일 앱의 BLE 비콘 스캐닝 및 근접 감지 구현 상세 사양입니다.

## 기술적 개요

근거리 BLE 비콘을 감지하고 신호 강도를 기반으로 거리를 계산하여 자동 출석을 처리합니다.

## 구현 세부사항

### 스캐닝 설정
```typescript
const scanConfig = {
  scanMode: BleScanMode.BALANCED,
  scanInterval: 1000, // ms
  rssiThreshold: -80,
  distanceThreshold: 5, // meters
  beaconFilter: {
    serviceUUIDs: ['0000FFFF-0000-1000-8000-00805F9B34FB'],
    manufacturerData: [0x004C, 0x02, 0x15] // iBeacon
  }
};
```

### 거리 계산 알고리즘
```typescript
function calculateDistance(rssi: number, txPower: number): number {
  if (rssi === 0) return -1;
  const ratio = rssi * 1.0 / txPower;
  return ratio < 1.0 
    ? Math.pow(ratio, 10) 
    : (0.89976) * Math.pow(ratio, 7.7095) + 0.111;
}
```

## 구성 매개변수
- 스캔 모드: BALANCED
- 스캔 간격: 1초
- RSSI 임계값: -80dBm
- 거리 임계값: 5m

## 통합 지점
- BLE Manager 서비스
- 위치 서비스
- 백그라운드 작업 스케줄러

## 모니터링 및 지표
- 스캔 성공률: 95% 이상
- 거리 계산 정확도: ±1m
- 배터리 소비: 시간당 3% 미만

## 관련 구현
- [출석 체크 API](../apis/attendance-check-api.md)
- [위치 서비스](./location-service-implementation.md)
