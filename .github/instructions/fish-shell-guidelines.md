# Fish Shell 사용 지침

## Heredoc 작성 방법

Fish shell에서는 bash/zsh와 다른 heredoc 문법을 사용합니다.

### ❌ 잘못된 방법 (bash/zsh 문법)
```bash
cat > file.txt << 'EOF'
내용
EOF
```

### ✅ 올바른 방법들

#### 1. printf 사용 (권장)
```fish
printf '%s\n' '# 제목
## 섹션
내용' > file.txt
```

#### 2. echo 사용
```fish
echo '# 제목
## 섹션  
내용' > file.txt
```

#### 3. 여러 줄 문자열 (Fish 3.1+)
```fish
set content '# 제목
## 섹션
내용'
echo $content > file.txt
```

#### 4. 임시 변수 활용
```fish
set -l content "# 제목
## 섹션
내용"
printf '%s\n' $content > file.txt
```

## 실제 사용 예시

### 마크다운 파일 생성
```fish
printf '%s\n' '# 보안 성능 최적화 개요

## 개요
이 문서는 보안 성능 최적화에 대한 내용입니다.

## 주요 내용
- 데이터 처리 최적화
- 모니터링 시스템
- 성능 튜닝' > security-performance-optimization.md
```

### 긴 내용의 경우 임시 변수 사용
```fish
set -l file_content '# 제목

## 개요
상세 설명...

## 섹션 1
내용...

## 섹션 2
내용...'

printf '%s\n' $file_content > target-file.md
```

## 특수 문자 처리

### 따옴표가 포함된 경우
```fish
printf '%s\n' "# 제목
이것은 \"따옴표\"가 포함된 내용입니다." > file.txt
```

### 백틱이 포함된 경우
```fish
printf '%s\n' '# 제목
코드 예시: `echo "hello"`' > file.txt
```

## 권장사항

1. **짧은 내용**: `echo` 사용
2. **긴 내용**: `printf '%s\n'` + 임시 변수 사용
3. **특수문자 많은 경우**: 단일 따옴표(`'`) 사용하여 이스케이프 방지
4. **변수 포함**: 이중 따옴표(`"`) 사용하되 이스케이프 주의

## 디버깅 팁

### 내용 미리 확인
```fish
set -l content '내용...'
echo $content  # 미리 확인
printf '%s\n' $content > file.txt  # 파일에 쓰기
```

### 파일 내용 검증
```fish
printf '%s\n' $content > file.txt
cat file.txt  # 결과 확인
```

---

## 🚨 실제 문제가 된 명령어 패턴들과 해결책

### 문제 1: find + grep + head 조합이 멈춤
```fish
# ❌ 문제가 되는 패턴
find . -name "*.md" -type f | grep -E "(technical-|accessibility-)" | head -20

# ✅ Fish shell에서 안전한 방법
find . -name "*.md" -type f | string match -r "(technical-|accessibility-)" | head -20

# 또는 더 안전하게 단계별로
find . -name "*technical*.md" -o -name "*accessibility*.md" | head -20
```

### 문제 2: while read 루프가 멈춤
```fish
# ❌ Bash 스타일 (Fish에서 문제)
find . -name "*.md" | while read file; do mv "$file" newdir/; done

# ✅ Fish 스타일
for file in (find . -name "*.md")
    mv "$file" newdir/
end

# 또는 더 간단하게
find . -name "*.md" -exec mv {} newdir/ \;
```

### 문제 3: 복잡한 파이프라인
```fish
# ❌ 복잡한 파이프라인
find . -name "*.md" -exec wc -l {} + | sort -nr | head -10

# ✅ Fish에서 안전한 방법
for file in *.md **/*.md
    if test -f "$file"
        echo (wc -l < "$file") "$file"
    end
end | sort -nr | head -10
```

### 문제 4: wc -l 명령어 응답 없음
```fish
# ❌ 문제가 되는 패턴
wc -l $(find . -name "*.md")

# ✅ 안전한 Fish 방법
for file in (find . -name "*.md" -type f)
    echo (wc -l < "$file") "$file"
end
```

## 🛠️ 디버깅 팁

### 명령어가 멈췄을 때
1. **즉시 Ctrl+C** 로 중단
2. **단계별 실행**으로 문제 지점 찾기
3. **timeout 사용**으로 안전장치 추가

```fish
# timeout으로 안전장치 추가
timeout 10s find . -name "*.md" | head -5

# 또는 조건부 실행
if timeout 5s test -d dirname
    echo "Directory exists and accessible"
end
```

### Fish 문법 확인
```fish
# 문법 검사
fish -n command.fish  # 문법만 검사
fish -c "command"     # 단일 명령어 테스트
```

## 📚 Fish Shell 리소스

- [Fish Tutorial](https://fishshell.com/docs/current/tutorial.html)
- [Fish vs Bash Syntax](https://fishshell.com/docs/current/fish_for_bash_users.html)
- 문제 발생시: `fish --help` 또는 `man fish`
