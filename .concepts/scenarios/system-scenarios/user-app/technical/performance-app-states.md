# User App 앱 상태별 감지 성능 최적화

## 📋 시나리오 개요

모바일 앱의 다양한 실행 상태(포그라운드, 백그라운드, 종료)에서 BLE 비콘 감지 성능을 최적화하는 기술적 구현 방안입니다.
iOS와 Android의 플랫폼별 제약사항과 대응 전략을 포함합니다.

---

## 1. 앱 상태별 감지 성능 최적화

### 1.1 성능 지표 및 목표

```json
{
  "performance_targets": {
    "foreground_state": {
      "success_rate": "95-98%",
      "response_time": "2-3초",
      "ble_range": "10-15m",
      "gps_accuracy": "±5m"
    },
    "background_state": {
      "success_rate": "80-85%",
      "response_time": "5-10초",
      "update_interval": "30초",
      "battery_impact": "최소"
    },
    "terminated_state": {
      "success_rate": "0%",
      "fallback_required": true,
      "notification_trigger": "BLE 신호 감지 시 앱 실행 유도"
    }
  }
}
```

**성능 최적화 전략:**
- **포그라운드**: 최대 성능 모드로 실시간 감지
- **백그라운드**: 배터리 효율성과 성능의 균형
- **종료 상태**: 시스템 알림을 통한 재활성화

### 1.2 iOS 백그라운드 제약 해결

```swift
// iOS Background App Refresh 최적화
class BackgroundLocationManager {
    private let locationManager = CLLocationManager()
    private let centralManager = CBCentralManager()
    
    func setupBackgroundTracking() {
        // 중요 위치 변화 감지
        locationManager.startMonitoringSignificantLocationChanges()
        
        // 지오펜스 활용
        let region = CLCircularRegion(
            center: eventLocation,
            radius: 100,
            identifier: "event_area"
        )
        locationManager.startMonitoring(for: region)
        
        // BLE 백그라운드 스캔 (제한적)
        centralManager.scanForPeripherals(
            withServices: [eventBeaconUUID],
            options: [CBCentralManagerScanOptionAllowDuplicatesKey: false]
        )
    }
    
    // 백그라운드 앱 새로 고침 최적화
    func handleBackgroundRefresh() {
        // 최소한의 작업만 수행
        performCriticalLocationCheck { result in
            // 중요한 변화가 있을 때만 서버 통신
            if result.significantChange {
                uploadLocationUpdate(result)
            }
        }
    }
    
    // 로컬 알림 활용
    func scheduleLocationNotification() {
        let content = UNMutableNotificationContent()
        content.title = "출석 확인"
        content.body = "행사장 근처에 계신 것 같습니다. 출석을 확인해주세요."
        content.sound = .default
        
        let trigger = UNLocationNotificationTrigger(
            region: eventGeofence,
            repeats: false
        )
        
        let request = UNNotificationRequest(
            identifier: "attendance_reminder",
            content: content,
            trigger: trigger
        )
        
        UNUserNotificationCenter.current().add(request)
    }
}

// iOS 15+ Background Processing 활용
class BackgroundProcessingManager {
    func registerBackgroundTasks() {
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.app.location-sync",
            using: nil
        ) { task in
            self.handleLocationSync(task: task as! BGProcessingTask)
        }
    }
    
    func scheduleLocationSync() {
        let request = BGProcessingTaskRequest(identifier: "com.app.location-sync")
        request.requiresNetworkConnectivity = true
        request.requiresExternalPower = false
        request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60) // 15분 후
        
        try? BGTaskScheduler.shared.submit(request)
    }
}
```

### 1.3 Android 배터리 최적화 대응

```kotlin
// Android Doze 모드 및 배터리 최적화 대응
class BatteryOptimizationHandler(private val context: Context) {
    
    // 배터리 최적화 예외 요청
    fun requestBatteryOptimizationExemption() {
        val powerManager = context.getSystemService(Context.POWER_SERVICE) as PowerManager
        
        if (!powerManager.isIgnoringBatteryOptimizations(context.packageName)) {
            val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS)
            intent.data = Uri.parse("package:${context.packageName}")
            context.startActivity(intent)
        }
    }
    
    // 포그라운드 서비스 활용
    fun startLocationService() {
        val serviceIntent = Intent(context, LocationTrackingService::class.java)
        ContextCompat.startForegroundService(context, serviceIntent)
    }
    
    // WorkManager로 주기적 체크
    fun schedulePeriodicLocationCheck() {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(false) // 배터리 부족해도 실행
            .build()
            
        val workRequest = PeriodicWorkRequestBuilder<LocationCheckWorker>(
            15, TimeUnit.MINUTES // 최소 15분 간격
        )
        .setConstraints(constraints)
        .setBackoffCriteria(
            BackoffPolicy.LINEAR,
            PeriodicWorkRequest.MIN_BACKOFF_MILLIS,
            TimeUnit.MILLISECONDS
        )
        .build()
        
        WorkManager.getInstance(context)
            .enqueueUniquePeriodicWork(
                "location_check",
                ExistingPeriodicWorkPolicy.REPLACE,
                workRequest
            )
    }
    
    // Doze 모드 감지 및 대응
    fun handleDozeMode() {
        val intentFilter = IntentFilter().apply {
            addAction(PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED)
        }
        
        context.registerReceiver(object : BroadcastReceiver() {
            override fun onReceive(context: Context?, intent: Intent?) {
                val powerManager = context?.getSystemService(Context.POWER_SERVICE) as PowerManager
                
                if (powerManager.isDeviceIdleMode) {
                    // Doze 모드 진입 - 최소한의 작업만 수행
                    scheduleHighPriorityLocationCheck()
                } else {
                    // Doze 모드 해제 - 정상 작업 재개
                    resumeNormalLocationTracking()
                }
            }
        }, intentFilter)
    }
}

// Android 12+ 정확한 알람 권한 처리
class AlarmPermissionManager {
    @RequiresApi(Build.VERSION_CODES.S)
    fun requestExactAlarmPermission() {
        val alarmManager = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager
        
        if (!alarmManager.canScheduleExactAlarms()) {
            val intent = Intent(Settings.ACTION_REQUEST_SCHEDULE_EXACT_ALARM)
            context.startActivity(intent)
        }
    }
}
```

---

## 📋 관련 개념

### 상위 수준 연결
- [Technical Performance Optimization 개요](./technical-performance-optimization.md) - 전체 성능 최적화 개요
- [User App API](../core-apis/user-app-api.md) - API 통합 지점

### 병렬 시나리오  
- [BLE 비콘 제약사항 해결](./technical-performance-ble-constraints.md) - BLE 비콘 한계 극복
- [GPS 정확도 제한 해결](./technical-performance-gps-accuracy.md) - GPS 정확도 향상

### 기술 패턴
- [BLE 통신 패턴](../../common/technical-patterns-ble-communication.md) - BLE 기본 통신 구현
- [보안 인증 패턴](../../common/technical-patterns-security-auth.md) - 인증 보안

---

## 📊 메트릭 및 성능 지표

| 앱 상태 | 성공률 목표 | 응답 시간 | 배터리 영향 | 측정 방법 |
|---------|-------------|-----------|-------------|-----------|
| 포그라운드 | 95-98% | 2-3초 | 높음 허용 | 실시간 모니터링 |
| 백그라운드 | 80-85% | 5-10초 | 최소화 | 주기적 체크 |
| 종료 상태 | 0% | N/A | 없음 | 알림 트리거 |
| 전체 평균 | 85-90% | 3-5초 | 중간 | 종합 분석 |
