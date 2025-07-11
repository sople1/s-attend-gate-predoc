# 출석 체크 API

BLE 비콘 및 수동 출석 체크를 위한 API 사양을 정의합니다.

## API 개요

여러 출석 체크 방식(BLE, QR, 수동)을 통합 관리하는 API 집합입니다.

## API 사양

### 1. BLE 자동 출석

```json
POST /api/v1/attendance/ble
{
  "user_id": "string",
  "event_id": "string",
  "beacon_data": {
    "uuid": "string",
    "major": "integer",
    "minor": "integer",
    "rssi": "integer"
  },
  "location": {
    "latitude": "number",
    "longitude": "number"
  },
  "timestamp": "datetime"
}

Response 200:
{
  "attendance_id": "string",
  "status": "success",
  "verified_at": "datetime"
}
```

### 2. QR 코드 출석

```json
POST /api/v1/attendance/qr
{
  "user_id": "string",
  "event_id": "string",
  "qr_token": "string",
  "scanned_at": "datetime"
}

Response 200:
{
  "attendance_id": "string",
  "status": "success",
  "verified_at": "datetime"
}
```

### 3. 수동 출석

```json
POST /api/v1/attendance/manual
{
  "user_id": "string",
  "event_id": "string",
  "operator_id": "string",
  "reason": "string"
}

Response 200:
{
  "attendance_id": "string",
  "status": "success",
  "recorded_at": "datetime"
}
```

## 성능 요구사항

- 응답 시간: 99%가 300ms 이내
- 처리량: 초당 100회 출석
- 동시성: 최대 500 동시 요청

## 오프라인 지원

- 오프라인 큐잉 지원
- 로컬 DB 캐싱
- 충돌 해결 전략

## 구현 참조
- [BLE 검증 로직](../implementations/ble-verification.md)
- [QR 토큰 관리](../security/qr-token.md)
- [출석 이력 관리](../data/attendance-history.md)
