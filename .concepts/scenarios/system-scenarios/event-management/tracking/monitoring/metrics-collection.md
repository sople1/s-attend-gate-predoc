# Event Management - Metrics Collection

## 📊 실시간 메트릭 수집

### 메트릭 수집 시스템

```typescript
class MetricsCollector {
  private metrics = new Map<string, number>();
  private counters = new Map<string, number>();

  // 응답 시간 측정
  recordResponseTime(endpoint: string, duration: number): void {
    const key = `response_time.${endpoint}`;
    this.metrics.set(key, duration);
    
    // Prometheus 메트릭 업데이트
    this.prometheusClient.histogram({
      name: 'http_request_duration_ms',
      help: 'Duration of HTTP requests in ms',
      labelNames: ['method', 'endpoint', 'status_code'],
      buckets: [0.1, 5, 15, 50, 100, 500]
    }).observe({ endpoint }, duration);
  }

  // 요청 수 카운트
  incrementCounter(metric: string, labels: Record<string, string> = {}): void {
    const key = this.buildMetricKey(metric, labels);
    this.counters.set(key, (this.counters.get(key) || 0) + 1);
  }

  // 실시간 대시보드용 메트릭
  getRealtimeMetrics(): RealtimeMetrics {
    return {
      activeConnections: this.getActiveConnections(),
      requestsPerSecond: this.getRequestsPerSecond(),
      averageResponseTime: this.getAverageResponseTime(),
      errorRate: this.getErrorRate()
    };
  }
}
```

### 구현 고려사항

1. 메트릭 수집 최적화
   - 메모리 사용량 최소화
   - 낮은 지연 시간
   - 효율적인 저장소 사용

2. 메트릭 유형
   - 카운터 (Counters)
   - 게이지 (Gauges)
   - 히스토그램 (Histograms)
   - 요약 (Summaries)

3. 저장 및 조회
   - 시계열 데이터베이스 활용
   - 효율적인 집계 쿼리
   - 데이터 보존 정책

## 📌 참고
- [알림 시스템](/tracking/monitoring/alert-system.md)
- [자동 스케일링](/tracking/monitoring/auto-scaling.md)
- [성능 최적화](/core/performance/database-optimization.md)
