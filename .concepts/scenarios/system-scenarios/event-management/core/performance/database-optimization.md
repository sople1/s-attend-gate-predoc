# Event Management - Database Optimization

## 🎯 데이터베이스 최적화 전략

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
```

### 파티셔닝 전략

```sql
-- 시간 기반 파티셔닝
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
             a.timestamp as last_attendance,
             a.gate_id
      FROM participants p
      LEFT JOIN LATERAL (
        SELECT timestamp, gate_id 
        FROM attendance_records ar 
        WHERE ar.participant_id = p.id
        ORDER BY timestamp DESC 
        LIMIT 1
      ) a ON true
      WHERE p.id = ANY($1)
    `;
    
    return this.db.query(query, [participantIds]);
  }
}
```

### 구현 고려사항

1. 인덱스 관리
   - 사용 패턴 기반 인덱스 설계
   - 불필요한 인덱스 제거
   - 인덱스 재구성 스케줄링

2. 파티셔닝
   - 시간 기반 파티셔닝
   - 이벤트 기반 파티셔닝
   - 파티션 유지보수 자동화

3. 쿼리 튜닝
   - 실행 계획 분석
   - N+1 문제 해결
   - 캐시 활용

## 📌 참고
- [캐싱 전략](/core/performance/caching-strategy.md)
- [연결 풀 최적화](/core/performance/connection-pool.md)
- [마이크로서비스 아키텍처](/core/architecture/microservices.md)
