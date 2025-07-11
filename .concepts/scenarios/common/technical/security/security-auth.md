# Common Technical Patterns - 보안 및 인증 패턴

## 🔐 개요

모든 서비스에서 공통으로 사용되는 보안, 인증, 개인정보 보호 패턴을 정의합니다.

**주요 포함 내용:**
- JWT 토큰 관리 및 인증
- 데이터 암호화 및 복호화
- 역할 기반 접근 제어 (RBAC)
- 개인정보 보호 및 GDPR 컴플라이언스
- 보안 이벤트 로깅 및 감사

---

## 🔐 공통 보안 패턴

### 모든 서비스에서 공통 사용되는 보안 모듈

#### JWT 토큰 관리

```typescript
// common/security/TokenManager.ts
class TokenManager {
  private secretKey: string;
  private issuer: string;
  
  constructor(config: TokenConfig) {
    this.secretKey = config.secretKey;
    this.issuer = config.issuer;
  }
  
  generateAccessToken(payload: TokenPayload): string {
    return jwt.sign(
      {
        ...payload,
        type: 'access'
      },
      this.secretKey,
      {
        expiresIn: '15m',
        issuer: this.issuer,
        audience: payload.audience || 'attend-gate-api'
      }
    );
  }
  
  generateRefreshToken(userId: string): string {
    return jwt.sign(
      {
        userId,
        type: 'refresh'
      },
      this.secretKey,
      {
        expiresIn: '7d',
        issuer: this.issuer
      }
    );
  }
  
  generateEventToken(eventId: string, participantId: string): string {
    return jwt.sign(
      {
        eventId,
        participantId,
        type: 'event_access'
      },
      this.secretKey,
      {
        expiresIn: '24h',
        issuer: this.issuer,
        audience: 'attend-gate-event'
      }
    );
  }
  
  verifyToken(token: string, expectedType?: string): TokenPayload {
    const decoded = jwt.verify(token, this.secretKey) as any;
    
    if (expectedType && decoded.type !== expectedType) {
      throw new Error(`Invalid token type. Expected: ${expectedType}, Got: ${decoded.type}`);
    }
    
    return decoded;
  }
  
  refreshToken(refreshToken: string): AuthTokens {
    const decoded = this.verifyToken(refreshToken, 'refresh');
    
    const newAccessToken = this.generateAccessToken({
      userId: decoded.userId,
      audience: 'attend-gate-api'
    });
    
    const newRefreshToken = this.generateRefreshToken(decoded.userId);
    
    return {
      accessToken: newAccessToken,
      refreshToken: newRefreshToken,
      expiresIn: 900 // 15분
    };
  }
}
```

#### 데이터 암호화 유틸리티

```typescript
// common/security/EncryptionUtils.ts
class EncryptionUtils {
  private static readonly ALGORITHM = 'aes-256-gcm';
  private static readonly KEY_LENGTH = 32;
  private static readonly IV_LENGTH = 16;
  private static readonly TAG_LENGTH = 16;
  
  static encrypt(data: string, key: string): EncryptedData {
    const iv = crypto.randomBytes(this.IV_LENGTH);
    const cipher = crypto.createCipher(this.ALGORITHM, key);
    cipher.setAAD(Buffer.from('attend-gate-aad')); // Additional Authenticated Data
    
    let encrypted = cipher.update(data, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return {
      data: encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex')
    };
  }
  
  static decrypt(encryptedData: EncryptedData, key: string): string {
    const decipher = crypto.createDecipher(this.ALGORITHM, key);
    decipher.setAAD(Buffer.from('attend-gate-aad'));
    decipher.setAuthTag(Buffer.from(encryptedData.authTag, 'hex'));
    
    let decrypted = decipher.update(encryptedData.data, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
  
  static hashPassword(password: string, salt?: string): Promise<HashedPassword> {
    return new Promise((resolve, reject) => {
      const saltValue = salt || crypto.randomBytes(16).toString('hex');
      
      crypto.pbkdf2(password, saltValue, 100000, 64, 'sha512', (err, derivedKey) => {
        if (err) reject(err);
        
        resolve({
          hash: derivedKey.toString('hex'),
          salt: saltValue
        });
      });
    });
  }
  
  static verifyPassword(password: string, hashedPassword: HashedPassword): Promise<boolean> {
    return new Promise((resolve, reject) => {
      crypto.pbkdf2(password, hashedPassword.salt, 100000, 64, 'sha512', (err, derivedKey) => {
        if (err) reject(err);
        
        const hash = derivedKey.toString('hex');
        resolve(hash === hashedPassword.hash);
      });
    });
  }
  
  static generateSecureKey(): string {
    return crypto.randomBytes(this.KEY_LENGTH).toString('hex');
  }
  
  static generateSecureToken(length: number = 32): string {
    return crypto.randomBytes(length).toString('hex');
  }
}
```

#### 개인정보 보호 관리

```typescript
// common/security/PrivacyManager.ts
class PrivacyManager {
  private encryptionKey: string;
  private dataDeletionQueue: DataDeletionQueue;
  
  constructor(config: PrivacyConfig) {
    this.encryptionKey = config.encryptionKey;
    this.dataDeletionQueue = new DataDeletionQueue(config.deletionConfig);
  }
  
  async anonymizeUserData(userId: string, userData: UserData): Promise<AnonymizedData> {
    // PII 제거 및 해시화
    return {
      userHash: this.generateUserHash(userId),
      ageGroup: this.generalizeAge(userData.age),
      location: this.generalizeLocation(userData.location),
      preferences: this.anonymizePreferences(userData.preferences),
