# 이벤트 생성 API

이벤트 생성 및 기본 설정을 위한 API 사양을 정의합니다.

## API 개요

이벤트의 기본 정보, 참가자 제한, 위치, 시간 등을 설정하는 API 집합입니다.

## API 사양

### 1. 이벤트 생성

```json
POST /api/v1/events
{
  "title": "string",
  "description": "string",
  "type": "string",
  "capacity": "integer",
  "datetime": {
    "start": "datetime",
    "end": "datetime"
  },
  "location": {
    "name": "string",
    "address": "string",
    "coordinates": {
      "latitude": "number",
      "longitude": "number"
    }
  },
  "attendance_policy": {
    "methods": ["ble", "qr", "manual"],
    "check_in_window": "integer",
    "auto_close": "boolean"
  }
}

Response 200:
{
  "event_id": "string",
  "status": "draft",
  "created_at": "datetime",
  "access_token": "string"
}
```

### 2. 이벤트 설정 업데이트

```json
PATCH /api/v1/events/{event_id}
{
  "title": "string",
  "description": "string",
  "capacity": "integer",
  "datetime": {
    "start": "datetime",
    "end": "datetime"
  }
}

Response 200:
{
  "event_id": "string",
  "updated_at": "datetime"
}
```

### 3. 이벤트 상태 변경

```json
POST /api/v1/events/{event_id}/status
{
  "status": "string",
  "reason": "string"
}

Response 200:
{
  "event_id": "string",
  "status": "string",
  "changed_at": "datetime"
}
```

## 성능 요구사항

- 생성 응답 시간: < 2초
- 설정 변경: < 1초
- 상태 변경: < 500ms
- 동시 생성: 10건/초

## 구현 고려사항

- 중복 생성 방지
- 설정 유효성 검증
- 상태 전이 규칙 적용
- 권한 검증

## 관련 구현
- [생명주기 관리](./event-lifecycle-implementation.md)
- [출석 정책](../tracking/attendance-policy.md)
- [알림 설정](../notifications/notification-settings.md)
