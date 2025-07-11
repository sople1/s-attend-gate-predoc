# Common Technical Patterns - 메트릭 수집 및 분석

## 📊 개요

모든 서비스에서 공통으로 사용되는 메트릭 수집, 성능 모니터링, 사용자 행동 분석 패턴을 정의합니다.

**주요 포함 내용:**
- 메트릭 수집 시스템
- 성능 모니터링
- 사용자 행동 분석
- 실시간 알림 시스템

---

## 📊 공통 분석 및 메트릭 수집

### 모든 서비스에서 사용하는 표준 메트릭 시스템

#### 메트릭 수집기

```typescript
// common/analytics/MetricsCollector.ts
class MetricsCollector {
  private metrics: Map<string, Metric[]> = new Map();
  private flushInterval: number = 60000; // 1분
  private batchSize: number = 100;
  
  constructor(private config: MetricsConfig) {
    this.startAutoFlush();
  }
  
  recordEvent(name: string, data: Record<string, any>): void {
    const metric: Metric = {
      name,
      timestamp: new Date().toISOString(),
      data,
      serviceId: this.config.serviceId,
      version: this.config.version
    };
    
    if (!this.metrics.has(name)) {
      this.metrics.set(name, []);
    }
    
    this.metrics.get(name)!.push(metric);
    
    // 배치 크기 초과 시 즉시 플러시
    if (this.getTotalMetricsCount() >= this.batchSize) {
      this.flush();
    }
  }
  
  recordPerformance(operation: string, duration: number, metadata?: any): void {
    this.recordEvent('performance', {
      operation,
      duration,
      metadata
    });
  }
  
  recordError(error: Error, context?: any): void {
    this.recordEvent('error', {
      message: error.message,
      stack: error.stack,
      name: error.name,
      context
    });
  }
  
  recordBusinessEvent(eventType: string, data: any): void {
    this.recordEvent('business_event', {
      eventType,
      data
    });
  }
  
  recordUserInteraction(action: string, element: string, metadata?: any): void {
    this.recordEvent('user_interaction', {
      action,
      element,
      metadata
    });
  }
  
  async flush(): Promise<void> {
    if (this.metrics.size === 0) return;
    
    const allMetrics = Array.from(this.metrics.entries()).flatMap(
      ([name, metrics]) => metrics
    );
    
    try {
      await this.sendMetrics(allMetrics);
      this.metrics.clear();
    } catch (error) {
      console.error('Failed to flush metrics:', error);
      // 메트릭 전송 실패 시 로컬에 저장
      await this.saveMetricsLocally(allMetrics);
    }
  }
  
  private async sendMetrics(metrics: Metric[]): Promise<void> {
    // Integrated Platform으로 메트릭 전송
    await fetch(`${this.config.platformURL}/api/metrics`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.config.apiKey}`
      },
      body: JSON.stringify({
        serviceId: this.config.serviceId,
        metrics,
        timestamp: new Date().toISOString()
      })
    });
  }
  
  private async saveMetricsLocally(metrics: Metric[]): Promise<void> {
    // 로컬 저장소에 메트릭 저장 (나중에 재시도)
    const localStorage = await this.getLocalStorage();
    
    for (const metric of metrics) {
      await localStorage.setItem(
        `metric_${Date.now()}_${Math.random()}`,
        JSON.stringify(metric)
      );
    }
  }
  
  private startAutoFlush(): void {
    setInterval(() => {
      this.flush().catch(console.error);
    }, this.flushInterval);
  }
  
  private getTotalMetricsCount(): number {
    return Array.from(this.metrics.values()).reduce(
      (total, metrics) => total + metrics.length,
      0
    );
  }
}
```

#### 성능 모니터링

```typescript
// common/analytics/PerformanceMonitor.ts
class PerformanceMonitor {
  private metricsCollector: MetricsCollector;
  private activeTimers: Map<string, number> = new Map();
  
  constructor(metricsCollector: MetricsCollector) {
    this.metricsCollector = metricsCollector;
  }
  
  startTimer(operationId: string): void {
    this.activeTimers.set(operationId, performance.now());
  }
  
  endTimer(operationId: string, metadata?: any): number {
    const startTime = this.activeTimers.get(operationId);
    if (!startTime) {
      console.warn(`Timer not found for operation: ${operationId}`);
      return 0;
    }
    
    const duration = performance.now() - startTime;
    this.activeTimers.delete(operationId);
    
    this.metricsCollector.recordPerformance(operationId, duration, metadata);
    
    return duration;
  }
  
  async measureAsync<T>(
    operationName: string,
    operation: () => Promise<T>,
    metadata?: any
  ): Promise<T> {
    const startTime = performance.now();
    
    try {
      const result = await operation();
      const duration = performance.now() - startTime;
      
      this.metricsCollector.recordPerformance(operationName, duration, {
        ...metadata,
        status: 'success'
      });
      
      return result;
    } catch (error) {
      const duration = performance.now() - startTime;
      
      this.metricsCollector.recordPerformance(operationName, duration, {
        ...metadata,
        status: 'error',
        error: error.message
      });
      
      throw error;
    }
  }
  
  measureSync<T>(
    operationName: string,
    operation: () => T,
    metadata?: any
  ): T {
    const startTime = performance.now();
    
    try {
      const result = operation();
      const duration = performance.now() - startTime;
      
      this.metricsCollector.recordPerformance(operationName, duration, {
        ...metadata,
        status: 'success'
      });
      
      return result;
    } catch (error) {
      const duration = performance.now() - startTime;
      
      this.metricsCollector.recordPerformance(operationName, duration, {
        ...metadata,
        status: 'error',
        error: error.message
      });
      
      throw error;
    }
  }
  
  recordCustomMetric(metricName: string, value: number, unit: string, metadata?: any): void {
    this.metricsCollector.recordEvent('custom_metric', {
      metricName,
      value,
      unit,
      metadata
    });
  }
  
  recordMemoryUsage(): void {
    if (typeof performance !== 'undefined' && performance.memory) {
      this.metricsCollector.recordEvent('memory_usage', {
        usedJSHeapSize: performance.memory.usedJSHeapSize,
        totalJSHeapSize: performance.memory.totalJSHeapSize,
        jsHeapSizeLimit: performance.memory.jsHeapSizeLimit
      });
    }
  }
}
```

#### 사용자 행동 분석

```typescript
// common/analytics/UserBehaviorAnalyzer.ts
class UserBehaviorAnalyzer {
  private metricsCollector: MetricsCollector;
  private sessionData: UserSession;
  
  constructor(metricsCollector: MetricsCollector) {
    this.metricsCollector = metricsCollector;
    this.initializeSession();
  }
  
  private initializeSession(): void {
    this.sessionData = {
      sessionId: this.generateSessionId(),
      startTime: new Date(),
      interactions: [],
      screenViews: [],
      errors: []
    };
  }
  
  recordScreenView(screenName: string, metadata?: any): void {
    const screenView = {
      screenName,
      timestamp: new Date(),
      metadata
    };
    
    this.sessionData.screenViews.push(screenView);
    
    this.metricsCollector.recordEvent('screen_view', {
      screenName,
      sessionId: this.sessionData.sessionId,
      metadata
    });
  }
  
  recordUserInteraction(
    action: string,
    target: string,
    context?: any
  ): void {
    const interaction = {
      action,
      target,
      timestamp: new Date(),
      context
    };
    
    this.sessionData.interactions.push(interaction);
    
    this.metricsCollector.recordUserInteraction(action, target, {
      sessionId: this.sessionData.sessionId,
      context
    });
  }
  
  recordFeatureUsage(featureName: string, usage: FeatureUsage): void {
    this.metricsCollector.recordEvent('feature_usage', {
      featureName,
      sessionId: this.sessionData.sessionId,
      usage
    });
  }
  
  recordConversionEvent(eventName: string, value?: number, metadata?: any): void {
    this.metricsCollector.recordEvent('conversion', {
      eventName,
      value,
      sessionId: this.sessionData.sessionId,
      metadata
    });
  }
  
  recordAttendancePattern(pattern: AttendancePattern): void {
    this.metricsCollector.recordBusinessEvent('attendance_pattern', {
      pattern,
      sessionId: this.sessionData.sessionId
    });
  }
  
  getSessionSummary(): SessionSummary {
    return {
      sessionId: this.sessionData.sessionId,
      duration: new Date().getTime() - this.sessionData.startTime.getTime(),
      screenViewCount: this.sessionData.screenViews.length,
      interactionCount: this.sessionData.interactions.length,
      errorCount: this.sessionData.errors.length,
      uniqueScreens: [...new Set(this.sessionData.screenViews.map(sv => sv.screenName))]
    };
  }
  
  private generateSessionId(): string {
    return `session_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

#### 실시간 알림 시스템

```typescript
// common/analytics/AlertSystem.ts
class AlertSystem {
  private metricsCollector: MetricsCollector;
  private alertThresholds: Map<string, AlertThreshold> = new Map();
  private activeAlerts: Map<string, Alert> = new Map();
  
  constructor(metricsCollector: MetricsCollector) {
    this.metricsCollector = metricsCollector;
    this.setupDefaultThresholds();
  }
  
  private setupDefaultThresholds(): void {
    // 성능 임계값
    this.addThreshold('performance_degradation', {
      metric: 'performance',
      condition: 'average_duration > 5000', // 5초 이상
      windowSize: 300000, // 5분
      severity: 'warning'
    });
    
    // 에러율 임계값
    this.addThreshold('high_error_rate', {
      metric: 'error',
      condition: 'count > 10', // 5분내 10개 이상
      windowSize: 300000,
      severity: 'critical'
    });
    
    // 동기화 실패 임계값
    this.addThreshold('sync_failure', {
      metric: 'sync_status',
      condition: 'failed_ratio > 0.1', // 10% 이상 실패
      windowSize: 600000, // 10분
      severity: 'warning'
    });
  }
  
  addThreshold(name: string, threshold: AlertThreshold): void {
    this.alertThresholds.set(name, threshold);
  }
  
  checkThresholds(metric: Metric): void {
    for (const [name, threshold] of this.alertThresholds) {
      if (this.shouldCheckThreshold(metric, threshold)) {
        this.evaluateThreshold(name, threshold, metric);
      }
    }
  }
  
  private shouldCheckThreshold(metric: Metric, threshold: AlertThreshold): boolean {
    return metric.name === threshold.metric;
  }
  
  private async evaluateThreshold(
    name: string,
    threshold: AlertThreshold,
    metric: Metric
  ): Promise<void> {
    const isTriggered = await this.evaluateCondition(threshold, metric);
    
    if (isTriggered && !this.activeAlerts.has(name)) {
      // 새로운 알림 생성
      const alert: Alert = {
        id: `alert_${Date.now()}`,
        name,
        severity: threshold.severity,
        message: `Threshold exceeded: ${name}`,
        triggeredAt: new Date(),
        metric,
        threshold
      };
      
      this.activeAlerts.set(name, alert);
      await this.sendAlert(alert);
      
    } else if (!isTriggered && this.activeAlerts.has(name)) {
      // 알림 해제
      const alert = this.activeAlerts.get(name)!;
      alert.resolvedAt = new Date();
      
      await this.sendAlertResolution(alert);
      this.activeAlerts.delete(name);
    }
  }
  
  private async evaluateCondition(
    threshold: AlertThreshold,
    metric: Metric
  ): Promise<boolean> {
    // 실제 조건 평가 로직
    // 여기서는 간단한 예시만 제공
    switch (threshold.condition) {
      case 'average_duration > 5000':
        return metric.data.duration > 5000;
      case 'count > 10':
        return await this.getMetricCountInWindow(threshold.metric, threshold.windowSize) > 10;
      default:
        return false;
    }
  }
  
  private async sendAlert(alert: Alert): Promise<void> {
    this.metricsCollector.recordEvent('alert_triggered', {
      alertId: alert.id,
      alertName: alert.name,
      severity: alert.severity,
      message: alert.message
    });
    
    // 외부 알림 시스템으로 전송 (예: Slack, 이메일 등)
    await this.notifyExternalSystems(alert);
  }
  
  private async sendAlertResolution(alert: Alert): Promise<void> {
    this.metricsCollector.recordEvent('alert_resolved', {
      alertId: alert.id,
      alertName: alert.name,
      duration: alert.resolvedAt!.getTime() - alert.triggeredAt.getTime()
    });
  }
  
  private async notifyExternalSystems(alert: Alert): Promise<void> {
    // 실제 구현에서는 Slack API, 이메일 서비스 등을 사용
    console.log(`🚨 Alert: ${alert.name} - ${alert.message}`);
  }
  
  private async getMetricCountInWindow(metricName: string, windowSize: number): Promise<number> {
    // 실제 구현에서는 로컬 DB나 메트릭 저장소에서 조회
    return 0; // 구현 필요
  }
  
  getActiveAlerts(): Alert[] {
    return Array.from(this.activeAlerts.values());
  }
}
```

## 📊 표준 메트릭 정의

### 성능 메트릭
- **응답 시간**: API 호출, BLE 통신, 동기화 작업
- **처리량**: 초당 요청 수, 출석 처리 건수
- **자원 사용률**: CPU, 메모리, 네트워크, 배터리

### 비즈니스 메트릭
- **출석률**: 실시간/일별/주별 출석률
- **기능 사용률**: 각 기능별 사용 빈도
- **오류율**: 기능별 오류 발생률

### 사용자 경험 메트릭
- **세션 길이**: 평균 사용 시간
- **화면 전환**: 네비게이션 패턴
- **이탈률**: 기능별 중도 이탈률

## 🔗 관련 시나리오

### 📄 Common 패턴 연계
- **오프라인 동기화**: [offline-sync.md](./offline-sync.md)
- **백업 복구**: [backup.md](./backup.md)
- **BLE 통신**: [ble-communication.md](./ble-communication.md)
- **보안 인증**: [security-auth.md](./security-auth.md)

### 🌐 시스템 연동 영역
- **통합 플랫폼**: 모든 메트릭 중앙 집중 관리
- **분석 대시보드**: 실시간 모니터링 및 리포팅
- **알림 시스템**: 임계값 기반 자동 알림

---

## 📊 성능 지표

| 지표 | 목표 | 측정 방법 |
|------|------|-----------|
| 메트릭 수집 지연 | < 1초 | 이벤트 발생부터 수집까지 |
| 배치 처리 시간 | < 30초 | 100개 메트릭 처리 시간 |
| 알림 발송 시간 | < 5초 | 임계값 초과부터 알림까지 |
| 데이터 정확도 | > 99.9% | 손실률 측정 |

---

**분할 정보**: 이 파일은 원본 `technical-patterns-data-sync.md` (842줄)에서 분할된 파일입니다.
**생성일**: 2024-12-19
**연관 파일**: technical-patterns-offline-sync.md, technical-patterns-backup.md
