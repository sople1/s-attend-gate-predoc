# Event Management 보안 및 데이터 보호 시나리오

## 📋 시나리오 개요

API 보안과 개인정보 보호를 위한 JWT 기반 인증, 데이터 암호화, 익명화 및 GDPR 준수 시나리오입니다.
감사 로깅과 보안 모니터링을 포함한 포괄적인 보안 체계를 다룹니다.

---

## 🔒 보안 및 데이터 보호 시나리오

### 1. API 보안

```typescript
// JWT 기반 인증 미들웨어
class AuthMiddleware {
  async validateAPIKey(req: Request, res: Response, next: NextFunction): Promise<void> {
    const apiKey = req.headers['x-api-key'] as string;
    const bearerToken = req.headers.authorization?.replace('Bearer ', '');
    
    // API 키 또는 Bearer 토큰 중 하나는 필수
    if (!apiKey && !bearerToken) {
      return res.status(401).json({ 
        error: 'Authentication required',
        code: 'AUTH_MISSING'
      });
    }

    let credentials: Credentials;

    try {
      if (bearerToken) {
        credentials = await this.validateJWT(bearerToken);
      } else {
        credentials = await this.validateAPIKey(apiKey);
      }
    } catch (error) {
      return res.status(401).json({ 
        error: 'Invalid credentials',
        code: 'AUTH_INVALID'
      });
    }

    // 요청 제한 확인 (Rate Limiting)
    const rateLimitOk = await this.checkRateLimit(credentials.clientId, req.path);
    if (!rateLimitOk) {
      return res.status(429).json({ 
        error: 'Rate limit exceeded',
        code: 'RATE_LIMIT_EXCEEDED'
      });
    }

    // 권한 확인
    const hasPermission = await this.checkPermissions(credentials, req);
    if (!hasPermission) {
      return res.status(403).json({ 
        error: 'Insufficient permissions',
        code: 'PERMISSION_DENIED'
      });
    }

    // 감사 로깅
    await this.auditLogger.logAPIAccess({
      clientId: credentials.clientId,
      endpoint: req.path,
      method: req.method,
      timestamp: new Date(),
      ip: req.ip,
      userAgent: req.headers['user-agent']
    });

    req.credentials = credentials;
    next();
  }

  private async checkRateLimit(clientId: string, endpoint: string): Promise<boolean> {
    const key = `rate_limit:${clientId}:${endpoint}`;
    const current = await this.redis.get(key) || 0;
    
    const limit = this.getRateLimitForEndpoint(endpoint);
