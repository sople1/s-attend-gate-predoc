# 비콘 모니터링 시스템

## 모니터링 지표

### 상태 지표
- 온라인/오프라인 상태
- 배터리 잔량
- 신호 강도
- 마지막 응답 시간

### 성능 지표
- 신호 안정성
- 응답 지연시간
- 신호 도달 범위
- 간섭 수준

### 오류 지표
- 통신 오류율
- 신호 손실 빈도
- 비정상 동작 횟수
- 배터리 경고 횟수

### 사용량 지표
- 감지 횟수
- 유니크 사용자 수
- 피크 시간대
- 구역별 활성도

## 모니터링 API
```json
{
  "get_beacon_status": {
    "method": "GET",
    "endpoint": "/api/v1/beacons/status",
    "parameters": {
      "beacon_id": "string"
    },
    "response": {
      "status": "string",
      "battery_level": "number",
      "rssi": "integer",
      "last_seen": "timestamp",
      "error_rate": "number",
      "active_users": "integer"
    }
  }
}
```

## 관련 문서
- [비콘 설정](./configuration.md)
- [구현 세부사항](./implementation.md)
- [기술 가이드](./technical-guide.md)
