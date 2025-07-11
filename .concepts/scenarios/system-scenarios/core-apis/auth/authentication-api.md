# Authentication API

BLE 기반 출석 시스템의 사용자 인증 및 권한 관리 API 사양을 정의합니다.

## API 개요

인증 토큰 발급, 갱신, 검증을 위한 RESTful API 엔드포인트들을 제공합니다.

## API 사양

### 1. 토큰 발급

```json
POST /api/v1/auth/token
{
  "grant_type": "password",
  "username": "string",
  "password": "string",
  "device_id": "string"
}

Response 200:
{
  "access_token": "string",
  "refresh_token": "string",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

### 2. 토큰 갱신

```json
POST /api/v1/auth/token/refresh
{
  "refresh_token": "string"
}

Response 200:
{
  "access_token": "string",
  "expires_in": 3600
}
```

## 성능 요구사항

- 응답 시간: 95%가 100ms 이내
- 처리량: 초당 500 요청
- 오류율: 0.1% 미만

## 보안 요구사항

- TLS 1.3 강제
- Rate limiting: IP당 분당 60 요청
- 토큰 수명: Access 1시간, Refresh 30일

## 구현 참조
- [인증 구현 상세](../implementations/auth-implementation.md)
- [보안 정책](../../security/auth-policies.md)
