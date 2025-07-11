# Event Management - 출석 추적

## 📋 개요

Event Management 시스템의 실시간 출석 추적과 외부 시스템 연동을 다룹니다. Gate Management와 Integrated Platform으로부터 출석 데이터를 수신하고 실시간으로 처리하는 핵심 기능을 제공합니다.

## 🎯 주요 목표

- **실시간 출석 처리**: QR/BLE 스캔 데이터 즉시 처리
- **외부 시스템 연동**: Gate Management, Integrated Platform 연동
- **분석 및 모니터링**: 실시간 통계, 성능 지표 제공
- **시스템 안정성**: 고가용성, 장애 복구, 데이터 무결성

## 📊 성능 지표

| 영역 | 목표 | 현재 상태 | 상세 링크 |
|------|------|-----------|----------|
| 실시간 처리 | <100ms 응답시간 | 85ms | [실시간 처리](./attendance-tracking-realtime-processing.md) |
| 시스템 연동 | 99.9% 성공률 | 99.95% | [시스템 연동](./attendance-tracking-system-integration.md) |
| 분석 모니터링 | <5초 지연 | 3.2초 | [분석 모니터링](./attendance-tracking-analytics-monitoring.md) |

## 🏗️ 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                   출석 추적 시스템                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  실시간     │  │  시스템     │  │  분석       │         │
│  │  처리       │  │  연동       │  │  모니터링   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│       │               │               │                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │WebSocket/API│  │Gate/Platform│  │메트릭/알람  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

## 🔗 전문화된 파일 구조

### 1. 실시간 출석 처리 (428 lines)
**파일**: [attendance-tracking-realtime-processing.md](./attendance-tracking-realtime-processing.md)

**주요 내용**:
- WebSocket을 통한 실시간 데이터 수신
- 출석 데이터 검증 및 처리
- 실시간 통계 업데이트
- 중복 처리 방지 로직
- 성능 최적화 전략

### 2. 외부 시스템 연동 (332 lines)
**파일**: [attendance-tracking-system-integration.md](./attendance-tracking-system-integration.md)

**주요 내용**:
- Gate Management 시스템 연동
- Integrated Platform 연동
- API 클라이언트 구현
- 오류 처리 및 재시도 로직
- 데이터 동기화 전략

### 3. 분석 및 모니터링 (40 lines)
**파일**: [attendance-tracking-analytics-monitoring.md](./attendance-tracking-analytics-monitoring.md)

**주요 내용**:
- 실시간 메트릭 수집
- 성능 지표 모니터링
- 알람 및 알림 시스템
- 시스템 헬스 체크
- 분석 리포트 생성

## 🔍 파일 분할 이력

### 원본 파일 정보
- **파일명**: `attendance-tracking.md`
- **원본 크기**: 781 lines
- **분할 일시**: 2025-07-10
- **백업 파일**: `attendance-tracking-old.md`

### 분할 기준
1. **실시간 처리**: WebSocket, 데이터 처리, 성능 최적화 관련 시나리오
2. **시스템 연동**: 외부 시스템과의 API 연동 및 데이터 동기화
3. **분석 모니터링**: 메트릭 수집, 모니터링, 알람 관련 기능

### 크로스 레퍼런스
- 실시간 처리 ↔ 시스템 연동: 데이터 수신 및 처리 플로우
- 시스템 연동 ↔ 분석 모니터링: 연동 상태 모니터링
- 분석 모니터링 ↔ 실시간 처리: 성능 지표 수집

---

## 🔗 관련 시나리오

- **[참가자 관리](./participant-management.md)** - 참가자 데이터 및 토큰 관리
- **[Gate Management](../gate-management/attendance-processing.md)** - QR/BLE 스캔 처리
- **[User App](../user-app/attendance-app-scenarios.md)** - 모바일 앱 출석 처리
- **[Integrated Platform](../integrated-platform/data-integration-api.md)** - 데이터 통합 플랫폼
