# User App 기본 접근성 기능 개요

## 개요

웹 접근성 가이드라인(WCAG 2.1 AA)을 준수하여 모든 사용자가 동등하게 출석 체크 시스템을 이용할 수 있도록 하는 기본적인 기술 구현 방안을 제공합니다.

**⚠️ 파일 분할 완료**: 이 문서는 관리 편의성을 위해 3개의 전문 영역으로 분할되었습니다.

## 📋 기본 접근성 영역

### 🔊 스크린 리더 지원
**파일**: [accessibility/screen-reader.md](accessibility/screen-reader.md)

NVDA, JAWS, VoiceOver 등 스크린 리더 사용자를 위한 구현:
- HTML 접근성 구조 및 ARIA 라이브 영역
- 키보드 네비게이션 및 포커스 관리
- 스크린 리더 호환성 테스트

### 👁️ 시각적 접근성
**파일**: [accessibility/visual.md](accessibility/visual.md)

저시력자, 색맹/색약자를 위한 시각적 개선:
- 고대비 모드 (다크/라이트 테마)
- 반응형 글꼴 크기 조절
- 색맹 친화적 컬러 팔레트 및 패턴

### 🔇 청각적 접근성
**파일**: [accessibility/audio.md](accessibility/audio.md)

청각 장애인, 난청자를 위한 대체 피드백:
- 진동 패턴 시스템
- 시각적 알림 및 화면 깨빡임 효과
- 브라우저 알림 및 설정 개인화

## 📊 접근성 성능 지표

| 영역 | WCAG 2.1 기준 | 구현 상태 | 테스트 도구 |
|------|---------------|-----------|-------------|
| **스크린 리더** | AA | ✅ 준수 | axe-core, NVDA |
| **키보드 네비게이션** | AA | ✅ 준수 | 수동 테스트 |
| **색상 대비** | AA (4.5:1) | ✅ 준수 | Colour Contrast Analyser |
| **폰트 크기** | 200% 확대 | ✅ 지원 | 브라우저 확대 |
| **진동 피드백** | - | ✅ 지원 | 실제 디바이스 |

## 🔗 관련 문서

### 📄 고급 접근성
- **[고급 접근성 기능](./accessibility-advanced.md)**: 운동/인지 장애 지원, 고급 테스트

### 🌐 연관 시나리오  
- **[User App 기술 성능](./technical-performance-optimization.md)**: 성능 최적화와 접근성
- **[User App 제약 해결](./technical-constraints-solutions.md)**: 기술적 제약 사항 해결

---

**파일 분할 이력**:
- **원본**: `accessibility-basic.md` (717줄, 2024-12-19 분할)
- **분할 결과**: 3개 전문 파일 + 1개 개요 파일
- **총 라인 수 감소**: 717줄 → 약 70줄 (90% 감소)
            </button>
            <p id="auto-check-desc">
                블루투스를 이용해 자동으로 출석을 체크합니다
            </p>
            
            <button 
                id="qr-scan-btn"
                aria-describedby="qr-scan-desc"
                onclick="startQRScan()">
                QR 코드 스캔
            </button>
            <p id="qr-scan-desc">
                카메라로 QR 코드를 스캔하여 출석을 체크합니다
            </p>
        </section>
    </main>
</body>
</html>
```

### 1.2 ARIA 라이브 영역 구현
```javascript
class AccessibilityManager {
    constructor() {
        this.liveRegion = document.getElementById('attendance-result');
        this.announcements = [];
        this.isAnnouncing = false;
    }
    
    announce(message, priority = 'polite') {
        this.announcements.push({ message, priority });
        this.processAnnouncements();
    }
    
    processAnnouncements() {
        if (this.isAnnouncing || this.announcements.length === 0) {
            return;
        }
        
        this.isAnnouncing = true;
        const { message, priority } = this.announcements.shift();
        
        // 라이브 영역 업데이트
        this.liveRegion.setAttribute('aria-live', priority);
        this.liveRegion.textContent = message;
        
        // 음성 합성 동시 실행
        if ('speechSynthesis' in window) {
            this.speakMessage(message);
        }
        
        // 다음 안내를 위한 대기
        setTimeout(() => {
            this.isAnnouncing = false;
            this.processAnnouncements();
        }, 2000);
    }
    
    speakMessage(message) {
        const utterance = new SpeechSynthesisUtterance(message);
        utterance.lang = 'ko-KR';
        utterance.rate = 0.9;
        utterance.pitch = 1.0;
        utterance.volume = 0.8;
        
        speechSynthesis.speak(utterance);
    }
    
    // 출석 상태 변경 시 호출
    updateAttendanceStatus(status, method = null) {
        let message = '';
        
        switch(status) {
            case 'checking':
                message = '출석을 확인하고 있습니다. 잠시만 기다려주세요.';
                break;
            case 'success':
                message = `출석이 완료되었습니다. ${method} 방식으로 처리되었습니다.`;
                break;
            case 'failed':
                message = '출석 확인에 실패했습니다. 다른 방법을 시도해보세요.';
                break;
            case 'error':
                message = '오류가 발생했습니다. 관리자에게 도움을 요청하세요.';
                break;
        }
        
        this.announce(message, status === 'error' ? 'assertive' : 'polite');
    }
}
```

### 1.3 키보드 네비게이션
```css
/* 포커스 인디케이터 */
*:focus {
    outline: 3px solid #005fcc;
    outline-offset: 2px;
}

/* 스킵 링크 */
.skip-link {
    position: absolute;
    top: -40px;
    left: 6px;
    background: #000;
    color: #fff;
    padding: 8px;
    text-decoration: none;
    z-index: 1000;
}

.skip-link:focus {
    top: 6px;
}

/* 키보드 전용 요소 */
.keyboard-only {
    position: absolute;
    left: -10000px;
    width: 1px;
    height: 1px;
    overflow: hidden;
}

.keyboard-only:focus {
    position: static;
    width: auto;
    height: auto;
}
```

```javascript
class KeyboardNavigation {
    constructor() {
        this.focusableElements = [
            'button',
            'input',
            'select',
            'textarea',
            'a[href]',
