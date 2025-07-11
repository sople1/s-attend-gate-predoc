# 보물찾기 - 시스템 시나리오

## 개요

이 폴더는 보물찾기 확장 기능의 **기술적 구현과 시스템 수준의 시나리오**를 포함합니다. BLE 비콘 기반 보물찾기 게임은 행사 참가자들에게 행사장 탐색을 장려하고 참여도를 높이는 게임화 요소를 제공합니다.

## 주요 시나리오 문서

### 비콘 시스템
- `beacon-configuration.md` - BLE 비콘 설정 및 배포
- `beacon-detection.md` - 비콘 감지 및 식별 알고리즘
- `location-tracking.md` - 실내 위치 추적 시스템

### 게임 시스템
- `quest-generation.md` - 동적 퀘스트 생성 시스템
- `rewards-system.md` - 보상 및 업적 시스템
- `progress-tracking.md` - 게임 진행 추적 메커니즘

### 관리 도구
- `admin-dashboard.md` - 운영자 게임 관리 도구
- `analytics-integration.md` - 참여 데이터 수집 및 분석
- `content-management.md` - 게임 콘텐츠 관리 시스템

## 기술 스택

- **비콘 하드웨어**: Eddystone/iBeacon 호환 BLE 비콘
- **모바일 감지**: Core Bluetooth (iOS), Bluetooth LE API (Android)
- **백엔드**: Node.js, Express, WebSocket
- **데이터베이스**: MongoDB (게임 상태), Redis (실시간 데이터)
- **관리 도구**: React 기반 웹 대시보드

## 성능 요구사항

- **비콘 감지 속도**: 90% 이상의 비콘 5초 내 감지
- **배터리 효율성**: 사용자 기기 배터리 소모 1시간당 최대 2%
- **동시 사용자**: 최대 1,000명의 동시 게임 참가자
- **게임 상태 동기화**: 5초 이내 모든 기기 동기화

## 관련 문서

- [보물찾기 게임 흐름 다이어그램](../mermaid-diagrams.md)
- [사용자 게임 경험](../user-scenarios/treasure-hunt-experience.md)
- [비콘 배치 전략](./beacon-configuration.md)
