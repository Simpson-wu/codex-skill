# Codex (GPT-5) 审核报告 — SKILL.md v1

> 审核日期：2026-03-02
> 审核模型：GPT-5 via Codex CLI 0.106.0
> 被审核文件：`~/.claude/skills/ask-codex/SKILL.md`（初始版本）

---

## 总体评价

推荐等级：**3/5**。总体思路清晰，但路径硬编码与默认跳过审批带来较大可移植性与安全风险，命令与输出解析也需要更稳健的落地细化。

## 逐项分析

### 架构合理性

- **优点**：定位为"Claude Code <-> Codex CLI"桥接层明确；四种模式（exec/exec stdin/review/exec full-auto）体现了常见用法分层，职责边界基本清楚（意图识别 -> 命令拼装 -> 执行 -> 结果提取）。
- **不足**：桥接层过多耦合运行时环境细节（WSL、HOME 重写、绝对路径），未抽象出"环境探测/路径解析/调用策略"的独立层。缺少对工作目录、路径转换与输出格式的可插拔策略设计。

### 完整性

- **优点**：含环境检查、意图分析、执行、格式化输出四步，覆盖主流程。
- **缺口**：
  - 未覆盖模型/提供商/推理强度等参数的显式注入与覆盖优先级；也未描述多轮上下文/会话粘性（即便使用 `--ephemeral`，也应明确何时需要非临时会话）。
  - 未覆盖跨平台工作目录转换（Windows -> WSL `wslpath`）与 CRLF -> LF 的 stdin 规范化，未描述大输入的分块/压缩策略。
  - 输出只基于"codex 标签"文本裁剪，缺少结构化输出（JSON/JSONL）与错误通道（stderr、非零 exit code）处理。

### 可用性

- **优点**：统一 WSL wrapper 思路简单直接；`exec stdin` 适合长文本，方向正确。
- **风险点**：
  - 当前 wrapper 以绝对路径和 `HOME` 指向 `/mnt/c/...`，在 WSL2 上性能与权限均不理想；且未绑定具体发行版（缺 `-d`），多发行版环境下不稳定。
  - 命令拼装对引号/转义脆弱，含空格/引号的 prompt 或路径可能注入或执行失败；`-C path` 需要 Linux 路径而非 Windows 路径，未见 `wslpath -u` 转换。
  - `exec stdin` 没有声明换行与编码处理（CRLF 剥离、UTF-8 保证）。

### 安全性

- **主要问题**：将 `--dangerously-bypass-approvals-and-sandbox` 设为"通用参数"默认开启，且存在 `exec full-auto` 文件操作模式，组合后风险高（越权写文件/执行命令）。
- 未见最小权限策略（受限工作目录/只读模式/显式 allowlist）与交互式确认（特别是 destructive 操作）。缺乏脱敏/日志审计策略。

### 可维护性

- **主要问题**：硬编码用户名与绝对路径（`/mnt/c/Users/<username>/...`）导致不可移植；`HOME` 强行指向 Windows 盘也会让后续维护脆弱。
- 建议以环境变量与探测流程（发行版、codex 二进制位置、工作目录映射）替代硬编码；提供本地化 wrapper 脚本与集中配置，减少分散修改点。

## 关键问题（TOP 3）

1. **安全风险** — 默认使用 `--dangerously-bypass-approvals-and-sandbox`，与"full-auto"文件操作结合，存在高破坏性风险且无确认/范围限制。
2. **硬编码路径** — 硬编码 Windows 用户路径与未做 `wslpath` 转换，导致跨机器不可用，且在多发行版/不同用户名/不同安装位置下易失败。
3. **输出解析脆弱** — 输出解析基于文本裁剪而非结构化通道，异常与边界情况（横幅变化、流式思考、错误信息）容易误判或丢失。

## 改进建议

### 环境与路径抽象

- 引入配置与探测：`CODEX_BIN`, `CODEX_HOME`, `CODEX_WSL_DISTRO`, `CODEX_WORKDIR` 等环境变量；启动时探测默认发行版（`wsl -l -q`）、二进制路径（`command -v codex` 或预设候选列表）。
- 统一路径转换：对传入工作目录一律经 `wslpath -u` 转换；避免把 `HOME` 指到 `/mnt/c`，优先把 Linux 版 codex 安装在 WSL 的 `$HOME`（如 `/home/<user>/.codex/bin/codex`），仅对工作目录做映射。
- 提供稳定 wrapper 示例：
  - `wsl -d "${CODEX_WSL_DISTRO:-Ubuntu}" -- bash -lc '"$CODEX_BIN" exec "$@"' -- <args>`
  - 传参统一走 stdin 以降低引号与注入风险；短 prompt 也优先 stdin。

### 安全默认与操作分级

- 取消全局默认 `--dangerously-bypass-approvals-and-sandbox`；仅在显式"full-auto"模式、且用户二次确认后启用。
- 只读默认，写入需显式确认。
- 支持受限工作目录。

### 命令与输入稳健性

- 统一对 stdin 做 `tr -d '\r'` 去 CRLF。
- `-C` 仅接收 Linux 路径；Windows 侧路径务必先 `wslpath -u`。

### 输出结构化与错误处理

- 若 Codex CLI 支持，开启结构化输出（如 `--json`）。
- 处理非零退出码与 stderr，把失败信息与重试策略返回给上层。

### 便携与维护

- 去除用户名硬编码；提供首启向导或一键安装脚本。
- 在仓库提供最小 README/故障排查。

---

> 此报告由 Codex (GPT-5) 自动生成，用于指导 SKILL.md v2 的改进。
