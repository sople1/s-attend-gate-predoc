# Gate Management

S-Attend-Gate의 게이트 관리 시스템입니다.

## 기능

- BLE 비콘 제어
- 게이트 하드웨어 제어
- 실시간 상태 모니터링
- 오프라인 동작 지원

## 기술 스택

- Runtime: Python
- Hardware Control: RPi.GPIO
- BLE: bluez
- Database: SQLite (로컬 캐시)
- IPC: ZeroMQ

## 설치 및 실행

추후 업데이트 예정

## API 참조

- [게이트 제어 API](/.concepts/scenarios/system-scenarios/gate-management/core/control-system.md)
- [상태 모니터링 API](/.concepts/scenarios/system-scenarios/gate-management/operations/operational-monitoring.md)

## 관련 문서

- [게이트 관리 시나리오](/.concepts/scenarios/system-scenarios/gate-management/)
- [운영 시나리오](/.concepts/scenarios/user-scenarios/gate-management/)
