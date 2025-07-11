# User App 네트워크 및 오프라인 최적화 시나리오

## 📋 개요

사용자 앱의 네트워크 연결 최적화, 오프라인 모드 구현, 데이터 압축 및 전송 최적화를 다룹니다.

---

## 4. 네트워크 연결 최적화

### 4.1 오프라인 모드 구현

```typescript
interface OfflineStorage {
  attendanceRecords: AttendanceRecord[];
  syncQueue: SyncOperation[];
  lastSyncTimestamp: number;
  cacheData: CacheData;
}

class OfflineManager {
  private storage: OfflineStorage;
  private syncInProgress = false;
  
  async storeOfflineAttendance(record: AttendanceRecord): Promise<void> {
    // 로컬 저장
    record.offline = true;
    record.localTimestamp = Date.now();
    record.uuid = this.generateOfflineUUID();
    
    await this.storage.attendanceRecords.push(record);
    await this.storage.syncQueue.push({
      type: 'attendance',
      data: record,
      priority: 'high',
      attempts: 0,
      maxAttempts: 5
    });
    
    // 로컬 알림
    this.showOfflineNotification(record);
  }
  
  async syncWhenOnline(): Promise<SyncResult> {
    if (!navigator.onLine || this.syncInProgress) {
      return { success: false, reason: 'offline_or_busy' };
    }
    
    this.syncInProgress = true;
    const results: SyncResult[] = [];
    
    try {
      const pendingOperations = this.storage.syncQueue
        .sort((a, b) => b.priority === 'high' ? 1 : -1);
      
      for (const operation of pendingOperations) {
        try {
          const result = await this.syncOperation(operation);
          if (result.success) {
            this.removeSyncOperation(operation);
            results.push(result);
          } else {
            operation.attempts++;
            if (operation.attempts >= operation.maxAttempts) {
              this.moveToFailedQueue(operation);
            }
          }
        } catch (error) {
          console.error('Sync operation failed:', error);
          operation.attempts++;
        }
      }
      
      return {
        success: true,
        syncedCount: results.length,
        pendingCount: this.storage.syncQueue.length
      };
    } finally {
      this.syncInProgress = false;
    }
  }
  
  private async syncOperation(operation: SyncOperation): Promise<SyncResult> {
    const networkQuality = await this.detectNetworkQuality();
    
    // 네트워크 품질에 따른 타임아웃 조정
    const timeout = this.calculateTimeout(networkQuality);
    
    return await Promise.race([
      this.uploadToServer(operation.data),
      this.timeoutPromise(timeout)
    ]);
  }
  
  private async detectNetworkQuality(): Promise<NetworkQuality> {
    const connection = (navigator as any).connection;
    
    if (!connection) {
      // 연결 품질 테스트
      const startTime = Date.now();
      try {
        await fetch('/api/ping', { 
          method: 'HEAD',
          cache: 'no-cache'
        });
        const rtt = Date.now() - startTime;
        
        return {
          type: rtt < 100 ? 'high' : rtt < 500 ? 'medium' : 'low',
          rtt,
          estimated: true
        };
      } catch {
        return { type: 'offline', rtt: Infinity, estimated: true };
      }
    }
    
    return {
      type: connection.effectiveType || 'unknown',
      downlink: connection.downlink || 0,
      rtt: connection.rtt || 0,
      saveData: connection.saveData || false,
      estimated: false
    };
  }
  
  // 지능형 재시도 로직
  private calculateRetryDelay(attempt: number, networkQuality: NetworkQuality): number {
    const baseDelay = 1000; // 1초
    const exponentialFactor = Math.pow(2, attempt);
    const networkMultiplier = networkQuality.type === 'slow-2g' ? 3 : 
                              networkQuality.type === '3g' ? 2 : 1;
    
    return baseDelay * exponentialFactor * networkMultiplier;
  }
  
  // 스마트 캐싱 시스템
  async cacheEssentialData(): Promise<void> {
    const essentialData = {
      eventInfo: await this.fetchEventInfo(),
      beaconConfigs: await this.fetchBeaconConfigs(),
      userProfile: await this.fetchUserProfile(),
      staticContent: await this.fetchStaticContent()
    };
    
    await this.storage.cacheData = essentialData;
    
    // 서비스 워커에 캐시 업데이트 알림
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.ready.then(registration => {
        registration.active?.postMessage({
          type: 'CACHE_UPDATE',
          data: essentialData
        });
      });
    }
  }
}

// 서비스 워커 캐싱 전략
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/api/attendance')) {
    // 출석 API는 네트워크 우선, 실패 시 오프라인 저장
    event.respondWith(
      fetch(event.request).catch(() => {
        return self.registration.sync.register('attendance-sync');
      })
    );
  } else if (event.request.url.includes('/api/static')) {
    // 정적 데이터는 캐시 우선
    event.respondWith(
      caches.match(event.request).then(response => {
        return response || fetch(event.request);
      })
    );
  }
});
```

### 4.2 데이터 압축 및 최적화

```javascript
// 적응형 데이터 압축
class AdaptiveDataCompression {
  constructor() {
    this.compressionStrategies = {
      'slow-2g': { level: 9, batch: 10 },
      '3g': { level: 6, batch: 25 },
      '4g': { level: 3, batch: 50 },
      'wifi': { level: 1, batch: 100 }
    };
  }
  
  compressAttendanceData(data, networkType = '4g') {
    const strategy = this.compressionStrategies[networkType];
    
    // 기본 데이터 압축 (JSON 키 단축)
    const compressed = {
      u: data.userId,           // user_id
      e: data.eventId,          // event_id  
      t: data.timestamp,        // timestamp
      l: [data.lat, data.lng],  // location
      m: data.method,           // attendance_method
      s: data.status,           // status
      b: this.compressBLEData(data.bleData), // ble_data
      a: data.accuracy,         // accuracy
      c: data.confidence        // confidence
    };
    
    // 네트워크 품질에 따른 추가 압축
    if (strategy.level > 6) {
      return this.applyHeavyCompression(compressed);
    } else if (strategy.level > 3) {
      return this.applyMediumCompression(compressed);
    }
    
    return compressed;
  }
  
  compressBLEData(bleData) {
    if (!bleData) return null;
    
    return {
      u: bleData.uuid.replace(/-/g, ''), // UUID without hyphens
      j: bleData.major,                   // major
      n: bleData.minor,                   // minor  
      r: Math.round(bleData.rssi),        // rounded RSSI
      d: Math.round(bleData.distance * 10) / 10 // rounded distance
    };
  }
  
  // 배치 전송 최적화
  createOptimalBatch(records, networkType) {
    const strategy = this.compressionStrategies[networkType];
    const batches = [];
    
    for (let i = 0; i < records.length; i += strategy.batch) {
      const batch = records.slice(i, i + strategy.batch);
      const compressedBatch = {
        t: Date.now(), // batch timestamp
        c: batch.length, // count
        d: batch.map(record => this.compressAttendanceData(record, networkType))
      };
      
      batches.push(compressedBatch);
    }
    
    return batches;
  }
  
  // 이미지 최적화 (QR 코드 스캔용)
  optimizeImage(imageData, networkType) {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    
    const qualityMap = {
      'slow-2g': { width: 400, height: 300, quality: 0.5 },
      '3g': { width: 600, height: 450, quality: 0.7 },
      '4g': { width: 800, height: 600, quality: 0.8 },
      'wifi': { width: 1024, height: 768, quality: 0.9 }
    };
    
    const config = qualityMap[networkType] || qualityMap['4g'];
    
    canvas.width = config.width;
    canvas.height = config.height;
    
    ctx.drawImage(imageData, 0, 0, config.width, config.height);
    
    return canvas.toDataURL('image/jpeg', config.quality);
  }
}

// 네트워크 상태 모니터링
class NetworkMonitor {
  constructor() {
    this.connectionHistory = [];
    this.isMonitoring = false;
  }
  
  startMonitoring() {
    if (this.isMonitoring) return;
    
    this.isMonitoring = true;
    
    // Connection API 모니터링
    if ('connection' in navigator) {
      navigator.connection.addEventListener('change', this.handleConnectionChange.bind(this));
    }
    
    // 온라인/오프라인 상태 모니터링
    window.addEventListener('online', this.handleOnline.bind(this));
    window.addEventListener('offline', this.handleOffline.bind(this));
    
    // 주기적 네트워크 품질 테스트
    this.startQualityTesting();
  }
  
  handleConnectionChange(event) {
    const connection = event.target;
    const connectionInfo = {
      type: connection.effectiveType,
      downlink: connection.downlink,
      rtt: connection.rtt,
      saveData: connection.saveData,
      timestamp: Date.now()
    };
    
    this.connectionHistory.push(connectionInfo);
    this.adjustApplicationBehavior(connectionInfo);
  }
  
  adjustApplicationBehavior(connectionInfo) {
    const app = window.AttendanceApp;
    
    switch (connectionInfo.type) {
      case 'slow-2g':
      case '2g':
        app.setBandwidthMode('minimal');
        app.increaseOfflineCaching();
        break;
      case '3g':
        app.setBandwidthMode('reduced');
        break;
      case '4g':
        app.setBandwidthMode('normal');
        break;
      default:
        app.setBandwidthMode('optimal');
    }
    
    if (connectionInfo.saveData) {
      app.enableDataSaverMode();
    }
  }
  
  async startQualityTesting() {
    setInterval(async () => {
      if (navigator.onLine) {
        const quality = await this.testNetworkQuality();
        this.recordQualityMetric(quality);
      }
    }, 30000); // 30초마다 테스트
  }
  
  async testNetworkQuality() {
    const startTime = Date.now();
    const testSize = 1024; // 1KB 테스트 파일
    
    try {
      const response = await fetch(`/api/speed-test?size=${testSize}`, {
        cache: 'no-cache'
      });
      
      const endTime = Date.now();
      const latency = endTime - startTime;
      const dataSize = parseInt(response.headers.get('content-length') || '0');
      const throughput = dataSize / (latency / 1000); // bytes per second
      
      return {
        latency,
        throughput,
        timestamp: endTime,
        success: response.ok
      };
    } catch (error) {
      return {
        latency: Infinity,
        throughput: 0,
        timestamp: Date.now(),
        success: false,
        error: error.message
      };
    }
  }
}
```

## 🔗 관련 시나리오

### 연결된 시나리오
- [성능 모니터링 및 보안](./technical-performance-monitoring.md) - 성능 지표 수집 및 자동 최적화
- [테스트 및 품질 보증](./technical-testing-quality.md) - 네트워크 및 오프라인 기능 테스트
- [기술적 제약사항 해결](./technical-constraints-solutions.md) - 기본 성능 최적화 및 제약사항 대응

### 기술 연동
- **BLE 스캐닝**: [technical-performance-optimization.md](./technical-performance-optimization.md)
- **위치 서비스**: [core-scenarios.md](./core-scenarios.md)
- **데이터 동기화**: [../integrated-platform/data-integration-api.md](../integrated-platform/data-integration-api.md)

## 📊 메트릭 및 성능 지표

### 네트워크 성능
- **동기화 성공률**: > 95%
- **오프라인 저장**: 무제한 지원
- **데이터 압축률**: 60-80% 절약
- **배치 처리**: 네트워크 품질별 최적화

---

*이 파일은 `technical-network-optimization.md` 분할의 첫 번째 부분입니다. 원본 백업: `technical-network-optimization-old.md`*
