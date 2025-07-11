# 출석 관련 사용자 행동 시나리오

## 시나리오 개요

사용자들이 실제 출석 과정에서 보이는 다양한 행동 패턴을 기록한 시나리오입니다.
기술적 구현 세부사항은 system-scenarios로 분리됩니다.

---

## 시나리오 1: 완벽한 자동 출석 행동

### 사용자 프로필
- Alex (25세, 개발자)
- 기술 친화적, 실험적 성향
- 앱 사용에 능숙함

### 행동 패턴
```
사용자 행동:
1. 행사장 주차장 도착 시 앱 상태 확인
2. 거리 정보 확인 후 입구로 이동
3. 진동 피드백 감지 시 즉시 앱 확인
4. 진행 상태 모니터링
5. 완료 확인 후 만족도 평가
6. 소셜 공유 진행

감정 변화:
- 시작: 호기심, 테스트 욕구
- 진행: 놀라움, 관찰자적 흥미
- 완료: 높은 만족도, 공유 욕구
```

---

## 시나리오 2: 백그라운드 지연 출석 행동

### 사용자 프로필
- Sarah (35세, 마케팅 매니저)
- 실용성 중시, 배터리 관리 주의
- 앱 사용에 보통 수준

### 행동 패턴
```
사용자 행동:
1. 늦어서 급하게 입구 통과
2. 알림 없음에 불안해하며 좌석 착석
3. 수동으로 앱 실행하여 상태 확인
4. 지연 처리 결과 확인 후 안도
5. 설정 메뉴에서 즉시 알림 활성화
6. 학습된 사용 패턴 적용

감정 변화:
- 시작: 급함, 걱정
- 중간: 불안, 의심
- 확인 후: 안도, 신뢰
- 설정 후: 통제감 회복
```

---

## 시나리오 3: 자동 감지 실패 대응 행동

### 사용자 프로필
- Robert (58세, 임원)
- 기술에 익숙하지 않음
- 인적 지원 선호

### 행동 패턴
```
사용자 행동:
1. 입구 통과 후 1분간 대기
2. 반응 없음에 당황하며 앱 실행
3. 오류 메시지 확인 후 해결책 탐색
4. 복잡한 옵션 회피하고 도움 요청 선택
5. 관리자 도착까지 대기
6. 관리자 지원으로 문제 해결
7. 추가 설정 도움 수락

감정 변화:
- 시작: 기대, 신뢰
- 실패 시: 당황, 좌절
- 도움 요청: 의존, 기대
- 해결 후: 감사, 만족
```

---

## 시나리오 4: QR 스캔 대체 행동

### 사용자 프로필
- Emma (22세, 대학생)
- 모바일 네이티브 세대
- 다양한 방법 적극 시도

### 행동 패턴
```
사용자 행동:
1. 자동 감지 실패 인지
2. 즉시 QR 스캔 모드로 전환
3. 입구 주변에서 QR코드 탐색
4. 스캔 완료 후 결과 확인
5. 자동 모드 재활성화
6. 다음 사용을 위한 설정 최적화

감정 변화:
- 실패 시: 약간의 당황
- 대안 발견: 문제 해결 의지
- 성공 시: 성취감, 자신감
```

---

## 시나리오 5: 네트워크 문제 상황 행동

### 사용자 프로필
- Michael (42세, 회계사)
- 중간 수준 기술 이해도
- 문제 해결 능력 보유

### 행동 패턴
```
사용자 행동:
1. 지하 행사장에서 네트워크 불안정 인지
2. 오프라인 모드 알림 확인
3. 오프라인 출석 처리 수락
4. 지상층 이동 시 자동 동기화 확인
5. 동기화 완료 알림 확인

감정 변화:
- 문제 인지: 우려, 걱정
- 대안 제시: 안심, 신뢰
- 자동 해결: 만족, 편리함
```

---

## 행동 분석 결과

### 성공적인 출석 행동의 공통점
1. **상황 인지**: 문제 발생 시 빠른 상황 파악
2. **대안 탐색**: 첫 번째 방법 실패 시 즉시 다른 방법 시도
3. **결과 확인**: 처리 완료 후 반드시 결과 확인
4. **학습 적용**: 경험을 바탕으로 다음 사용 시 개선

### 실패로 이어지는 행동 패턴
1. **포기**: 첫 번째 방법 실패 시 추가 시도 없음
2. **무관심**: 피드백 메시지 무시
3. **회피**: 도움 요청이나 대안 방법 기피
4. **반복**: 동일한 실수 지속

### 페르소나별 행동 특성

#### Alex (기술 친화적)
- 적극적 탐색과 실험
- 세부 정보 확인 선호
- 피드백과 공유 활발

#### Sarah (실용 중심)
- 빠른 문제 해결 추구
- 안정성과 확실성 중시
- 설정을 통한 통제 선호

#### Robert (보수적)
- 인적 지원 의존성 높음
- 단순한 해결책 선호
- 점진적 학습과 적응

#### Emma (적응력 높음)
- 다양한 방법 빠른 시도
- 모바일 UI에 직관적 반응
- 자기 주도적 문제 해결

### 설계 시사점
1. **다단계 백업**: 자동 → QR → 관리자 지원
2. **명확한 피드백**: 각 단계별 명확한 상태 표시
3. **개인화**: 사용자 성향에 맞는 기본 설정
4. **학습 지원**: 실패 후 올바른 사용법 가이드
