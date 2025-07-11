---
applyTo: '**'
---
# Base Instructions

- Follow coding standards and best practices.
- Incorporate domain knowledge relevant to the task.
- Adhere to user preferences and guidelines.

# Terminal Environment

- This system uses Fish shell.

## Fish Shell Syntax Rules

### 🚨 Critical: Avoid Commands that Cause Shell Hangs

**Common problematic patterns that cause hangs:**

1. **Heredoc (<<)**: Fish doesn't support bash-style heredoc
   - ❌ `cat > file.txt << 'EOF'`
   - ✅ `printf '%s\n' 'content' > file.txt`

2. **Complex pipe chains with grep/find**: Can hang if patterns don't match
   - ❌ `find . -name "*.md" | grep -E "(long-pattern)" | head -20`
   - ✅ Break into smaller commands, use `-q` for quiet mode

3. **Background processes without proper handling**: 
   - ❌ `command &`
   - ✅ Use `command &; disown` or proper job control

### ✅ Safe Fish Shell Patterns

**File creation:**
```fish
# Multi-line content
printf '%s\n' '# Title
Content here
More content' > file.txt

# Single line
echo 'content' > file.txt
```

**File operations:**
```fish
# Safe file search
find . -name "*.md" -type f
find . -maxdepth 2 -name "pattern*"

# Safe line counting
for file in *.md; wc -l "$file"; end
```

**Variable assignment:**
```fish
# Fish syntax
set variable_name "value"
set -l local_var "value"

# Arrays
set list_var item1 item2 item3
```

**Conditional execution:**
```fish
# Fish if syntax
if test -f file.txt
    echo "File exists"
end

# Short circuit
test -f file.txt; and echo "exists"
```

### 🛡️ Troubleshooting Hangs

If a command hangs:
1. Use Ctrl+C to interrupt
2. Check for Fish-specific syntax issues
3. Break complex commands into simpler parts
4. Use `timeout` wrapper for potentially problematic commands:
   ```fish
   timeout 10s your_command
   ```