# ETL 프로세스 기술적 구현

이 문서는 고급 분석 대시보드를 위한 데이터 추출, 변환, 적재(ETL) 프로세스의 기술적 구현을 설명합니다.

## 기술적 개요

ETL 파이프라인은 다양한 소스(User App, Gate Management, Event Management, Integrated Platform)에서 데이터를 수집, 정제, 변환하여 분석에 최적화된 형태로 OLAP 데이터베이스에 적재합니다. 실시간 처리와 배치 처리를 혼합한 하이브리드 방식으로 운영됩니다.

## 구현 세부사항

### 데이터 소스 연결

```json
{
  "data_sources": [
    {
      "source_id": "user_app",
      "type": "websocket",
      "endpoint": "wss://api.s-attend-gate.com/user-app/events",
      "auth": "bearer_token",
      "real_time": true,
      "batch_schedule": "1h",
      "data_schema": "user_app_schema_v2.json"
    },
    {
      "source_id": "gate_management",
      "type": "rest_api",
      "endpoint": "https://api.s-attend-gate.com/gate/metrics",
      "auth": "api_key",
      "real_time": false,
      "batch_schedule": "5m",
      "data_schema": "gate_metrics_schema.json"
    },
    {
      "source_id": "event_management",
      "type": "kafka",
      "endpoint": "kafka://events.s-attend-gate.com:9092/event-data",
      "auth": "sasl_scram",
      "real_time": true,
      "batch_schedule": null,
      "data_schema": "event_data_schema.json"
    },
    {
      "source_id": "integrated_platform",
      "type": "rest_api",
      "endpoint": "https://api.s-attend-gate.com/platform/analytics",
      "auth": "oauth2",
      "real_time": false,
      "batch_schedule": "1h",
      "data_schema": "platform_analytics_schema.json"
    }
  ]
}
```

### 데이터 변환 파이프라인

```python
def transform_attendance_data(raw_data):
    """
    출석 데이터 변환 및 보강 로직
    
    - 타임스탬프 정규화 (UTC 기준)
    - 사용자 식별자 익명화
    - 메타데이터 보강 (위치, 디바이스 정보)
    - 세션 및 행사 정보 연결
    - 파생 지표 계산 (체류 시간, 참여도 점수)
    """
    transformed_data = {}
    
    # 타임스탬프 정규화
    transformed_data["timestamp_utc"] = normalize_timestamp(raw_data["timestamp"])
    
    # 사용자 식별자 익명화
    transformed_data["user_hash"] = anonymize_user_id(raw_data["user_id"])
    
    # 메타데이터 보강
    transformed_data["metadata"] = enrich_metadata(
        raw_data["device_info"],
        raw_data["location"]
    )
    
    # 세션 및 행사 정보 연결
    transformed_data["event_context"] = link_event_context(
        raw_data["event_id"],
        raw_data["timestamp_utc"]
    )
    
    # 파생 지표 계산
    transformed_data["derived_metrics"] = calculate_derived_metrics(
        raw_data,
        transformed_data["event_context"]
    )
    
    return transformed_data
```

### 저장소 적재 전략

- **실시간 데이터**: ClickHouse의 MergeTree 엔진 활용, 파티셔닝 키로 이벤트 ID와 날짜 사용
- **집계 데이터**: 시간, 일, 주, 월 단위 사전 집계 테이블 유지
- **차원 데이터**: 이벤트, 사용자, 위치 등 차원 정보를 별도 테이블로 관리
- **데이터 생명주기**: 세부 원시 데이터 3개월 보관, 집계 데이터 3년 보관

## 구성

### 성능 튜닝

```yaml
etl_config:
  # 스케일링 설정
  workers: 8
  max_workers: 24
  autoscale: true
  
  # 성능 최적화
  batch_size: 5000
  buffer_size_mb: 256
  flush_interval_sec: 15
  
  # 오류 처리
  retry_attempts: 3
  retry_backoff_ms: 1000
  dead_letter_queue: "dlq-analytics"
  
  # 모니터링
  metrics_interval_sec: 30
  health_check_endpoint: "/health"
```

### 장애 복구 메커니즘

1. **점진적 재시도**: 오류 발생 시 지수 백오프로 최대 3회 재시도
2. **데드레터 큐**: 지속적 실패 데이터는 별도 저장소에 보관 후 수동 검토
3. **체크포인트**: 5분 단위 처리 상태 저장으로 장애 시 복구 지점 확보
4. **부분 처리**: 배치 내 일부 레코드 실패 시 성공 항목만 커밋

## 통합 지점

- **모니터링 시스템**: Prometheus와 Grafana를 통한 ETL 성능 메트릭 제공
- **알림 시스템**: 중요 임계치 초과 시 PagerDuty로 알림 전송
- **로깅 시스템**: Elasticsearch에 구조화된 로그 저장, Kibana로 조회
- **데이터 품질**: Great Expectations 프레임워크로 데이터 검증 및 문서화

## 모니터링 및 지표

### 주요 성능 지표

- **처리 지연 시간**: 데이터 소스에서 적재까지 소요 시간 (목표: P95 < 5초)
- **처리량**: 초당 처리 레코드 수 (목표: 5,000 RPS)
- **오류율**: 전체 레코드 중 처리 실패 비율 (목표: < 0.1%)
- **리소스 사용률**: CPU, 메모리, 디스크 I/O 사용량

### 데이터 품질 지표

- **완전성**: 누락 필드 비율 (목표: < 1%)
- **정확성**: 스키마 검증 통과율 (목표: > 99.9%)
- **일관성**: 중복 레코드 비율 (목표: < 0.01%)
- **적시성**: 예상 대비 데이터 도착 지연 (목표: < 2분)

## 관련 문서

- [OLAP 저장소 아키텍처](./olap-storage.md)
- [데이터 동기화 메커니즘](./data-sync-mechanism.md)
- [데이터 파이프라인 다이어그램](../mermaid-diagrams.md)
