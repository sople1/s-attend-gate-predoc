# Common Technical Patterns - 오프라인 동기화

## 📊 개요

모든 서비스에서 공통으로 사용되는 오프라인 동기화 패턴을 정의합니다.

**주요 포함 내용:**
- 오프라인 동기화 엔진
- 로컬 데이터베이스 스키마
- 충돌 해결 전략
- 네트워크 복구 패턴

---

## 🔄 오프라인 동기화 패턴

### User App, Gate Management, Event Management에서 공통 사용

#### 로컬 데이터베이스 스키마

```sql
-- common/database/offline-schema.sql

-- 동기화 대기열 테이블
CREATE TABLE sync_queue (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  operation_type TEXT NOT NULL, -- 'INSERT', 'UPDATE', 'DELETE'
  table_name TEXT NOT NULL,
  record_id TEXT NOT NULL,
  data TEXT NOT NULL, -- JSON 형태의 데이터
  priority INTEGER DEFAULT 1, -- 우선순위 (1=높음, 5=낮음)
  retry_count INTEGER DEFAULT 0,
  max_retries INTEGER DEFAULT 3,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  last_attempt_at DATETIME,
  status TEXT DEFAULT 'pending' -- 'pending', 'syncing', 'completed', 'failed'
);

-- 출석 데이터 로컬 저장
CREATE TABLE local_attendance (
  id TEXT PRIMARY KEY,
  participant_id TEXT NOT NULL,
  event_id TEXT NOT NULL,
  check_in_time DATETIME,
  check_out_time DATETIME,
  gate_id TEXT,
  method TEXT, -- 'ble', 'qr', 'manual'
  sync_status TEXT DEFAULT 'pending', -- 'pending', 'synced', 'failed'
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 동기화 로그
CREATE TABLE sync_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  operation_id TEXT NOT NULL,
  operation_type TEXT NOT NULL,
  status TEXT NOT NULL, -- 'success', 'error'
  error_message TEXT,
  sync_duration INTEGER, -- milliseconds
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 참가자 캐시 테이블
CREATE TABLE participant_cache (
  id TEXT PRIMARY KEY,
  event_id TEXT NOT NULL,
  name TEXT NOT NULL,
  email TEXT,
  phone TEXT,
  qr_token TEXT,
  cache_expires_at DATETIME,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 오프라인 설정 캐시
CREATE TABLE config_cache (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  expires_at DATETIME,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

#### 동기화 엔진

```typescript
// common/sync/SyncEngine.ts
class SyncEngine {
  private db: SQLiteDatabase;
  private apiClient: APIClient;
  private isOnline: boolean = false;
  private syncInProgress: boolean = false;
  
  constructor(db: SQLiteDatabase, apiClient: APIClient) {
    this.db = db;
    this.apiClient = apiClient;
    
    // 네트워크 상태 모니터링
    this.monitorNetworkStatus();
    
    // 주기적 동기화 (온라인 상태일 때)
    this.startPeriodicSync();
  }
  
  async queueOperation(
    operation: SyncOperation
  ): Promise<void> {
    await this.db.executeSql(`
      INSERT INTO sync_queue (
        operation_type, table_name, record_id, data, priority
      ) VALUES (?, ?, ?, ?, ?)
    `, [
      operation.type,
      operation.tableName,
      operation.recordId,
      JSON.stringify(operation.data),
      operation.priority || 1
    ]);
    
    // 온라인 상태면 즉시 동기화 시도
    if (this.isOnline && !this.syncInProgress) {
      await this.syncPendingOperations();
    }
  }
  
  private async syncPendingOperations(): Promise<void> {
    if (this.syncInProgress) return;
    
    this.syncInProgress = true;
    
    try {
      // 우선순위 순으로 대기열에서 작업 가져오기
      const operations = await this.db.executeSql(`
        SELECT * FROM sync_queue 
        WHERE status = 'pending' AND retry_count < max_retries
        ORDER BY priority ASC, created_at ASC
        LIMIT 50
      `);
      
      for (const operation of operations.rows._array) {
        await this.processSyncOperation(operation);
      }
    } finally {
      this.syncInProgress = false;
    }
  }
  
  private async processSyncOperation(operation: SyncQueueItem): Promise<void> {
    const startTime = Date.now();
    
    try {
      // 상태를 'syncing'으로 업데이트
      await this.db.executeSql(`
        UPDATE sync_queue 
        SET status = 'syncing', last_attempt_at = CURRENT_TIMESTAMP
        WHERE id = ?
      `, [operation.id]);
      
      // 실제 동기화 실행
      await this.executeSyncOperation(operation);
      
      // 성공 시 큐에서 제거
      await this.db.executeSql(`
        DELETE FROM sync_queue WHERE id = ?
      `, [operation.id]);
      
      // 성공 로그 기록
      await this.logSyncResult(operation, 'success', Date.now() - startTime);
      
    } catch (error) {
      const newRetryCount = operation.retry_count + 1;
      
      if (newRetryCount >= operation.max_retries) {
        // 최대 재시도 횟수 초과 - 실패로 마킹
        await this.db.executeSql(`
          UPDATE sync_queue 
          SET status = 'failed', retry_count = ?
          WHERE id = ?
        `, [newRetryCount, operation.id]);
        
        await this.logSyncResult(operation, 'failed', Date.now() - startTime, error.message);
      } else {
        // 재시도 카운트 증가
        await this.db.executeSql(`
          UPDATE sync_queue 
          SET retry_count = ?, status = 'pending', last_attempt_at = CURRENT_TIMESTAMP
          WHERE id = ?
        `, [newRetryCount, operation.id]);
      }
    }
  }
  
  private async executeSyncOperation(operation: SyncQueueItem): Promise<void> {
    const data = JSON.parse(operation.data);
    
    switch (operation.operation_type) {
      case 'INSERT':
        await this.apiClient.post(`/api/${operation.table_name}`, data);
        break;
        
      case 'UPDATE':
        await this.apiClient.put(`/api/${operation.table_name}/${operation.record_id}`, data);
        break;
        
      case 'DELETE':
        await this.apiClient.delete(`/api/${operation.table_name}/${operation.record_id}`);
        break;
        
      default:
        throw new Error(`Unknown operation type: ${operation.operation_type}`);
    }
  }
  
  async getSyncStatus(): Promise<SyncStatus> {
    const pendingCount = await this.db.executeSql(`
      SELECT COUNT(*) as count FROM sync_queue WHERE status = 'pending'
    `);
    
    const failedCount = await this.db.executeSql(`
      SELECT COUNT(*) as count FROM sync_queue WHERE status = 'failed'
    `);
    
    const lastSync = await this.db.executeSql(`
      SELECT MAX(created_at) as last_sync FROM sync_log WHERE status = 'success'
    `);
    
    return {
      isOnline: this.isOnline,
      pendingOperations: pendingCount.rows.item(0).count,
      failedOperations: failedCount.rows.item(0).count,
      lastSuccessfulSync: lastSync.rows.item(0).last_sync,
      syncInProgress: this.syncInProgress
    };
  }
}
```

#### 충돌 해결 전략

```typescript
// common/sync/ConflictResolver.ts
class ConflictResolver {
  async resolveConflict(
    localData: any,
    serverData: any,
    conflictType: ConflictType
  ): Promise<ConflictResolution> {
    switch (conflictType) {
      case 'attendance_duplicate':
        return this.resolveAttendanceConflict(localData, serverData);
        
      case 'participant_update':
        return this.resolveParticipantConflict(localData, serverData);
        
      case 'config_mismatch':
        return this.resolveConfigConflict(localData, serverData);
        
      default:
        return this.useServerWins(serverData);
    }
  }
  
  private resolveAttendanceConflict(
    localData: AttendanceRecord,
    serverData: AttendanceRecord
  ): ConflictResolution {
    // 출석 기록의 경우 더 빠른 타임스탬프를 우선
    const localTime = new Date(localData.timestamp).getTime();
    const serverTime = new Date(serverData.timestamp).getTime();
    
    if (localTime < serverTime) {
      return {
        resolution: 'use_local',
        data: localData,
        reason: 'Earlier timestamp'
      };
    } else {
      return {
        resolution: 'use_server',
        data: serverData,
        reason: 'Later timestamp'
      };
    }
  }
  
  private resolveParticipantConflict(
    localData: ParticipantRecord,
    serverData: ParticipantRecord
  ): ConflictResolution {
    // 참가자 정보는 더 최근 업데이트를 우선
    const localUpdate = new Date(localData.updated_at).getTime();
    const serverUpdate = new Date(serverData.updated_at).getTime();
    
    if (localUpdate > serverUpdate) {
      return {
        resolution: 'use_local',
        data: localData,
        reason: 'More recent update'
      };
    } else {
      return {
        resolution: 'use_server',
        data: serverData,
        reason: 'More recent update'
      };
    }
  }
  
  private resolveConfigConflict(
    localData: ConfigRecord,
    serverData: ConfigRecord
  ): ConflictResolution {
    // 설정은 항상 서버를 우선
    return {
      resolution: 'use_server',
      data: serverData,
      reason: 'Server configuration takes precedence'
    };
  }
  
  private useServerWins(serverData: any): ConflictResolution {
    return {
      resolution: 'use_server',
      data: serverData,
      reason: 'Default: server wins'
    };
  }
}
```

## 🔗 관련 시나리오

### 📄 Common 패턴 연계
- **메트릭 수집**: [technical-patterns-metrics.md](./technical-patterns-metrics.md)
- **백업 복구**: [technical-patterns-backup.md](./technical-patterns-backup.md)
- **BLE 통신**: [technical-patterns-ble-communication.md](./technical-patterns-ble-communication.md)
- **보안 인증**: [technical-patterns-security-auth.md](./technical-patterns-security-auth.md)

### 🌐 시스템 연동 영역
- **사용자 앱**: Gate Management, Event Management 연동
- **게이트 관리**: User App 출석 데이터 수신
- **이벤트 관리**: 모든 출석 데이터 통합 관리

---

## 📊 성능 지표

| 지표 | 목표 | 측정 방법 |
|------|------|-----------|
| 동기화 지연 시간 | < 5초 | 큐 대기 시간 측정 |
| 오프라인 데이터 보존 | 7일 | 로컬 스토리지 용량 |
| 동기화 성공률 | > 99% | 성공/실패 비율 |
| 충돌 해결률 | > 95% | 자동 해결 비율 |

---

**분할 정보**: 이 파일은 원본 `technical-patterns-data-sync.md` (842줄)에서 분할된 파일입니다.
**생성일**: 2024-12-19
**연관 파일**: technical-patterns-metrics.md, technical-patterns-backup.md
