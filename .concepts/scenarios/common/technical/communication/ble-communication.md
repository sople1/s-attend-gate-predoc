# Common Technical Patterns - BLE 및 통신 패턴

## 🔧 개요

4개 서비스 그룹에서 공통으로 사용되는 BLE 통신 패턴과 API 클라이언트 구현 패턴을 정의합니다.

**주요 포함 내용:**
- BLE 스캐닝 및 연결 관리
- 근접 감지 및 거리 계산
- 표준 통신 프로토콜
- API 클라이언트 패턴
- 오류 처리 및 재시도 로직

---

## 📱 BLE (Bluetooth Low Energy) 통합 패턴

### User App과 Gate Management에서 공통 사용

#### BLE 스캐닝 및 연결 관리

```typescript
// common/ble/BLEManager.ts
interface BLEDevice {
  id: string;
  name: string;
  rssi: number;
  advertisementData: AdvertisementData;
  txPowerLevel?: number;
}

class BLEManager {
  private scanningActive: boolean = false;
  private connectedDevices: Map<string, BLEDevice> = new Map();
  
  async startScanning(options: BLEScanOptions): Promise<void> {
    if (this.scanningActive) return;
    
    this.scanningActive = true;
    
    const scanOptions = {
      serviceUUIDs: options.serviceUUIDs || [ATTEND_GATE_SERVICE_UUID],
      allowDuplicates: options.allowDuplicates || false,
      scanMode: options.scanMode || 'balanced', // low_power, balanced, low_latency
      timeout: options.timeout || 0 // 0 = continuous
    };
    
    await BluetoothManager.startDeviceScan(
      scanOptions.serviceUUIDs,
      scanOptions,
      this.onDeviceDiscovered.bind(this)
    );
  }
  
  private async onDeviceDiscovered(
    error: BleError | null,
    device: Device | null
  ): Promise<void> {
    if (error) {
      console.error('BLE scan error:', error);
      return;
    }
    
    if (!device) return;
    
    const bleDevice: BLEDevice = {
      id: device.id,
      name: device.name || device.localName || 'Unknown',
      rssi: device.rssi || -100,
      advertisementData: {
        manufacturerData: device.manufacturerData,
        serviceData: device.serviceData,
        serviceUUIDs: device.serviceUUIDs
      },
      txPowerLevel: device.txPowerLevel
    };
    
    // 거리 계산 (RSSI 기반)
    const distance = this.calculateDistance(bleDevice.rssi, bleDevice.txPowerLevel);
    
    // 임계 거리 내의 기기만 처리
    if (distance <= this.getProximityThreshold()) {
      await this.handleProximityDevice(bleDevice, distance);
    }
  }
  
  private calculateDistance(rssi: number, txPower?: number): number {
    if (!txPower) txPower = -59; // 기본값 (1m에서의 RSSI)
    
    if (rssi === 0) return -1.0;
    
    const ratio = (txPower - rssi) / 20.0;
    return Math.pow(10, ratio);
  }
  
  async connectToDevice(deviceId: string): Promise<Device> {
    const device = await BluetoothManager.connectToDevice(deviceId);
    
    await device.discoverAllServicesAndCharacteristics();
    
    this.connectedDevices.set(deviceId, device);
    
    return device;
  }
  
  async writeAttendanceData(
    deviceId: string,
    attendanceData: AttendanceData
  ): Promise<void> {
    const device = this.connectedDevices.get(deviceId);
    if (!device) throw new Error('Device not connected');
    
    const characteristic = await device.writeCharacteristicWithResponseForService(
      ATTEND_GATE_SERVICE_UUID,
      ATTENDANCE_CHARACTERISTIC_UUID,
      base64.encode(JSON.stringify(attendanceData))
    );
    
    console.log('Attendance data written successfully');
  }
  
  async readDeviceStatus(deviceId: string): Promise<DeviceStatus> {
    const device = this.connectedDevices.get(deviceId);
    if (!device) throw new Error('Device not connected');
    
    const characteristic = await device.readCharacteristicForService(
      ATTEND_GATE_SERVICE_UUID,
      STATUS_CHARACTERISTIC_UUID
    );
    
    const statusData = JSON.parse(base64.decode(characteristic.value));
    
    return {
      batteryLevel: statusData.battery,
      signalStrength: statusData.rssi,
      lastActivity: new Date(statusData.lastActivity),
      isOnline: statusData.online
    };
  }
}
```

#### BLE 설정 및 상수

```typescript
// common/ble/BLEConstants.ts
export const BLE_CONFIG = {
  // 서비스 UUID
  ATTEND_GATE_SERVICE_UUID: '6ba7b810-9dad-11d1-80b4-00c04fd430c8',
  
  // 특성 UUID
  ATTENDANCE_CHARACTERISTIC_UUID: '6ba7b811-9dad-11d1-80b4-00c04fd430c8',
  STATUS_CHARACTERISTIC_UUID: '6ba7b812-9dad-11d1-80b4-00c04fd430c8',
  CONFIG_CHARACTERISTIC_UUID: '6ba7b813-9dad-11d1-80b4-00c04fd430c8',
  
  // 스캔 설정
  SCAN_SETTINGS: {
    PROXIMITY_THRESHOLD: 3.0, // 3미터
    SCAN_INTERVAL: 5000, // 5초
    CONNECTION_TIMEOUT: 10000, // 10초
    RETRY_ATTEMPTS: 3
  },
  
  // 전송 전력 레벨
  TX_POWER_LEVELS: {
    ULTRA_LOW: -40,  // 실내용 (1-2m)
    LOW: -20,        // 근거리용 (2-5m)
    MEDIUM: -10,     // 중거리용 (5-10m)
    HIGH: 0          // 장거리용 (10-20m)
  }
};

export interface AttendanceData {
  participantId: string;
  eventId: string;
  timestamp: string;
  action: 'check_in' | 'check_out';
  gateId?: string;
  metadata?: Record<string, any>;
}
```

#### 근접 감지 및 처리

```typescript
// common/ble/ProximityHandler.ts
class ProximityHandler {
  private proximityDevices: Map<string, ProximityDevice> = new Map();
  private detectionCallbacks: ProximityCallback[] = [];
  
  registerProximityCallback(callback: ProximityCallback): void {
    this.detectionCallbacks.push(callback);
  }
  
  async handleProximityDevice(device: BLEDevice, distance: number): Promise<void> {
    const deviceId = device.id;
    const currentTime = Date.now();
    
    // 기존 감지 기록 확인
    const existingDevice = this.proximityDevices.get(deviceId);
    
    if (existingDevice) {
      // 너무 짧은 간격의 중복 감지 방지 (디바운싱)
      if (currentTime - existingDevice.lastDetected < 5000) {
        return;
      }
      
      // 거리 변화 추적
      existingDevice.distanceHistory.push({
        distance,
        timestamp: currentTime,
        rssi: device.rssi
      });
      
      // 히스토리 크기 제한
      if (existingDevice.distanceHistory.length > 10) {
        existingDevice.distanceHistory.shift();
      }
      
      existingDevice.lastDetected = currentTime;
      existingDevice.currentDistance = distance;
    } else {
      // 새로운 기기 감지
      const proximityDevice: ProximityDevice = {
        deviceId,
        name: device.name,
        firstDetected: currentTime,
        lastDetected: currentTime,
        currentDistance: distance,
        distanceHistory: [{
          distance,
          timestamp: currentTime,
          rssi: device.rssi
        }],
        stableProximity: false
      };
      
      this.proximityDevices.set(deviceId, proximityDevice);
    }
    
    // 안정적인 근접 상태 판단
    const isStableProximity = this.isStableProximity(deviceId);
    
    if (isStableProximity && !existingDevice?.stableProximity) {
      // 안정적인 근접 상태로 전환
      this.proximityDevices.get(deviceId)!.stableProximity = true;
      
      // 콜백 실행
      for (const callback of this.detectionCallbacks) {
        await callback({
          device,
          distance,
          eventType: 'proximity_stable',
          metadata: this.getProximityMetadata(deviceId)
        });
      }
    }
  }
  
  private isStableProximity(deviceId: string): boolean {
    const device = this.proximityDevices.get(deviceId);
    if (!device || device.distanceHistory.length < 3) return false;
    
    // 최근 3개 측정값이 모두 임계값 내에 있는지 확인
    const recentMeasurements = device.distanceHistory.slice(-3);
    const avgDistance = recentMeasurements.reduce((sum, m) => sum + m.distance, 0) / recentMeasurements.length;
    const variance = recentMeasurements.reduce((sum, m) => sum + Math.pow(m.distance - avgDistance, 2), 0) / recentMeasurements.length;
    
    // 평균 거리가 임계값 내이고 분산이 낮으면 안정적
    return avgDistance <= BLE_CONFIG.SCAN_SETTINGS.PROXIMITY_THRESHOLD && variance < 0.5;
  }
  
  private getProximityMetadata(deviceId: string): ProximityMetadata {
    const device = this.proximityDevices.get(deviceId);
    if (!device) return {};
    
    return {
      firstDetected: device.firstDetected,
      proximityDuration: Date.now() - device.firstDetected,
      averageDistance: device.distanceHistory.reduce((sum, h) => sum + h.distance, 0) / device.distanceHistory.length,
      measurementCount: device.distanceHistory.length
    };
  }
}
```

---

## 🌐 API 클라이언트 패턴

### 모든 서비스에서 공통 사용되는 HTTP 클라이언트

#### 기본 API 클라이언트

```typescript
// common/api/APIClient.ts
class APIClient {
  private baseURL: string;
  private authToken: string | null = null;
  private retryConfig: RetryConfig;
  
  constructor(config: APIClientConfig) {
    this.baseURL = config.baseURL;
    this.retryConfig = config.retryConfig || {
      maxRetries: 3,
      initialDelay: 1000,
      maxDelay: 10000,
      backoffFactor: 2
    };
  }
  
  setAuthToken(token: string): void {
    this.authToken = token;
  }
  
  private async request<T>(options: RequestOptions): Promise<T> {
    const { method, path, body, headers = {}, retries = 0 } = options;
    
    const url = `${this.baseURL}${path}`;
    
    const requestConfig: RequestInit = {
      method,
      headers: {
        'Content-Type': 'application/json',
        ...headers,
        ...(this.authToken ? { Authorization: `Bearer ${this.authToken}` } : {})
      },
      body: body ? JSON.stringify(body) : undefined
    };
    
    try {
      const response = await fetch(url, requestConfig);
      
      if (!response.ok) {
        const error = await this.handleErrorResponse(response);
        
        // 재시도 가능한 에러인지 확인
        if (this.shouldRetry(error) && retries < this.retryConfig.maxRetries) {
          await this.delay(this.calculateDelay(retries));
          return this.request({ ...options, retries: retries + 1 });
        }
        
        throw error;
      }
      
      return await response.json();
    } catch (error) {
      if (error instanceof TypeError && retries < this.retryConfig.maxRetries) {
        // 네트워크 에러 재시도
        await this.delay(this.calculateDelay(retries));
        return this.request({ ...options, retries: retries + 1 });
      }
      
      throw error;
    }
  }
  
  private async handleErrorResponse(response: Response): Promise<APIError> {
    let errorData: any = {};
    
    try {
      errorData = await response.json();
    } catch {
      // JSON 파싱 실패 시 기본 메시지 사용
    }
    
    return new APIError(
      response.status,
      errorData.message || response.statusText,
      errorData
    );
  }
  
  private shouldRetry(error: APIError): boolean {
    // 5xx 서버 에러만 재시도
    return error.status >= 500;
  }
  
  private calculateDelay(attempt: number): number {
    // 지수 백오프 with 지터
    const baseDelay = Math.pow(2, attempt) * 1000;
    const jitter = Math.random() * 1000;
    return Math.min(baseDelay + jitter, this.retryConfig.maxDelay);
  }
  
  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
  
  async get<T>(path: string, options?: Partial<RequestOptions>): Promise<T> {
    return this.request({ ...options, method: 'GET', path });
  }
  
  async post<T>(path: string, body: any, options?: Partial<RequestOptions>): Promise<T> {
    return this.request({ ...options, method: 'POST', path, body });
  }
  
  async put<T>(path: string, body: any, options?: Partial<RequestOptions>): Promise<T> {
    return this.request({ ...options, method: 'PUT', path, body });
  }
  
  async delete<T>(path: string, options?: Partial<RequestOptions>): Promise<T> {
    return this.request({ ...options, method: 'DELETE', path });
  }
}

class APIError extends Error {
  constructor(
    public status: number,
    message: string,
    public data?: any
  ) {
    super(message);
    this.name = 'APIError';
  }
}
```

#### 실시간 통신 (WebSocket)

```typescript
// common/api/WebSocketClient.ts
class WebSocketClient {
  private ws: WebSocket | null = null;
  private reconnectInterval: number = 5000;
  private maxReconnectAttempts: number = 10;
  private reconnectAttempts: number = 0;
  private eventHandlers: Map<string, EventHandler[]> = new Map();
  
  constructor(private url: string) {}
  
  connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(this.url);
      
      this.ws.onopen = () => {
        console.log('WebSocket connected');
        this.reconnectAttempts = 0;
        resolve();
      };
      
      this.ws.onmessage = (event) => {
        this.handleMessage(event.data);
      };
      
      this.ws.onclose = () => {
        console.log('WebSocket disconnected');
        this.attemptReconnect();
      };
      
      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        reject(error);
      };
    });
  }
  
  private handleMessage(data: string): void {
    try {
      const message = JSON.parse(data);
      const handlers = this.eventHandlers.get(message.type) || [];
      
      handlers.forEach(handler => {
        try {
          handler(message.payload);
        } catch (error) {
          console.error('Event handler error:', error);
        }
      });
    } catch (error) {
      console.error('Message parsing error:', error);
    }
  }
  
  private attemptReconnect(): void {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnection attempts reached');
      return;
    }
    
    this.reconnectAttempts++;
    
    setTimeout(() => {
      console.log(`Attempting to reconnect... (${this.reconnectAttempts}/${this.maxReconnectAttempts})`);
      this.connect().catch(console.error);
    }, this.reconnectInterval);
  }
  
  on(eventType: string, handler: EventHandler): void {
    if (!this.eventHandlers.has(eventType)) {
      this.eventHandlers.set(eventType, []);
    }
    
    this.eventHandlers.get(eventType)!.push(handler);
  }
  
  off(eventType: string, handler: EventHandler): void {
    const handlers = this.eventHandlers.get(eventType);
    if (handlers) {
      const index = handlers.indexOf(handler);
      if (index !== -1) {
        handlers.splice(index, 1);
      }
    }
  }
  
  send(eventType: string, payload: any): void {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify({
        type: eventType,
        payload,
        timestamp: new Date().toISOString()
      }));
    } else {
      console.warn('WebSocket not connected. Message not sent:', eventType);
    }
  }
  
  disconnect(): void {
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }
}
```

---

## 🔗 관련 시나리오

### 📄 Common 패턴 연계
- **[🔐 technical-patterns-security-auth.md](./technical-patterns-security-auth.md)**: 보안 및 인증 패턴
- **[📊 technical-patterns-data-sync.md](./technical-patterns-data-sync.md)**: 데이터 동기화 및 분석 패턴

### 🌐 시스템 적용 영역
- **User App**: BLE 근접 감지 및 통신
- **Gate Management**: BLE 스캐닝 및 데이터 수집
- **Event Management**: API 클라이언트 및 실시간 통신
- **Integrated Platform**: 통합 통신 및 메시징 허브

---

## 📊 성능 지표

| 영역 | 목표 지표 | 측정 방법 |
|------|-----------|-----------|
| **BLE 연결** | 95% 성공률 | 연결 시도 대비 성공률 |
| **근접 감지** | 3초 이내 감지 | 임계 거리 진입부터 감지까지 |
| **API 응답** | 평균 200ms | 요청부터 응답까지 |
| **WebSocket** | 99% 연결 유지 | 연결 안정성 모니터링 |
