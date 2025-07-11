# Integrated Platform - Circuit Breaker 및 헬스 체크

## 🔧 장애 감지 및 복구 시스템

### Circuit Breaker 패턴

**목표**: 시스템 장애 시 자동 복구 및 서비스 연속성 보장

```typescript
// src/services/resilience/CircuitBreaker.ts
class CircuitBreaker {
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
  private failureCount: number = 0;
  private lastFailureTime: number = 0;
  private successCount: number = 0;
  
  constructor(
    private failureThreshold: number = 5,
    private recoveryTimeout: number = 60000, // 1분
    private halfOpenMaxCalls: number = 3
  ) {}
  
  async execute<T>(
    operation: () => Promise<T>,
    fallback?: () => Promise<T>
  ): Promise<T> {
    if (this.state === 'OPEN') {
      if (this.shouldAttemptReset()) {
        this.state = 'HALF_OPEN';
        this.successCount = 0;
      } else {
        return this.executeFallback(fallback);
      }
    }
    
    try {
      const result = await operation();
      
      if (this.state === 'HALF_OPEN') {
        this.successCount++;
        if (this.successCount >= this.halfOpenMaxCalls) {
          this.reset();
        }
      } else {
        this.reset();
      }
      
      return result;
    } catch (error) {
      this.recordFailure();
      
      if (this.state === 'HALF_OPEN') {
        this.state = 'OPEN';
        this.lastFailureTime = Date.now();
      } else if (this.failureCount >= this.failureThreshold) {
        this.state = 'OPEN';
        this.lastFailureTime = Date.now();
      }
      
      return this.executeFallback(fallback, error);
    }
  }
  
  private shouldAttemptReset(): boolean {
    return Date.now() - this.lastFailureTime >= this.recoveryTimeout;
  }
  
  private recordFailure(): void {
    this.failureCount++;
  }
  
  private reset(): void {
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
  }
  
  private async executeFallback<T>(
    fallback?: () => Promise<T>,
    originalError?: any
  ): Promise<T> {
    if (fallback) {
      try {
        return await fallback();
      } catch (fallbackError) {
        throw new Error(`Circuit breaker open. Original error: ${originalError?.message}. Fallback error: ${fallbackError.message}`);
      }
    }
    
    throw new Error(`Circuit breaker open. Service unavailable. Original error: ${originalError?.message}`);
  }
  
  getState(): { state: string; failureCount: number; lastFailureTime: number } {
    return {
      state: this.state,
      failureCount: this.failureCount,
      lastFailureTime: this.lastFailureTime
    };
  }
}
```

### 헬스 체크 및 서비스 디스커버리

```typescript
// src/services/health/HealthCheckService.ts
class HealthCheckService {
  private healthChecks: Map<string, HealthCheck> = new Map();
  private serviceRegistry: ServiceRegistry;
  
  constructor() {
    this.registerDefaultHealthChecks();
  }
  
  private registerDefaultHealthChecks(): void {
    // 데이터베이스 헬스 체크
    this.healthChecks.set('database', {
      name: 'database',
      timeout: 5000,
      interval: 30000,
      check: async () => {
        const start = Date.now();
        await this.database.query('SELECT 1');
        const duration = Date.now() - start;
        
        return {
          status: duration < 1000 ? 'healthy' : 'degraded',
          responseTime: duration,
          details: { query: 'SELECT 1' }
        };
      }
    });
    
    // Redis 헬스 체크
    this.healthChecks.set('redis', {
      name: 'redis',
      timeout: 3000,
      interval: 30000,
      check: async () => {
        const start = Date.now();
        await this.redis.ping();
        const duration = Date.now() - start;
        
        return {
          status: duration < 500 ? 'healthy' : 'degraded',
          responseTime: duration,
          details: { command: 'PING' }
        };
      }
    });
    
    // 외부 API 헬스 체크
    this.healthChecks.set('external_api', {
      name: 'external_api',
      timeout: 10000,
      interval: 60000,
      check: async () => {
        const checks = await Promise.allSettled([
          this.checkExternalService('payment-service'),
          this.checkExternalService('notification-service'),
          this.checkExternalService('analytics-service')
        ]);
        
        const healthy = checks.filter(c => c.status === 'fulfilled').length;
        const total = checks.length;
        
        return {
          status: this.calculateOverallHealth(healthy, total),
          responseTime: 0,
          details: { healthy, total, services: checks }
        };
      }
    });
  }
  
  async checkHealth(): Promise<HealthStatus> {
    const results = new Map<string, HealthCheckResult>();
    const promises = Array.from(this.healthChecks.entries()).map(
      async ([name, check]) => {
        try {
          const result = await Promise.race([
            check.check(),
            this.timeout(check.timeout)
          ]);
          results.set(name, { ...result, timestamp: new Date().toISOString() });
        } catch (error) {
          results.set(name, {
            status: 'unhealthy',
            responseTime: check.timeout,
            timestamp: new Date().toISOString(),
            error: error.message
          });
        }
      }
    );
    
    await Promise.all(promises);
    
    const overallStatus = this.calculateOverallStatus(results);
    
    return {
      status: overallStatus,
      timestamp: new Date().toISOString(),
      checks: Object.fromEntries(results)
    };
  }
  
  private async timeout(ms: number): Promise<never> {
    return new Promise((_, reject) => 
      setTimeout(() => reject(new Error('Health check timeout')), ms)
    );
  }
  
  private calculateOverallStatus(results: Map<string, HealthCheckResult>): HealthStatusType {
    const statuses = Array.from(results.values()).map(r => r.status);
    
    if (statuses.every(s => s === 'healthy')) {
      return 'healthy';
    }
    
    if (statuses.some(s => s === 'unhealthy')) {
      return 'unhealthy';
    }
    
    return 'degraded';
  }
  
  private calculateOverallHealth(healthy: number, total: number): HealthStatusType {
    if (healthy === total) return 'healthy';
    if (healthy === 0) return 'unhealthy';
    if (healthy / total < 0.5) return 'degraded';
    return 'healthy';
  }
  
  async startMonitoring(): Promise<void> {
    for (const [name, check] of this.healthChecks) {
      setInterval(async () => {
        try {
          const result = await check.check();
          await this.recordHealthMetric(name, result);
          
          if (result.status !== 'healthy') {
            await this.handleUnhealthyService(name, result);
          }
        } catch (error) {
          await this.handleHealthCheckFailure(name, error);
        }
      }, check.interval);
    }
  }
  
  private async handleUnhealthyService(
    serviceName: string, 
    result: HealthCheckResult
  ): Promise<void> {
    const alert = {
      severity: result.status === 'unhealthy' ? 'critical' : 'warning',
      service: serviceName,
      status: result.status,
      responseTime: result.responseTime,
      timestamp: result.timestamp
    };
    
    await this.alertManager.sendAlert(alert);
    
    // 서비스 레지스트리에서 일시적으로 제거
    if (result.status === 'unhealthy') {
      await this.serviceRegistry.markServiceUnavailable(serviceName);
    }
  }
}
```

---

## 🔗 관련 파일

- **[자동 복구 및 백업](./disaster-recovery-backup.md)** - 복구 전략 및 백업 시스템
- **[인시던트 관리](./disaster-recovery-incident.md)** - 인시던트 대응 및 관리
- **[통합 플랫폼 장애 대응 개요](./security-performance-disaster-recovery.md)** - 전체 개요
- **[성능 최적화](./security-performance-optimization.md)** - 시스템 성능 연동
