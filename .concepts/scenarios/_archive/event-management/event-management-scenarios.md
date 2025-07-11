# Event Management Service 시나리오

## 🎯 서비스 개요

**Event Management Service**는 단일 행사를 관리하는 독립 서비스입니다. 각 행사마다 별도로 배포되며, 해당 행사의 참가자 데이터 관리, 출석 추적, 실시간 모니터링을 담당합니다.

### 핵심 특징
- **단일 행사 집중**: 하나의 행사만을 위한 전용 서비스
- **독립 배포**: 행사별로 별도 서버에 배포 가능
- **API 중심**: 다른 서비스와 API로 연동
- **실시간 처리**: 출석 데이터 실시간 수집 및 분석

---

## 📊 데이터 관리 시나리오

### 시나리오 1: 참가자 데이터 업로드 (CSV/API)
**목표**: 행사 참가자 명단을 시스템에 등록

```
1. 관리자가 참가자 CSV 파일 준비
   - 필수 필드: 이름, 이메일, 소속, 참가권한
   - 선택 필드: 전화번호, 특이사항, 그룹

2. CSV 업로드 또는 API 호출
   POST /api/participants/bulk-upload
   Content-Type: multipart/form-data
   {
     "file": "participants.csv",
     "eventId": "tech-conference-2024"
   }

3. 데이터 검증 및 처리
   - 중복 이메일 체크
   - 필수 필드 누락 검증
   - 데이터 형식 검증

4. 참가자 토큰 생성
   - 각 참가자별 고유 토큰 발급
   - QR 코드 생성
   - 이메일 발송 준비

결과: 참가자 데이터베이스 구축 완료
```

### 시나리오 2: 외부 시스템 연동 데이터 동기화
**목표**: 등록 시스템이나 CRM과 실시간 동기화

```
1. Webhook 설정
   POST /api/webhooks/register
   {
     "eventSource": "eventbrite",
     "endpoint": "https://our-event-service.com/webhook/participants",
     "events": ["participant.registered", "participant.updated"]
   }

2. 실시간 데이터 수신
   외부 시스템 → Event Management Service
   {
     "type": "participant.registered",
     "participant": {
       "email": "john@example.com",
       "name": "John Doe",
       "registrationTime": "2024-01-15T10:00:00Z"
     }
   }

3. 데이터 정규화 및 저장
   - 내부 데이터 형식으로 변환
   - 토큰 생성 및 QR 코드 준비
   - User App에 참가 권한 추가

결과: 외부 등록과 실시간 동기화 완료
```

---

## 🎫 토큰 및 인증 시나리오

### 시나리오 3: 참가자 토큰 생성 및 배포
**목표**: 참가자가 앱에서 사용할 인증 토큰 발급

```
1. 대량 토큰 생성
   POST /api/tokens/generate-batch
   {
     "eventId": "tech-conference-2024",
     "participants": ["all"] // 또는 특정 참가자 목록
   }

2. 토큰 구조 생성
   {
     "participantToken": "TCF24-ABCD-1234-EFGH",
     "qrCode": "data:image/png;base64,iVBORw0KGgoAAAA...",
     "eventInfo": {
       "eventId": "tech-conference-2024",
       "eventName": "Tech Conference 2024",
       "serverEndpoint": "https://tc24.events.com/api"
     },
     "participantInfo": {
       "name": "John Doe",
       "email": "john@example.com",
       "permissions": ["attendance", "session-access"]
     }
   }

3. 배포 채널 선택
   a) 이메일 자동 발송
      - QR 코드 첨부
      - 앱 다운로드 링크
      - 행사 정보 포함
   
   b) API 제공 (User App 직접 연동)
      GET /api/tokens/{email}
      → User App이 직접 토큰 조회

결과: 참가자별 고유 토큰 및 QR 코드 준비 완료
```

### 시나리오 4: 토큰 검증 및 갱신
**목표**: 유효한 토큰만 출석 체크 허용

```
1. 토큰 유효성 검증 (Gate Management 요청)
   POST /api/tokens/verify
   {
     "token": "TCF24-ABCD-1234-EFGH",
     "gateId": "main-entrance",
     "timestamp": "2024-01-20T09:30:00Z"
   }

2. 검증 프로세스
   - 토큰 형식 검사
   - 만료 시간 확인
   - 행사 매칭 확인
   - 중복 출석 체크

3. 응답 처리
   성공 시:
   {
     "valid": true,
     "participant": {
       "name": "John Doe",
       "id": "P001",
       "firstAttendance": false
     },
     "permissions": ["session-a", "lunch"]
   }
   
   실패 시:
   {
     "valid": false,
     "reason": "token_expired",
     "message": "토큰이 만료되었습니다"
   }

결과: 안전한 출석 체크 인증 완료
```

---

## 📈 실시간 모니터링 시나리오

### 시나리오 5: 실시간 출석 현황 추적
**목표**: 행사 진행 중 실시간 출석 데이터 수집

```
1. 출석 데이터 수신 (Gate Management → Event Management)
   POST /api/attendance/checkin
   {
     "participantToken": "TCF24-ABCD-1234-EFGH",
     "gateId": "main-entrance",
     "timestamp": "2024-01-20T09:30:00Z",
     "method": "qr_scan",
     "deviceId": "tablet-001"
   }

2. 실시간 처리
   - 출석 기록 저장
   - 중복 체크 방지
   - 실시간 통계 업데이트
   - 알림 조건 확인

3. 실시간 대시보드 데이터 제공
   GET /api/analytics/realtime
   {
     "totalRegistered": 500,
     "currentAttendees": 234,
     "attendanceRate": 46.8,
     "lastUpdate": "2024-01-20T09:31:00Z",
     "gateActivity": [
       {
         "gateId": "main-entrance",
         "count": 180,
         "rate": "3.2/min"
       }
     ]
   }

결과: 실시간 출석 현황 파악 가능
```

### 시나리오 6: 출석 패턴 분석
**목표**: 행사 중 출석 패턴을 분석하여 운영 최적화

```
1. 시간대별 출석 분석
   GET /api/analytics/attendance-pattern
   Query: timeRange=hourly&date=2024-01-20
   
   응답:
   {
     "patterns": [
       {
         "hour": "09:00-10:00",
         "checkins": 89,
         "peak": true,
         "avgWaitTime": "2.3min"
       },
       {
         "hour": "10:00-11:00", 
         "checkins": 45,
         "peak": false,
         "avgWaitTime": "0.8min"
       }
     ]
   }

2. 게이트별 부하 분석
   - 각 게이트의 처리량 비교
   - 대기 시간 예측
   - 게이트 증설 권고

3. 예측 알고리즘 실행
   - 다음 1시간 예상 출석자 수
   - 필요 게이트 수 권고
   - 스태프 배치 최적화

결과: 데이터 기반 운영 최적화 달성
```

---

## 🔄 외부 서비스 연동 시나리오

### 시나리오 7: User App 연동
**목표**: User App이 다중 행사 중 이 행사에 참여

```
1. User App에서 행사 추가 요청
   POST /api/user-integration/add-event
   {
     "userToken": "USER-GLOBAL-TOKEN",
     "eventToken": "TCF24-ABCD-1234-EFGH"
   }

2. 토큰 교환 및 권한 확인
   - Event token 유효성 검증
   - User의 참가 권한 확인
   - 행사 정보 패키지 생성

3. User App으로 행사 데이터 전송
   {
     "eventId": "tech-conference-2024",
     "eventName": "Tech Conference 2024",
     "schedule": [...],
     "apiEndpoint": "https://tc24.events.com/api",
     "participantInfo": {
       "name": "John Doe",
       "permissions": ["attendance", "session-access"]
     }
   }

결과: User App에 행사 추가 완료
```

### 시나리오 8: Integrated Platform 데이터 제공
**목표**: 통합 플랫폼에 행사 데이터 전송

```
1. 정기 데이터 동기화
   매시간 정각에 실행:
   POST https://integrated-platform.com/api/events/sync
   {
     "eventId": "tech-conference-2024",
     "syncType": "incremental",
     "data": {
       "attendanceUpdates": [...],
       "newParticipants": [...],
       "eventStatus": "ongoing"
     }
   }

2. 실시간 중요 이벤트 전송
   출석률 임계점 도달 시:
   POST https://integrated-platform.com/api/events/alert
   {
     "eventId": "tech-conference-2024",
     "alertType": "attendance_milestone",
     "data": {
       "currentRate": 75.0,
       "milestone": "75_percent"
     }
   }

결과: 통합 플랫폼에서 다중 행사 분석 가능
```

---

## 💾 오프라인 및 동기화 시나리오

### 시나리오 9: 오프라인 모드 지원
**목표**: 네트워크 연결이 불안정한 환경에서도 서비스 지속

```
1. 오프라인 데이터 패키지 생성
   GET /api/offline/package
   {
     "participants": [...], // 전체 참가자 목록
     "tokens": [...],       // 유효한 토큰 목록
     "configurations": {    // 게이트 설정
       "allowOfflineCheckin": true,
       "syncInterval": 300
     }
   }

2. Gate Management 오프라인 모드
   - 로컬 토큰 검증
   - 출석 데이터 로컬 저장
   - 주기적 연결 시도

3. 재연결 시 동기화
   POST /api/sync/offline-data
   {
     "deviceId": "tablet-001",
     "offlineRecords": [
       {
         "token": "TCF24-ABCD-1234-EFGH",
         "timestamp": "2024-01-20T10:15:00Z",
         "gateId": "main-entrance"
       }
     ]
   }

결과: 네트워크 장애에도 불구하고 출석 체크 지속
```

---

## 🔧 시스템 관리 시나리오

### 시나리오 10: 행사 종료 후 데이터 처리
**목표**: 행사 완료 후 최종 데이터 정리 및 아카이브

```
1. 최종 출석 통계 생성
   POST /api/admin/finalize-event
   {
     "eventId": "tech-conference-2024",
     "finalizeType": "complete"
   }

2. 데이터 아카이브 패키지 생성
   {
     "eventSummary": {
       "totalRegistered": 500,
       "totalAttended": 387,
       "attendanceRate": 77.4,
       "peakTime": "09:30-10:00"
     },
     "exportFiles": [
       "attendance-records.csv",
       "participant-list.csv", 
       "analytics-report.pdf"
     ]
   }

3. 정리 작업
   - 임시 데이터 삭제
   - 로그 아카이브
   - 토큰 무효화
   - 서비스 종료 준비

결과: 깔끔한 행사 마무리 및 데이터 보존
```

---

## 🎯 주요 특징 정리

### 독립성
- 각 행사마다 별도 Event Management Service 배포
- 다른 행사나 서비스와 완전 독립적 운영
- 서버 장애 시 다른 행사에 영향 없음

### 확장성
- 참가자 수에 따른 서버 스펙 조정 가능
- API 기반으로 다양한 외부 시스템과 연동
- 오프라인 모드로 네트워크 의존성 최소화

### 보안성
- 행사별 독립 토큰 시스템
- 데이터 격리로 보안 리스크 최소화
- API 키 기반 서비스 간 인증

이 Event Management Service는 단일 행사에 집중하여 최적의 성능과 안정성을 제공하며, 필요에 따라 다른 서비스들과 유연하게 연동할 수 있는 구조입니다.
