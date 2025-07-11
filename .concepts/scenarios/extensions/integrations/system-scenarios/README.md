# 외부 시스템 연동 - 시스템 시나리오

## 개요

이 폴더는 외부 시스템 연동 확장 기능의 **기술적 구현과 시스템 수준의 시나리오**를 포함합니다. 이 확장 기능은 s-attend-gate 시스템을 CRM, 마케팅 도구, ERP, LMS 등의 외부 시스템과 통합하여 데이터 흐름을 원활하게 합니다.

## 주요 시나리오 문서

### 연동 아키텍처
- `integration-architecture.md` - 전체 통합 시스템 아키텍처
- `adapter-design.md` - 시스템별 어댑터 설계 패턴
- `api-gateway.md` - 중앙 API 게이트웨이 구현

### 데이터 통합
- `data-transformation.md` - 데이터 변환 및 매핑 엔진
- `sync-patterns.md` - 동기화 패턴 (실시간, 배치, 이벤트 기반)
- `data-validation.md` - 통합 데이터 검증 및 품질 관리

### 시스템별 연동
- `crm-integration.md` - CRM 시스템 연동 (Salesforce, HubSpot 등)
- `marketing-integration.md` - 마케팅 도구 연동 (Marketo, Mailchimp 등)
- `erp-integration.md` - ERP 시스템 연동 (SAP, Oracle 등)
- `lms-integration.md` - 학습 관리 시스템 연동 (Canvas, Moodle 등)

## 기술 스택

- **통합 플랫폼**: Apache Camel, Spring Integration
- **API 관리**: Kong, Apigee
- **데이터 변환**: Apache NiFi, Talend
- **메시징**: RabbitMQ, Apache Kafka
- **보안**: OAuth 2.0, JWT, API 키 관리

## 성능 요구사항

- **응답 시간**: API 호출 95% 200ms 이내 응답
- **처리량**: 초당 최대 100개 통합 요청 처리
- **동기화 지연**: 양방향 데이터 동기화 5분 이내
- **가용성**: 99.9% 이상의 통합 서비스 가용성

## 관련 문서

- [통합 아키텍처 다이어그램](../mermaid-diagrams.md)
- [관리자 통합 경험](../user-scenarios/admin-integration-experience.md)
- [데이터 거버넌스 정책](../../common/data-governance.md)
