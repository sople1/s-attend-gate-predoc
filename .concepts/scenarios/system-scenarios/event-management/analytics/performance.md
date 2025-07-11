# Event Management 성능 최적화 시나리오

## 📋 시나리오 개요

시스템 성능 최적화와 모니터링을 위한 데이터베이스 최적화, 계층적 캐싱 전략, 성능 모니터링 시나리오입니다.
대용량 데이터 처리와 실시간 분석을 위한 포괄적인 최적화 방안을 다룹니다.

---

## 🚀 성능 최적화 시나리오

### 1. 데이터베이스 최적화

```sql
-- 출석 기록 조회 최적화를 위한 인덱스
CREATE INDEX CONCURRENTLY idx_attendance_participant_timestamp 
ON attendance_records(participant_id, timestamp DESC);

CREATE INDEX CONCURRENTLY idx_attendance_gate_timestamp 
ON attendance_records(gate_id, timestamp DESC);

CREATE INDEX CONCURRENTLY idx_attendance_method_timestamp 
ON attendance_records(method, timestamp DESC);

-- 참가자 토큰 조회 최적화 (부분 인덱스)
CREATE UNIQUE INDEX CONCURRENTLY idx_participants_token_active 
ON participants(token) WHERE status = 'active';

-- 전문 검색을 위한 GIN 인덱스
CREATE INDEX CONCURRENTLY idx_participants_search 
ON participants USING gin(to_tsvector('english', name || ' ' || email));

-- 파티셔닝 (날짜별)
CREATE TABLE attendance_records_2024_q1 PARTITION OF attendance_records
FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE attendance_records_2024_q2 PARTITION OF attendance_records
FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

```typescript
class DatabaseOptimizationService {
  // 쿼리 최적화 및 실행 계획 분석
  async analyzeQueryPerformance(): Promise<QueryAnalysis[]> {
    const criticalQueries = [
      'SELECT * FROM attendance_records WHERE participant_id = ? ORDER BY timestamp DESC',
      'SELECT COUNT(*) FROM attendance_records WHERE gate_id = ? AND timestamp >= ?',
      'SELECT * FROM participants WHERE token = ? AND status = \'active\'',
      'SELECT gate_id, COUNT(*) FROM attendance_records WHERE timestamp >= ? GROUP BY gate_id'
    ];

    const analyses = await Promise.all(
      criticalQueries.map(async (query) => {
        const plan = await this.db.raw(`EXPLAIN (ANALYZE, BUFFERS) ${query}`);
        return {
          query,
          executionTime: this.extractExecutionTime(plan),
          indexUsage: this.analyzeIndexUsage(plan),
          suggestions: this.generateOptimizationSuggestions(plan)
        };
      })
    );

    return analyses;
  }

  // 자동 파티션 관리
  async managePartitions(): Promise<void> {
    const currentDate = new Date();
    const nextQuarter = new Date(currentDate);
    nextQuarter.setMonth(currentDate.getMonth() + 3);

    const partitionName = `attendance_records_${nextQuarter.getFullYear()}_q${Math.ceil((nextQuarter.getMonth() + 1) / 3)}`;
    const startDate = new Date(nextQuarter.getFullYear(), Math.floor(nextQuarter.getMonth() / 3) * 3, 1);
    const endDate = new Date(startDate);
    endDate.setMonth(startDate.getMonth() + 3);

    await this.db.raw(`
      CREATE TABLE IF NOT EXISTS ${partitionName} PARTITION OF attendance_records
      FOR VALUES FROM ('${startDate.toISOString().split('T')[0]}') TO ('${endDate.toISOString().split('T')[0]}')
    `);

    // 오래된 파티션 정리
    await this.cleanupOldPartitions();
  }

  // 통계 업데이트 및 VACUUM 자동화
  async performMaintenance(): Promise<void> {
    const tables = ['attendance_records', 'participants', 'gates'];
    
    for (const table of tables) {
      // 통계 업데이트
      await this.db.raw(`ANALYZE ${table}`);
      
      // 자동 VACUUM (dead tuple 정리)
      await this.db.raw(`VACUUM (ANALYZE) ${table}`);
    }
  }
}
```

### 2. 계층적 캐싱 전략

```typescript
class AdvancedCacheStrategy {
  private memoryCache: LRUCache;
  private redisCluster: RedisCluster;
  private cacheMetrics: CacheMetrics;

  constructor() {
    this.memoryCache = new LRUCache({ max: 1000, ttl: 300000 }); // 5분
    this.redisCluster = new RedisCluster(/* config */);
    this.cacheMetrics = new CacheMetrics();
  }

  // 계층적 캐싱 with 메트릭
  async get<T>(key: string): Promise<T | null> {
    const startTime = Date.now();

    // L1: 메모리 캐시 (가장 빠름)
    let value = this.memoryCache.get(key);
    if (value) {
      this.cacheMetrics.recordHit('L1', Date.now() - startTime);
      return value;
    }

    // L2: Redis 클러스터 (중간 속도)
    value = await this.redisCluster.get(key);
    if (value) {
      this.memoryCache.set(key, value);
      this.cacheMetrics.recordHit('L2', Date.now() - startTime);
      return JSON.parse(value);
    }

    this.cacheMetrics.recordMiss(Date.now() - startTime);
    return null;
  }

  async set<T>(key: string, value: T, ttl: number): Promise<void> {
    // 메모리 캐시에 저장 (짧은 TTL)
    this.memoryCache.set(key, value, Math.min(ttl, 300000));

    // Redis에 저장 (긴 TTL)
    await this.redisCluster.setex(key, ttl, JSON.stringify(value));
  }

  // 미리 로딩 (Preloading)
  async preloadCriticalData(): Promise<void> {
    const criticalKeys = [
      'stats:overall',
      'active_gates',
      'event_config'
    ];

    await Promise.all(criticalKeys.map(async (key) => {
      const value = await this.redisCluster.get(key);
      if (value) {
        this.memoryCache.set(key, JSON.parse(value));
      }
    }));
  }

  // 캐시 무효화 패턴
  async invalidatePattern(pattern: string): Promise<void> {
    // 메모리 캐시에서 패턴 매칭으로 삭제
    for (const key of this.memoryCache.keys()) {
      if (key.includes(pattern)) {
        this.memoryCache.delete(key);
      }
    }

    // Redis에서 패턴 매칭으로 삭제
    await this.redisCluster.eval(`
      local keys = redis.call('keys', ARGV[1])
      for i=1,#keys,5000 do
        redis.call('del', unpack(keys, i, math.min(i+4999, #keys)))
      end
      return #keys
    `, 0, pattern);
  }

  // 캐시 워밍 전략
  async warmCache(): Promise<void> {
    console.log('Starting cache warming...');
    
    // 자주 접근되는 데이터 미리 로딩
    const commonQueries = [
      () => this.loadAllActiveParticipants(),
      () => this.loadGateConfigurations(),
      () => this.loadRecentStatistics(),
      () => this.loadEventConfiguration()
    ];

    await Promise.all(commonQueries.map(query => query()));
    
    console.log('Cache warming completed');
  }

  // 스마트 캐시 관리
  async manageSmartCache(): Promise<void> {
    const cacheStats = await this.cacheMetrics.getStats();
    
    // 히트율이 낮은 키 제거
    if (cacheStats.hitRate < 0.7) {
      await this.evictLowHitRateKeys();
    }

    // 자주 액세스되는 데이터의 TTL 연장
    await this.extendHotDataTTL();

    // 메모리 사용량 모니터링
    if (cacheStats.memoryUsage > 0.8) {
      await this.optimizeMemoryUsage();
    }
  }
}
```

### 3. 성능 모니터링 및 최적화

```typescript
class PerformanceMonitor {
  private metrics: Map<string, Metric[]> = new Map();

  // API 응답 시간 모니터링
  measureAPIResponse(endpoint: string) {
    return (req: Request, res: Response, next: NextFunction) => {
      const startTime = Date.now();
      
      res.on('finish', () => {
        const duration = Date.now() - startTime;
        this.recordMetric('api_response_time', {
          endpoint,
          duration,
          statusCode: res.statusCode,
          timestamp: new Date()
        });

        // 느린 요청 알림
        if (duration > 1000) { // 1초 이상
          this.alertSlowRequest(endpoint, duration);
        }
      });

      next();
    };
  }

  // 데이터베이스 쿼리 모니터링
  async measureDBQuery<T>(
    queryName: string, 
    queryFn: () => Promise<T>
  ): Promise<T> {
    const startTime = Date.now();
    
    try {
      const result = await queryFn();
      const duration = Date.now() - startTime;
      
      this.recordMetric('db_query_time', {
        queryName,
        duration,
        success: true,
        timestamp: new Date()
      });

      return result;
    } catch (error) {
      const duration = Date.now() - startTime;
      
      this.recordMetric('db_query_time', {
        queryName,
        duration,
        success: false,
        error: error.message,
        timestamp: new Date()
      });

      throw error;
    }
  }

  // 실시간 성능 메트릭 대시보드
  getPerformanceMetrics(): PerformanceDashboard {
    const now = Date.now();
    const fiveMinutesAgo = now - 5 * 60 * 1000;

    return {
      apiPerformance: this.aggregateAPIMetrics(fiveMinutesAgo, now),
      dbPerformance: this.aggregateDBMetrics(fiveMinutesAgo, now),
      cachePerformance: this.aggregateCacheMetrics(fiveMinutesAgo, now),
      systemHealth: this.getSystemHealthMetrics()
    };
  }

  private recordMetric(type: string, data: any): void {
    if (!this.metrics.has(type)) {
      this.metrics.set(type, []);
    }

    const metrics = this.metrics.get(type)!;
    metrics.push({ ...data, type });

    // 메모리 관리: 5분 이상 된 메트릭 제거
    const fiveMinutesAgo = Date.now() - 5 * 60 * 1000;
    this.metrics.set(type, metrics.filter(m => m.timestamp.getTime() > fiveMinutesAgo));
  }

  // 자동 최적화 제안
  async generateOptimizationSuggestions(): Promise<OptimizationSuggestion[]> {
    const suggestions: OptimizationSuggestion[] = [];
    const metrics = this.getPerformanceMetrics();

    // 데이터베이스 최적화 제안
    if (metrics.dbPerformance.averageQueryTime > 500) {
      suggestions.push({
        type: 'database',
        priority: 'high',
        description: '데이터베이스 쿼리 성능이 저하되었습니다.',
        actions: [
          '인덱스 추가 검토',
          '쿼리 최적화',
          '파티셔닝 고려'
        ],
        expectedImpact: '쿼리 성능 50% 향상'
      });
    }

    // 캐시 최적화 제안
    if (metrics.cachePerformance.hitRate < 0.7) {
      suggestions.push({
        type: 'cache',
        priority: 'medium',
        description: '캐시 히트율이 낮습니다.',
        actions: [
          '캐시 키 전략 재검토',
          'TTL 조정',
          '캐시 워밍 강화'
        ],
        expectedImpact: '응답 시간 30% 단축'
      });
    }

    return suggestions;
  }
}
```

### 4. 부하 분산 및 확장성

```typescript
class LoadBalancingService {
  private healthyNodes: Set<string> = new Set();
  private nodeMetrics: Map<string, NodeMetrics> = new Map();

  // 동적 부하 분산
  async getOptimalNode(requestType: string): Promise<string> {
    const availableNodes = Array.from(this.healthyNodes);
    
    if (availableNodes.length === 0) {
      throw new Error('No healthy nodes available');
    }

    // 요청 유형에 따른 노드 선택
    switch (requestType) {
      case 'analytics':
        return this.selectAnalyticsNode(availableNodes);
      case 'database':
        return this.selectDatabaseNode(availableNodes);
      case 'cache':
        return this.selectCacheNode(availableNodes);
      default:
        return this.selectGeneralNode(availableNodes);
    }
  }

  private selectAnalyticsNode(nodes: string[]): string {
    // CPU 사용률이 낮은 노드 선택
    return nodes.reduce((best, current) => {
      const bestMetrics = this.nodeMetrics.get(best);
      const currentMetrics = this.nodeMetrics.get(current);
      
      if (!bestMetrics || !currentMetrics) return best;
      
      return currentMetrics.cpuUsage < bestMetrics.cpuUsage ? current : best;
    });
  }

  private selectDatabaseNode(nodes: string[]): string {
    // 연결 수가 적은 노드 선택
    return nodes.reduce((best, current) => {
      const bestMetrics = this.nodeMetrics.get(best);
      const currentMetrics = this.nodeMetrics.get(current);
      
      if (!bestMetrics || !currentMetrics) return best;
      
      return currentMetrics.activeConnections < bestMetrics.activeConnections ? current : best;
    });
  }

  // 자동 스케일링
  async autoScale(): Promise<void> {
    const systemMetrics = await this.getSystemMetrics();
    
    // 스케일 업 조건
    if (systemMetrics.avgCpuUsage > 80 || systemMetrics.avgMemoryUsage > 85) {
      await this.scaleUp();
    }
    
    // 스케일 다운 조건
    if (systemMetrics.avgCpuUsage < 30 && systemMetrics.avgMemoryUsage < 40) {
      await this.scaleDown();
    }
  }

  private async scaleUp(): Promise<void> {
    const newNodeCount = Math.min(this.healthyNodes.size * 1.5, 10); // 최대 10개
    
    for (let i = this.healthyNodes.size; i < newNodeCount; i++) {
      await this.deployNewNode();
    }
  }

  private async scaleDown(): Promise<void> {
    const targetNodeCount = Math.max(Math.floor(this.healthyNodes.size * 0.7), 2); // 최소 2개
    
    while (this.healthyNodes.size > targetNodeCount) {
      await this.terminateNode();
    }
  }
}
```

### 5. 실시간 알림 및 자동 복구

```typescript
class PerformanceAlertService {
  private alertRules: AlertRule[] = [
    {
      name: 'high_response_time',
      condition: (metrics) => metrics.avgResponseTime > 2000,
      severity: 'warning',
      action: 'AUTO_SCALE'
    },
    {
      name: 'database_slow_query',
      condition: (metrics) => metrics.slowQueryCount > 10,
      severity: 'critical',
      action: 'OPTIMIZE_QUERIES'
    },
    {
      name: 'memory_usage_high',
      condition: (metrics) => metrics.memoryUsage > 90,
      severity: 'critical',
      action: 'RESTART_SERVICE'
    }
  ];

  async checkAlerts(): Promise<void> {
    const currentMetrics = await this.performanceMonitor.getPerformanceMetrics();
    
    for (const rule of this.alertRules) {
      if (rule.condition(currentMetrics)) {
        await this.triggerAlert(rule, currentMetrics);
      }
    }
  }

  private async triggerAlert(rule: AlertRule, metrics: PerformanceMetrics): Promise<void> {
    // 알림 발송
    await this.sendAlert({
      title: `Performance Alert: ${rule.name}`,
      severity: rule.severity,
      metrics: metrics,
      timestamp: new Date()
    });

    // 자동 복구 액션
    await this.executeAutoRecovery(rule.action, metrics);
  }

  private async executeAutoRecovery(action: string, metrics: PerformanceMetrics): Promise<void> {
    switch (action) {
      case 'AUTO_SCALE':
        await this.loadBalancer.autoScale();
        break;
      case 'OPTIMIZE_QUERIES':
        await this.databaseOptimizer.optimizeSlowQueries();
        break;
      case 'RESTART_SERVICE':
        await this.serviceManager.gracefulRestart();
        break;
    }
  }
}
```

---

## 📋 관련 개념

### 상위 수준 연결
- [Analytics Reporting 개요](./analytics-reporting.md) - 전체 시스템 개요
- [Event Management API](../core-apis/event-management-api.md) - 성능 최적화 적용점

### 병렬 시나리오  
- [분석 및 리포팅](./analytics-reporting-analytics.md) - 성능이 중요한 분석 기능
- [보안 및 데이터 보호](./analytics-reporting-security.md) - 보안과 성능의 균형

### 기술 패턴
- [데이터 동기화 패턴](../../common/technical-patterns-data-sync.md) - 최적화된 동기화 전략
- [BLE 통신 패턴](../../common/technical-patterns-ble-communication.md) - 효율적인 데이터 수집

---

## 🚀 성능 메트릭 및 최적화 지표

| 구분 | 목표 | 측정 방법 | 현재 상태 |
|------|------|-----------|-----------|
| **API 응답 시간** | < 500ms | 평균 응답 시간 모니터링 | 🟢 달성 |
| **데이터베이스 쿼리** | < 100ms | 쿼리 실행 시간 측정 | 🟡 개선 필요 |
| **캐시 히트율** | > 85% | 캐시 히트/미스 비율 | 🟢 달성 |
| **메모리 사용률** | < 80% | 시스템 메모리 모니터링 | 🟢 달성 |
| **CPU 사용률** | < 70% | CPU 사용률 모니터링 | 🟡 개선 필요 |
| **동시 사용자** | 10,000+ | 부하 테스트 결과 | 🟢 달성 |
| **데이터 처리량** | 1,000 TPS | 초당 트랜잭션 수 | 🟡 개선 필요 |

### 최적화 우선순위
1. **High Priority**: 데이터베이스 쿼리 최적화 (인덱스 추가, 파티셔닝)
2. **Medium Priority**: 캐시 전략 개선 (스마트 캐싱, 무효화 최적화)
3. **Low Priority**: 자동 스케일링 정책 조정
