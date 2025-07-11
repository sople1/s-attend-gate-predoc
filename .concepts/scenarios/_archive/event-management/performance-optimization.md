# Event Management - Performance Optimization

## 🚀 성능 최적화 전략

이벤트 관리 시스템의 성능을 최적화하기 위한 종합적인 접근 방법을 다룹니다.

### 📋 목차

1. [데이터베이스 최적화](#1-데이터베이스-최적화)
2. [캐싱 전략](#2-캐싱-전략)
3. [연결 풀 최적화](#3-연결-풀-최적화)

---

## 1. 데이터베이스 최적화

### 인덱스 설계

```sql
-- 출석 기록 조회 최적화를 위한 인덱스
CREATE INDEX idx_attendance_participant_timestamp 
ON attendance_records(participant_id, timestamp DESC);

CREATE INDEX idx_attendance_gate_timestamp 
ON attendance_records(gate_id, timestamp DESC);

-- 참가자 토큰 조회 최적화
CREATE UNIQUE INDEX idx_participants_token 
ON participants(token) WHERE status = 'active';

-- 복합 인덱스로 빈번한 쿼리 최적화
CREATE INDEX idx_attendance_compound 
ON attendance_records(event_id, gate_id, timestamp DESC, method);

-- 파티셔닝으로 대용량 데이터 관리
CREATE TABLE attendance_records_2024_q1 PARTITION OF attendance_records
FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE attendance_records_2024_q2 PARTITION OF attendance_records
FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

### 쿼리 최적화

```typescript
class OptimizedQueries {
  // 배치 조회로 N+1 문제 해결
  async getParticipantsWithAttendance(participantIds: string[]): Promise<ParticipantWithAttendance[]> {
    const query = `
      SELECT p.*, 
             COALESCE(a.attendance_count, 0) as attendance_count,
             a.first_attendance,
             a.last_attendance
      FROM participants p
      LEFT JOIN (
        SELECT participant_id,
               COUNT(*) as attendance_count,
               MIN(timestamp) as first_attendance,
               MAX(timestamp) as last_attendance
        FROM attendance_records
        WHERE participant_id = ANY($1)
        GROUP BY participant_id
      ) a ON p.id = a.participant_id
      WHERE p.id = ANY($1)
    `;
    
    return await this.db.query(query, [participantIds]);
  }

  // 페이지네이션 최적화 (OFFSET 대신 커서 기반)
  async getAttendanceRecordsPaginated(
    lastTimestamp?: string,
    limit: number = 100
  ): Promise<AttendanceRecord[]> {
    const query = `
      SELECT * FROM attendance_records
      WHERE ($1::timestamp IS NULL OR timestamp < $1)
      ORDER BY timestamp DESC
      LIMIT $2
    `;
    
    return await this.db.query(query, [lastTimestamp, limit]);
  }
}
```

## 2. 캐싱 전략

### 계층적 캐싱 시스템

```typescript
class CacheStrategy {
  private memoryCache = new Map<string, any>();
  private readonly CACHE_TTL = {
    PARTICIPANT: 300,    // 5분
    GATE_CONFIG: 1800,   // 30분
    ANALYTICS: 60,       // 1분
    STATIC_DATA: 3600    // 1시간
  };

  // 계층적 캐싱 (L1: Memory, L2: Redis, L3: Database)
  async getParticipant(id: string): Promise<Participant | null> {
    // L1: 메모리 캐시 (빠름, 작은 용량)
    let participant = this.memoryCache.get(`participant:${id}`);
    if (participant) {
      console.log('Cache hit: Memory');
      return participant;
    }

    // L2: Redis 캐시 (중간 속도, 큰 용량)
    participant = await this.redisCache.get(`participant:${id}`);
    if (participant) {
      console.log('Cache hit: Redis');
      this.memoryCache.set(`participant:${id}`, participant, this.CACHE_TTL.PARTICIPANT);
      return participant;
    }

    // L3: 데이터베이스 (느림, 영구 저장)
    console.log('Cache miss: Database query');
    participant = await this.db.getParticipant(id);
    if (participant) {
      await this.redisCache.setex(`participant:${id}`, this.CACHE_TTL.PARTICIPANT * 6, participant);
      this.memoryCache.set(`participant:${id}`, participant, this.CACHE_TTL.PARTICIPANT);
    }

    return participant;
  }

  // 캐시 무효화 전략
  async invalidateParticipantCache(participantId: string): Promise<void> {
    // 계층적 캐시 무효화
    this.memoryCache.delete(`participant:${participantId}`);
    await this.redisCache.del(`participant:${participantId}`);
    
    // 관련 캐시도 무효화
    await this.redisCache.del(`participant:token:${participantId}`);
    await this.redisCache.del(`analytics:participant:${participantId}`);
  }

  // 미리 데이터 로딩 (Cache Warming)
  async warmupCache(): Promise<void> {
    console.log('Starting cache warmup...');
    
    // 활성 참가자 미리 로딩
    const activeParticipants = await this.db.getActiveParticipants();
    await Promise.all(
      activeParticipants.map(p => this.getParticipant(p.id))
    );

    // 게이트 설정 미리 로딩
    const gates = await this.db.getActiveGates();
    await Promise.all(
      gates.map(g => this.getGateConfig(g.id))
    );

    console.log('Cache warmup completed');
  }
}
```

## 3. 연결 풀 최적화

### 데이터베이스 연결 관리

```typescript
class DatabaseConnectionManager {
  private pool: Pool;

  constructor() {
    this.pool = new Pool({
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT || '5432'),
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      
      // 연결 풀 최적화
      min: 10,          // 최소 연결 수
      max: 100,         // 최대 연결 수
      idleTimeoutMillis: 30000,  // 유휴 연결 타임아웃
      connectionTimeoutMillis: 2000, // 연결 타임아웃
      
      // 연결 유지
      keepAlive: true,
      keepAliveInitialDelayMillis: 0,
    });

    // 연결 상태 모니터링
    this.pool.on('connect', (client) => {
      console.log('New database connection established');
    });

    this.pool.on('error', (err) => {
      console.error('Database pool error:', err);
    });
  }

  // 읽기 전용 쿼리용 분리된 연결 풀
  private readPool = new Pool({ /* 읽기 전용 DB 설정 */ });

  async query(text: string, params?: any[], readOnly = false): Promise<any> {
    const pool = readOnly ? this.readPool : this.pool;
    const client = await pool.connect();
    
    try {
      const start = Date.now();
      const result = await client.query(text, params);
      const duration = Date.now() - start;
      
      // 느린 쿼리 로깅
      if (duration > 1000) {
        console.warn(`Slow query detected: ${duration}ms`, { text, params });
      }
      
      return result;
    } finally {
      client.release();
    }
  }
}
```

## 🔗 관련 파일

### 확장성 및 모니터링
- [확장성 아키텍처](./scalability-architecture.md) - 마이크로서비스, 메시지 큐, 부하 처리
- [모니터링 및 스케일링](./monitoring-scaling.md) - 실시간 모니터링, 자동 스케일링

### 시스템 성능
- [메인 성능 시나리오](./core-scenarios-performance-scalability.md) - 전체 성능 최적화 개요
