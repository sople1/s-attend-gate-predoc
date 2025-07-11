# User App 테스트 및 품질 보증 시나리오

## 📋 개요

사용자 앱의 자동화 테스트 시나리오, 부하 테스트, 성능 회귀 방지 및 품질 보증 전략을 다룹니다.

---

## 7. 테스트 및 품질 보증

### 7.1 자동화 테스트 시나리오

```python
import pytest
import asyncio
from unittest.mock import Mock, patch
from attendance_system import AttendanceSystem, NetworkCondition, DeviceState

class TestAttendanceSystem:
    
    @pytest.fixture
    def attendance_system(self):
        return AttendanceSystem()
    
    @pytest.mark.parametrize("device_state", ["foreground", "background", "locked"])
    @pytest.mark.parametrize("network_condition", ["wifi", "4g", "3g", "slow_2g", "offline"])
    async def test_attendance_detection_comprehensive(self, attendance_system, device_state, network_condition):
        \"\"\"다양한 디바이스 상태와 네트워크 조건에서 출석 감지 테스트\"\"\"
        
        # 테스트 환경 설정
        simulator = AttendanceSimulator()
        simulator.set_device_state(DeviceState[device_state.upper()])
        simulator.set_network_condition(NetworkCondition[network_condition.upper()])
        
        # BLE 비콘 신호 시뮬레이션
        beacon_signal = simulator.generate_beacon_signal(
            rssi=-65, distance=5, duration=10, beacon_id="test_beacon_001"
        )
        
        # 출석 감지 테스트 실행
        result = await attendance_system.process_attendance(beacon_signal)
        
        # 기대 성공률 계산
        expected_success_rate = self.calculate_expected_success_rate(
            device_state, network_condition
        )
        
        # 결과 검증
        assert result.success_rate >= expected_success_rate, \
            f"Success rate {result.success_rate} below expected {expected_success_rate}"
        
        max_response_time = self.get_max_response_time(network_condition)
        assert result.response_time <= max_response_time, \
            f"Response time {result.response_time}ms exceeds limit {max_response_time}ms"
        
        # 네트워크별 추가 검증
        if network_condition == "offline":
            assert result.stored_offline == True
            assert result.sync_pending == True
        else:
            assert result.server_confirmed == True
    
    def test_location_accuracy_multi_sensor(self, attendance_system):
        \"\"\"다중 센서 융합 위치 정확도 테스트\"\"\"
        test_locations = [
            (37.5665, 126.9780, "서울시청"),  
            (37.5642, 126.9753, "명동"),
            (37.5172, 127.0473, "강남역")
        ]
        
        for true_lat, true_lng, location_name in test_locations:
            true_location = (true_lat, true_lng)
            
            # 센서 데이터 시뮬레이션
            gps_data = self.add_gps_noise(true_location, noise_level=5)
            wifi_data = self.simulate_wifi_scan(true_location)
            ble_data = self.simulate_ble_scan(true_location)
            
            # 위치 계산
            calculated_location = attendance_system.location_system.calculate_position(
                gps=gps_data,
                wifi_aps=wifi_data,
                ble_beacons=ble_data
            )
            
            # 정확도 검증
            accuracy = self.calculate_distance(true_location, calculated_location)
            assert accuracy <= 10.0, \
                f"Location accuracy {accuracy}m exceeds 10m limit at {location_name}"
            
            # 신뢰도 검증
            assert calculated_location.confidence >= 0.7, \
                f"Location confidence {calculated_location.confidence} too low"
    
    @pytest.mark.stress
    async def test_concurrent_attendance_processing(self, attendance_system):
        \"\"\"동시 다중 출석 처리 스트레스 테스트\"\"\"
        concurrent_users = 100
        
        async def simulate_user_attendance(user_id):
            beacon_signal = AttendanceSimulator().generate_beacon_signal(
                rssi=-60 + random.randint(-10, 10),
                distance=random.uniform(1, 15),
                duration=random.uniform(5, 15)
            )
            beacon_signal.user_id = f"user_{user_id}"
            
            return await attendance_system.process_attendance(beacon_signal)
        
        # 동시 처리 실행
        tasks = [simulate_user_attendance(i) for i in range(concurrent_users)]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # 결과 분석
        successful_results = [r for r in results if not isinstance(r, Exception)]
        error_results = [r for r in results if isinstance(r, Exception)]
        
        success_rate = len(successful_results) / len(results)
        assert success_rate >= 0.95, \
            f"Concurrent processing success rate {success_rate} below 95%"
        
        # 응답 시간 분석
        response_times = [r.response_time for r in successful_results]
        avg_response_time = sum(response_times) / len(response_times)
        p95_response_time = sorted(response_times)[int(len(response_times) * 0.95)]
        
        assert avg_response_time <= 1000, \
            f"Average response time {avg_response_time}ms exceeds 1000ms"
        assert p95_response_time <= 2000, \
            f"95th percentile response time {p95_response_time}ms exceeds 2000ms"
    
    def test_battery_optimization(self, attendance_system):
        \"\"\"배터리 최적화 테스트\"\"\"
        battery_levels = [100, 50, 20, 10, 5]
        
        for battery_level in battery_levels:
            attendance_system.battery_manager.update_battery_level(battery_level)
            
            # 배터리 수준에 따른 스캔 간격 조정 확인
            expected_interval = self.calculate_expected_scan_interval(battery_level)
            actual_interval = attendance_system.ble_scanner.get_scan_interval()
            
            assert actual_interval >= expected_interval, \
                f"Scan interval {actual_interval}ms should be >= {expected_interval}ms at {battery_level}% battery"
    
    @pytest.mark.security
    def test_data_encryption_privacy(self, attendance_system):
        \"\"\"데이터 암호화 및 개인정보 보호 테스트\"\"\"
        sensitive_data = {
            'user_id': 'test_user_12345',
            'location': {'lat': 37.5665, 'lng': 126.9780},
            'device_id': 'device_fingerprint_xyz',
            'personal_info': {'email': 'test@example.com', 'phone': '010-1234-5678'}
        }
        
        # 암호화 테스트
        encrypted_data = attendance_system.security_manager.encrypt_personal_data(sensitive_data)
        assert encrypted_data != sensitive_data, "Data should be encrypted"
        
        # 복호화 테스트
        decrypted_data = attendance_system.security_manager.decrypt_personal_data(encrypted_data)
        assert decrypted_data == sensitive_data, "Decrypted data should match original"
        
        # 익명화 테스트
        anonymized_data = attendance_system.security_manager.anonymize_data(sensitive_data)
        assert anonymized_data['user_id'] != sensitive_data['user_id'], "User ID should be anonymized"
        assert 'personal_info' not in anonymized_data, "Personal info should be removed"
    
    def calculate_expected_success_rate(self, device_state, network_condition):
        \"\"\"기대 성공률 계산\"\"\"
        base_rates = {
            "foreground": 0.95,
            "background": 0.85,
            "locked": 0.70
        }
        
        network_multipliers = {
            "wifi": 1.0,
            "4g": 0.98,
            "3g": 0.90,
            "slow_2g": 0.80,
            "offline": 0.95  # 오프라인 저장 성공률
        }
        
        return base_rates[device_state] * network_multipliers[network_condition]
    
    def calculate_expected_scan_interval(self, battery_level):
        \"\"\"배터리 수준별 기대 스캔 간격\"\"\"
        if battery_level >= 50:
            return 1000  # 1초
        elif battery_level >= 20:
            return 2000  # 2초
        elif battery_level >= 10:
            return 5000  # 5초
        else:
            return 10000 # 10초
```

### 7.2 부하 테스트

```javascript
// K6 부하 테스트 스크립트
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

// 커스텀 메트릭
const errorRate = new Rate('errors');
const attendanceSuccessRate = new Rate('attendance_success');

export let options = {
  scenarios: {
    // 점진적 부하 증가
    ramp_up: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },   // 2분 동안 100명까지 증가
        { duration: '5m', target: 500 },   // 5분 동안 500명까지 증가
        { duration: '10m', target: 1000 }, // 10분 동안 1000명 유지
        { duration: '5m', target: 100 },   // 5분 동안 100명으로 감소
        { duration: '2m', target: 0 },     // 2분 동안 0명으로 감소
      ],
    },
    
    // 스파이크 테스트
    spike_test: {
      executor: 'ramping-vus',
      startTime: '25m',
      stages: [
        { duration: '10s', target: 2000 }, // 급격한 증가
        { duration: '1m', target: 2000 },  // 유지
        { duration: '10s', target: 100 },  // 급격한 감소
      ],
    },
    
    // 안정성 테스트 (장시간 실행)
    soak_test: {
      executor: 'constant-vus',
      vus: 200,
      duration: '2h',
      startTime: '30m',
    }
  },
  thresholds: {
    'http_req_duration': ['p(95)<2000'], // 95%의 요청이 2초 이내
    'http_req_failed': ['rate<0.01'],    // 실패율 1% 미만
    'attendance_success': ['rate>0.95'], // 출석 성공률 95% 이상
  },
};

export default function() {
  const userId = `user_${__VU}_${__ITER}`;
  const eventId = 'load_test_event_2024';
  
  // 실제 사용자 행동 시뮬레이션
  const attendanceData = generateRealisticAttendanceData(userId, eventId);
  
  // 출석 체크 API 호출
  const response = http.post(
    'https://api.s-attend-gate.com/v1/attendance',
    JSON.stringify(attendanceData),
    {
      headers: { 
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${getAuthToken(userId)}`
      },
      timeout: '10s'
    }
  );
  
  // 응답 검증
  const success = check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 2000ms': (r) => r.timings.duration < 2000,
    'attendance recorded successfully': (r) => {
      const body = r.json();
      return body && body.success === true;
    },
    'no server errors': (r) => r.status < 500,
    'proper response format': (r) => {
      const body = r.json();
      return body && typeof body.timestamp === 'number';
    }
  });
  
  // 메트릭 기록
  errorRate.add(!success);
  attendanceSuccessRate.add(success);
  
  // 오프라인 시나리오 테스트
  if (Math.random() < 0.1) { // 10% 확률로 오프라인 시뮬레이션
    testOfflineMode(userId, eventId);
  }
  
  // 네트워크 상태 시뮬레이션
  const networkDelay = simulateNetworkConditions();
  sleep(networkDelay);
}

function generateRealisticAttendanceData(userId, eventId) {
  // 실제 사용자 위치 패턴 시뮬레이션
  const baseLocation = getEventLocation(eventId);
  const userLocation = addRealisticLocationVariation(baseLocation);
  
  return {
    userId: userId,
    eventId: eventId,
    timestamp: Date.now(),
    location: userLocation,
    method: selectRandomMethod(['ble_auto', 'qr_scan', 'manual']),
    beaconData: generateBeaconData(),
    deviceInfo: generateDeviceFingerprint(),
    networkQuality: getCurrentNetworkQuality()
  };
}

function testOfflineMode(userId, eventId) {
  // 오프라인 출석 저장 테스트
  const offlineData = {
    userId: userId,
    eventId: eventId,
    timestamp: Date.now(),
    offline: true,
    location: generateLocation(),
    method: 'ble_auto'
  };
  
  // 로컬 저장소에 저장 시뮬레이션
  const stored = storeOfflineAttendance(offlineData);
  check(stored, {
    'offline storage successful': (result) => result.success === true
  });
}

function simulateNetworkConditions() {
  // 실제 네트워크 상황 시뮬레이션
  const conditions = ['wifi', '4g', '3g', 'slow'];
  const condition = conditions[Math.floor(Math.random() * conditions.length)];
  
  const delays = {
    'wifi': Math.random() * 0.1,      // 0-100ms
    '4g': Math.random() * 0.5,        // 0-500ms
    '3g': Math.random() * 1.0,        // 0-1000ms
    'slow': Math.random() * 3.0       // 0-3000ms
  };
  
  return delays[condition];
}

// 성능 메트릭 수집
export function handleSummary(data) {
  return {
    'load-test-results.json': JSON.stringify(data, null, 2),
    'load-test-summary.html': generateHTMLReport(data),
  };
}

function generateHTMLReport(data) {
  const template = `
  <!DOCTYPE html>
  <html>
  <head>
    <title>Load Test Results</title>
    <style>
      body { font-family: Arial, sans-serif; margin: 20px; }
      .metric { margin: 10px 0; padding: 10px; border: 1px solid #ddd; }
      .pass { background-color: #d4edda; }
      .fail { background-color: #f8d7da; }
      .summary { background-color: #f8f9fa; padding: 15px; margin-bottom: 20px; }
    </style>
  </head>
  <body>
    <h1>S-Attend-Gate Load Test Results</h1>
    
    <div class="summary">
      <h2>Test Summary</h2>
      <p><strong>Total Requests:</strong> ${data.metrics.http_reqs.values.count}</p>
      <p><strong>Test Duration:</strong> ${(data.state.testRunDurationMs / 1000).toFixed(2)}s</p>
      <p><strong>Average VUs:</strong> ${data.metrics.vus.values.avg.toFixed(0)}</p>
    </div>
    
    <div class="metric ${data.metrics.http_req_duration.thresholds.p95?.ok ? 'pass' : 'fail'}">
      <h3>Response Time (95th percentile)</h3>
      <p>${data.metrics.http_req_duration.values.p95.toFixed(2)}ms</p>
      <p><em>Threshold: < 2000ms</em></p>
    </div>
    
    <div class="metric ${data.metrics.http_req_failed.thresholds.rate?.ok ? 'pass' : 'fail'}">
      <h3>Error Rate</h3>
      <p>${(data.metrics.http_req_failed.values.rate * 100).toFixed(2)}%</p>
      <p><em>Threshold: < 1%</em></p>
    </div>
    
    <div class="metric ${data.metrics.attendance_success?.thresholds?.rate?.ok ? 'pass' : 'fail'}">
      <h3>Attendance Success Rate</h3>
      <p>${((data.metrics.attendance_success?.values?.rate || 0) * 100).toFixed(2)}%</p>
      <p><em>Threshold: > 95%</em></p>
    </div>
    
    <div class="metric">
      <h3>Request Rate</h3>
      <p>${data.metrics.http_reqs.values.rate.toFixed(2)} requests/second</p>
    </div>
    
    <div class="metric">
      <h3>Data Transfer</h3>
      <p>Received: ${(data.metrics.data_received.values.count / 1024 / 1024).toFixed(2)} MB</p>
      <p>Sent: ${(data.metrics.data_sent.values.count / 1024 / 1024).toFixed(2)} MB</p>
    </div>
  </body>
  </html>
  `;
  
  return template;
}
```

### 7.3 성능 회귀 방지

```yaml
# CI/CD 파이프라인에서 성능 테스트 통합
performance_tests:
  pre_deployment:
    - name: "Unit Performance Tests"
      command: "pytest tests/performance/ -v"
      threshold:
        max_duration: "5m"
        memory_limit: "1GB"
    
    - name: "Integration Performance Tests"  
      command: "pytest tests/integration/performance/ -v"
      threshold:
        response_time_p95: "1500ms"
        throughput_min: "100 req/s"
    
    - name: "Load Test (Staging)"
      command: "k6 run --out influxdb tests/load/staging.js"
      threshold:
        concurrent_users: 500
        duration: "10m"
        error_rate_max: "0.5%"
  
  post_deployment:
    - name: "Production Smoke Test"
      command: "k6 run tests/load/production_smoke.js"
      threshold:
        duration: "2m"
        error_rate_max: "0.1%"
    
    - name: "Performance Monitoring"
      monitoring:
        - metric: "response_time_p95"
          alert_threshold: "2000ms"
        - metric: "error_rate"
          alert_threshold: "1%"
        - metric: "attendance_success_rate"
          alert_threshold: "95%"

# 성능 기준선 관리
performance_baselines:
  mobile_app:
    ble_scan_latency: "500ms"
    location_fix_time: "3s"
    battery_drain_rate: "5%/hour"
    memory_usage_max: "150MB"
    
  api_endpoints:
    attendance_creation: "200ms"
    user_authentication: "100ms"
    event_data_sync: "500ms"
    
  offline_capabilities:
    data_sync_batch_size: "1000 records"
    offline_storage_limit: "10MB"
    sync_success_rate: "99%"
```

## 🔗 관련 시나리오

### 연결된 시나리오
- [네트워크 및 오프라인 최적화](./technical-network-offline.md) - 네트워크 테스트 및 오프라인 기능 검증
- [성능 모니터링 및 보안](./technical-performance-monitoring.md) - 보안 테스트 및 성능 지표 검증
- [기술적 제약사항 해결](./technical-constraints-solutions.md) - 배터리 및 위치 정확도 테스트

### 기술 연동
- **자동화 테스트**: Pytest + Mock + AsyncIO
- **부하 테스트**: K6 + InfluxDB + Grafana  
- **CI/CD 통합**: GitHub Actions + Performance Gates
- **모니터링**: Prometheus + Alertmanager

## 📊 메트릭 및 성능 지표

### 테스트 커버리지
- **단위 테스트**: > 90% 코드 커버리지
- **통합 테스트**: > 80% 시나리오 커버리지
- **E2E 테스트**: 주요 사용자 여정 100%
- **성능 테스트**: 모든 API 엔드포인트

### 품질 게이트
- **기능 테스트**: 100% 통과
- **성능 회귀**: 0% 허용
- **보안 취약점**: Critical 0개
- **접근성 준수**: WCAG 2.1 AA 100%

### 부하 테스트 기준
- **동시 사용자**: 1000명 지원
- **응답 시간**: P95 < 2초
- **에러율**: < 1%
- **출석 성공률**: > 95%

---

*이 파일은 `technical-network-optimization.md` 분할의 세 번째 부분입니다. 원본 백업: `technical-network-optimization-old.md`*
