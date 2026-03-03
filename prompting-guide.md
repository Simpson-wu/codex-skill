# Codex Prompt 构造指南

## 基本结构

一个好的 Codex prompt 由三部分组成：

```
[角色与目标] — 一句话说明你要 Codex 做什么
[上下文与内容] — 需要审核/分析的具体内容
[输出要求] — 期望的回复格式和重点
```

## 审核类 Prompt 模板

审核类任务应明确列出审核维度，例如：

```
你是一位资深技术架构师。请审核以下实施方案，从以下维度给出意见：

1. 技术可行性 — 方案是否可实现，有无技术障碍
2. 完整性 — 是否有遗漏的环节或边界情况
3. 风险点 — 潜在的失败模式和风险
4. 改进建议 — 具体可操作的优化方向

---

（方案内容）

---

请按上述维度逐项分析，最后给出总体评价。
```

更多模板化场景请参考 `templates/` 目录下的对应模板（review-plan.md、review-test.md、code-review.md）。

## 长文本处理

当内容超过 4000 字符，或包含 shell 特殊字符（反引号 `` ` ``、`$`、单引号等）时：

1. 将完整 prompt（包括角色说明和输出要求）写入临时文件
2. 通过 WSL + stdin 传入 Codex
3. 清理临时文件

```bash
# 写入临时文件
TMPFILE=$(mktemp /tmp/codex-prompt-XXXXXX.md)
cat > "$TMPFILE" << 'PROMPT_EOF'
你是...请审核以下内容...

（长文本内容）

请按以下格式输出...
PROMPT_EOF

# 通过 WSL 执行（需要先完成 SKILL.md Step 1 环境探测）
WSL_TMP=$(wsl -d "$CODEX_WSL_DISTRO" -- wslpath -u "$TMPFILE")
wsl -d "$CODEX_WSL_DISTRO" -- bash -c \
  "export HOME='$(dirname "$CODEX_WSL_HOME")'; tr -d '\r' < '$WSL_TMP' | '$CODEX_LINUX_BIN' exec -s read-only --ephemeral -"

rm -f "$TMPFILE"
```

## 注意事项

- **WSL 必需**：Windows 原生 Codex 二进制存在已知崩溃（GitHub #12962），所有命令必须通过 WSL 执行。调用前需完成 SKILL.md Step 1 环境探测，设置 `CODEX_WSL_DISTRO`、`CODEX_LINUX_BIN`、`CODEX_WSL_HOME` 三个变量。
- Codex CLI 的 `exec` 模式默认 model 是 gpt-5（在 ~/.codex/config.toml 中配置）
- 不需要手动指定 `-m gpt-5`，除非要使用其他模型
- `--ephemeral` 确保每次调用独立，不受历史会话影响
- 避免在 prompt 中包含需要 Codex 执行的命令，审核类任务使用 `-s read-only` 沙箱
- **特殊字符处理**：当 prompt 包含反引号、`$`、单引号等 shell 特殊字符时，即使不足 4000 字符，也应使用临时文件 + stdin 方式（上述长文本处理方法），避免 shell 展开导致 prompt 被破坏
