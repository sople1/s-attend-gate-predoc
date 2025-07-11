# User App GPS 정확도 제한 해결

## 📋 시나리오 개요

GPS의 실내 환경 제약을 극복하기 위한 하이브리드 위치 확인 시스템과 고급 센서 융합 기술 구현 방안입니다.
기계 학습 기반 위치 보정과 크라우드소싱 데이터 활용을 포함합니다.

---

## 3. GPS 정확도 제한 해결

### 3.1 실내 위치 확인 보완 시스템

```python
class IndoorLocationSystem:
    def __init__(self):
        self.wifi_fingerprints = {}
        self.ble_rssi_map = {}
        self.calibration_points = []
        self.sensor_fusion_filter = ExtendedKalmanFilter()
    
    def wifi_based_location(self, wifi_scan_results):
        """WiFi RSSI 기반 위치 추정"""
        location_candidates = []
        
        for ap_mac, rssi in wifi_scan_results.items():
            if ap_mac in self.wifi_fingerprints:
                candidates = self.wifi_fingerprints[ap_mac]
                weighted_locations = self.calculate_weighted_position(
                    candidates, rssi
                )
                location_candidates.extend(weighted_locations)
        
        if len(location_candidates) >= 3:
            return self.triangulate_position(location_candidates)
        else:
            return self.nearest_neighbor_estimation(location_candidates)
    
    def ble_triangulation(self, beacon_signals):
        """BLE 비콘 삼각측량"""
        if len(beacon_signals) < 3:
            return self.estimate_with_limited_beacons(beacon_signals)
            
        positions = []
        for beacon_id, rssi in beacon_signals.items():
            distance = self.rssi_to_distance(rssi, beacon_id)
            beacon_position = self.get_beacon_position(beacon_id)
            confidence = self.calculate_distance_confidence(rssi)
            positions.append((beacon_position, distance, confidence))
        
        return self.weighted_trilaterate(positions)
    
    def hybrid_location(self, gps, wifi, ble, accelerometer, gyroscope):
        """하이브리드 위치 결정 with 센서 융합"""
        confidence_scores = {}
        
        # GPS 신뢰도 (실내/실외 환경 감지)
        indoor_probability = self.detect_indoor_environment(wifi, ble)
        confidence_scores['gps'] = (1 - indoor_probability) * 0.9
            
        # WiFi 신뢰도
        wifi_aps = len(wifi)
        wifi_strength = sum(rssi for rssi in wifi.values()) / len(wifi) if wifi else -100
        confidence_scores['wifi'] = min(wifi_aps / 5.0, 0.8) * self.normalize_rssi(wifi_strength)
        
        # BLE 신뢰도  
        ble_beacons = len(ble)
        ble_avg_rssi = sum(rssi for rssi in ble.values()) / len(ble) if ble else -100
        confidence_scores['ble'] = min(ble_beacons / 3.0, 0.9) * self.normalize_rssi(ble_avg_rssi)
        
        # 관성 센서 보정
        motion_estimate = self.dead_reckoning_update(accelerometer, gyroscope)
        
        return self.sensor_fusion_filter.update(
            [gps, wifi, ble, motion_estimate], 
            confidence_scores
        )
    
    def rssi_to_distance(self, rssi, beacon_id):
        """RSSI를 거리로 변환 (환경별 보정 적용)"""
        beacon_config = self.get_beacon_config(beacon_id)
        tx_power = beacon_config['tx_power']
        
        # 기본 공식: d = 10^((TxPower - RSSI) / (10 * n))
        # n: 경로 손실 지수 (실내: 2-4, 실외: 2-3)
        path_loss_exponent = self.get_environment_path_loss_exponent()
        
        if rssi == 0:
            return -1  # 신호 없음
        
        ratio = tx_power * 1.0 / rssi
        if ratio < 1.0:
            return pow(ratio, 10)
        else:
            accuracy = (0.89976) * pow(ratio, 7.7095) + 0.111
            return accuracy
    
    def calibrate_environment(self, calibration_data):
        """환경별 보정 데이터 학습"""
        # 기계 학습을 통한 RSSI-거리 관계 모델링
        features = []
        targets = []
        
        for point in calibration_data:
            feature_vector = [
                point['rssi'],
                point['beacon_count'],
                point['wifi_density'],
                point['environment_type']  # 0: 실외, 1: 실내, 2: 지하
            ]
            features.append(feature_vector)
            targets.append(point['actual_distance'])
        
        self.distance_model = self.train_distance_model(features, targets)
    
    def detect_indoor_environment(self, wifi_aps, ble_beacons):
        """실내/실외 환경 감지"""
        # WiFi AP 밀도가 높으면 실내일 가능성 높음
        wifi_density = len(wifi_aps)
        
        # BLE 비콘 존재는 실내 지표
        beacon_presence = len(ble_beacons) > 0
        
        # GPS 신호 강도 (낮으면 실내)
        gps_strength = self.get_current_gps_strength()
        
        indoor_score = (
            min(wifi_density / 10.0, 1.0) * 0.4 +
            (1.0 if beacon_presence else 0.0) * 0.4 +
            (1.0 - min(gps_strength / 50.0, 1.0)) * 0.2
        )
        
        return indoor_score
```

### 3.2 위치 정확도 개선 알고리즘

```json
{
  "location_accuracy_improvement": {
    "kalman_filter": {
      "purpose": "GPS 노이즈 제거 및 예측",
      "parameters": {
        "process_noise": 0.01,
        "measurement_noise": 5.0,
        "prediction_model": "constant_velocity",
        "update_frequency": "1Hz"
      },
      "state_vector": ["x", "y", "velocity_x", "velocity_y"]
    },
    "particle_filter": {
      "purpose": "실내 복잡 환경 대응",
      "parameters": {
        "particle_count": 1000,
        "resampling_threshold": 0.5,
        "motion_model": "pedestrian_dead_reckoning",
        "observation_model": "multi_sensor_likelihood"
      }
    },
    "sensor_fusion": {
      "sensors": [
        "GPS", "WiFi", "BLE", "Accelerometer", 
        "Gyroscope", "Magnetometer", "Barometer"
      ],
      "fusion_algorithm": "Extended_Kalman_Filter",
      "update_frequency": "10Hz",
      "adaptive_weighting": true
    },
    "machine_learning": {
      "algorithm": "Random_Forest_Regression",
      "features": [
        "rssi_values", "wifi_fingerprint", "beacon_distances",
        "motion_pattern", "time_of_day", "user_context"
      ],
      "training_data": "crowdsourced_location_corrections"
    }
  }
}
```

**고급 위치 추정 구현:**
```kotlin
class AdvancedLocationEstimator {
    private val kalmanFilter = LocationKalmanFilter()
    private val particleFilter = ParticleFilter(1000)
    private val mlModel = LocationMLModel()
    
    data class LocationEstimate(
        val position: LatLng,
        val accuracy: Float,
        val confidence: Float,
        val timestamp: Long,
        val sources: List<LocationSource>
    )
    
    fun estimateLocation(sensorData: SensorData): LocationEstimate {
        // 1. 각 센서별 위치 추정
        val gpsEstimate = processGPS(sensorData.gps)
        val wifiEstimate = processWiFi(sensorData.wifi)
        val bleEstimate = processBLE(sensorData.ble)
        val motionEstimate = processMotion(sensorData.imu)
        
        // 2. 신뢰도 계산
        val confidences = calculateConfidences(
            gpsEstimate, wifiEstimate, bleEstimate, motionEstimate
        )
        
        // 3. 센서 융합
        val fusedLocation = when {
            isHighAccuracyRequired() -> {
                particleFilter.update(
                    listOf(gpsEstimate, wifiEstimate, bleEstimate, motionEstimate),
                    confidences
                )
            }
            else -> {
                kalmanFilter.update(
                    listOf(gpsEstimate, wifiEstimate, bleEstimate),
                    confidences
                )
            }
        }
        
        // 4. 기계 학습 모델로 후처리
        val refinedLocation = mlModel.refineLocation(
            fusedLocation, 
            sensorData,
            getUserContext()
        )
        
        return LocationEstimate(
            position = refinedLocation.position,
            accuracy = refinedLocation.accuracy,
            confidence = calculateOverallConfidence(confidences),
            timestamp = System.currentTimeMillis(),
            sources = listOf(
                LocationSource.GPS, LocationSource.WiFi, 
                LocationSource.BLE, LocationSource.Motion
            )
        )
    }
    
    private fun calculateConfidences(vararg estimates: LocationEstimate?): Map<LocationSource, Float> {
        return mapOf(
            LocationSource.GPS to calculateGPSConfidence(estimates[0]),
            LocationSource.WiFi to calculateWiFiConfidence(estimates[1]),
            LocationSource.BLE to calculateBLEConfidence(estimates[2]),
            LocationSource.Motion to calculateMotionConfidence(estimates[3])
        )
    }
    
    private fun calculateGPSConfidence(estimate: LocationEstimate?): Float {
        return when {
            estimate == null -> 0f
            estimate.accuracy > 50f -> 0.2f  // 낮은 정확도
            estimate.accuracy > 20f -> 0.6f  // 보통 정확도
            else -> 0.9f                     // 높은 정확도
        }
    }
    
    // 적응형 가중치 조정
    private fun adaptWeights(locationHistory: List<LocationEstimate>) {
        val recentAccuracy = locationHistory.takeLast(10)
            .groupBy { it.sources.firstOrNull() }
            .mapValues { (_, estimates) -> 
                estimates.map { it.accuracy }.average() 
            }
        
        // 최근 성능이 좋은 센서에 더 높은 가중치 부여
        kalmanFilter.updateSensorWeights(recentAccuracy)
    }
}

// 크라우드소싱 기반 위치 보정
class CrowdsourcedLocationCorrection {
    fun submitLocationCorrection(
        estimatedLocation: LatLng,
        actualLocation: LatLng,
        sensorData: SensorData
    ) {
        val correction = LocationCorrection(
            estimatedLocation = estimatedLocation,
            actualLocation = actualLocation,
            error = estimatedLocation.distanceTo(actualLocation),
            sensorSignatures = extractSensorSignatures(sensorData),
            timestamp = System.currentTimeMillis(),
            userId = getCurrentUserId()
        )
        
        uploadLocationCorrection(correction)
    }
    
    private fun extractSensorSignatures(sensorData: SensorData): SensorSignatures {
        return SensorSignatures(
            wifiFingerprint = sensorData.wifi.map { 
                WifiAP(it.ssid, it.bssid, it.level) 
            },
            bleBeacons = sensorData.ble.map { 
                BleBeacon(it.uuid, it.major, it.minor, it.rssi) 
            },
            environmentType = detectEnvironmentType(sensorData),
            motionPattern = analyzeMotionPattern(sensorData.imu)
        )
    }
}
```

---

## 📋 관련 개념

### 상위 수준 연결
- [Technical Performance Optimization 개요](./technical-performance-optimization.md) - 전체 성능 최적화 개요
- [User App API](../core-apis/user-app-api.md) - API 통합 지점

### 병렬 시나리오  
- [앱 상태별 감지 성능 최적화](./technical-performance-app-states.md) - 앱 상태별 최적화
- [BLE 비콘 제약사항 해결](./technical-performance-ble-constraints.md) - BLE 비콘 한계 극복

### 기술 패턴
- [BLE 통신 패턴](../../common/technical-patterns-ble-communication.md) - BLE 기본 통신 구현
- [데이터 동기화 패턴](../../common/technical-patterns-data-sync.md) - 오프라인 동기화

---

## 🔗 관련 시나리오

### 연결된 시나리오
- **[네트워크 최적화](./technical-network-optimization.md)**: 오프라인 모드 및 데이터 최적화
- **[접근성 기본](./accessibility-basic.md)**: 성능과 접근성의 균형
- **[접근성 고급](./accessibility-advanced.md)**: 보조 기술 성능 최적화
- **[Gate Management](../gate-management/system-operations.md)**: BLE 비콘 하드웨어 연동

### 기술 연동
- **센서 융합**: Kalman Filter + Particle Filter
- **기계 학습**: TensorFlow Lite for on-device inference
- **신호 처리**: RSSI filtering and triangulation
- **배터리 최적화**: Adaptive scanning and background processing

---

## 📊 메트릭 및 성능 지표

### 위치 정확도
| 환경 | 목표 정확도 | 신뢰도 | 응답 시간 | 측정 방법 |
|------|-------------|--------|-----------|-----------|
| 실외 GPS | ±3-5m | 95% | < 3초 | GPS 위성 신호 |
| 실내 하이브리드 | ±5-10m | 90% | < 5초 | WiFi + BLE + IMU |
| BLE 근접 감지 | ±2-3m | 95% | < 2초 | RSSI 삼각측량 |
| 응급 대체 | ±20-50m | 80% | < 10초 | 네트워크 기반 |

### 배터리 효율성
| 모드 | 연속 사용 시간 | 시간당 배터리 소모 | 최적화 기법 |
|------|-----------------|-------------------|---------------|
| 포그라운드 사용 | 8-10시간 | 10-12% | 고성능 모드 |
| 백그라운드 추적 | 24-48시간 | 2-5% | 적응형 스캔 |
| 최적화 모드 | 48-72시간 | 1-2% | 최소 스캔 |

### 시스템 안정성
| 지표 | 목표 | 측정 방법 | 개선 방안 |
|------|------|-----------|-----------|
| 크래시율 | < 0.1% | 앱 분석 도구 | 예외 처리 강화 |
| 메모리 사용량 | < 100MB | 프로파일링 | 메모리 풀 관리 |
| 네트워크 효율성 | < 1MB/hour | 트래픽 모니터링 | 데이터 압축 |
| 감지 성공률 | > 95% | 실제 사용 통계 | 센서 융합 개선 |
