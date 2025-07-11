# 리더보드 시스템

## 리더보드 유형

### 전체 순위
- 총점 기준
- 발견 수 기준
- 시간 기준
- 효율성 기준

### 시즌 순위
- 시즌별 총점
- 시즌 업적
- 시즌 특별 미션
- 시즌 참여도

### 그룹별 순위
- 난이도별
- 지역별
- 레벨별
- 팀별

## 순위 계산

### 점수 가중치
```json
{
  "weights": {
    "total_points": 0.4,
    "discoveries": 0.3,
    "time_efficiency": 0.2,
    "achievements": 0.1
  }
}
```

### 갱신 주기
- 실시간 순위: 즉시 갱신
- 일일 순위: 매일 00:00
- 주간 순위: 매주 월요일
- 시즌 순위: 시즌 종료시

### 보상 정책
- TOP 10: 특별 보상
- TOP 50: 고급 보상
- TOP 100: 일반 보상
- 참가상: 기본 보상

## 리더보드 API
```json
{
  "get_leaderboard": {
    "method": "GET",
    "endpoint": "/api/v1/leaderboard",
    "parameters": {
      "board_type": "string",
      "time_range": "string",
      "limit": "integer"
    }
  }
}
```

## 관련 문서
- [게임 로직](./game-logic.md)
- [보상 시스템](./rewards.md)
- [구현 세부사항](./implementation.md)
