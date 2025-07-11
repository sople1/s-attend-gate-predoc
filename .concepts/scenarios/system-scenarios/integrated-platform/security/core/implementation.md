# Integrated Platform - 보안 시스템

## 🔒 보안 및 권한 관리 시나리오

### 시나리오 6: 다층 보안 시스템

**목표**: 엔터프라이즈급 보안 요구사항을 만족하는 다층 보안 시스템 구현

#### 6.1 통합 인증 및 권한 시스템

```typescript
// src/security/AuthenticationService.ts
class AuthenticationService {
  private jwtService: JWTService;
  private rbacService: RBACService;
  
  async authenticateUser(credentials: LoginCredentials): Promise<AuthResult> {
    // 1. 기본 인증 (이메일/패스워드)
    const user = await this.validateCredentials(credentials);
    if (!user) {
      throw new UnauthorizedError('Invalid credentials');
    }
    
    // 2. 2FA 검증 (활성화된 경우)
    if (user.twoFactorEnabled) {
      await this.validate2FA(user.id, credentials.twoFactorCode);
    }
    
    // 3. 권한 로드
    const permissions = await this.rbacService.getUserPermissions(user.id);
    
    // 4. JWT 토큰 생성
    const accessToken = await this.jwtService.generateAccessToken({
      userId: user.id,
      email: user.email,
      roles: user.roles,
      permissions: permissions.map(p => p.name)
    });
    
    const refreshToken = await this.jwtService.generateRefreshToken(user.id);
    
    // 5. 세션 저장
    await this.saveUserSession(user.id, accessToken, refreshToken);
    
    return {
      user: this.sanitizeUser(user),
      accessToken,
      refreshToken,
      permissions,
      expiresIn: this.jwtService.getTokenExpiry()
    };
  }
  
  async authorizeAction(
    userId: string, 
    action: string, 
    resource: string
  ): Promise<boolean> {
    const userPermissions = await this.rbacService.getUserPermissions(userId);
    
    return this.rbacService.checkPermission(userPermissions, action, resource);
  }
}
```

#### 6.2 데이터 암호화 및 개인정보 보호

```typescript
// src/security/DataProtectionService.ts
class DataProtectionService {
  private encryptionService: EncryptionService;
  
  async encryptSensitiveData(data: SensitiveData): Promise<EncryptedData> {
    const encryptedFields: Record<string, string> = {};
    
    for (const [field, value] of Object.entries(data)) {
      if (this.isSensitiveField(field)) {
        encryptedFields[field] = await this.encryptionService.encrypt(
          value, 
          this.getEncryptionKey(field)
        );
      } else {
        encryptedFields[field] = value;
      }
    }
    
    return encryptedFields as EncryptedData;
  }
  
  async anonymizeParticipantData(
    participantData: ParticipantData[]
  ): Promise<AnonymizedData[]> {
    return participantData.map(participant => ({
      // 고유 식별자는 해시로 변환
      participantHash: this.generateParticipantHash(participant.id),
      
      // 개인정보는 제거하고 분석 가능한 데이터만 유지
      ageGroup: this.getAgeGroup(participant.age),
      region: this.getRegion(participant.address),
      interests: participant.interests,
      attendanceHistory: participant.attendanceHistory.map(event => ({
        eventType: event.type,
        date: this.truncateToMonth(event.date),
        duration: event.duration
      }))
    }));
  }
  
  private isSensitiveField(field: string): boolean {
    const sensitiveFields = [
      'ssn', 'nationalId', 'passport', 'creditCard',
      'email', 'phone', 'address', 'birthDate'
    ];
    
    return sensitiveFields.includes(field);
  }
  
  private getEncryptionKey(field: string): string {
    // 필드별로 다른 암호화 키 사용
    return process.env[`ENCRYPTION_KEY_${field.toUpperCase()}`] || 
           process.env.DEFAULT_ENCRYPTION_KEY;
  }
  
  private generateParticipantHash(id: string): string {
    return crypto
      .createHash('sha256')
      .update(id + process.env.HASH_SALT)
      .digest('hex');
  }
}
```

#### 6.3 API 보안 및 접근 제어

```typescript
// src/security/APISecurityMiddleware.ts
class APISecurityMiddleware {
  private rateLimiter: Map<string, RateLimiter>;
  
  constructor() {
    this.rateLimiter = new Map([
      ['general', new RateLimiter({ windowMs: 15 * 60 * 1000, max: 100 })],
      ['auth', new RateLimiter({ windowMs: 15 * 60 * 1000, max: 5 })],
      ['premium', new RateLimiter({ windowMs: 15 * 60 * 1000, max: 1000 })]
    ]);
  }
  
  async securityMiddleware(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      // 1. Security headers 추가
      this.addSecurityHeaders(res);
      
      // 2. Rate limiting 적용
      await this.applyRateLimit(req, res);
      
      // 3. 입력 검증
      await this.validateInput(req);
      
      // 4. JWT 토큰 검증 (필요한 경우)
      if (this.requiresAuthentication(req.path)) {
        await this.validateJWTToken(req);
      }
      
      // 5. 권한 확인
      if (this.requiresAuthorization(req.path)) {
        await this.checkPermissions(req);
      }
      
      next();
    } catch (error) {
      this.handleSecurityError(error, res);
    }
  }
  
  private async applyRateLimit(req: Request, res: Response): Promise<void> {
    let limiterType = 'general';
    
    if (req.path.startsWith('/api/auth')) {
      limiterType = 'auth';
    } else if (req.partner?.tier === 'premium') {
      limiterType = 'premium';
    }
    
    const limiter = this.rateLimiter.get(limiterType);
    const allowed = await limiter.checkLimit(req);
    
    if (!allowed) {
      throw new Error('Rate limit exceeded');
    }
  }
  
  private addSecurityHeaders(res: Response): void {
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('X-Frame-Options', 'DENY');
    res.setHeader('X-XSS-Protection', '1; mode=block');
    res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    res.setHeader('Content-Security-Policy', "default-src 'self'");
  }
  
  private async validateInput(req: Request): Promise<void> {
    // SQL Injection 방지
    const sqlInjectionPatterns = [
      /(\bSELECT\b|\bINSERT\b|\bUPDATE\b|\bDELETE\b|\bDROP\b)/i,
      /(UNION\s+SELECT)/i,
      /(\bOR\b|\bAND\b)\s+\d+\s*=\s*\d+/i
    ];
    
    // XSS 방지
    const xssPatterns = [
      /<script[^>]*>.*?<\/script>/gi,
      /javascript:/gi,
      /on\w+\s*=/gi
    ];
    
    const allPatterns = [...sqlInjectionPatterns, ...xssPatterns];
    const inputData = JSON.stringify(req.body) + JSON.stringify(req.query);
    
    for (const pattern of allPatterns) {
      if (pattern.test(inputData)) {
        throw new SecurityError('Malicious input detected');
      }
    }
  }
  
  private async validateJWTToken(req: Request): Promise<void> {
    const token = this.extractToken(req);
    if (!token) {
      throw new UnauthorizedError('Token required');
    }
    
    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      req.user = decoded;
    } catch (error) {
      throw new UnauthorizedError('Invalid token');
    }
  }
  
  private async checkPermissions(req: Request): Promise<void> {
    const requiredPermission = this.getRequiredPermission(req.path, req.method);
    const userPermissions = req.user?.permissions || [];
    
    if (!userPermissions.includes(requiredPermission)) {
      throw new ForbiddenError('Insufficient permissions');
    }
  }
}
```

#### 6.4 감사 로그 및 보안 모니터링

```typescript
// src/security/AuditService.ts
class AuditService {
  private auditLogger: Logger;
  private securityMonitor: SecurityMonitor;
  
  async logSecurityEvent(event: SecurityEvent): Promise<void> {
    const auditEntry = {
      id: uuidv4(),
      timestamp: new Date().toISOString(),
      type: event.type,
      userId: event.userId,
      ipAddress: event.ipAddress,
      userAgent: event.userAgent,
      action: event.action,
      resource: event.resource,
      result: event.result,
      details: event.details,
      severity: this.calculateSeverity(event)
    };
    
    // 감사 로그 저장
    await this.auditLogger.log(auditEntry);
    
    // 보안 이벤트 분석
    await this.analyzeSecurityEvent(auditEntry);
    
    // 실시간 모니터링
    await this.securityMonitor.processEvent(auditEntry);
  }
  
  private async analyzeSecurityEvent(event: AuditEntry): Promise<void> {
    // 의심스러운 활동 패턴 감지
    const suspiciousPatterns = await this.detectSuspiciousPatterns(event);
    
    if (suspiciousPatterns.length > 0) {
      await this.handleSuspiciousActivity(event, suspiciousPatterns);
    }
    
    // 보안 메트릭 업데이트
    await this.updateSecurityMetrics(event);
  }
  
  private async detectSuspiciousPatterns(event: AuditEntry): Promise<string[]> {
    const patterns: string[] = [];
    
    // 1. 짧은 시간 내 다수 로그인 실패
    const recentFailures = await this.getRecentLoginFailures(
      event.ipAddress, 
      '15 minutes'
    );
    if (recentFailures > 5) {
      patterns.push('brute_force_attack');
    }
    
    // 2. 비정상적인 시간대 접근
    const hour = new Date().getHours();
    if (hour < 6 || hour > 22) {
      patterns.push('unusual_time_access');
    }
    
    // 3. 권한 상승 시도
    if (event.action.includes('admin') && !event.userId.includes('admin')) {
      patterns.push('privilege_escalation_attempt');
    }
    
    // 4. 대량 데이터 접근
    if (event.details?.recordCount > 1000) {
      patterns.push('bulk_data_access');
    }
    
    return patterns;
  }
  
  private async handleSuspiciousActivity(
    event: AuditEntry, 
    patterns: string[]
  ): Promise<void> {
    const alert = {
      severity: 'high',
      title: 'Suspicious Activity Detected',
      description: `Detected patterns: ${patterns.join(', ')}`,
      userId: event.userId,
      ipAddress: event.ipAddress,
      timestamp: event.timestamp,
      patterns
    };
    
    // 즉시 알림 발송
    await this.notificationService.sendSecurityAlert(alert);
    
    // 임시 계정 잠금 (심각한 경우)
    if (patterns.includes('brute_force_attack')) {
      await this.temporaryAccountLock(event.userId, event.ipAddress);
    }
    
    // 보안팀에 에스컬레이션
    if (patterns.includes('privilege_escalation_attempt')) {
      await this.escalateToSecurityTeam(alert);
    }
  }
  
  async generateSecurityReport(period: 'daily' | 'weekly' | 'monthly'): Promise<SecurityReport> {
    const startDate = this.getStartDate(period);
    const auditLogs = await this.getAuditLogs(startDate);
    
    return {
      period,
      startDate: startDate.toISOString(),
      endDate: new Date().toISOString(),
      summary: {
        totalEvents: auditLogs.length,
        successfulLogins: auditLogs.filter(e => e.type === 'login' && e.result === 'success').length,
        failedLogins: auditLogs.filter(e => e.type === 'login' && e.result === 'failure').length,
        suspiciousActivities: auditLogs.filter(e => e.severity === 'high').length,
        dataAccesses: auditLogs.filter(e => e.type === 'data_access').length
      },
      topRisks: await this.identifyTopRisks(auditLogs),
      recommendations: await this.generateSecurityRecommendations(auditLogs)
    };
  }
}
```

#### 6.5 컴플라이언스 및 규정 준수

```typescript
// src/security/ComplianceService.ts
class ComplianceService {
  private complianceRules: Map<string, ComplianceRule>;
  
  constructor() {
    this.complianceRules = new Map([
      ['gdpr', new GDPRComplianceRule()],
      ['hipaa', new HIPAAComplianceRule()],
      ['pci_dss', new PCIDSSComplianceRule()],
      ['iso27001', new ISO27001ComplianceRule()]
    ]);
  }
  
  async validateCompliance(
    dataType: string, 
    operation: string, 
    context: ComplianceContext
  ): Promise<ComplianceResult> {
    const applicableRules = this.getApplicableRules(dataType, operation);
    const results: ComplianceCheckResult[] = [];
    
    for (const rule of applicableRules) {
      const result = await rule.validate(dataType, operation, context);
      results.push(result);
    }
    
    return {
      compliant: results.every(r => r.compliant),
      results,
      recommendations: this.generateComplianceRecommendations(results)
    };
  }
  
  async ensureDataRetentionCompliance(): Promise<void> {
    const retentionPolicies = await this.getDataRetentionPolicies();
    
    for (const policy of retentionPolicies) {
      const expiredData = await this.findExpiredData(policy);
      
      if (expiredData.length > 0) {
        await this.handleExpiredData(policy, expiredData);
      }
    }
  }
  
  private async handleExpiredData(
    policy: DataRetentionPolicy, 
    expiredData: ExpiredDataItem[]
  ): Promise<void> {
    switch (policy.action) {
      case 'delete':
        await this.securelyDeleteData(expiredData);
        break;
      case 'anonymize':
        await this.anonymizeData(expiredData);
        break;
      case 'archive':
        await this.archiveData(expiredData, policy.archiveLocation);
        break;
    }
    
    // 컴플라이언스 로그 기록
    await this.logComplianceAction({
      action: policy.action,
      dataType: policy.dataType,
      itemCount: expiredData.length,
      timestamp: new Date().toISOString()
    });
  }
  
  async generateComplianceReport(): Promise<ComplianceReport> {
    const allRules = Array.from(this.complianceRules.values());
    const reportSections: ComplianceReportSection[] = [];
    
    for (const rule of allRules) {
      const section = await rule.generateReportSection();
      reportSections.push(section);
    }
    
    return {
      generatedAt: new Date().toISOString(),
      overallStatus: this.calculateOverallComplianceStatus(reportSections),
      sections: reportSections,
      actionItems: this.extractActionItems(reportSections)
    };
  }
}
```

---

## 🔗 관련 파일

- **[성능 최적화 및 모니터링](./security-performance-optimization.md)** - 시스템 성능 최적화 전략
- **[장애 대응 및 복구](./security-performance-disaster-recovery.md)** - 장애 복구 및 연속성 보장
- **[통합 플랫폼 보안 성능 개요](./security-performance.md)** - 전체 개요
- **[데이터 통합 및 API 허브](./data-integration-api.md)** - 데이터 보안 연동
