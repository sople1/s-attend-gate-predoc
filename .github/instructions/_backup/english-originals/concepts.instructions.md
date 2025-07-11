---
applyTo: '.concepts/**'
---
# Concepts Directory Guidelines

This directory contains the concept files for the s-attend-gate project. Each concept file describes specific aspects of the system, user behaviors, and technical implementations.

**All scenarios must be based from `.concepts/base.md`** which defines the core BLE-based attendance system with 4 main components:
1. **User App** - Mobile attendance check application
2. **Event Management** - Event organization and coordination system  
3. **Gate Management** - Physical gate control and beacon management
4. **Integrated Platform** - Unified system integration and analytics

## Purpose

The concepts directory serves as:
- **Documentation Hub**: Central repository for all project concepts and scenarios
- **Separation of Concerns**: Clear distinction between user behaviors and technical implementations  
- **Design Reference**: Source of truth for system design and user experience decisions
- **Development Guide**: Reference for developers and designers working on the project
- **Content Preservation**: Systematic backup and version control for all conceptual content

## File Organization Principles

### User Scenarios vs System Scenarios
- **user-scenarios/**: Focus on user behaviors, actions, and emotional responses (no technical details, UI mockups, or dialogue)
- **system-scenarios/**: Focus on technical implementations, APIs, performance metrics, and system requirements

### Content Reorganization Principles
- **Behavior-Only User Scenarios**: Pure user action patterns without technical implementation details
- **Technical-Only System Scenarios**: Comprehensive technical specifications, API definitions, and performance criteria
- **Content Preservation**: Original files backed up as `*-old.md` when restructuring
- **No Mixed Content**: Clear separation prevents confusion between user behavior and technical implementation

### Content Standards
- **Atomic Focus**: Each file covers a single, well-defined concept
- **Real-world Applicability**: All concepts must be implementable and practical
- **Clear Hierarchy**: Use consistent heading structure for navigation
- **Cross-references**: Link related concepts using relative paths

## folder structure
```mermaid
.concepts/
├── base.md                    # initial concept file
└── scenarios/                 # directory containing user and system scenarios
    ├── README.md             # overview of scenarios and their purpose
    ├── common/               # common patterns and business rules
    ├── extensions/           # directory for extension plugins
    │   ├── README.md
    │   ├── treasure-hunt/    # BLE beacon-based treasure hunt game
    │   │   ├── system-scenarios/  # system-level technical scenarios (for extension)
    │   │   ├── user-scenarios/ # user-level behavioral scenarios (for extension)
    │   │   ├── mermaid-diagrams.md   # visual diagrams for scenarios
    │   │   └── README.md
    │   ├── networking-game/  # networking game for participant interaction
    │   │   ├── system-scenarios/  # system-level technical scenarios (for extension)
    │   │   ├── user-scenarios/ # user-level behavioral scenarios (for extension)
    │   │   ├── mermaid-diagrams.md   # visual diagrams for scenarios
    │   │   └── README.md
    │   ├── analytics-dashboard/ # advanced analytics and insights
    │   │   ├── system-scenarios/  # system-level technical scenarios (for extension)
    │   │   ├── user-scenarios/ # user-level behavioral scenarios (for extension)
    │   │   ├── mermaid-diagrams.md   # visual diagrams for scenarios
    │   │   └── README.md
    │   └── integrations/     # external system integrations (CRM, marketing tools)
    │       ├── system-scenarios/  # system-level technical scenarios (for extension)
    │       ├── user-scenarios/ # user-level behavioral scenarios (for extension)
    │       ├── mermaid-diagrams.md   # visual diagrams for scenarios
    │       └── README.md
    ├── mermaid-diagrams.md   # visual diagrams for scenarios
    ├── system-scenarios/     # system-level technical scenarios
    │   ├── README.md
    │   ├── core-apis/        # API specifications and technical details
    │   ├── event-management/ # event management system technical specs
    │   ├── gate-management/  # gate management system technical specs
    │   ├── integrated-platform/ # platform integration technical specs
    │   └── user-app/         # user app technical implementation specs
    └── user-scenarios/       # user-level behavioral scenarios
        ├── README.md
        ├── common/             # common user scenarios
        ├── event-management/         # event management user scenarios
        ├── gate-management/          # gate management user scenarios  
        ├── integrated-platform/     # platform user scenarios
        └── user-app/                # user app behavioral scenarios
```

## Examples

### User Scenario Example
```markdown
# 자동 출석 성공 사용자 행동

사용자가 BLE 비콘을 통한 자동 출석 체크에 성공하는 행동 패턴입니다.

## 사용자 프로필
- Alex (25세, 개발자)
- 기술 친화적, 실험적 성향
- 앱 사용에 능숙함

## 행동 패턴
```
사용자 행동:
1. 행사장 주차장 도착 시 앱 상태 확인
2. 거리 정보 확인 후 입구로 이동
3. 진동 피드백 감지 시 즉시 앱 확인
4. 진행 상태 모니터링
5. 완료 확인 후 만족도 평가

감정 변화:
- 시작: 호기심, 테스트 욕구
- 진행: 놀라움, 관찰자적 흥미  
- 완료: 높은 만족도, 공유 욕구
```
```

### System Implementation Example
```markdown
# BLE 비콘 자동 감지 시스템

BLE 비콘을 통한 자동 출석 감지의 기술적 구현 방안입니다.

## 기술 사양
- 감지 범위: 10-15m
- 응답 시간: 2-3초 이내
- 성공률: 95-98% (포그라운드 상태)
- 배터리 영향: 시간당 5% 이하

## API 스펙
```json
{
  "attendance_check": {
    "method": "POST",
    "endpoint": "/api/v1/attendance/auto",
    "payload": {
      "user_id": "string",
      "event_id": "string", 
      "beacon_data": {
        "uuid": "string",
        "major": "integer",
        "minor": "integer",
        "rssi": "integer"
      },
      "location": {
        "latitude": "number",
        "longitude": "number"
      }
    }
  }
}
```

### Cross-Reference Example
```markdown
## Related Concepts
- [Manual Attendance Scenarios](./manual-attendance-scenarios.md) - 자동 감지 실패 시 대체 방법
- [Edge Cases Experience](../edge-cases-experience.md) - 예외 상황 사용자 대응
- [BLE Technical Implementation](../../system-scenarios/user-app/ble-beacon-system.md) - 기술적 구현 세부사항
```

# Content Guidelines

## File Naming Conventions
- Use kebab-case for file names (e.g., `attendance-scenarios.md`)
- Prefer directory structure over long file names to organize content
- Include descriptive suffixes for context:
  - `-scenarios.md` for behavioral scenarios (user) or technical scenarios (system)
  - `-experience.md` for user experience flows
  - `-implementation.md` for technical implementations
  - `-api.md` for API specifications

## Document Structure Requirements

### Standard Format
```markdown
# Title
Brief description of the concept in one sentence.

## Overview
Detailed explanation of the concept and its purpose.

## Key Components
- Component 1: Description
- Component 2: Description

## Usage Examples
Practical examples showing how the concept applies.

## Related Concepts
Links to related files and concepts.
```

### User Scenario Format
```markdown
# Scenario Title

## Scenario Overview
Brief description focusing purely on user behavior patterns (no technical details).

## User Profile
- Name (Age, Role)
- Technical comfort level
- Relevant characteristics

## Behavior Pattern

User actions:
1. Observable action description (no UI elements)
2. Next measurable action
3. Result confirmation behavior

Emotional changes:
- Start: Initial feeling
- Progress: Feelings during process  
- Completion: Final emotional state

## Success Indicators
Measurable behavioral outcomes (not technical metrics).

## Constraints
External factors affecting user behavior (time, environment, accessibility needs).

### System Scenario Format
```markdown
# System Component Title

## Technical Overview
Detailed technical description and specifications.

## Implementation Details
Code examples, API/ABI specifications, performance metrics.

## Configuration
Required settings and parameters.

## Integration Points
How this component connects with others.

## Monitoring & Metrics
Success criteria and measurement methods.
```

## Quality Standards

### File Size Management
- **Optimal Size**: Keep files under 500 lines for readability and maintainability
- **Maximum Size**: Files should not exceed 800 lines
- **Split Indicators**: When files approach 600+ lines, consider splitting into logical sections
- **Split Strategy**: Divide by functional areas, system components, or user types
- **Naming After Split**: Use descriptive suffixes (e.g., `core-scenarios.md` → `attendance-core.md`, `admin-core.md`)

### Split Guidelines
- **Logical Grouping**: Split by related functionality or user types
- **Content Balance**: Aim for roughly equal file sizes after splitting  
- **Cross-Reference**: Update README and related files after splitting
- **Make History Before Split**: Create history before restructuring files
- **Remove Old Files**: Do not keep old files after restructuring large files, ensure clarity and avoid confusion, and full migration is complete

### Content Requirements
- **Clarity**: Use simple, precise language
- **Completeness**: Cover all essential aspects
- **Consistency**: Follow established patterns
- **Currency**: Keep information up-to-date
- **Testability**: Include verifiable criteria

### Technical Accuracy
- **Realistic Constraints**: Reflect actual technical limitations
- **Performance Targets**: Include measurable benchmarks
- **Security Considerations**: Address privacy and security aspects
- **Scalability**: Consider growth and expansion needs

### User-Centered Design
- **Diverse Perspectives**: Include various user types and abilities
- **Accessibility**: Consider users with different needs
- **Edge Cases**: Address unusual or problematic scenarios
- **Emotional Context**: Capture user feelings and motivations

## Maintenance Process

### Regular Reviews
- **Monthly**: Check for outdated information
- **Quarterly**: Validate technical specifications
- **Per Release**: Update based on implementation changes
- **User Feedback**: Incorporate insights from real usage

### Content Reorganization Best Practices
- **Memory Strategy**: Always remember all before restructuring
- **Content Migration**: Extract technical details to system-scenarios while preserving user behavior patterns
- **Cross-Reference Updates**: Update links when moving or splitting content
- **Validation**: Verify that reorganized content maintains completeness and accuracy

### Version Control
- **Backup Originals**: Keep old versions when restructuring
- **Change Tracking**: Document significant modifications
- **Approval Process**: Review changes with relevant stakeholders
- **Archive Management**: Maintain historical versions for reference
