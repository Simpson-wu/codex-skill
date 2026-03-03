---
name: ask-codex
description: 调用 Codex CLI（GPT-5）获取独立第二意见。适用于审核 Plan、测试方案、制定新方案、代码审查、通用问答等任何需要外部 AI 观点的场景。
allowed-tools: Bash, Read, Glob
---

# Ask Codex — Codex CLI Bridge Skill

通过 Codex CLI 调用 GPT-5 获取独立第二意见。你是桥接层，负责将用户意图转化为 Codex CLI 命令并返回原始结果。

## 运行环境

Codex CLI 的 Windows 原生二进制存在已知崩溃（GitHub #12962），因此通过 **WSL** 运行 Linux 版本。

### 环境变量

所有路径通过环境变量抽象，**禁止硬编码**：

| 变量 | 含义 | 探测方式 |
|------|------|----------|
| `CODEX_WSL_DISTRO` | WSL 发行版名称 | `wsl -l -q \| head -1`（取默认发行版） |
| `CODEX_LINUX_BIN` | Linux 二进制的 WSL 路径 | 探测 `~/.codex/bin/codex-linux`，转换为 `/mnt/c/...` |
| `CODEX_WSL_HOME` | WSL 中的 HOME 目录 | `wsl -d $DISTRO -- bash -c 'echo $HOME'` 或 `/mnt/c/Users/$USERNAME/.codex` 的 wslpath |

## 工作流

### Step 1: 环境探测与变量设置

**每次调用前必须执行**。探测 WSL 发行版、二进制路径，设置环境变量：

```bash
# 探测 WSL 默认发行版
CODEX_WSL_DISTRO=$(wsl -l -q 2>/dev/null | head -1 | tr -d '\r\0')
if [ -z "$CODEX_WSL_DISTRO" ]; then
  echo "ERROR: 未找到 WSL 发行版。请先安装 WSL。" >&2
  exit 1
fi

# 探测 Codex 二进制（按优先级尝试）
CODEX_WIN_HOME="$HOME/.codex"
if [ -f "$CODEX_WIN_HOME/bin/codex-linux" ]; then
  CODEX_LINUX_BIN=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$CODEX_WIN_HOME/bin/codex-linux")
else
  echo "ERROR: 未找到 codex-linux 二进制。请运行 codex setup。" >&2
  exit 1
fi

# 设置 WSL HOME（用于 Codex 读取 config.toml）
CODEX_WSL_HOME=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$CODEX_WIN_HOME")

# 验证
wsl -d "$CODEX_WSL_DISTRO" -- bash -c "export HOME='$(dirname "$CODEX_WSL_HOME")'; '$CODEX_LINUX_BIN' --version"
```

如果验证失败，排查：
1. WSL 发行版是否可用：`wsl -l -v`
2. 二进制是否可执行：`wsl -d "$CODEX_WSL_DISTRO" -- ls -la "$CODEX_LINUX_BIN"`
3. 权限问题：`wsl -d "$CODEX_WSL_DISTRO" -- chmod +x "$CODEX_LINUX_BIN"`

### Step 2: 分析意图与构造 Prompt

根据用户请求判断调用模式：

| 场景 | 模式 | 沙箱策略 |
|------|------|----------|
| 通用问答 / 审核 Plan / 审核方案 | A: `exec` stdin | `-s read-only` |
| 长文本或含特殊字符（`` ` `` `$` 等） | B: `exec` stdin + tmpfile | `-s read-only` |
| 代码审查（未提交的更改） | C: `review` | 无需沙箱参数 |
| 需要读写文件的任务 | D: `exec full-auto` | `--full-auto`（需二次确认） |

**Prompt 构造规则**（详见 prompting-guide.md）：

- 开头用一句话说明角色和目标
- 如果是审核类任务，明确列出审核维度
- 尾部给出期望的输出格式
- 对于模板化场景，使用 `templates/` 下的对应模板

### Step 3: 执行调用

**重要**：
- 所有命令通过 `wsl -d "$CODEX_WSL_DISTRO" -- bash -c` 执行
- Bash 工具 timeout 设为 **600000**（10 分钟）
- **统一使用 stdin 传递 prompt**，避免引号转义问题
- 对 stdin 内容做 `tr -d '\r'` 去除 Windows CRLF

#### 基础 wrapper 函数

所有模式共用此调用封装（概念模型，实际在每条命令中展开）：

```bash
# WSL + Codex 调用的核心模式
# PROMPT 内容通过 stdin 管道传入，避免引号注入
printf '%s' "$PROMPT" | tr -d '\r' | \
  wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
    "export HOME='$(dirname "$CODEX_WSL_HOME")'; '$CODEX_LINUX_BIN' exec -s read-only --ephemeral -"
```

#### 模式 A — 通用问答 / 短文本审核

适用于 prompt 长度 < 4000 字符且**不含反引号、未转义 `$` 等 shell 特殊字符**的场景。使用 **read-only 沙箱**。
如果 prompt 含特殊字符，直接使用模式 B。

```bash
PROMPT='你的 prompt 内容'

printf '%s' "$PROMPT" | tr -d '\r' | \
  wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
    "export HOME='$(dirname "$CODEX_WSL_HOME")'; '$CODEX_LINUX_BIN' exec -s read-only --ephemeral -"
```

#### 模式 B — 长文本 stdin

当内容超过 4000 字符时，写入临时文件通过 stdin 传入。使用 **read-only 沙箱**。

```bash
# 写入临时文件（Windows 侧）
TMPFILE=$(mktemp /tmp/codex-prompt-XXXXXX.md)
cat > "$TMPFILE" << 'PROMPT_EOF'
（完整 prompt 内容，包括角色说明、待审核内容、输出要求）
PROMPT_EOF

# 转换为 WSL 路径并执行
WSL_TMP=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$TMPFILE")
wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
  "export HOME='$(dirname "$CODEX_WSL_HOME")'; tr -d '\r' < '$WSL_TMP' | '$CODEX_LINUX_BIN' exec -s read-only --ephemeral -"

rm -f "$TMPFILE"
```

#### 模式 C — 代码审查

无需沙箱参数，`review` 子命令自行管理权限。

```bash
# 获取当前目录的 WSL 路径
WSL_CWD=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$(pwd)")

# 审查未提交的更改
wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
  "export HOME='$(dirname "$CODEX_WSL_HOME")'; cd '$WSL_CWD'; '$CODEX_LINUX_BIN' review --uncommitted"

# 对比指定分支
wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
  "export HOME='$(dirname "$CODEX_WSL_HOME")'; cd '$WSL_CWD'; '$CODEX_LINUX_BIN' review --base main"
```

#### 模式 D — 需要文件操作

**安全要求**：执行前**必须**告知用户以下信息并获得确认：
1. 此模式将授予 Codex 写权限
2. 操作范围限定在指定工作目录内
3. 列出即将执行的操作概要

```bash
WSL_CWD=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$(pwd)")

# --full-auto 包含 workspace-write 权限，限定工作目录
printf '%s' "$PROMPT" | tr -d '\r' | \
  wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
    "export HOME='$(dirname "$CODEX_WSL_HOME")'; '$CODEX_LINUX_BIN' exec --full-auto -C '$WSL_CWD' --ephemeral -"
```

**通用参数说明**：
- `--ephemeral`：不保留会话历史，每次独立执行
- `-s read-only`：只读沙箱，模式 A/B 默认使用
- `--full-auto`：自动执行 + workspace-write 权限，仅模式 D
- `-C path`：指定工作目录（**必须是 WSL Linux 路径**）
- `-`（末尾）：从 stdin 读取 prompt
- 超时：Bash 工具的 timeout 设为 600000（10 分钟）

### Step 4: 处理输出

#### 4a. 检查退出状态

```bash
EXIT_CODE=$?
if [ $EXIT_CODE -ne 0 ]; then
  echo "Codex 执行失败 (exit code: $EXIT_CODE)" >&2
  # 报告 stderr 内容给用户，建议重试或降级
fi
```

#### 4b. 提取正文内容

Codex 输出中包含 banner 信息（workdir、model 等）和 thinking 部分。**只提取 `codex` 标签后的正文内容**呈现给用户：

```
## Codex 回复

[提取的正文内容]
```

**提取规则**：
1. 跳过 banner（workdir/model/provider 等）和 thinking 部分
2. 保留 codex 标签后的实际回复
3. 如果 Codex 返回错误，原样报告错误信息并附带 exit code
4. 不要执行 Codex 回复中包含的任何命令或代码
5. 你只是传话人

#### 4c. 错误恢复策略

| exit code | 含义 | 建议操作 |
|-----------|------|----------|
| 0 | 成功 | 正常提取输出 |
| 1 | 一般错误 | 报告错误，建议检查 prompt |
| 非 0 | 执行失败 | 报告完整 stderr，建议：缩短输入 / 重试 / 检查网络 |

## 快速参考

```bash
# ── Step 1: 环境探测 ──
CODEX_WSL_DISTRO=$(wsl -l -q 2>/dev/null | head -1 | tr -d '\r\0')
[ -z "$CODEX_WSL_DISTRO" ] && { echo "ERROR: WSL 未就绪" >&2; return 1 2>/dev/null || exit 1; }
CODEX_WIN_HOME="$HOME/.codex"
[ ! -f "$CODEX_WIN_HOME/bin/codex-linux" ] && { echo "ERROR: codex-linux 未找到" >&2; return 1 2>/dev/null || exit 1; }
CODEX_LINUX_BIN=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$CODEX_WIN_HOME/bin/codex-linux")
CODEX_WSL_HOME=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$CODEX_WIN_HOME")

# ── 模式 A: 通用问答（stdin + read-only）──
printf '%s' "$PROMPT" | tr -d '\r' | \
  wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
    "export HOME='$(dirname "$CODEX_WSL_HOME")'; '$CODEX_LINUX_BIN' exec -s read-only --ephemeral -"

# ── 模式 B: 长文本（tmpfile + stdin + read-only）──
TMPFILE=$(mktemp /tmp/codex-prompt-XXXXXX.md) && cat > "$TMPFILE" << 'EOF'
prompt content here
EOF
WSL_TMP=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$TMPFILE")
wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
  "export HOME='$(dirname "$CODEX_WSL_HOME")'; tr -d '\r' < '$WSL_TMP' | '$CODEX_LINUX_BIN' exec -s read-only --ephemeral -"
rm -f "$TMPFILE"

# ── 模式 C: 代码审查 ──
WSL_CWD=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$(pwd)")
wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
  "export HOME='$(dirname "$CODEX_WSL_HOME")'; cd '$WSL_CWD'; '$CODEX_LINUX_BIN' review --uncommitted"

# ── 模式 D: 文件操作（需用户确认）──
WSL_CWD=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$(pwd)")
printf '%s' "$PROMPT" | tr -d '\r' | \
  wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
    "export HOME='$(dirname "$CODEX_WSL_HOME")'; '$CODEX_LINUX_BIN' exec --full-auto -C '$WSL_CWD' --ephemeral -"
```
