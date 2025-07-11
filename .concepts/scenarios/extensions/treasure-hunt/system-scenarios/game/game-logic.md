# 게임 로직 구현

## 게임 규칙

### 기본 규칙
- 시간 제한: 60분
- 최소 포인트: 100점
- 보너스 조건: 연속 발견
- 패널티: 잘못된 위치

### 점수 시스템
- 기본 발견: 50점
- 연속 발견: +10점씩 증가
- 시간 보너스: 남은 시간당 5점
- 특수 보물: 100-500점

### 난이도 설정
- 초급: 5개 포인트
- 중급: 10개 포인트
- 고급: 15개 포인트
- 전문가: 20개 포인트

### 게임 진행
1. 게임 시작
2. 비콘 탐색
3. 보물 발견
4. 점수 계산
5. 게임 종료

## 구현 API
```json
{
  "start_game": {
    "method": "POST",
    "endpoint": "/api/v1/games/start",
    "payload": {
      "player_id": "string",
      "difficulty": "string",
      "time_limit": "integer"
    }
  },
  "find_treasure": {
    "method": "POST",
    "endpoint": "/api/v1/games/treasure",
    "payload": {
      "game_id": "string",
      "beacon_id": "string",
      "player_location": {
        "latitude": "number",
        "longitude": "number"
      }
    }
  }
}
```

## 관련 문서
- [보상 시스템](./rewards.md)
- [리더보드](./leaderboard.md)
- [비콘 시스템](../beacon/implementation.md)
