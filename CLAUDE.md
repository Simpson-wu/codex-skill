# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目定位

这是一个 Claude Code Skill，封装 OpenAI Codex CLI，让 Claude Code 用户在工作流中调用 Codex 获取独立第二意见。典型场景：

- 审核 Claude Code 生成的 Plan（实施方案）
- 审核测试大纲 / 测试方案
- 根据用户要求制定新的 Plan
- 通用任务：让 Codex 作为独立 agent 完成指定工作

## Codex CLI 环境

已安装：`codex-cli 0.106.0`（npm 全局安装）

配置文件 `~/.codex/config.toml`：
- 模型提供商：用户自行配置（支持 OpenAI 官方或兼容 endpoint）
- 默认模型：`gpt-5`，reasoning effort = high
- 认证：`codex login` 管理，或在 config.toml 中配置 API key

### 核心命令

```bash
# 非交互执行（skill 主要使用此模式）
codex exec "你的指令"
codex exec -m gpt-5 "审核以下方案..."

# 代码审查
codex review --uncommitted          # 审查未提交的更改
codex review --base main            # 对比 main 分支
codex review "自定义审查指令"

# 输出控制
codex exec --json "指令"                    # JSONL 输出
codex exec -o output.md "指令"              # 结果写入文件
codex exec --output-schema schema.json "指令"  # 结构化输出

# 沙箱策略
codex exec -s read-only "指令"              # 只读（最安全）
codex exec -s workspace-write "指令"        # 可写工作区
codex exec --full-auto "指令"               # 自动执行 + workspace-write

# 工作目录
codex exec -C /path/to/project "指令"
```

## Skill 文件格式

遵循 Claude Code Skill 规范，核心文件为 `SKILL.md`：

```markdown
---
name: skill-name
description: 一句话描述，决定 Skill 何时被触发
---

# Skill 标题

（Skill 的完整指令内容）
```

## 架构设计要点

1. **Skill → Codex CLI 的桥接**：Skill 在 SKILL.md 中定义指令模板，通过 Bash 工具调用 `codex exec` 执行
2. **上下文传递**：将需要审核的内容（Plan、测试大纲等）通过 stdin 管道或临时文件传给 Codex
3. **结果回收**：通过 `-o` 参数或 stdout 捕获 Codex 输出，带回 Claude Code 对话

## 父项目

本项目位于 `claude-code-config/正在研发/skills/codex-skill/`，父仓库是 Claude Code 配置包，包含全局 settings、commands、其他 skills。Skill 开发完成后需安装到 Claude Code 的 plugin 系统中使用。
