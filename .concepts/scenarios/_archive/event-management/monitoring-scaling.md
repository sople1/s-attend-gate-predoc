# Event Management - Monitoring & Auto Scaling

## 📊 모니터링 및 자동 스케일링

시스템의 실시간 모니터링과 자동 스케일링을 통한 안정적인 서비스 운영 전략을 다룹니다.

### 📋 목차

1. [실시간 메트릭 수집](#1-실시간-메트릭-수집)
2. [알림 시스템](#2-알림-시스템)
3. [수평 확장 (Auto Scaling)](#3-수평-확장-auto-scaling)
4. [예측적 스케일링](#4-예측적-스케일링)

---

## 1. 실시간 메트릭 수집

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
      errorRate: this.getErrorRate(),
      memoryUsage: process.memoryUsage(),
      cpuUsage: process.cpuUsage()
    };
  }
}

// 성능 측정 미들웨어
class PerformanceMiddleware {
  static measurePerformance() {
    return (req: Request, res: Response, next: NextFunction) => {
      const start = Date.now();

      res.on('finish', () => {
        const duration = Date.now() - start;
        const endpoint = req.route?.path || req.path;
        
        // 메트릭 수집
        metricsCollector.recordResponseTime(endpoint, duration);
        metricsCollector.incrementCounter('http_requests_total', {
          method: req.method,
          endpoint,
          status_code: res.statusCode.toString()
        });

        // 느린 요청 경고
        if (duration > 1000) {
          console.warn(`Slow request detected: ${req.method} ${endpoint} - ${duration}ms`);
        }
      });

      next();
    };
  }
}
```

## 2. 알림 시스템

### 규칙 기반 알림

```typescript
class AlertingSystem {
  private alertRules: AlertRule[] = [
    {
      name: 'high_response_time',
      condition: 'avg_response_time > 1000',
      threshold: 1000,
      duration: '5m',
      severity: 'warning'
    },
    {
      name: 'high_error_rate',
      condition: 'error_rate > 0.05',
      threshold: 0.05,
      duration: '2m',
      severity: 'critical'
    },
    {
      name: 'database_connection_failure',
      condition: 'db_connection_errors > 0',
      threshold: 0,
      duration: '1m',
      severity: 'critical'
    }
  ];

  async checkAlerts(): Promise<void> {
    for (const rule of this.alertRules) {
      const triggered = await this.evaluateRule(rule);
      
      if (triggered) {
        await this.sendAlert(rule);
      }
    }
  }

  private async sendAlert(rule: AlertRule): Promise<void> {
    const alert: Alert = {
      name: rule.name,
      severity: rule.severity,
      message: `Alert: ${rule.name} - Threshold exceeded`,
      timestamp: new Date(),
      metrics: await this.getCurrentMetrics()
    };

    // 다양한 채널로 알림 발송
    await Promise.all([
      this.sendSlackAlert(alert),
      this.sendEmailAlert(alert),
      this.updateDashboard(alert)
    ]);
  }
}
```

## 3. 수평 확장 (Auto Scaling)

### 자동 스케일링 관리

```typescript
class AutoScalingManager {
  private readonly MIN_INSTANCES = 2;
  private readonly MAX_INSTANCES = 20;
  private readonly SCALE_UP_THRESHOLD = 0.7;   // 70% CPU 사용률
  private readonly SCALE_DOWN_THRESHOLD = 0.3; // 30% CPU 사용률

  async checkScalingNeeds(): Promise<void> {
    const metrics = await this.getClusterMetrics();
    const currentInstances = await this.getCurrentInstanceCount();

    // 확장 필요성 판단
    if (metrics.cpuUtilization > this.SCALE_UP_THRESHOLD) {
      await this.scaleUp(currentInstances);
    } else if (metrics.cpuUtilization < this.SCALE_DOWN_THRESHOLD) {
      await this.scaleDown(currentInstances);
    }
  }

  private async scaleUp(currentInstances: number): Promise<void> {
    const targetInstances = Math.min(
      currentInstances + 1,
      this.MAX_INSTANCES
    );

    if (targetInstances > currentInstances) {
      console.log(`Scaling up from ${currentInstances} to ${targetInstances} instances`);
      await this.updateInstanceCount(targetInstances);
      
      // 알림 발송
      await this.notifyScalingEvent('scale-up', currentInstances, targetInstances);
    }
  }

  private async scaleDown(currentInstances: number): Promise<void> {
    const targetInstances = Math.max(
      currentInstances - 1,
      this.MIN_INSTANCES
    );

    if (targetInstances < currentInstances) {
      console.log(`Scaling down from ${currentInstances} to ${targetInstances} instances`);
      
      // Graceful shutdown을 위한 지연
      await this.drainConnections();
      await this.updateInstanceCount(targetInstances);
      
      // 알림 발송
      await this.notifyScalingEvent('scale-down', currentInstances, targetInstances);
    }
  }
}
```

## 4. 예측적 스케일링

### 트래픽 패턴 예측

```typescript
class PredictiveScaling {
  // 과거 데이터를 기반으로 부하 예측
  async predictTrafficPattern(): Promise<TrafficPrediction> {
    const historicalData = await this.getHistoricalTraffic();
    const eventSchedule = await this.getUpcomingEvents();

    // 시간대별 평균 트래픽 계산
    const hourlyPattern = this.calculateHourlyPattern(historicalData);
    
    // 행사 일정을 고려한 트래픽 예측
    const eventImpact = this.calculateEventImpact(eventSchedule);

    return {
      nextHourPrediction: this.predictNextHour(hourlyPattern, eventImpact),
      next24HoursPrediction: this.predict24Hours(hourlyPattern, eventImpact),
      recommendedScaling: this.recommendScaling(hourlyPattern, eventImpact)
    };
  }

  // 행사 시작 전 미리 확장
  async preScaleForEvent(eventId: string): Promise<void> {
    const event = await this.getEventDetails(eventId);
    const expectedParticipants = event.registeredCount;
    
    // 참가자 수 기반 인스턴스 계산
    const recommendedInstances = Math.ceil(expectedParticipants / 1000);
    const currentInstances = await this.getCurrentInstanceCount();

    if (recommendedInstances > currentInstances) {
      console.log(`Pre-scaling for event ${eventId}: ${currentInstances} -> ${recommendedInstances}`);
      await this.updateInstanceCount(recommendedInstances);
    }
  }
}
```

### 📈 성능 지표 및 목표

| 지표 | 목표 | 모니터링 방법 | 알림 임계값 |
|------|------|---------------|-------------|
| **응답 시간** | < 200ms (95%ile) | Prometheus + Grafana | > 1초 |
| **처리량** | > 10,000 TPS | 로드 테스트 | < 5,000 TPS |
| **가용성** | 99.9% | 업타임 모니터링 | < 99.5% |
| **동시 사용자** | 50,000명 | 실시간 메트릭 | > 45,000명 |
| **확장 시간** | < 2분 | 자동 스케일링 로그 | > 5분 |
| **CPU 사용률** | < 70% | 시스템 모니터링 | > 80% |
| **메모리 사용률** | < 80% | 시스템 모니터링 | > 90% |

### 🔧 관련 시나리오

- **[Event Lifecycle](./core-scenarios-lifecycle.md)**: 행사 생명주기 전반의 성능 요구사항
- **[Data Integration](./core-scenarios-data-integration.md)**: 외부 시스템과의 데이터 연동 성능
- **[Gate Management](../gate-management/core-scenarios.md)**: 게이트 시스템과의 실시간 통신 최적화
- **[Analytics Reporting](./analytics-reporting.md)**: 분석 데이터 처리 성능 최적화

## 🔗 관련 파일

### 성능 및 확장성
- [성능 최적화](./performance-optimization.md) - 데이터베이스 최적화, 캐싱 전략, 연결 풀 관리
- [확장성 아키텍처](./scalability-architecture.md) - 마이크로서비스, 메시지 큐, 부하 처리

### 시스템 성능
- [메인 성능 시나리오](./core-scenarios-performance-scalability.md) - 전체 성능 최적화 개요
