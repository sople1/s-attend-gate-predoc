# 출석 추적 API

실시간 출석 현황 추적 및 통계를 위한 API 사양을 정의합니다.

## API 개요

출석 데이터의 실시간 집계, 이력 조회, 통계 분석을 위한 API 집합입니다.

## API 사양

### 1. 실시간 현황 조회

```json
GET /api/v1/attendance/status
{
  "event_id": "string",
  "filters": {
    "from": "datetime",
    "to": "datetime",
    "status": ["string"]
  }
}

Response 200:
{
  "total_attendees": "integer",
  "checked_in": "integer",
  "pending": "integer",
  "breakdown": {
    "ble": "integer",
    "qr": "integer",
    "manual": "integer"
  },
  "latest_updates": [{
    "user_id": "string",
    "type": "string",
    "timestamp": "datetime"
  }]
}
```

### 2. 출석 이력 조회

```json
GET /api/v1/attendance/history
{
  "user_id": "string",
  "event_id": "string",
  "period": {
    "from": "datetime",
    "to": "datetime"
  }
}

Response 200:
{
  "history": [{
    "attendance_id": "string",
    "type": "string",
    "status": "string",
    "timestamp": "datetime",
    "location": {
      "latitude": "number",
      "longitude": "number"
    }
  }]
}
```

### 3. 통계 분석

```json
GET /api/v1/attendance/analytics
{
  "event_id": "string",
  "metrics": ["string"],
  "group_by": "string",
  "period": {
    "from": "datetime",
    "to": "datetime"
  }
}

Response 200:
{
  "summary": {
    "total_events": "integer",
    "total_attendees": "integer",
    "attendance_rate": "number"
  },
  "trends": [{
    "period": "string",
    "count": "integer",
    "rate": "number"
  }]
}
```

## 성능 요구사항

- 실시간 조회: 200ms 이내
- 이력 조회: 500ms 이내
- 통계 분석: 2초 이내
- 데이터 정확도: 99.99%

## 캐싱 전략

- 실시간 현황: 10초 TTL
- 이력 데이터: 1시간 TTL
- 통계 데이터: 일별 갱신

## 구현 참조
- [실시간 집계 로직](../implementations/realtime-aggregation.md)
- [통계 처리 파이프라인](../data/statistics-pipeline.md)
- [데이터 캐싱 전략](../cache/attendance-cache.md)
