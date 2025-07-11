# 보상 시스템

## 보상 유형

### 기본 보상
- 포인트 적립
- 경험치 획득
- 업적 달성
- 레벨 업

### 특별 보상
- 희귀 아이템
- 특별 권한
- 이벤트 참여권
- VIP 혜택

### 시즌 보상
- 시즌 랭킹 보상
- 시즌 한정 아이템
- 특별 타이틀
- 누적 보상

## 보상 정책

### 포인트 정책
```json
{
  "points": {
    "basic_find": 50,
    "combo_bonus": 10,
    "time_bonus": 5,
    "special_item": {
      "rare": 100,
      "epic": 300,
      "legendary": 500
    }
  }
}
```

### 레벨 시스템
- 레벨 1-10: 100포인트/레벨
- 레벨 11-20: 200포인트/레벨
- 레벨 21-30: 300포인트/레벨
- 레벨 31+: 500포인트/레벨

### 업적 시스템
- 일일 업적
- 주간 업적
- 월간 업적
- 특별 업적

## 보상 API
```json
{
  "claim_reward": {
    "method": "POST",
    "endpoint": "/api/v1/rewards/claim",
    "payload": {
      "player_id": "string",
      "reward_id": "string",
      "game_id": "string"
    }
  }
}
```

## 관련 문서
- [게임 로직](./game-logic.md)
- [리더보드](./leaderboard.md)
- [구현 세부사항](./implementation.md)
