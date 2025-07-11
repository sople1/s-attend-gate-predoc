# CRM 시스템 연동 구현

이 문서는 s-attend-gate 시스템과 주요 CRM 플랫폼 간의 통합을 위한 기술적 구현을 설명합니다.

## 기술적 개요

CRM 통합 모듈은 Salesforce, HubSpot, Microsoft Dynamics 등 주요 CRM 시스템과의 양방향 데이터 동기화를 지원합니다. 통합은 REST API, 웹훅, 배치 프로세싱 등 다양한 방법을 통해 이루어지며, 실시간 및 정기적 동기화를 모두 지원합니다.

## 구현 세부사항

### 지원하는 CRM 시스템

```json
{
  "supported_crm_systems": [
    {
      "name": "Salesforce",
      "api_versions": ["v52.0", "v53.0", "v54.0"],
      "auth_methods": ["OAuth 2.0", "JWT"],
      "integration_methods": ["REST API", "Bulk API", "Streaming API"],
      "sync_capabilities": ["Real-time", "Scheduled", "Triggered"],
      "entity_mappings": [
        {"s_attend": "Participant", "crm": "Contact"},
        {"s_attend": "Event", "crm": "Campaign"},
        {"s_attend": "Session", "crm": "Campaign Member"}
      ]
    },
    {
      "name": "HubSpot",
      "api_versions": ["v3"],
      "auth_methods": ["API Key", "OAuth 2.0"],
      "integration_methods": ["REST API", "Webhooks"],
      "sync_capabilities": ["Real-time", "Scheduled"],
      "entity_mappings": [
        {"s_attend": "Participant", "crm": "Contact"},
        {"s_attend": "Event", "crm": "Deal"},
        {"s_attend": "Attendance", "crm": "Engagement"}
      ]
    },
    {
      "name": "Microsoft Dynamics 365",
      "api_versions": ["v9.1", "v9.2"],
      "auth_methods": ["OAuth 2.0", "Client Credentials"],
      "integration_methods": ["Web API", "Batch Operations"],
      "sync_capabilities": ["Scheduled", "Triggered"],
      "entity_mappings": [
        {"s_attend": "Participant", "crm": "Contact"},
        {"s_attend": "Event", "crm": "Campaign"},
        {"s_attend": "Attendance", "crm": "Campaign Response"}
      ]
    },
    {
      "name": "Zoho CRM",
      "api_versions": ["v2", "v2.1"],
      "auth_methods": ["OAuth 2.0", "API Key"],
      "integration_methods": ["REST API", "Webhooks"],
      "sync_capabilities": ["Real-time", "Scheduled"],
      "entity_mappings": [
        {"s_attend": "Participant", "crm": "Contact"},
        {"s_attend": "Event", "crm": "Campaign"},
        {"s_attend": "Session", "crm": "Event"}
      ]
    }
  ]
}
```

### 데이터 매핑 엔진

```typescript
class CrmDataMapper {
  private mappingConfig: MappingConfig;
  private transformationRules: TransformationRule[];
  
  constructor(crmType: string, mappingConfig: MappingConfig) {
    this.mappingConfig = mappingConfig;
    this.transformationRules = this.loadTransformationRules(crmType);
  }
  
  /**
   * s-attend-gate 엔티티를 CRM 엔티티로 변환
   */
  mapToCrm(sourceEntity: string, data: any): any {
    // 매핑 구성 검색
    const entityMapping = this.findEntityMapping(sourceEntity);
    if (!entityMapping) {
      throw new Error(`No mapping found for entity: ${sourceEntity}`);
    }
    
    // 결과 객체 초기화
    const result: any = {
      entity_type: entityMapping.target_entity,
      fields: {}
    };
    
    // 필드 매핑 적용
    for (const fieldMapping of entityMapping.field_mappings) {
      const sourceValue = this.getNestedValue(data, fieldMapping.source_field);
      
      // 변환 규칙 적용
      let targetValue = sourceValue;
      if (fieldMapping.transformation) {
        targetValue = this.applyTransformation(
          sourceValue, 
          fieldMapping.transformation,
          data
        );
      }
      
      // 결과에 설정
      if (targetValue !== undefined) {
        this.setNestedValue(result.fields, fieldMapping.target_field, targetValue);
      }
    }
    
    // 추가 메타데이터
    result.metadata = {
      source_system: "s-attend-gate",
      sync_timestamp: new Date().toISOString(),
      mapping_version: this.mappingConfig.version
    };
    
    return result;
  }
  
  /**
   * CRM 엔티티를 s-attend-gate 엔티티로 변환
   */
  mapFromCrm(targetEntity: string, crmData: any): any {
    // 매핑 구성 검색
    const entityMapping = this.findReverseEntityMapping(targetEntity);
    if (!entityMapping) {
      throw new Error(`No reverse mapping found for entity: ${targetEntity}`);
    }
    
    // 결과 객체 초기화
    const result: any = {};
    
    // 필드 매핑 적용 (역방향)
    for (const fieldMapping of entityMapping.field_mappings) {
      if (!fieldMapping.bidirectional) {
        continue; // 양방향이 아닌 필드 건너뛰기
      }
      
      const crmValue = this.getNestedValue(crmData, fieldMapping.target_field);
      
      // 역변환 규칙 적용
      let sourceValue = crmValue;
      if (fieldMapping.reverse_transformation) {
        sourceValue = this.applyTransformation(
          crmValue,
          fieldMapping.reverse_transformation,
          crmData
        );
      }
      
      // 결과에 설정
      if (sourceValue !== undefined) {
        this.setNestedValue(result, fieldMapping.source_field, sourceValue);
      }
    }
    
    return result;
  }
  
  // 중첩된 객체 경로에서 값 추출
  private getNestedValue(obj: any, path: string): any {
    const parts = path.split('.');
    return parts.reduce((o, key) => o && o[key] !== undefined ? o[key] : undefined, obj);
  }
  
  // 중첩된 객체 경로에 값 설정
  private setNestedValue(obj: any, path: string, value: any): void {
    const parts = path.split('.');
    const lastKey = parts.pop()!;
    const target = parts.reduce((o, key) => {
      o[key] = o[key] || {};
      return o[key];
    }, obj);
    target[lastKey] = value;
  }
  
  // 엔티티 매핑 찾기
  private findEntityMapping(sourceEntity: string) {
    return this.mappingConfig.entity_mappings.find(
      m => m.source_entity === sourceEntity
    );
  }
  
  // 역방향 엔티티 매핑 찾기
  private findReverseEntityMapping(targetEntity: string) {
    return this.mappingConfig.entity_mappings.find(
      m => m.target_entity === targetEntity
    );
  }
  
  // 변환 규칙 적용
  private applyTransformation(value: any, transformation: string, context?: any): any {
    const rule = this.transformationRules.find(r => r.name === transformation);
    if (!rule) {
      throw new Error(`Transformation rule not found: ${transformation}`);
    }
    
    return rule.transform(value, context);
  }
  
  // CRM 유형별 변환 규칙 로드
  private loadTransformationRules(crmType: string): TransformationRule[] {
    // CRM 유형에 따른 특화된 변환 규칙 로드
    const commonRules = [
      {
        name: "dateTimeFormat",
        transform: (value: string, context: any) => {
          if (!value) return null;
          const date = new Date(value);
          return date.toISOString();
        }
      },
      {
        name: "phoneFormat",
        transform: (value: string, context: any) => {
          if (!value) return null;
          // 전화번호 형식을 대상 CRM에 맞게 변환
          return value.replace(/[^\d+]/g, '');
        }
      },
      {
        name: "attendanceStatus",
        transform: (value: string, context: any) => {
          const statusMap: {[key: string]: string} = {
            "attended": "Attended",
            "no-show": "No-Show",
            "canceled": "Cancelled",
            "pending": "Registered"
          };
          return statusMap[value] || "Unknown";
        }
      }
    ];
    
    // CRM 특화 규칙
    const crmSpecificRules: {[key: string]: TransformationRule[]} = {
      "salesforce": [
        {
          name: "picklistValue",
          transform: (value: string, context: any) => {
            // Salesforce 픽리스트 값 매핑
            const fieldName = context?.fieldName;
            const picklistMappings: {[key: string]: {[key: string]: string}} = {
              "participantType": {
                "attendee": "Attendee",
                "speaker": "Speaker",
                "sponsor": "Sponsor",
                "staff": "Staff"
              }
            };
            
            return picklistMappings[fieldName]?.[value] || value;
          }
        }
      ],
      "hubspot": [
        {
          name: "dealStage",
          transform: (value: string, context: any) => {
            // HubSpot의 거래 단계 매핑
            const stageMap: {[key: string]: string} = {
              "registered": "appointmentscheduled",
              "attended": "closedwon",
              "no-show": "closedlost"
            };
            return stageMap[value] || "appointmentscheduled";
          }
        }
      ]
    };
    
    return [...commonRules, ...(crmSpecificRules[crmType.toLowerCase()] || [])];
  }
}

interface MappingConfig {
  version: string;
  entity_mappings: EntityMapping[];
}

interface EntityMapping {
  source_entity: string;
  target_entity: string;
  field_mappings: FieldMapping[];
}

interface FieldMapping {
  source_field: string;
  target_field: string;
  transformation?: string;
  reverse_transformation?: string;
  bidirectional: boolean;
}

interface TransformationRule {
  name: string;
  transform: (value: any, context?: any) => any;
}
```

### 동기화 프로세스

```typescript
class CrmSyncManager {
  private adapter: CrmAdapter;
  private dataMapper: CrmDataMapper;
  private syncConfig: SyncConfig;
  private logger: Logger;
  
  constructor(
    crmType: string,
    connectionConfig: ConnectionConfig,
    syncConfig: SyncConfig,
    logger: Logger
  ) {
    this.adapter = this.createAdapter(crmType, connectionConfig);
    this.dataMapper = new CrmDataMapper(crmType, this.loadMappingConfig(crmType));
    this.syncConfig = syncConfig;
    this.logger = logger;
  }
  
  /**
   * s-attend-gate에서 CRM으로 데이터 동기화
   */
  async syncToCrm(): Promise<SyncResult> {
    this.logger.info("Starting sync to CRM");
    const result: SyncResult = {
      started_at: new Date().toISOString(),
      entity_results: [],
      success: false,
      error: null
    };
    
    try {
      // 각 엔티티 유형에 대해 처리
      for (const entityConfig of this.syncConfig.entities) {
        this.logger.info(`Processing entity: ${entityConfig.entity_type}`);
        
        // 마지막 동기화 이후 변경된 레코드 조회
        const changedRecords = await this.fetchChangedRecords(
          entityConfig.entity_type,
          entityConfig.last_sync_time
        );
        
        const entityResult: EntitySyncResult = {
          entity_type: entityConfig.entity_type,
          records_processed: changedRecords.length,
          records_succeeded: 0,
          records_failed: 0,
          errors: []
        };
        
        // 각 레코드 처리
        for (const record of changedRecords) {
          try {
            // CRM 형식으로 변환
            const crmData = this.dataMapper.mapToCrm(entityConfig.entity_type, record);
            
            // 기존 레코드 검색 (중복 방지)
            const existingRecord = await this.adapter.findExistingRecord(
              crmData.entity_type,
              this.getExternalId(entityConfig.entity_type, record)
            );
            
            // 생성 또는 업데이트
            if (existingRecord) {
              await this.adapter.updateRecord(
                crmData.entity_type,
                existingRecord.id,
                crmData.fields
              );
            } else {
              await this.adapter.createRecord(
                crmData.entity_type,
                crmData.fields
              );
            }
            
            entityResult.records_succeeded++;
          } catch (error) {
            entityResult.records_failed++;
            entityResult.errors.push({
              record_id: record.id,
              error: error.message
            });
            this.logger.error(`Failed to sync record ${record.id}: ${error.message}`);
          }
        }
        
        result.entity_results.push(entityResult);
      }
      
      result.success = true;
      result.completed_at = new Date().toISOString();
      
      // 마지막 동기화 시간 업데이트
      await this.updateLastSyncTime();
      
      this.logger.info("Sync to CRM completed successfully");
    } catch (error) {
      result.success = false;
      result.error = error.message;
      result.completed_at = new Date().toISOString();
      this.logger.error(`Sync to CRM failed: ${error.message}`);
    }
    
    return result;
  }
  
  /**
   * CRM에서 s-attend-gate로 데이터 동기화
   */
  async syncFromCrm(): Promise<SyncResult> {
    this.logger.info("Starting sync from CRM");
    const result: SyncResult = {
      started_at: new Date().toISOString(),
      entity_results: [],
      success: false,
      error: null
    };
    
    try {
      // 각 엔티티 유형에 대해 처리
      for (const entityConfig of this.syncConfig.entities) {
        // 양방향 동기화 설정된 엔티티만 처리
        if (!entityConfig.bidirectional) {
          continue;
        }
        
        this.logger.info(`Processing CRM entity for: ${entityConfig.entity_type}`);
        
        // CRM에서 마지막 동기화 이후 변경된 레코드 조회
        const crmEntityType = this.getCrmEntityType(entityConfig.entity_type);
        const changedCrmRecords = await this.adapter.fetchChangedRecords(
          crmEntityType,
          entityConfig.last_sync_time
        );
        
        const entityResult: EntitySyncResult = {
          entity_type: entityConfig.entity_type,
          records_processed: changedCrmRecords.length,
          records_succeeded: 0,
          records_failed: 0,
          errors: []
        };
        
        // 각 레코드 처리
        for (const crmRecord of changedCrmRecords) {
          try {
            // s-attend-gate 형식으로 변환
            const localData = this.dataMapper.mapFromCrm(entityConfig.entity_type, crmRecord);
            
            // 기존 레코드 검색
            const externalId = this.getExternalIdFromCrm(
              entityConfig.entity_type,
              crmRecord
            );
            const existingRecord = await this.findLocalRecord(
              entityConfig.entity_type,
              externalId
            );
            
            // 생성 또는 업데이트
            if (existingRecord) {
              await this.updateLocalRecord(
                entityConfig.entity_type,
                existingRecord.id,
                localData
              );
            } else {
              await this.createLocalRecord(
                entityConfig.entity_type,
                localData
              );
            }
            
            entityResult.records_succeeded++;
          } catch (error) {
            entityResult.records_failed++;
            entityResult.errors.push({
              record_id: crmRecord.id,
              error: error.message
            });
            this.logger.error(`Failed to sync CRM record ${crmRecord.id}: ${error.message}`);
          }
        }
        
        result.entity_results.push(entityResult);
      }
      
      result.success = true;
      result.completed_at = new Date().toISOString();
      
      // 마지막 동기화 시간 업데이트
      await this.updateLastSyncTime();
      
      this.logger.info("Sync from CRM completed successfully");
    } catch (error) {
      result.success = false;
      result.error = error.message;
      result.completed_at = new Date().toISOString();
      this.logger.error(`Sync from CRM failed: ${error.message}`);
    }
    
    return result;
  }
  
  // 필요한 헬퍼 메서드들은 생략...
}
```

## 구성

### 연결 구성

```yaml
connection_config:
  # Salesforce 설정 예시
  salesforce:
    auth_type: "oauth2"
    client_id: "${SALESFORCE_CLIENT_ID}"
    client_secret: "${SALESFORCE_CLIENT_SECRET}"
    username: "${SALESFORCE_USERNAME}"
    password: "${SALESFORCE_PASSWORD}"
    security_token: "${SALESFORCE_SECURITY_TOKEN}"
    sandbox: false
    api_version: "v54.0"
    connection_timeout_sec: 30
    request_timeout_sec: 120
    max_retry_attempts: 3
    
  # HubSpot 설정 예시
  hubspot:
    auth_type: "api_key"
    api_key: "${HUBSPOT_API_KEY}"
    connection_timeout_sec: 20
    request_timeout_sec: 60
    max_retry_attempts: 3
    
  # 기타 CRM 설정...
```

### 동기화 구성

```yaml
sync_config:
  # 기본 설정
  default_schedule: "0 0 * * *"  # 매일 자정
  conflict_strategy: "most_recent_wins"
  batch_size: 100
  max_records_per_sync: 10000
  
  # 엔티티별 설정
  entities:
    - entity_type: "Participant"
      bidirectional: true
      conflict_strategy: "crm_wins"  # CRM 데이터 우선
      id_mapping_field: "email"
      fields_to_exclude: ["internal_notes", "payment_details"]
      sync_schedule: "0 */2 * * *"  # 2시간마다
      
    - entity_type: "Event"
      bidirectional: false  # 단방향 (s-attend-gate → CRM)
      id_mapping_field: "event_code"
      sync_schedule: "0 0 * * *"  # 매일
      
    - entity_type: "Attendance"
      bidirectional: false
      id_mapping_field: "composite:participant_id,event_id"
      sync_schedule: "0 */4 * * *"  # 4시간마다
      
    - entity_type: "Session"
      bidirectional: false
      id_mapping_field: "session_code"
      sync_schedule: "0 0 * * *"  # 매일
```

### 데이터 매핑 구성

```yaml
mapping_config:
  version: "1.3.0"
  
  # Participant <-> Contact 매핑
  - source_entity: "Participant"
    target_entity: "Contact"
    field_mappings:
      - source_field: "first_name"
        target_field: "FirstName"
        bidirectional: true
        
      - source_field: "last_name"
        target_field: "LastName"
        bidirectional: true
        
      - source_field: "email"
        target_field: "Email"
        bidirectional: true
        
      - source_field: "phone"
        target_field: "Phone"
        transformation: "phoneFormat"
        reverse_transformation: "phoneFormat"
        bidirectional: true
        
      - source_field: "company"
        target_field: "Account.Name"
        bidirectional: true
        
      - source_field: "job_title"
        target_field: "Title"
        bidirectional: true
        
      - source_field: "type"
        target_field: "Type__c"
        transformation: "picklistValue"
        reverse_transformation: "picklistValue"
        bidirectional: true
        
      - source_field: "tags"
        target_field: "Tags__c"
        transformation: "arrayToSemicolonString"
        reverse_transformation: "semicolonStringToArray"
        bidirectional: true
        
      - source_field: "created_at"
        target_field: "s_attend_Created_Date__c"
        transformation: "dateTimeFormat"
        bidirectional: false
  
  # Event <-> Campaign 매핑
  - source_entity: "Event"
    target_entity: "Campaign"
    field_mappings:
      - source_field: "name"
        target_field: "Name"
        bidirectional: false
        
      - source_field: "description"
        target_field: "Description"
        bidirectional: false
        
      - source_field: "start_date"
        target_field: "StartDate"
        transformation: "dateTimeFormat"
        bidirectional: false
        
      - source_field: "end_date"
        target_field: "EndDate"
        transformation: "dateTimeFormat"
        bidirectional: false
        
      - source_field: "venue"
        target_field: "Location__c"
        bidirectional: false
        
      - source_field: "event_code"
        target_field: "s_attend_Event_Code__c"
        bidirectional: false
        
      - source_field: "event_type"
        target_field: "Type"
        transformation: "eventTypeToCampaignType"
        bidirectional: false
```

## 통합 지점

- **인증 서비스**: 외부 시스템에 대한 OAuth 2.0 및 API 키 관리
- **스케줄러**: 정기적인 동기화 작업 예약 및 관리
- **이벤트 시스템**: 실시간 변경 감지 및 트리거 기반 동기화
- **모니터링 서비스**: 통합 상태 및 성능 모니터링

## 모니터링 및 지표

### 주요 성능 지표

- **동기화 시간**: 전체 동기화 프로세스 완료 시간 (목표: < 5분)
- **레코드 처리량**: 초당 처리된 레코드 수 (목표: > 50 records/s)
- **성공률**: 성공적으로 동기화된 레코드 비율 (목표: > 99%)
- **API 사용량**: CRM API 호출량 및 제한 임계값

### 에러 처리 및 복구

1. **단계적 백오프**: API 실패 시 점진적 재시도 (최대 3회)
2. **일부 실패 허용**: 일부 레코드 실패가 전체 동기화를 중단하지 않음
3. **실패 대기열**: 처리 실패한 레코드는 별도 대기열로 이동하여 후속 처리
4. **경고 알림**: 지정된 임계값 이상의 오류 발생 시 관리자에게 알림

### 감사 및 로깅

- 모든 동기화 작업에 대한 감사 추적 기록
- 레코드 수준의 상세 변경 이력
- 양방향 동기화 시 데이터 출처 추적
- 주기적인 데이터 일관성 검사 및 보고

## 관련 문서

- [통합 아키텍처 개요](./integration-architecture.md)
- [어댑터 디자인 패턴](./adapter-design.md)
- [데이터 변환 엔진](./data-transformation.md)
