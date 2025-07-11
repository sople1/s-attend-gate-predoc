# Integrated Platform - 성능 최적화

## 📈 성능 최적화 시나리오

### 시나리오 7: 대용량 데이터 처리 최적화

**목표**: 수백 개의 동시 행사와 수십만 명의 참가자 데이터를 효율적으로 처리

#### 7.1 캐싱 전략 및 성능 최적화

```typescript
// src/services/performance/CacheManager.ts
class CacheManager {
  private redisCluster: RedisCluster;
  private memoryCache: NodeCache;
  
  async getWithCache<T>(
    key: string,
    fetchFunction: () => Promise<T>,
    options: CacheOptions = {}
  ): Promise<T> {
    const cacheKey = this.buildCacheKey(key, options.namespace);
    
    // 1. 메모리 캐시 확인 (가장 빠름)
    if (options.useMemoryCache) {
      const memoryResult = this.memoryCache.get<T>(cacheKey);
      if (memoryResult) {
        return memoryResult;
      }
    }
    
    // 2. Redis 캐시 확인
    const redisResult = await this.redisCluster.get(cacheKey);
    if (redisResult) {
      const parsed = JSON.parse(redisResult) as T;
      
      // 메모리 캐시에도 저장 (짧은 TTL)
      if (options.useMemoryCache) {
        this.memoryCache.set(cacheKey, parsed, 60); // 1분
      }
      
      return parsed;
    }
    
    // 3. 실제 데이터 조회
    const data = await fetchFunction();
    
    // 4. 캐시에 저장
    await this.setCache(cacheKey, data, options);
    
    return data;
  }
  
  async invalidatePattern(pattern: string): Promise<void> {
    // Redis 클러스터의 모든 노드에서 패턴 매칭 키 삭제
    const nodes = this.redisCluster.nodes();
    
    await Promise.all(
      nodes.map(async (node) => {
        const keys = await node.keys(pattern);
        if (keys.length > 0) {
          await node.del(...keys);
        }
      })
    );
    
    // 메모리 캐시도 정리
    this.memoryCache.flushAll();
  }
  
  async warmCache(): Promise<void> {
    // 자주 사용되는 데이터를 미리 캐시에 로드
    const criticalData = [
      'active_events',
      'popular_event_types',
      'system_configuration',
      'user_permissions'
    ];
    
    await Promise.all(
      criticalData.map(async (dataType) => {
        try {
          await this.preloadData(dataType);
        } catch (error) {
          console.error(`Failed to warm cache for ${dataType}:`, error);
        }
      })
    );
  }
  
  private async preloadData(dataType: string): Promise<void> {
    switch (dataType) {
      case 'active_events':
        const events = await this.eventService.getActiveEvents();
        await this.setCache('active_events', events, { ttl: 300 });
        break;
      
      case 'popular_event_types':
        const eventTypes = await this.analyticsService.getPopularEventTypes();
        await this.setCache('popular_event_types', eventTypes, { ttl: 3600 });
        break;
      
      case 'system_configuration':
        const config = await this.configService.getSystemConfiguration();
        await this.setCache('system_config', config, { ttl: 1800 });
        break;
    }
  }
}
```

#### 7.2 데이터베이스 최적화

```typescript
// src/services/performance/DatabaseOptimizer.ts
class DatabaseOptimizer {
  private database: Database;
  private queryAnalyzer: QueryAnalyzer;
  
  async optimizeQueries(): Promise<OptimizationResult> {
    // 1. 느린 쿼리 식별
    const slowQueries = await this.identifySlowQueries();
    
    // 2. 인덱스 최적화 제안
    const indexSuggestions = await this.suggestIndexes(slowQueries);
    
    // 3. 쿼리 최적화 적용
    const optimizedQueries = await this.optimizeQueryPlans(slowQueries);
    
    // 4. 파티셔닝 검토
    const partitioningSuggestions = await this.suggestPartitioning();
    
    return {
      slowQueriesFound: slowQueries.length,
      indexesCreated: indexSuggestions.applied.length,
      queriesOptimized: optimizedQueries.length,
      partitioningSuggestions: partitioningSuggestions.length
    };
  }
  
  private async identifySlowQueries(): Promise<SlowQuery[]> {
    const slowQueryLog = await this.database.query(`
      SELECT 
        query_time,
        lock_time,
        rows_sent,
        rows_examined,
        sql_text
      FROM mysql.slow_log 
      WHERE start_time >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
      ORDER BY query_time DESC
      LIMIT 50
    `);
    
    return slowQueryLog.map(row => ({
      queryTime: row.query_time,
      lockTime: row.lock_time,
      rowsExamined: row.rows_examined,
      rowsSent: row.rows_sent,
      sqlText: row.sql_text,
      optimization: this.analyzeQuery(row.sql_text)
    }));
  }
  
  private async suggestIndexes(slowQueries: SlowQuery[]): Promise<IndexSuggestion> {
    const suggestions: IndexCreation[] = [];
    
    for (const query of slowQueries) {
      const analysis = await this.queryAnalyzer.analyze(query.sqlText);
      
      // WHERE 절 분석
      if (analysis.whereColumns.length > 0) {
        suggestions.push({
          table: analysis.table,
          columns: analysis.whereColumns,
          type: 'BTREE',
          reason: 'Improve WHERE clause performance'
        });
      }
      
      // JOIN 조건 분석
      if (analysis.joinColumns.length > 0) {
        suggestions.push({
          table: analysis.table,
          columns: analysis.joinColumns,
          type: 'BTREE',
          reason: 'Improve JOIN performance'
        });
      }
      
      // ORDER BY 분석
      if (analysis.orderByColumns.length > 0) {
        suggestions.push({
          table: analysis.table,
          columns: analysis.orderByColumns,
          type: 'BTREE',
          reason: 'Improve ORDER BY performance'
        });
      }
    }
    
    // 중복 제거 및 우선순위 정렬
    const uniqueSuggestions = this.deduplicateIndexSuggestions(suggestions);
    const prioritized = this.prioritizeIndexSuggestions(uniqueSuggestions);
    
    // 상위 10개 인덱스만 적용
    const toApply = prioritized.slice(0, 10);
    
    return {
      suggested: suggestions.length,
      applied: await this.createIndexes(toApply)
    };
  }
  
  private async createIndexes(suggestions: IndexCreation[]): Promise<IndexCreation[]> {
    const created: IndexCreation[] = [];
    
    const indexOptimizations = suggestions.map(suggestion => {
      const indexName = `idx_${suggestion.table}_${suggestion.columns.join('_')}`;
      return `CREATE INDEX ${indexName} ON ${suggestion.table} (${suggestion.columns.join(', ')})`;
    });
    
    for (const optimization of indexOptimizations) {
      try {
        await this.database.query(optimization);
      } catch (error) {
        if (!error.message.includes('already exists')) {
          console.error('Index optimization failed:', error);
        }
      }
    }
    
    // 통계 정보 업데이트
    await this.database.query('ANALYZE');
  }
}
```

#### 7.3 Auto-Scaling 및 부하 분산

```typescript
// src/services/scaling/AutoScaler.ts
class AutoScaler {
  private metrics: MetricsCollector;
  private kubernetesClient: K8sApi;
  
  async monitorAndScale(): Promise<void> {
    const currentMetrics = await this.metrics.getCurrentMetrics();
    
    // CPU, 메모리, 네트워크 사용률 확인
    const cpuUtilization = currentMetrics.cpu.average;
    const memoryUtilization = currentMetrics.memory.average;
    const requestRate = currentMetrics.network.requestsPerSecond;
    
    // 스케일링 결정
    const scalingDecision = this.calculateScalingDecision({
      cpuUtilization,
      memoryUtilization,
      requestRate
    });
    
    if (scalingDecision.action !== 'none') {
      await this.executeScaling(scalingDecision);
    }
  }
  
  private calculateScalingDecision(metrics: SystemMetrics): ScalingDecision {
    // 스케일 업 조건
    if (metrics.cpuUtilization > 70 || metrics.memoryUtilization > 80) {
      return {
        action: 'scale_up',
        targetReplicas: Math.min(this.currentReplicas * 2, this.maxReplicas),
        reason: 'High resource utilization'
      };
    }
    
    // 스케일 다운 조건 (좀 더 보수적)
    if (metrics.cpuUtilization < 30 && metrics.memoryUtilization < 40 && 
        metrics.requestRate < this.baselineRequestRate * 0.5) {
      return {
        action: 'scale_down',
        targetReplicas: Math.max(this.currentReplicas / 2, this.minReplicas),
        reason: 'Low resource utilization'
      };
    }
    
    return { action: 'none' };
  }
  
  private async executeScaling(decision: ScalingDecision): Promise<void> {
    try {
      await this.kubernetesClient.patchNamespacedDeploymentScale(
        this.deploymentName,
        this.namespace,
        {
          spec: {
            replicas: decision.targetReplicas
          }
        }
      );
