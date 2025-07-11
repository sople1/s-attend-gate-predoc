// filepath: /Users/sople1/Library/CloudStorage/OneDrive-개인/Workspace/s-attend-gate/.github/instructions/base.instructions.md
---
applyTo: '**'
---
# 기본 지침

- 코딩 표준과 모범 사례를 따르세요.
- 작업과 관련된 도메인 지식을 활용하세요.
- 사용자 선호도와 가이드라인을 준수하세요.
- 명확하고 간결한 코드를 작성하세요.
- 코드 리뷰와 피드백을 적극적으로 반영하세요.
- 테스트와 디버깅을 철저히 수행하세요.
- 문서화와 주석을 통해 코드 이해도를 높이세요.
- 지속적인 학습과 개선을 추구하세요.
- 기능별로 명확한 책임 분리를 유지하세요.
- 불필요한 중복을 피하고, 재사용에 우선한 코드를 작성하세요.

# 터미널 환경

- 이 시스템은 Fish shell을 사용합니다.

## Fish Shell 문법 규칙

### 🚨 중요: 셸 멈춤을 일으키는 명령어 회피

**멈춤을 일으키는 일반적인 문제 패턴:**

1. **Heredoc (<<)**: Fish는 bash 스타일 heredoc을 지원하지 않음
   - ❌ `cat > file.txt << 'EOF'`
   - ✅ `printf '%s\n' 'content' > file.txt`

2. **grep/find를 사용한 복잡한 파이프 체인**: 패턴이 일치하지 않으면 멈출 수 있음
   - ❌ `find . -name "*.md" | grep -E "(long-pattern)" | head -20`
   - ✅ 더 작은 명령어로 나누고, `-q` 옵션을 사용하여 조용한 모드 활용

3. **적절하지 않은 백그라운드 프로세스 처리**: 
   - ❌ `command &`
   - ✅ `command &; disown` 또는 적절한 작업 제어 사용

### ✅ 안전한 Fish Shell 패턴

**파일 생성:**
```fish
# 여러 줄 내용
printf '%s\n' '# 제목
여기에 내용
추가 내용' > file.txt

# 한 줄
echo 'content' > file.txt
```

**파일 작업:**
```fish
# 안전한 파일 검색
find . -name "*.md" -type f
find . -maxdepth 2 -name "pattern*"

# 안전한 줄 수 세기
for file in *.md; wc -l "$file"; end
```

**변수 할당:**
```fish
# Fish 문법
set variable_name "value"
set -l local_var "value"

# 배열
set list_var item1 item2 item3
```

**조건부 실행:**
```fish
# Fish if 문법
if test -f file.txt
    echo "파일이 존재합니다"
end

# 단축 실행
test -f file.txt; and echo "존재함"
```

### 🛡️ 멈춤 문제 해결

명령어가 멈춘 경우:
1. Ctrl+C를 사용하여 중단
2. Fish 관련 문법 문제 확인
3. 복잡한 명령어를 더 간단한 부분으로 나누기
4. 문제가 될 수 있는 명령어에 `timeout` 래퍼 사용:
   ```fish
   timeout 10s your_command
   ```
