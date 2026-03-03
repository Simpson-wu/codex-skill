# 代码审查指令模板

> 所有命令需要先完成 SKILL.md Step 1 环境探测，设置 `CODEX_WSL_DISTRO`、`CODEX_LINUX_BIN`、`CODEX_WSL_HOME`。

## 快速审查 — 未提交的更改

```bash
WSL_CWD=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$(pwd)")
wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
  "export HOME='$(dirname "$CODEX_WSL_HOME")'; cd '$WSL_CWD'; '$CODEX_LINUX_BIN' review --uncommitted"
```

## 分支对比审查

```bash
WSL_CWD=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$(pwd)")
wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
  "export HOME='$(dirname "$CODEX_WSL_HOME")'; cd '$WSL_CWD'; '$CODEX_LINUX_BIN' review --base main"
```

## 自定义审查重点

通过 stdin 传入自定义指令（避免 shell 特殊字符问题）：

```bash
WSL_CWD=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$(pwd)")

TMPFILE=$(mktemp /tmp/codex-review-XXXXXX.md)
cat > "$TMPFILE" << 'PROMPT_EOF'
请重点检查以下方面：
1. 安全漏洞（注入、XSS、敏感信息泄露）
2. 性能问题（N+1 查询、不必要的循环、内存泄漏）
3. 错误处理（是否有未捕获的异常、错误信息是否有用）
4. 代码风格（命名规范、函数长度、职责单一性）
PROMPT_EOF

WSL_TMP=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$TMPFILE")
wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
  "export HOME='$(dirname "$CODEX_WSL_HOME")'; cd '$WSL_CWD'; tr -d '\r' < '$WSL_TMP' | '$CODEX_LINUX_BIN' review -"
rm -f "$TMPFILE"
```

## 针对特定文件的审查

获取 diff 内容后，通过临时文件 + stdin 传入（`git diff` 输出包含 `$` 等特殊字符，不能直接插值）：

```bash
WSL_CWD=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$(pwd)")

# 收集 diff 并写入临时文件
TMPFILE=$(mktemp /tmp/codex-review-XXXXXX.md)
{
  echo "请审查以下代码变更，重点关注逻辑正确性和边界情况："
  echo ""
  git diff -- path/to/file.ts
} > "$TMPFILE"

WSL_TMP=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$TMPFILE")
wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
  "export HOME='$(dirname "$CODEX_WSL_HOME")'; cd '$WSL_CWD'; tr -d '\r' < '$WSL_TMP' | '$CODEX_LINUX_BIN' exec -s read-only --ephemeral -"
rm -f "$TMPFILE"
```
