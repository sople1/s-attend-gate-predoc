# 참가자 매칭 알고리즘

이 문서는 네트워킹 게임의 핵심 기술 요소인 참가자 매칭 알고리즘의 기술적 구현을 설명합니다.

## 기술적 개요

매칭 알고리즘은 참가자 프로필, 행동 데이터, 미션 목표를 종합적으로 분석하여 의미 있는 네트워킹 기회를 제공합니다. 다차원 벡터 모델과 그래프 기반 추천 시스템을 결합한 하이브리드 접근 방식을 사용합니다.

## 구현 세부사항

### 프로필 벡터화

```python
def vectorize_profile(profile_data):
    """
    참가자 프로필을 다차원 벡터로 변환
    
    입력:
    - profile_data: 참가자 프로필 (관심사, 직무, 경험 등)
    
    출력:
    - 128차원 임베딩 벡터
    """
    # 텍스트 필드 처리 (관심사, 직무, 기술 등)
    text_fields = [
        profile_data.get("interests", ""),
        profile_data.get("job_title", ""),
        profile_data.get("skills", ""),
        profile_data.get("seeking", ""),
        profile_data.get("bio", "")
    ]
    
    # 텍스트 임베딩 (사전 훈련된 BERT 모델 사용)
    text_embedding = text_embedding_model.encode(" ".join(text_fields))
    
    # 범주형 데이터 처리 (산업, 역할, 경력 수준 등)
    categorical_embedding = encode_categorical_features([
        profile_data.get("industry", ""),
        profile_data.get("role_category", ""),
        profile_data.get("experience_level", "")
    ])
    
    # 수치 데이터 처리 (참가 행사 수, 경력 기간 등)
    numerical_embedding = normalize_numerical_features([
        profile_data.get("events_attended", 0),
        profile_data.get("years_experience", 0)
    ])
    
    # 벡터 결합
    return np.concatenate([
        text_embedding * WEIGHTS["text"],
        categorical_embedding * WEIGHTS["categorical"],
        numerical_embedding * WEIGHTS["numerical"]
    ])
```

### 유사도 계산

```python
def calculate_similarity(vector_a, vector_b, context=None):
    """
    두 참가자 벡터 간의 유사도 계산
    
    입력:
    - vector_a, vector_b: 참가자 프로필 벡터
    - context: 선택적 컨텍스트 정보 (특정 미션, 시간, 위치 등)
    
    출력:
    - 유사도 점수 (0-1)
    """
    # 기본 코사인 유사도
    base_similarity = cosine_similarity(vector_a, vector_b)
    
    if context is None:
        return base_similarity
    
    # 컨텍스트 기반 가중치 적용
    weights = get_context_weights(context)
    
    # 미션 관련 보정
    if "mission" in context:
        mission_boost = calculate_mission_alignment(
            vector_a, vector_b, context["mission"]
        )
        return base_similarity * weights["base"] + mission_boost * weights["mission"]
    
    # 행사 주제 관련 보정
    if "event_topics" in context:
        topic_boost = calculate_topic_alignment(
            vector_a, vector_b, context["event_topics"]
        )
        return base_similarity * weights["base"] + topic_boost * weights["topic"]
    
    return base_similarity
```

### 추천 알고리즘

```python
def generate_recommendations(user_id, count=5, context=None):
    """
    특정 사용자에게 추천할 연결 생성
    
    입력:
    - user_id: 대상 사용자 ID
    - count: 추천할 연결 수
    - context: 추천 컨텍스트 (현재 위치, 시간, 활성 미션 등)
    
    출력:
    - 추천 연결 목록 (사용자 ID와 유사도 점수)
    """
    # 사용자 프로필 벡터 조회
    user_vector = get_user_vector(user_id)
    
    # 컨텍스트 확장
    if context is None:
        context = {}
    
    # 현재 위치 추가 (가능한 경우)
    if "current_location" not in context and has_recent_location(user_id):
        context["current_location"] = get_recent_location(user_id)
    
    # 현재 활성 미션 추가
    if "active_missions" not in context:
        context["active_missions"] = get_active_missions(user_id)
    
    # 후보 풀 생성
    candidates = []
    
    # 1. 위치 기반 후보 (근처 사용자)
    if "current_location" in context:
        nearby_users = find_nearby_users(
            context["current_location"],
            radius=50,  # 미터
            exclude_ids=[user_id]
        )
        candidates.extend([
            (u_id, 0, "proximity") for u_id in nearby_users
        ])
    
    # 2. 미션 기반 후보
    if "active_missions" in context:
        for mission in context["active_missions"]:
            mission_candidates = find_mission_candidates(user_id, mission)
            candidates.extend([
                (u_id, 0, "mission") for u_id in mission_candidates
            ])
    
    # 3. 프로필 유사도 기반 후보
    similar_profiles = find_similar_profiles(
        user_vector,
        top_k=count * 3,
        exclude_ids=[user_id]
    )
    candidates.extend([
        (u_id, 0, "similarity") for u_id in similar_profiles
    ])
    
    # 후보별 최종 점수 계산
    scored_candidates = []
    for candidate_id, _, source in candidates:
        # 중복 제거
        if candidate_id in [c[0] for c in scored_candidates]:
            continue
            
        candidate_vector = get_user_vector(candidate_id)
        
        # 유사도 계산
        similarity = calculate_similarity(
            user_vector,
            candidate_vector,
            context
        )
        
        # 추가 요소 고려
        final_score = adjust_score(
            similarity,
            user_id=user_id,
            candidate_id=candidate_id,
            interaction_history=get_interaction_history(user_id, candidate_id),
            source=source
        )
        
        scored_candidates.append((candidate_id, final_score))
    
    # 최종 정렬 및 상위 결과 반환
    return sorted(scored_candidates, key=lambda x: x[1], reverse=True)[:count]
```

### 미션 생성 및 할당

미션은 사용자 프로필, 행사 맥락, 현재 네트워킹 상태를 고려하여 동적으로 생성됩니다:

- **개인 미션**: 특정 역할, 산업 또는 관심사를 가진 참가자 만나기
- **그룹 미션**: 특정 조건을 만족하는 3-5명의 그룹 형성하기
- **행사 미션**: 주최자가 사전 정의한 특별 미션 (VIP 만남, 스폰서 부스 방문 등)

미션은 다음 기준에 따라 참가자에게 할당됩니다:
1. **완료 가능성**: 현재 행사에 해당 미션을 완료할 수 있는 충분한 대상이 있는지
2. **개인 관련성**: 사용자의 프로필과 목표에 얼마나 부합하는지
3. **난이도 밸런싱**: 쉬운 미션과 도전적인 미션의 적절한 조합
4. **진행 상태**: 사용자의 현재 게임 진행 상태와 참여도

## 구성

### 성능 최적화

```yaml
matching_config:
  # 벡터 연산 최적화
  vector_dimension: 128
  index_type: "HNSW"  # Hierarchical Navigable Small World
  distance_metric: "cosine"
  
  # 캐싱 전략
  profile_cache_ttl_sec: 300  # 5분
  results_cache_ttl_sec: 60   # 1분
  
  # 추천 다양성
  diversity_factor: 0.3  # 유사도와 다양성 간의 균형
  
  # 계산 제한
  max_candidates_per_source: 100
  max_computation_time_ms: 200
```

### 개인정보 보호

- 모든 매칭 데이터는 익명화된 상태로 처리
- 사용자 명시적 동의 없이 연락처 정보 공유 없음
- 사용자별 매칭 제외 목록 지원
- 미팅 요청에 대한 거부 권한 보장

## 통합 지점

- **User App**: 프로필 정보 및 사용자 설정
- **Event Management**: 행사 컨텍스트 및 참가자 목록
- **분석 대시보드**: 네트워킹 패턴 및 효과 분석
- **외부 연동**: LinkedIn 등 선택적 프로필 강화

## 모니터링 및 지표

### 주요 성능 지표

- **응답 시간**: 매칭 요청에서 결과 제공까지 소요 시간 (목표: P95 < 200ms)
- **관련성 점수**: 추천된 매칭의 실제 상호작용 전환율
- **자원 사용**: 서버 부하 및 메모리 사용량
- **처리량**: 초당 처리 가능한 매칭 요청 수

### 매칭 품질 지표

- **상호 연결율**: 추천 후 실제 QR 교환으로 이어진 비율 (목표: > 30%)
- **만족도 점수**: 연결 후 사용자 평가 (목표: 평균 4/5 이상)
- **미션 완료율**: 할당된 미션 대비 완료된 미션 비율 (목표: > 60%)
- **장기 연결성**: 행사 후 1개월 내 재접촉률 (목표: > 15%)

## 관련 문서

- [프로필 관리 시스템](./profile-management.md)
- [미션 시스템](./mission-system.md)
- [네트워킹 경험 다이어그램](../mermaid-diagrams.md)
