
```javascript
class FontSizeController {
    constructor() {
        this.currentSize = this.getSavedFontSize() || 2;
        this.applyFontSize();
        this.createFontSizeControls();
    }
    
    createFontSizeControls() {
        const controls = document.createElement('div');
        controls.className = 'font-size-controls';
        controls.innerHTML = `
            <button aria-label="글자 크기 줄이기" onclick="fontController.decreaseSize()">A-</button>
            <span aria-live="polite">글자 크기: ${this.getSizeLabel()}</span>
            <button aria-label="글자 크기 키우기" onclick="fontController.increaseSize()">A+</button>
        `;
        
        document.body.appendChild(controls);
    }
    
    increaseSize() {
        if (this.currentSize < 5) {
            this.currentSize++;
            this.applyFontSize();
            this.saveFontSize();
        }
    }
    
    decreaseSize() {
        if (this.currentSize > 1) {
            this.currentSize--;
            this.applyFontSize();
            this.saveFontSize();
        }
    }
    
    applyFontSize() {
        document.body.className = document.body.className
            .replace(/font-size-\d+/, '');
        document.body.classList.add(`font-size-${this.currentSize}`);
        
        // 스크린 리더에 변경 사항 알림
        this.announceChange();
    }
    
    getSizeLabel() {
        const labels = ['매우 작게', '작게', '보통', '크게', '매우 크게'];
        return labels[this.currentSize - 1];
    }
    
    announceChange() {
        const message = `글자 크기가 ${this.getSizeLabel()}로 변경되었습니다`;
        if (window.accessibilityManager) {
            window.accessibilityManager.announce(message);
        }
    }
    
    saveFontSize() {
        localStorage.setItem('fontSize', this.currentSize);
    }
    
    getSavedFontSize() {
        return parseInt(localStorage.getItem('fontSize'));
    }
}
```

---

## 3. 청각적 접근성

### 3.1 진동 패턴 시스템
```javascript
class HapticFeedback {
    constructor() {
        this.patterns = {
            success: [100, 50, 100],           // 짧은 진동 2번
            error: [300],                      // 긴 진동 1번
            notification: [50, 100, 50, 100, 50], // 점진동 3번
            warning: [200, 100, 200, 100, 200, 100, 200], // 경고 패턴
            emergency: [500, 200, 500, 200, 500] // 비상 패턴
        };
        
        this.isSupported = 'vibrate' in navigator;
    }
    
    vibrate(patternName) {
        if (!this.isSupported) {
            console.log('진동이 지원되지 않는 기기입니다');
            return;
        }
        
        const pattern = this.patterns[patternName];
        if (pattern) {
            navigator.vibrate(pattern);
        }
    }
    
    // 사용자 정의 패턴
    customVibrate(pattern) {
        if (this.isSupported) {
            navigator.vibrate(pattern);
        }
    }
    
    // 진동 설정 확인
    checkVibrationSettings() {
        const userPreferences = this.getUserPreferences();
        return userPreferences.vibrationEnabled !== false;
    }
    
    getUserPreferences() {
        const prefs = localStorage.getItem('accessibilityPreferences');
        return prefs ? JSON.parse(prefs) : {};
    }
}

// 출석 체크 시 진동 피드백
class AttendanceVibration extends HapticFeedback {
    onAttendanceStart() {
        if (this.checkVibrationSettings()) {
            this.vibrate('notification');
        }
    }
    
    onAttendanceSuccess() {
        if (this.checkVibrationSettings()) {
            this.vibrate('success');
        }
    }
    
    onAttendanceError() {
        if (this.checkVibrationSettings()) {
            this.vibrate('error');
        }
    }
    
    onNetworkError() {
        if (this.checkVibrationSettings()) {
            this.vibrate('warning');
        }
    }
}
```

### 3.2 시각적 알림 시스템
```css
/* 시각적 알림 스타일 */
.visual-notification {
    position: fixed;
    top: 20px;
    right: 20px;
    padding: 16px 24px;
    border-radius: 8px;
    border: 3px solid;
    font-weight: bold;
    font-size: 18px;
    min-width: 300px;
    z-index: 9999;
    animation: slideIn 0.3s ease-out;
}

.visual-notification.success {
    background-color: var(--success-color);
    border-color: var(--success-color);
    color: var(--background-color);
}

.visual-notification.error {
    background-color: var(--error-color);
    border-color: var(--error-color);
    color: var(--background-color);
    animation: shake 0.5s ease-in-out;
}

.visual-notification.warning {
    background-color: var(--warning-color);
    border-color: var(--warning-color);
    color: var(--text-color);
    animation: pulse 1s ease-in-out infinite;
}

@keyframes slideIn {
    from {
        transform: translateX(100%);
        opacity: 0;
    }
    to {
        transform: translateX(0);
        opacity: 1;
    }
}

@keyframes shake {
    0%, 100% { transform: translateX(0); }
    25% { transform: translateX(-5px); }
    75% { transform: translateX(5px); }
}

@keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.7; }
}

/* 화면 전체 플래시 효과 (긴급 상황) */
.screen-flash {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background-color: var(--error-color);
    opacity: 0.8;
    z-index: 99999;
    animation: flash 0.2s ease-in-out 3;
    pointer-events: none;
}

@keyframes flash {
    0%, 100% { opacity: 0; }
    50% { opacity: 0.8; }
}
```

```javascript
class VisualNotificationManager {
    constructor() {
        this.notifications = [];
        this.maxNotifications = 3;
    }
    
    show(message, type = 'info', duration = 5000) {
        const notification = this.createNotification(message, type);
        document.body.appendChild(notification);
        
        this.notifications.push(notification);
        this.positionNotifications();
        
        // 자동 제거
        setTimeout(() => {
            this.remove(notification);
        }, duration);
        
        // 진동 피드백과 연동
        if (window.hapticFeedback) {
            window.hapticFeedback.vibrate(type);
        }
    }
    
    createNotification(message, type) {
        const notification = document.createElement('div');
        notification.className = `visual-notification ${type}`;
        notification.setAttribute('role', 'alert');
        notification.setAttribute('aria-live', 'assertive');
        
        const icon = this.getIcon(type);
        notification.innerHTML = `
            <div style="display: flex; align-items: center; gap: 12px;">
                <span style="font-size: 24px;" aria-hidden="true">${icon}</span>
                <span>${message}</span>
                <button 
                    onclick="this.parentElement.parentElement.remove()"
                    aria-label="알림 닫기"
                    style="margin-left: auto; background: none; border: none; color: inherit; font-size: 20px; cursor: pointer;">
                    ✕
                </button>
            </div>
        `;
        
        return notification;
    }
    
    getIcon(type) {
        const icons = {
            success: '✅',
            error: '❌',
            warning: '⚠️',
            info: 'ℹ️'
        };
        return icons[type] || icons.info;
    }
    
    remove(notification) {
        if (notification && notification.parentNode) {
            notification.style.animation = 'slideOut 0.3s ease-in';
            setTimeout(() => {
                notification.remove();
                this.notifications = this.notifications.filter(n => n !== notification);
                this.positionNotifications();
            }, 300);
        }
    }
    
    positionNotifications() {
        this.notifications.forEach((notification, index) => {
            notification.style.top = `${20 + (index * 80)}px`;
        });
    }
    
    // 긴급 상황 전체 화면 플래시
    flashScreen() {
        const flash = document.createElement('div');
        flash.className = 'screen-flash';
        document.body.appendChild(flash);
        
        setTimeout(() => {
            flash.remove();
        }, 600);
    }
}
```

---

## 🔗 관련 문서

- **[고급 접근성 및 테스트](./accessibility-advanced.md)** - 운동 능력 접근성, 인지적 접근성, 테스트 및 검증
- **[User App 핵심 시나리오](./core-scenarios.md)** - 사용자 앱 핵심 기능 시나리오  
- **[User App Service 개요](./README.md)** - 서비스 전체 개요
- **[기술적 제약사항 해결](./technical-constraints-solutions.md)** - 기술적 제약사항 및 해결 방안

---

*이 분할은 파일 크기 최적화를 위해 수행되었으며, 원본 내용은 `accessibility-implementation-old.md`에 백업되어 있습니다.*
