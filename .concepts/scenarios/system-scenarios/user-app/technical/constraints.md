# User App 기술적 제약사항 및 성능 최적화

## 개요

Base.md에서 정의된 핵심 기술적 제약사항과 해결책을 사용자 앱 관점에서 구체적인 기술 사양으로 제공합니다. 실제 앱 구현 시 발생할 수 있는 기술적 한계와 이를 극복하는 방법을 다룹니다.

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

### 1.2 iOS 백그라운드 제약 해결
```swift
// iOS Background App Refresh 최적화
class BackgroundLocationManager {
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
```

### 1.3 Android 배터리 최적화 대응
```kotlin
// Android Doze 모드 대응
class BatteryOptimizationHandler {
    // 화이트리스트 요청
    fun requestBatteryOptimizationExemption() {
        val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS)
        intent.data = Uri.parse("package:$packageName")
        startActivity(intent)
    }
    
    // 포그라운드 서비스 활용
    fun startLocationService() {
        val serviceIntent = Intent(this, LocationTrackingService::class.java)
        ContextCompat.startForegroundService(this, serviceIntent)
    }
    
    // WorkManager로 주기적 체크
    val workRequest = PeriodicWorkRequestBuilder<LocationCheckWorker>(
        15, TimeUnit.MINUTES
    ).build()
}
