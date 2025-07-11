# Fish Shell 명령어 사용 요약서

## 🎯 목적
이 문서는 s-attend-gate 프로젝트에서 Fish shell 사용 시 발생했던 **터미널 멈춤 현상**을 방지하기 위한 실무 가이드입니다.

## 🚨 문제가 되었던 패턴들과 해결책

### 1. 복잡한 find + grep + head 조합
**문제:** 패턴 매칭 실패 시 무한 대기
```fish
# ❌ 문제 패턴
find . -name "*.md" -type f | grep -E "(technical-|accessibility-)" | head -20

# ✅ 안전한 Fish 방법
find . -name "*technical*.md" -o -name "*accessibility*.md" | head -20
```

### 2. wc -l 명령어 응답 없음
**문제:** 대량 파일 처리 시 메모리 부족
```fish
# ❌ 문제 패턴 
wc -l $(find . -name "*.md")

# ✅ 안전한 방법
for file in (find . -name "*.md" -type f | head -50)
    echo (wc -l < "$file") "$file"
end
```

### 3. Heredoc 사용 시 구문 오류
**문제:** Fish는 bash 스타일 heredoc 미지원
```fish
# ❌ Bash 스타일
cat > file.txt << 'EOF'
content
EOF

# ✅ Fish 방법
printf '%s\n' 'content
more content' > file.txt
```

### 4. while read 루프 멈춤
**문제:** 파이프라인에서 while read가 Fish에서 문제
```fish
# ❌ 문제 패턴
find . -name "*.md" | while read file; do mv "$file" newdir/; done

# ✅ Fish 방법
for file in (find . -name "*.md")
    mv "$file" newdir/
end
```

## ✅ 권장 Fish Shell 패턴

### 파일 작업
```fish
# 안전한 파일 검색
find . -maxdepth 3 -name "*.md" -type f

# 파일 크기 확인
for file in *.md
    if test -f "$file"
        echo (wc -l < "$file") "$file"
    end
end | sort -nr | head -10
```

### 디렉토리 작업
```fish
# 디렉토리 생성 및 파일 이동
mkdir -p new/directory/structure
mv long-filename.md new/directory/short-name.md
```

### 조건부 실행
```fish
# Fish 스타일 조건문
if test -f filename.md
    mv filename.md new-location/
end

# 단축 형태
test -f filename.md; and mv filename.md new-location/
```

## 🛡️ 안전장치

### Timeout 사용
```fish
# 10초 제한
timeout 10s find . -name "*.md"

# 조건부 실행
if timeout 5s test -d directory
    echo "접근 가능"
end
```

### 단계별 검증
```fish
# 1단계: 파일 존재 확인
test -f file.md; and echo "파일 존재"

# 2단계: 작업 수행
if test -f file.md
    mv file.md new-location/
    echo "이동 완료"
end
```

## 📋 체크리스트

### 명령어 실행 전
- [ ] Fish 문법인지 확인 (`set` vs `export`, `for` vs `while`)
- [ ] 복잡한 파이프라인 분할 고려
- [ ] timeout 적용 필요성 검토
- [ ] 파일 존재성 사전 확인

### 문제 발생 시
- [ ] **즉시 Ctrl+C**로 중단
- [ ] 단계별로 명령어 분할 실행
- [ ] Fish 문법 검사: `fish -n script.fish`
- [ ] 필요시 bash로 전환: `bash -c "command"`

## 🔗 참고 자료

- **기본 지침**: `/Users/sople1/Library/CloudStorage/OneDrive-개인/Workspace/s-attend-gate/.github/instructions/base.instructions.md`
- **상세 가이드**: `/Users/sople1/Library/CloudStorage/OneDrive-개인/Workspace/s-attend-gate/.github/instructions/fish-shell-guidelines.md`
- **Fish 공식 문서**: https://fishshell.com/docs/current/

---

**작성일**: 2025.07.10  
**적용 범위**: s-attend-gate 프로젝트 전체
