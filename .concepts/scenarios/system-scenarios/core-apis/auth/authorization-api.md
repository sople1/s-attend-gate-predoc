# 권한 관리 API

역할 기반 접근 제어(RBAC) 및 리소스 권한 관리를 위한 API 사양을 정의합니다.

## API 개요

시스템 내 모든 리소스에 대한 세밀한 접근 제어와 권한 관리를 제공합니다.

## API 사양

### 1. 역할 관리

```json
POST /api/v1/auth/roles
{
  "name": "string",
  "description": "string",
  "permissions": ["string"],
  "scope": "string"
}

Response 200:
{
  "role_id": "string",
  "name": "string",
  "permissions": ["string"],
  "created_at": "datetime",
  "updated_at": "datetime"
}
```

### 2. 권한 할당

```json
POST /api/v1/auth/assignments
{
  "user_id": "string",
  "role_id": "string",
  "resource_id": "string",
  "expires_at": "datetime"
}

Response 200:
{
  "assignment_id": "string",
  "status": "active",
  "created_at": "datetime"
}
```

### 3. 권한 검증

```json
POST /api/v1/auth/check
{
  "user_id": "string",
  "action": "string",
  "resource": "string"
}

Response 200:
{
  "allowed": "boolean",
  "reason": "string"
}
```

## 성능 요구사항

- 응답 시간: 95%가 100ms 이내
- 처리량: 초당 1000 요청
- 캐시 적중률: 99%

## 보안 요구사항

- 권한 변경은 감사 로그 필수
- 권한 캐시 TTL 최대 5분
- 중요 작업은 MFA 필수

## 구현 참조
- [RBAC 구현 상세](../implementations/rbac-implementation.md)
- [권한 캐싱 전략](../cache/permission-cache.md)
- [감사 로깅](../audit/audit-logging.md)
