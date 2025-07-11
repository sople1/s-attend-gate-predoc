---
title: "외부 시스템 통신 흐름 스토리보드"
flow-type: "system"
screens: 4
resolution: "1920x1080"
created: "2025-07-11"
updated: "2025-07-11"
---

# 외부 시스템 통신 흐름

## 개요

s-attend-gate 시스템의 각 구성 요소가 외부 시스템과 통신하는 흐름을 시각화합니다.

## User App 통신 흐름

```mermaid
sequenceDiagram
    participant UA as User App
    participant GPS as 위치 서비스
    participant BLE as 블루투스
    participant FCM as 푸시 알림
    participant Auth as 소셜 로그인

    UA->>GPS: 위치 정보 요청
    GPS-->>UA: 위치 데이터 반환
    UA->>BLE: 비콘 스캔
    BLE-->>UA: 비콘 신호 감지
    FCM->>UA: 푸시 알림 수신
    UA->>Auth: 소셜 로그인 요청
    Auth-->>UA: 인증 토큰 반환
```

### 핵심 통신 포인트
1. 위치 기반 서비스
   - GPS 위치 추적
   - 게이트 근접 감지
   - 위치 데이터 캐싱

2. BLE 통신
   - 비콘 신호 스캔
   - 신호 강도 측정
   - 범위 계산

3. 푸시 알림
   - 토큰 관리
   - 알림 수신/표시
   - 백그라운드 처리

## Event Management 통신 흐름

```mermaid
sequenceDiagram
    participant EM as Event Management
    participant PG as 결제 게이트웨이
    participant Mail as 이메일 서비스
    participant SMS as SMS 서비스
    participant CRM as CRM 시스템

    EM->>PG: 결제 요청
    PG-->>EM: 결제 승인/거절
    EM->>Mail: 이메일 발송
    Mail-->>EM: 발송 상태
    EM->>SMS: 알림 메시지 전송
    SMS-->>EM: 전송 상태
    EM->>CRM: 고객 데이터 동기화
    CRM-->>EM: 동기화 완료
```

### 핵심 통신 포인트
1. 결제 처리
   - 결제 요청/승인
   - 환불 처리
   - 정산 관리

2. 커뮤니케이션
   - 대량 메일 발송
   - SMS 알림
   - 상태 추적

3. CRM 연동
   - 고객 데이터 동기화
   - 이벤트 정보 공유
   - 마케팅 자동화

## Gate Management 통신 흐름

```mermaid
sequenceDiagram
    participant GM as Gate Management
    participant Beacon as BLE 비콘
    participant Gate as 물리적 게이트
    participant CCTV as CCTV 시스템
    participant Alert as 긴급 알림

    GM->>Beacon: 비콘 설정
    Beacon-->>GM: 상태 보고
    GM->>Gate: 게이트 제어
    Gate-->>GM: 동작 상태
    GM->>CCTV: 영상 스트림 요청
    CCTV-->>GM: 영상 데이터
    GM->>Alert: 긴급 상황 알림
    Alert-->>GM: 알림 확인
```

### 핵심 통신 포인트
1. 하드웨어 제어
   - 비콘 관리
   - 게이트 동작
   - 센서 모니터링

2. 보안 시스템
   - CCTV 연동
   - 접근 제어
   - 이상 감지

3. 긴급 대응
   - 알림 발송
   - 백업 시스템
   - 장애 복구

## Integrated Platform 통신 흐름

```mermaid
sequenceDiagram
    participant IP as Integrated Platform
    participant DW as 데이터 웨어하우스
    participant S3 as 백업 스토리지
    participant Mon as 모니터링
    participant API as API 게이트웨이

    IP->>DW: 데이터 적재
    DW-->>IP: 쿼리 결과
    IP->>S3: 백업 데이터 저장
    S3-->>IP: 저장 확인
    IP->>Mon: 메트릭 전송
    Mon-->>IP: 알림 발생
    IP->>API: API 요청 라우팅
    API-->>IP: 응답 데이터
```

### 핵심 통신 포인트
1. 데이터 처리
   - 데이터 수집
   - 분석 처리
   - 저장/백업

2. 모니터링
   - 성능 측정
   - 알림 관리
   - 로그 분석

3. API 관리
   - 요청 라우팅
   - 인증/인가
   - 요율 제한

## 데이터 흐름 고려사항

### 보안
- 모든 외부 통신 SSL/TLS 적용
- 민감 데이터 암호화
- 접근 토큰 관리

### 성능
- 비동기 처리
- 캐싱 전략
- 재시도 메커니즘

### 안정성
- 장애 복구
- 데이터 정합성
- 백업/복원
