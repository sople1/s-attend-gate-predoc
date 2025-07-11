# Integrated Platform Service 개요

## 🎯 서비스 정의

**Integrated Platform Service**는 여러 Event Management Service들을 통합하여 관리하고 분석하는 플랫폼 서비스입니다. **다중 행사 통합 관리**를 핵심으로, 크로스 이벤트 분석, 통합 대시보드, API 허브 역할을 담당하며 조직의 행사 운영을 전략적으로 지원합니다.

## 📁 디렉토리 구조

```
integrated-platform/
├── ai/                # AI and ML features
│   ├── core/         # Core AI infrastructure
│   ├── ml/           # Machine learning models
│   └── services/     # AI service endpoints
├── analytics/         # Analytics and reporting
│   ├── core/         # Core analytics functionality
│   ├── bi/           # Business intelligence
│   └── reporting/    # Reporting features
├── core/             # Core platform features
│   └── integration/  # Integration components
├── disaster-recovery/ # DR and backup
│   ├── core/         # Core DR functionality
│   └── system/       # System recovery
├── monitoring/       # System monitoring
│   ├── alerts/      # Alert management
│   ├── dashboard/   # Monitoring dashboards
│   └── logging/     # Logging system
├── performance/      # Performance management
│   └── core/        # Core performance features
└── security/        # Security features
    └── core/        # Core security features
```

## 🔑 핵심 구성요소

### AI 및 분석 통합
- [AI 코어 인프라](ai/core/implementation.md) - AI 시스템 기반
- [ML 모델](ai/ml/models.md) - 기계학습 모델 정의
- [AI 서비스](ai/services/api.md) - AI 서비스 API
- [분석 코어](analytics/core/implementation.md) - 분석 시스템 기반
- [BI 기능](analytics/bi/implementation.md) - 비즈니스 인텔리전스

### 플랫폼 인프라
- [코어 플랫폼](core/implementation.md) - 기본 플랫폼 기능
- [통합 API](core/integration/api.md) - 시스템 통합
- [성능 관리](performance/core/implementation.md) - 성능 최적화

### 모니터링 및 보안
- [알림 시스템](monitoring/alerts/implementation.md) - 실시간 알림
- [로깅](monitoring/logging/implementation.md) - 시스템 로깅
- [보안](security/core/implementation.md) - 보안 기능
- [재해 복구](disaster-recovery/core/implementation.md) - DR 계획

## 🔄 통합 지점

### 데이터 흐름
1. 사용자 앱 → 코어 플랫폼 → 분석
2. 분석 → AI 처리 → 인사이트
3. 인사이트 → 모니터링 → 알림
4. 전체 구성요소 → 로깅 → 분석

### 주요 기능
- 실시간 데이터 처리
- 머신러닝 통합
- 성능 모니터링
- 보안 관리
- 재해 복구

## 📊 시각화 문서
자세한 아키텍처 및 흐름도는 [mermaid-diagrams.md](mermaid-diagrams.md)를 참조하세요.

## 🔗 관련 구성요소
- [이벤트 관리 시스템](../event-management/README.md)
- [게이트 관리 시스템](../gate-management/README.md)
- [사용자 애플리케이션](../user-app/README.md)
