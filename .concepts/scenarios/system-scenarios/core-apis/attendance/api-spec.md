# 출석 관리 API 사양

자동 및 수동 출석 처리를 위한 API 사양을 정의합니다.

## 기술적 개요
- RESTful API 기반 출석 관리 시스템
- 실시간 처리 및 동기화 지원
- 다중 출석 방식 통합

## 구현 세부사항

### 자동 출석 API
```json
{
  "method": "POST",
  "endpoint": "/api/v1/attendance/auto",
  "description": "BLE 비콘 기반 자동 출석 처리",
  "request": {
    "content-type": "application/json",
    "body": {
      "user_id": "string",
      "event_id": "string",
      "beacon_data": {
        "uuid": "string",
        "major": "number",
        "minor": "number",
        "rssi": "number"
      },
      "location": {
        "latitude": "number",
        "longitude": "number"
      }
    }
  },
  "response": {
    "success": {
      "status": 200,
      "body": {
        "attendance_id": "string",
        "timestamp": "string",
        "status": "success"
      }
    },
    "error": {
      "status": 400,
      "body": {
        "error": "string",
        "message": "string"
      }
    }
  }
}
```

### 수동 출석 API
```json
{
  "method": "POST",
  "endpoint": "/api/v1/attendance/manual",
  "description": "QR 코드 또는 코드 입력 기반 수동 출석",
  "request": {
    "content-type": "application/json",
    "body": {
      "user_id": "string",
      "event_id": "string",
      "code": "string",
      "type": "qr|manual"
    }
  },
  "response": {
    "success": {
      "status": 200,
      "body": {
        "attendance_id": "string",
        "timestamp": "string",
        "status": "success"
      }
    },
    "error": {
      "status": 400,
      "body": {
        "error": "string",
        "message": "string"
      }
    }
  }
}
```

## 성능 요구사항
- 응답 시간: < 500ms
- 동시 처리: 1000 req/s
- 가용성: 99.9%

## 보안 요구사항
- JWT 기반 인증
- 요청 암호화
- Rate limiting

## 통합 지점
- 사용자 인증 시스템
- 이벤트 관리 시스템
- 게이트 제어 시스템

## 오류 처리
- 재시도 로직
- 실패 로깅
- 알림 발송

## 모니터링 지표
- 응답 시간
- 오류율
- 동시 요청수
- 성공률
