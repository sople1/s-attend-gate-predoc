# 출석 추적 API

실시간 출석 현황 추적 및 관리를 위한 API 사양을 정의합니다.

## API 개요

이벤트 진행 중 참가자들의 출석 상태를 추적하고 관리하는 API 집합입니다.

## API 사양

### 1. 실시간 출석 현황 조회

```json
GET /api/v1/events/{event_id}/attendance/status
{
  "filters": {
    "status": ["checked_in", "pending", "missed"],
    "method": ["ble", "qr", "manual"]
  }
}

Response 200:
{
  "total_participants": "integer",
  "checked_in": "integer",
  "pending": "integer",
  "missed": "integer",
  "breakdown": {
    "ble": "integer",
    "qr": "integer",
    "manual": "integer"
  },
  "recent_activities": [{
    "timestamp": "datetime",
    "participant_id": "string",
    "action": "string",
    "method": "string"
  }]
}
```

### 2. 개별 참가자 출석 기록

```json
GET /api/v1/events/{event_id}/participants/{participant_id}/attendance
{
  "period": {
    "from": "datetime",
    "to": "datetime"
  }
}

Response 200:
{
  "participant_info": {
    "id": "string",
    "name": "string",
    "status": "string"
  },
  "attendance_records": [{
    "timestamp": "datetime",
    "method": "string",
    "location": {
      "latitude": "number",
      "longitude": "number"
    },
    "device_info": {
      "type": "string",
      "id": "string"
    }
  }]
}
```

### 3. 출석 상태 업데이트

```json
POST /api/v1/events/{event_id}/attendance
{
  "participant_id": "string",
  "status": "string",
  "method": "string",
  "location": {
    "latitude": "number",
    "longitude": "number"
  },
  "metadata": {
    "device_id": "string",
    "beacon_id": "string"
  }
}

Response 200:
{
  "attendance_id": "string",
  "status": "success",
  "timestamp": "datetime"
}
```

## 성능 요구사항

- 상태 조회: < 500ms
- 기록 조회: < 1초
- 상태 업데이트: < 300ms
- 동시 처리: 100건/초

## 데이터 동기화

- 실시간 업데이트
- 오프라인 캐싱
- 충돌 해결
- 데이터 정합성

## 관련 구현
- [모니터링 구현](./monitoring-implementation.md)
- [데이터 동기화](../core/data-sync.md)
- [알림 처리](../notifications/notification-handling.md)
