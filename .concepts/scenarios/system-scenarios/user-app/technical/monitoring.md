# User App 성능 모니터링 및 보안 시나리오

## 📋 개요

사용자 앱의 실시간 성능 지표 수집, 자동 최적화 시스템, 데이터 암호화 및 개인정보 보호를 다룹니다.

---

## 5. 성능 모니터링 및 최적화

### 5.1 실시간 성능 지표 수집

```python
class PerformanceMonitor:
    def __init__(self):
        self.metrics = {
            'ble_scan_success_rate': RollingAverage(window=100),
            'gps_accuracy': RollingAverage(window=50),
            'network_latency': RollingAverage(window=20),
            'battery_drain_rate': RollingAverage(window=10),
            'memory_usage': RollingAverage(window=30),
            'cpu_usage': RollingAverage(window=30)
        }
        self.event_counters = defaultdict(int)
        self.performance_alerts = []
    
    def record_ble_scan(self, success: bool, rssi: float, duration: float, beacon_id: str):
        self.metrics['ble_scan_success_rate'].add(1.0 if success else 0.0)
        
        if success:
            self.record_custom_metric('ble_rssi', rssi)
            self.record_custom_metric('ble_scan_duration', duration)
            self.increment_counter(f'ble_beacon_{beacon_id}')
        else:
            self.increment_counter('ble_scan_failures')
            self.check_ble_performance_degradation()
    
    def record_location_fix(self, accuracy: float, method: str, duration: float):
        self.metrics['gps_accuracy'].add(accuracy)
        self.increment_counter(f'location_method_{method}')
        self.record_custom_metric(f'location_fix_time_{method}', duration)
        
        # 위치 정확도 임계값 체크
        if accuracy > 20.0:  # 20m 이상 오차
            self.record_performance_alert(
                'poor_location_accuracy',
                f'Location accuracy: {accuracy}m with method: {method}'
            )
    
    def record_network_operation(self, operation: str, latency: float, success: bool, data_size: int):
        self.metrics['network_latency'].add(latency)
        self.increment_counter(f'network_{operation}_{\"success\" if success else \"failure\"}')
        
        if success:
            throughput = data_size / (latency / 1000) if latency > 0 else 0
            self.record_custom_metric('network_throughput', throughput)
        
        # 네트워크 성능 임계값 체크
        if latency > 5000:  # 5초 이상 지연
            self.record_performance_alert(
                'high_network_latency',
                f'Network operation {operation} took {latency}ms'
            )
    
    def record_battery_info(self, level: float, is_charging: bool, temperature: float):
        if hasattr(self, 'last_battery_level') and hasattr(self, 'last_battery_time'):
            time_diff = time.time() - self.last_battery_time
            level_diff = self.last_battery_level - level
            
            if time_diff > 0 and not is_charging:
                drain_rate = (level_diff / time_diff) * 3600  # per hour
                self.metrics['battery_drain_rate'].add(drain_rate)
                
                # 배터리 드레인 임계값 체크
                if drain_rate > 10:  # 시간당 10% 이상 소모
                    self.record_performance_alert(
                        'high_battery_drain',
                        f'Battery draining at {drain_rate:.1f}% per hour'
                    )
        
