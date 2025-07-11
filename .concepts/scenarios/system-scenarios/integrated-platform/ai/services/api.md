# AI Service API

통합 플랫폼의 AI 서비스 API 명세입니다.

## API 개요

AI 서비스는 다음 기능을 위한 RESTful 엔드포인트를 제공합니다:
- 모델 예측
- 학습 관리
- 상태 모니터링
- 설정 관리

## 엔드포인트

### 1. 예측 API

#### 1.1 패턴 예측
```http
POST /api/v1/ai/predict/pattern
Content-Type: application/json
Authorization: Bearer {token}

{
  "model_id": "attendance_pattern_v1",
  "input_data": {
    "user_id": "string",
    "event_id": "string",
    "timestamp": "string",
    "features": {
      "historical_attendance": ["array"],
      "event_type": "string",
      "location": {
        "latitude": "number",
        "longitude": "number"
      }
    }
  },
  "options": {
    "confidence_threshold": "number",
    "max_results": "integer"
  }
}

Response: 200 OK
{
  "predictions": [{
    "pattern": "string",
    "probability": "number",
    "confidence": "number",
    "timestamp": "string"
  }],
  "model_info": {
    "version": "string",
    "last_trained": "string",
    "accuracy": "number"
  }
}
```

#### 1.2 행동 예측
```http
POST /api/v1/ai/predict/behavior
Content-Type: application/json
Authorization: Bearer {token}

{
  "model_id": "behavior_prediction_v1",
  "user_data": {
    "user_id": "string",
    "historical_behavior": {
      "attendance_rate": "number",
      "event_preferences": ["array"],
      "interaction_history": ["array"]
    }
  },
  "event_data": {
    "event_id": "string",
    "type": "string",
    "datetime": "string"
  }
}

Response: 200 OK
{
  "predictions": {
    "attendance_probability": "number",
    "likely_behavior": "string",
    "confidence": "number"
  },
  "recommendations": [{
    "action": "string",
    "impact_score": "number"
  }]
}
```
  "features": {
    "attendance_history": ["array"],
    "user_metrics": {
      "login_frequency": "number",
      "attendance_rate": "number"
    },
    "context": {
      "event_type": "string",
      "location": "string"
    }
  }
}

Response:
{
  "prediction": {
    "value": "any",
    "confidence": "number",
    "timestamp": "string"
  },
  "model_info": {
    "version": "string",
    "last_trained": "string"
  }
}
```

### 2. 모델 관리 API

#### 2.1 모델 학습
```http
POST /api/v1/ai/models/train
Content-Type: application/json
Authorization: Bearer {token}

{
  "model_id": "string",
  "training_config": {
    "dataset_id": "string",
    "parameters": {
      "epochs": "integer",
      "batch_size": "integer",
      "learning_rate": "number"
    },
    "validation": {
      "split_ratio": "number",
      "metrics": ["array"]
    }
  }
}

Response: 202 Accepted
{
  "job_id": "string",
  "status": "string",
  "estimated_completion": "string"
}
```

#### 2.2 모델 상태 조회
```http
GET /api/v1/ai/models/{model_id}/status
Authorization: Bearer {token}

Response: 200 OK
{
  "model_id": "string",
  "status": "string",
  "version": "string",
  "metrics": {
    "accuracy": "number",
    "performance": {
      "latency_p95": "number",
      "throughput": "number"
    }
  },
  "last_updated": "string"
}
```

### 3. 모니터링 API

#### 3.1 성능 메트릭
```http
GET /api/v1/ai/monitoring/metrics
Authorization: Bearer {token}

Query Parameters:
- model_id: string (optional)
- start_time: string
- end_time: string
- metrics: array of strings

Response: 200 OK
{
  "time_range": {
    "start": "string",
    "end": "string"
  },
  "metrics": {
    "prediction_latency": [{
      "timestamp": "string",
      "value": "number"
    }],
    "accuracy": [{
      "timestamp": "string",
      "value": "number"
    }],
    "throughput": [{
      "timestamp": "string",
      "value": "number"
    }]
  }
}
```

#### 3.2 알림 설정
```http
POST /api/v1/ai/monitoring/alerts
Content-Type: application/json
Authorization: Bearer {token}

{
  "model_id": "string",
  "alert_config": {
    "metric": "string",
    "condition": {
      "type": "threshold|trend",
      "value": "number",
      "window": "string"
    },
    "notification": {
      "channels": ["email", "slack"],
      "recipients": ["array"]
    }
  }
}

Response: 201 Created
{
  "alert_id": "string",
  "status": "active"
}
```

### Training API

```json
POST /api/v1/ai/train
{
  "model_id": "string",
  "training_config": {
    "epochs": "number",
    "batch_size": "number",
    "validation_split": "number"
  }
}

Response:
{
  "job_id": "string",
  "status": "string",
  "eta": "number"
}
```

## 에러 응답

### 일반 에러 형식
```json
{
  "error": {
    "code": "string",
    "message": "string",
    "details": {
      "field": "string",
      "reason": "string"
    }
  }
}
```

### 주요 에러 코드
- 400: 잘못된 요청
- 401: 인증 필요
- 403: 권한 없음
- 404: 리소스 없음
- 429: 요청 횟수 초과
- 500: 서버 오류

## 인증 및 권한

### API 키 인증
```http
GET /api/v1/ai/auth/key
Authorization: Bearer {token}

Response: 200 OK
{
  "api_key": "string",
  "expires_at": "string",
  "permissions": ["array"]
}
```

### 권한 수준
```yaml
permissions:
  - predict:read  # 예측 요청
  - model:read    # 모델 상태 조회
  - model:write   # 모델 학습/수정
  - admin:write   # 관리자 설정
```

## 사용량 제한

### 기본 제한
- 예측 API: 1000 요청/분
- 학습 API: 10 요청/시간
- 모니터링 API: 100 요청/분

### 헤더 정보
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1625097600
```

## 통합 예시

### Python 클라이언트
```python
from s_attend_ai import AIClient

client = AIClient(api_key="your_api_key")

# 패턴 예측
prediction = client.predict_pattern(
    user_id="user123",
    event_id="event456",
    features={
        "historical_attendance": [...],
        "event_type": "conference"
    }
)

# 행동 예측
behavior = client.predict_behavior(
    user_id="user123",
    event_data={
        "type": "seminar",
        "datetime": "2025-07-15T14:00:00Z"
    }
)
```

### curl 예시
```bash
# 패턴 예측
curl -X POST "https://api.s-attend.ai/v1/ai/predict/pattern" \
     -H "Authorization: Bearer {token}" \
     -H "Content-Type: application/json" \
     -d '{
       "model_id": "attendance_pattern_v1",
       "input_data": {
         "user_id": "user123",
         "event_id": "event456"
       }
     }'
```

## 관련 문서
- [AI Core Implementation](../core/implementation.md)
- [ML Models](../ml/models.md)
- [Integration Guide](./integration.md)
