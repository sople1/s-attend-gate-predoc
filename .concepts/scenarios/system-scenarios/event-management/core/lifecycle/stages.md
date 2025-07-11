# Event Management - 이벤트 생명주기 관리

## 🔄 이벤트 생명주기 시나리오

### 1. 이벤트 생성 및 설정

```typescript
interface EventConfiguration {
  name: string;
  description: string;
  startDate: Date;
  endDate: Date;
  maxParticipants: number;
  registrationSettings: {
    isOpen: boolean;
    requiresApproval: boolean;
    allowWaitlist: boolean;
    cutoffDate?: Date;
  };
  checkInSettings: {
    allowEarlyCheckIn: boolean;
    earlyCheckInMinutes: number;
    requireQRCode: boolean;
    allowManualCheckIn: boolean;
  };
  notificationSettings: {
    sendWelcomeEmail: boolean;
    sendReminderEmail: boolean;
    reminderHoursBefore: number;
  };
}

class EventLifecycleManager {
  async createEvent(config: EventConfiguration): Promise<Event> {
    // 1. 이벤트 생성
    const event = await this.eventRepository.create({
      ...config,
      status: 'draft',
      participantCount: 0,
      createdAt: new Date()
    });

    // 2. 기본 게이트 설정 생성
    await this.setupDefaultGates(event.id);

    // 3. 토큰 생성 규칙 설정
    await this.setupTokenGenerationRules(event.id);

    // 4. 이메일 템플릿 설정
    await this.setupEmailTemplates(event.id, config.notificationSettings);

    return event;
  }

  async activateEvent(eventId: string): Promise<void> {
    const event = await this.eventRepository.findById(eventId);
    
    // 이벤트 유효성 검증
    await this.validateEventForActivation(event);

    // 상태 변경
    await this.eventRepository.update(eventId, {
      status: 'active',
      activatedAt: new Date()
    });

    // 게이트 시스템에 이벤트 정보 전송
    await this.notifyGateManagement(event);

    // 통합 플랫폼에 이벤트 등록
    await this.registerWithIntegratedPlatform(event);
  }
}
```

### 2. 참가자 등록 관리

```typescript
class ParticipantRegistrationManager {
  async registerParticipant(
    eventId: string, 
    participantData: ParticipantRegistrationData
  ): Promise<RegistrationResult> {
    
    // 1. 등록 가능 여부 확인
    const eligibility = await this.checkRegistrationEligibility(eventId, participantData);
    if (!eligibility.eligible) {
      return { success: false, reason: eligibility.reason };
    }

    const transaction = await this.db.beginTransaction();

    try {
      // 2. 참가자 데이터 저장
      const participant = await this.participantRepository.create({
        ...participantData,
        eventId,
        registeredAt: new Date(),
        status: eligibility.requiresApproval ? 'pending' : 'confirmed'
      }, transaction);

      // 3. 토큰 생성
      const token = await this.generateParticipantToken(participant.id);
      await this.participantRepository.updateToken(participant.id, token, transaction);

      // 4. 참가자 수 업데이트
      await this.eventRepository.incrementParticipantCount(eventId, transaction);

      // 5. 환영 이메일 스케줄링
      if (participant.status === 'confirmed') {
        await this.scheduleWelcomeEmail(participant);
      }

      await transaction.commit();

      return {
        success: true,
        participant,
        token,
        qrCode: await this.generateQRCode(token)
      };

    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }

  private async checkRegistrationEligibility(
    eventId: string, 
    participantData: ParticipantRegistrationData
  ): Promise<EligibilityResult> {
    
    const event = await this.eventRepository.findById(eventId);
    
    // 등록 기간 확인
    if (!event.registrationSettings.isOpen) {
      return { eligible: false, reason: 'Registration is closed' };
    }

    // 마감일 확인
    if (event.registrationSettings.cutoffDate && 
        new Date() > event.registrationSettings.cutoffDate) {
      return { eligible: false, reason: 'Registration deadline passed' };
    }

    // 정원 확인
    if (event.participantCount >= event.maxParticipants) {
      if (event.registrationSettings.allowWaitlist) {
        return { 
          eligible: true, 
          requiresApproval: true,
          status: 'waitlist' 
        };
      }
      return { eligible: false, reason: 'Event is full' };
    }

    // 중복 등록 확인
    const existingParticipant = await this.participantRepository.findByEmailAndEvent(
      participantData.email, 
      eventId
    );
    
    if (existingParticipant) {
      return { eligible: false, reason: 'Already registered' };
    }

    return {
      eligible: true,
      requiresApproval: event.registrationSettings.requiresApproval
    };
  }
}
```

### 3. 이벤트 진행 중 관리

```typescript
class EventProgressManager {
  private eventListeners = new Map<string, EventListener[]>();

  async startEvent(eventId: string): Promise<void> {
    const event = await this.eventRepository.findById(eventId);
    
    // 1. 이벤트 상태 변경
    await this.eventRepository.update(eventId, {
      status: 'in_progress',
      actualStartTime: new Date()
    });

    // 2. 실시간 모니터링 시작
    await this.startRealTimeMonitoring(eventId);

    // 3. 게이트 시스템 활성화
    await this.gateManagementService.activateGates(eventId);

    // 4. 출석 추적 시작
    await this.attendanceTracker.startTracking(eventId);

    // 5. 알림 발송
    await this.notifyEventStart(event);
  }

  async handleRealTimeAttendance(attendanceData: AttendanceRecord): Promise<void> {
    // 1. 출석 기록 저장
    await this.attendanceRepository.create(attendanceData);

    // 2. 실시간 통계 업데이트
    await this.updateRealTimeStats(attendanceData.eventId);

    // 3. 이벤트 리스너들에게 알림
    const listeners = this.eventListeners.get(attendanceData.eventId) || [];
    for (const listener of listeners) {
      await listener.onAttendanceUpdate(attendanceData);
    }

    // 4. 임계치 확인 및 알림
    await this.checkCapacityThresholds(attendanceData.eventId);
  }

  private async updateRealTimeStats(eventId: string): Promise<void> {
    const stats = await this.calculateCurrentStats(eventId);
    
    // Redis에 실시간 통계 저장
    await this.cacheService.set(
      `event:${eventId}:stats`, 
      stats, 
      60 // 1분 TTL
    );

    // WebSocket으로 실시간 대시보드 업데이트
    await this.websocketService.broadcast(`event:${eventId}:stats`, stats);
  }

  async endEvent(eventId: string): Promise<EventSummary> {
    const event = await this.eventRepository.findById(eventId);
    
    // 1. 이벤트 상태 변경
    await this.eventRepository.update(eventId, {
      status: 'completed',
      actualEndTime: new Date()
    });

    // 2. 최종 통계 계산
    const finalStats = await this.calculateFinalStats(eventId);

    // 3. 출석 추적 중단
    await this.attendanceTracker.stopTracking(eventId);

    // 4. 게이트 비활성화
    await this.gateManagementService.deactivateGates(eventId);

    // 5. 최종 보고서 생성 스케줄링
    await this.scheduleEventReport(eventId);

    // 6. 후속 작업 실행
    await this.executePostEventTasks(eventId);

    return {
      event,
      finalStats,
      participantCount: finalStats.totalParticipants,
      attendanceRate: finalStats.attendanceRate,
      duration: finalStats.actualDuration
    };
  }
}
```

### 4. 이벤트 종료 후 처리

```typescript
class PostEventProcessor {
  async generateEventReport(eventId: string): Promise<EventReport> {
    const [event, participants, attendanceRecords, stats] = await Promise.all([
      this.eventRepository.findById(eventId),
      this.participantRepository.findByEventId(eventId),
      this.attendanceRepository.findByEventId(eventId),
      this.calculateDetailedStats(eventId)
    ]);

    const report: EventReport = {
      event: {
        id: event.id,
        name: event.name,
        description: event.description,
        plannedDuration: event.endDate.getTime() - event.startDate.getTime(),
        actualDuration: stats.actualDuration
      },
      participation: {
        registered: participants.length,
        attended: stats.attendedCount,
        attendanceRate: stats.attendanceRate,
        checkInTimes: stats.checkInDistribution,
        noShows: stats.noShowCount
      },
      demographics: {
        ageGroups: stats.ageGroupDistribution,
        locations: stats.locationDistribution,
        registrationSources: stats.registrationSources
      },
      performance: {
        averageCheckInTime: stats.averageCheckInTime,
        peakAttendanceTime: stats.peakAttendanceTime,
        systemUptime: stats.systemUptime,
        errorRate: stats.errorRate
      },
      recommendations: await this.generateRecommendations(stats)
    };

    // 보고서 저장
    await this.reportRepository.create(report);

    // 관리자에게 이메일 발송
    await this.emailService.sendEventReport(event.organizerEmail, report);

    return report;
  }

  private async generateRecommendations(stats: DetailedStats): Promise<string[]> {
    const recommendations: string[] = [];

    if (stats.attendanceRate < 0.7) {
      recommendations.push('Consider improving marketing and reminder strategies');
    }

    if (stats.averageCheckInTime > 30) {
      recommendations.push('Optimize check-in process or add more gates');
    }

    if (stats.errorRate > 0.05) {
      recommendations.push('Review system reliability and error handling');
    }

    if (stats.peakAttendanceTime) {
      recommendations.push(`Plan for peak attendance around ${stats.peakAttendanceTime}`);
    }

    return recommendations;
  }

  async archiveEventData(eventId: string): Promise<void> {
    // 개인정보 보호를 위한 데이터 익명화
    await this.anonymizePersonalData(eventId);

    // 장기 보관을 위한 데이터 압축
    await this.compressEventData(eventId);

    // 접근 빈도가 낮은 데이터를 콜드 스토리지로 이동
    await this.moveToArchive(eventId);

    // 보존 정책에 따른 삭제 스케줄링
    await this.scheduleDataRetention(eventId);
  }
}
```

## 성능 지표

### 이벤트 생명주기 KPI
- **이벤트 생성 시간**: < 5초
- **등록 처리 시간**: < 2초  
- **실시간 업데이트 지연**: < 1초
- **보고서 생성 시간**: < 30초

### 참가자 관리 메트릭
- **등록 완료율**: > 95%
- **토큰 생성 성공률**: 100%
- **이메일 발송 성공률**: > 98%
- **중복 등록 방지율**: 100%

---

## 🔗 관련 파일

- **[데이터 처리 및 통합](./core-scenarios-data-integration.md)** - 데이터 흐름 및 외부 시스템 연동
- **[성능 및 확장성](./core-scenarios-performance-scalability.md)** - 시스템 최적화 및 확장 전략
- **[Event Management 개요](./core-scenarios.md)** - 전체 시스템 개요
