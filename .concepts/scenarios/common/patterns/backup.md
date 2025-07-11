# Common Technical Patterns - 데이터 백업 및 복구

## 📊 개요

모든 서비스에서 공통으로 사용되는 데이터 백업, 복구, 아카이브 패턴을 정의합니다.

**주요 포함 내용:**
- 자동화된 백업 시스템
- 데이터 복구 전략
- 아카이브 관리
- 재해 복구 계획

---

## 💾 데이터 백업 및 복구

#### 백업 매니저

```typescript
// common/data/BackupManager.ts
class BackupManager {
  private config: BackupConfig;
  private compressionEnabled: boolean;
  
  constructor(config: BackupConfig) {
    this.config = config;
    this.compressionEnabled = config.compression || true;
  }
  
  async createBackup(dataType: BackupDataType): Promise<BackupResult> {
    const backupId = this.generateBackupId();
    const startTime = Date.now();
    
    try {
      let data: any;
      
      switch (dataType) {
        case 'attendance_records':
          data = await this.exportAttendanceRecords();
          break;
        case 'participant_data':
          data = await this.exportParticipantData();
          break;
        case 'configuration':
          data = await this.exportConfiguration();
          break;
        case 'full_backup':
          data = await this.exportFullData();
          break;
        default:
          throw new Error(`Unknown backup type: ${dataType}`);
      }
      
      const backup: BackupData = {
        id: backupId,
        type: dataType,
        timestamp: new Date(),
        version: this.config.version,
        data
      };
      
      const backupContent = this.compressionEnabled
        ? await this.compressData(backup)
        : JSON.stringify(backup);
      
      await this.saveBackup(backupId, backupContent);
      
      const duration = Date.now() - startTime;
      
      return {
        id: backupId,
        success: true,
        duration,
        size: backupContent.length,
        compressed: this.compressionEnabled
      };
      
    } catch (error) {
      return {
        id: backupId,
        success: false,
        error: error.message,
        duration: Date.now() - startTime
      };
    }
  }
  
  async restoreBackup(backupId: string): Promise<RestoreResult> {
    const startTime = Date.now();
    
    try {
      const backupContent = await this.loadBackup(backupId);
      
      const backup: BackupData = this.compressionEnabled
        ? await this.decompressData(backupContent)
        : JSON.parse(backupContent);
      
      await this.validateBackup(backup);
      
      const restoreResult = await this.restoreData(backup);
      
      const duration = Date.now() - startTime;
      
      return {
        success: true,
        duration,
        restoredRecords: restoreResult.recordCount,
        backupVersion: backup.version
      };
      
    } catch (error) {
      return {
        success: false,
        error: error.message,
        duration: Date.now() - startTime
      };
    }
  }
  
  async scheduleAutoBackup(): Promise<void> {
    const interval = this.config.autoBackupInterval || 24 * 60 * 60 * 1000; // 24시간
    
    setInterval(async () => {
      try {
        await this.createBackup('full_backup');
        console.log('Auto backup completed successfully');
      } catch (error) {
        console.error('Auto backup failed:', error);
      }
    }, interval);
  }
  
  private async exportAttendanceRecords(): Promise<AttendanceRecord[]> {
    // 출석 기록 내보내기
    const db = await this.getDatabase();
    return db.getAllAttendanceRecords();
  }
  
  private async exportParticipantData(): Promise<ParticipantData[]> {
    // 참가자 데이터 내보내기
    const db = await this.getDatabase();
    return db.getAllParticipants();
  }
  
  private async exportConfiguration(): Promise<ConfigData> {
    // 시스템 설정 내보내기
    const db = await this.getDatabase();
    return db.getAllConfigurations();
  }
  
  private async exportFullData(): Promise<FullBackupData> {
    // 전체 데이터 내보내기
    return {
      attendanceRecords: await this.exportAttendanceRecords(),
      participantData: await this.exportParticipantData(),
      configuration: await this.exportConfiguration(),
      metadata: {
        exportedAt: new Date(),
        version: this.config.version,
        schemaVersion: this.config.schemaVersion
      }
    };
  }
  
  private async restoreData(backup: BackupData): Promise<RestoreResult> {
    const db = await this.getDatabase();
    let recordCount = 0;
    
    await db.transaction(async (tx) => {
      switch (backup.type) {
        case 'attendance_records':
          recordCount = await this.restoreAttendanceRecords(tx, backup.data);
          break;
        case 'participant_data':
          recordCount = await this.restoreParticipantData(tx, backup.data);
          break;
        case 'configuration':
          recordCount = await this.restoreConfiguration(tx, backup.data);
          break;
        case 'full_backup':
          recordCount = await this.restoreFullData(tx, backup.data);
          break;
      }
    });
    
    return { recordCount };
  }
  
  private async compressData(data: any): Promise<string> {
    const jsonString = JSON.stringify(data);
    // 실제 구현에서는 gzip 또는 다른 압축 알고리즘 사용
    return btoa(jsonString); // 단순 base64 인코딩 (예시)
  }
  
  private async decompressData(compressedData: string): Promise<any> {
    const jsonString = atob(compressedData); // base64 디코딩 (예시)
    return JSON.parse(jsonString);
  }
  
  private async validateBackup(backup: BackupData): Promise<void> {
    // 백업 데이터 무결성 검증
    if (!backup.id || !backup.timestamp || !backup.data) {
      throw new Error('Invalid backup format');
    }
    
    // 버전 호환성 검사
    if (!this.isVersionCompatible(backup.version)) {
      throw new Error(`Incompatible backup version: ${backup.version}`);
    }
  }
  
  private isVersionCompatible(backupVersion: string): boolean {
    // 버전 호환성 로직
    const currentVersion = this.config.version;
    return this.compareVersions(backupVersion, currentVersion) >= -1; // 1버전 차이까지 허용
  }
  
  private compareVersions(v1: string, v2: string): number {
    const parts1 = v1.split('.').map(Number);
    const parts2 = v2.split('.').map(Number);
    
    for (let i = 0; i < Math.max(parts1.length, parts2.length); i++) {
      const part1 = parts1[i] || 0;
      const part2 = parts2[i] || 0;
      
      if (part1 > part2) return 1;
      if (part1 < part2) return -1;
    }
    
    return 0;
  }
  
  private generateBackupId(): string {
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const random = Math.random().toString(36).substr(2, 6);
    return `backup_${timestamp}_${random}`;
  }
}
```

#### 아카이브 매니저

```typescript
// common/data/ArchiveManager.ts
class ArchiveManager {
  private config: ArchiveConfig;
  private backupManager: BackupManager;
  
  constructor(config: ArchiveConfig, backupManager: BackupManager) {
    this.config = config;
    this.backupManager = backupManager;
  }
  
  async archiveOldData(): Promise<ArchiveResult> {
    const cutoffDate = this.getCutoffDate();
    
    try {
      // 오래된 데이터 식별
      const oldData = await this.identifyOldData(cutoffDate);
      
      // 아카이브 백업 생성
      const archiveBackup = await this.createArchiveBackup(oldData);
      
      // 원본 데이터베이스에서 제거
      await this.removeOldData(oldData);
      
      return {
        success: true,
        archivedRecords: oldData.recordCount,
        backupId: archiveBackup.id,
        freedSpace: await this.calculateFreedSpace(oldData)
      };
      
    } catch (error) {
      return {
        success: false,
        error: error.message
      };
    }
  }
  
  async retrieveArchivedData(
    startDate: Date,
    endDate: Date
  ): Promise<ArchivedData> {
    // 아카이브에서 특정 기간 데이터 검색
    const relevantBackups = await this.findRelevantBackups(startDate, endDate);
    
    const retrievedData: ArchivedData = {
      attendanceRecords: [],
      participantData: [],
      timeRange: { startDate, endDate }
    };
    
    for (const backup of relevantBackups) {
      const backupData = await this.backupManager.restoreBackup(backup.id);
      const filteredData = this.filterDataByDateRange(
        backupData,
        startDate,
        endDate
      );
      
      retrievedData.attendanceRecords.push(...filteredData.attendanceRecords);
      retrievedData.participantData.push(...filteredData.participantData);
    }
    
    return retrievedData;
  }
  
  async cleanupOldArchives(): Promise<CleanupResult> {
    const retentionPeriod = this.config.retentionPeriod || 365; // 1년
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - retentionPeriod);
    
    const oldArchives = await this.findArchivesOlderThan(cutoffDate);
    let deletedCount = 0;
    let errors: string[] = [];
    
    for (const archive of oldArchives) {
      try {
        await this.deleteArchive(archive.id);
        deletedCount++;
      } catch (error) {
        errors.push(`Failed to delete archive ${archive.id}: ${error.message}`);
      }
    }
    
    return {
      deletedCount,
      errors,
      success: errors.length === 0
    };
  }
  
  private getCutoffDate(): Date {
    const archiveAge = this.config.archiveAfterDays || 90; // 90일 후 아카이브
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - archiveAge);
    return cutoffDate;
  }
  
  private async identifyOldData(cutoffDate: Date): Promise<OldDataSet> {
    const db = await this.getDatabase();
    
    return {
      attendanceRecords: await db.getAttendanceRecordsOlderThan(cutoffDate),
      participantData: await db.getInactiveParticipantsOlderThan(cutoffDate),
      logs: await db.getLogsOlderThan(cutoffDate),
      recordCount: 0 // 계산 필요
    };
  }
  
  private async createArchiveBackup(oldData: OldDataSet): Promise<BackupResult> {
    return this.backupManager.createBackup('archive_data');
  }
}
```

#### 재해 복구 시스템

```typescript
// common/data/DisasterRecoveryManager.ts
class DisasterRecoveryManager {
  private backupManager: BackupManager;
  private replicationConfig: ReplicationConfig;
  
  constructor(
    backupManager: BackupManager,
    replicationConfig: ReplicationConfig
  ) {
    this.backupManager = backupManager;
    this.replicationConfig = replicationConfig;
  }
  
  async createDisasterRecoveryPlan(): Promise<DRPlan> {
    return {
      id: this.generatePlanId(),
      createdAt: new Date(),
      backupStrategy: {
        frequency: 'hourly',
        retention: '30days',
        destinations: this.replicationConfig.backupDestinations
      },
      recoveryObjectives: {
        RTO: 4 * 60 * 60 * 1000, // 4시간 (Recovery Time Objective)
        RPO: 1 * 60 * 60 * 1000  // 1시간 (Recovery Point Objective)
      },
      procedures: this.getRecoveryProcedures()
    };
  }
  
  async executeRecovery(scenario: DisasterScenario): Promise<RecoveryResult> {
    const startTime = Date.now();
    
    try {
      // 1. 시스템 상태 평가
      const systemStatus = await this.assessSystemStatus();
      
      // 2. 복구 전략 선택
      const strategy = this.selectRecoveryStrategy(scenario, systemStatus);
      
      // 3. 복구 실행
      const recoverySteps = await this.executeRecoverySteps(strategy);
      
      // 4. 시스템 검증
      const verification = await this.verifySystemIntegrity();
      
      const duration = Date.now() - startTime;
      
      return {
        success: true,
        scenario,
        strategy: strategy.name,
        duration,
        recoveredData: recoverySteps.recoveredRecords,
        verification
      };
      
    } catch (error) {
      return {
        success: false,
        scenario,
        error: error.message,
        duration: Date.now() - startTime
      };
    }
  }
  
  async testRecoveryProcedure(): Promise<TestResult> {
    // 재해 복구 절차 테스트
    const testEnvironment = await this.createTestEnvironment();
    
    try {
      // 모의 재해 시나리오 실행
      const mockDisaster = this.createMockDisaster();
      const recoveryResult = await this.executeRecovery(mockDisaster);
      
      // 복구 결과 검증
      const validation = await this.validateRecovery(testEnvironment, recoveryResult);
      
      return {
        success: validation.isValid,
        scenario: mockDisaster,
        duration: recoveryResult.duration,
        issues: validation.issues,
        recommendations: validation.recommendations
      };
      
    } finally {
      await this.cleanupTestEnvironment(testEnvironment);
    }
  }
  
  private async assessSystemStatus(): Promise<SystemStatus> {
    return {
      database: await this.checkDatabaseStatus(),
      storage: await this.checkStorageStatus(),
      network: await this.checkNetworkStatus(),
      backups: await this.checkBackupAvailability()
    };
  }
  
  private selectRecoveryStrategy(
    scenario: DisasterScenario,
    status: SystemStatus
  ): RecoveryStrategy {
    if (scenario.severity === 'total_loss') {
      return this.getFullRecoveryStrategy();
    } else if (scenario.type === 'data_corruption') {
      return this.getDataRestoreStrategy();
    } else {
      return this.getPartialRecoveryStrategy();
    }
  }
  
  private getRecoveryProcedures(): RecoveryProcedure[] {
    return [
      {
        step: 1,
        name: 'Assess Damage',
        description: 'Evaluate system status and data integrity',
        estimatedTime: 30 // minutes
      },
      {
        step: 2,
        name: 'Restore Infrastructure',
        description: 'Rebuild or repair system infrastructure',
        estimatedTime: 120
      },
      {
        step: 3,
        name: 'Restore Data',
        description: 'Recover data from latest valid backup',
        estimatedTime: 60
      },
      {
        step: 4,
        name: 'Verify System',
        description: 'Test system functionality and data integrity',
        estimatedTime: 45
      },
      {
        step: 5,
        name: 'Resume Operations',
        description: 'Return system to normal operations',
        estimatedTime: 15
      }
    ];
  }
  
  private generatePlanId(): string {
    return `drp_${Date.now()}_${Math.random().toString(36).substr(2, 6)}`;
  }
}
```

## 📊 백업 및 복구 전략

### 백업 스케줄
- **증분 백업**: 매시간
- **차등 백업**: 매일
- **전체 백업**: 매주
- **아카이브 백업**: 매월

### 복구 시간 목표 (RTO/RPO)
- **RTO (Recovery Time Objective)**: 4시간
- **RPO (Recovery Point Objective)**: 1시간
- **데이터 무결성**: 99.99%
- **복구 성공률**: 99.9%

## 🔗 관련 시나리오

### 📄 Common 패턴 연계
- **오프라인 동기화**: [technical-patterns-offline-sync.md](./technical-patterns-offline-sync.md)
- **메트릭 수집**: [technical-patterns-metrics.md](./technical-patterns-metrics.md)
- **BLE 통신**: [technical-patterns-ble-communication.md](./technical-patterns-ble-communication.md)
- **보안 인증**: [technical-patterns-security-auth.md](./technical-patterns-security-auth.md)

### 🌐 시스템 연동 영역
- **통합 플랫폼**: 중앙 집중식 백업 관리
- **모든 서비스**: 백업 데이터 제공 및 복구

---

## 📊 성능 지표

| 지표 | 목표 | 측정 방법 |
|------|------|-----------|
| 백업 완료 시간 | < 10분 (전체) | 백업 시작부터 완료까지 |
| 복구 완료 시간 | < 4시간 | 재해 발생부터 서비스 복구까지 |
| 데이터 손실률 | < 0.01% | RPO 내 데이터 보존율 |
| 백업 성공률 | > 99.9% | 예약된 백업의 성공 비율 |

---

**분할 정보**: 이 파일은 원본 `technical-patterns-data-sync.md` (842줄)에서 분할된 파일입니다.
**생성일**: 2024-12-19
**연관 파일**: technical-patterns-offline-sync.md, technical-patterns-metrics.md
