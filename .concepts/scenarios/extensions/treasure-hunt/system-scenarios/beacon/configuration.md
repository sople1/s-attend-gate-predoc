# 비콘 설정 가이드

## 설정 매개변수

### 기본 설정
- 광고 간격: 300ms
- 전송 파워: -12dBm ~ 4dBm
- 연결 모드: 비연결형
- 암호화: AES-128

### 위치 설정
- 고정 위치 좌표 등록
- 층별 매핑
- 구역 설정
- 중첩 영역 처리

### 신호 설정
- RSSI 캘리브레이션
- 거리 매핑
- 필터링 임계값
- 노이즈 제거 수준

### 배터리 관리
- 전력 소비 모니터링
- 절전 모드 설정
- 배터리 경고 임계값
- 수명 예측

## 설정 API
```json
{
  "configure_beacon": {
    "method": "POST",
    "endpoint": "/api/v1/beacons/configure",
    "payload": {
      "beacon_id": "string",
      "advertising_interval": "integer",
      "tx_power": "integer",
      "encryption_key": "string",
      "location": {
        "latitude": "number",
        "longitude": "number",
        "floor": "integer",
        "zone": "string"
      },
      "thresholds": {
        "rssi_min": "integer",
        "battery_warning": "integer"
      }
    }
  }
}
```

## 관련 문서
- [구현 세부사항](./implementation.md)
- [모니터링 시스템](./monitoring.md)
- [기술 가이드](./technical-guide.md)
