            '[tabindex]:not([tabindex="-1"])'
        ];
        
        this.initKeyboardHandlers();
    }
    
    initKeyboardHandlers() {
        document.addEventListener('keydown', (e) => {
            switch(e.key) {
                case 'Enter':
                case ' ':
                    if (e.target.matches('button')) {
                        e.preventDefault();
                        e.target.click();
                    }
                    break;
                    
                case 'Escape':
                    this.closeModal();
                    break;
                    
                case 'Tab':
                    this.handleTabNavigation(e);
                    break;
                    
                case 'ArrowUp':
                case 'ArrowDown':
                    if (e.target.matches('[role="listbox"] [role="option"]')) {
                        this.handleArrowNavigation(e);
                    }
                    break;
            }
        });
    }
    
    trapFocus(container) {
        const focusableElements = container.querySelectorAll(
            this.focusableElements.join(',')
        );
        
        const firstElement = focusableElements[0];
        const lastElement = focusableElements[focusableElements.length - 1];
        
        container.addEventListener('keydown', (e) => {
            if (e.key === 'Tab') {
                if (e.shiftKey) {
                    if (document.activeElement === firstElement) {
                        e.preventDefault();
                        lastElement.focus();
                    }
                } else {
                    if (document.activeElement === lastElement) {
                        e.preventDefault();
                        firstElement.focus();
                    }
                }
            }
        });
    }
}
```

---

## 2. 시각적 접근성

### 2.1 고대비 모드 구현
```css
/* 기본 색상 팔레트 */
:root {
    --primary-color: #005fcc;
    --success-color: #28a745;
    --warning-color: #ffc107;
    --error-color: #dc3545;
    --text-color: #212529;
    --background-color: #ffffff;
    --border-color: #dee2e6;
}

/* 고대비 모드 */
@media (prefers-contrast: high) {
    :root {
        --primary-color: #0000ff;
        --success-color: #008000;
        --warning-color: #ffff00;
        --error-color: #ff0000;
        --text-color: #000000;
        --background-color: #ffffff;
        --border-color: #000000;
    }
}

/* 다크 모드 */
@media (prefers-color-scheme: dark) {
    :root {
        --primary-color: #66b3ff;
        --success-color: #40e057;
        --warning-color: #ffd43b;
        --error-color: #ff6b6b;
        --text-color: #ffffff;
        --background-color: #1a1a1a;
        --border-color: #404040;
    }
}

/* 고대비 다크 모드 */
@media (prefers-color-scheme: dark) and (prefers-contrast: high) {
    :root {
        --primary-color: #ffffff;
        --success-color: #00ff00;
        --warning-color: #ffff00;
        --error-color: #ff0000;
        --text-color: #ffffff;
        --background-color: #000000;
        --border-color: #ffffff;
    }
}

/* 버튼 스타일 */
.button {
    background-color: var(--primary-color);
    color: var(--background-color);
    border: 2px solid var(--primary-color);
    padding: 12px 24px;
    font-size: 16px;
    font-weight: bold;
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.2s ease;
}

.button:hover {
    background-color: var(--background-color);
    color: var(--primary-color);
}

.button:focus {
    outline: 3px solid var(--warning-color);
    outline-offset: 2px;
}

/* 상태 표시 */
.status-success {
    background-color: var(--success-color);
    color: var(--background-color);
    border: 2px solid var(--success-color);
}

.status-error {
    background-color: var(--error-color);
    color: var(--background-color);
    border: 2px solid var(--error-color);
}

.status-warning {
    background-color: var(--warning-color);
    color: var(--text-color);
    border: 2px solid var(--warning-color);
}
```

### 2.2 반응형 글꼴 크기
```css
/* 기본 글꼴 크기 */
html {
    font-size: 16px;
}

/* 사용자 설정 반영 */
@media (prefers-reduced-motion: reduce) {
    * {
        animation-duration: 0.01ms !important;
        animation-iteration-count: 1 !important;
        transition-duration: 0.01ms !important;
    }
}

/* 텍스트 크기 조절 지원 */
.text-size-small { font-size: 0.875rem; }
.text-size-normal { font-size: 1rem; }
.text-size-large { font-size: 1.25rem; }
.text-size-extra-large { font-size: 1.5rem; }

/* 동적 글꼴 크기 조절 */
body.font-size-1 { font-size: 14px; }
body.font-size-2 { font-size: 16px; }
body.font-size-3 { font-size: 18px; }
body.font-size-4 { font-size: 20px; }
body.font-size-5 { font-size: 24px; }

/* 줄 간격 조절 */
body {
    line-height: 1.6;
}

body.line-height-normal { line-height: 1.4; }
body.line-height-relaxed { line-height: 1.6; }
body.line-height-loose { line-height: 1.8; }
```

