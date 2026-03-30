# opus-plan-codex-work — Claude Code 插件市场

> 🇬🇧 [English Documentation](./README.md)

这是一个 Claude Code **插件市场 (Plugin Marketplace)**，托管了 **opus-coding** 插件。该插件实现了一套四阶段多模型协作编码协议：以 Claude Opus 负责规划，GPT-5.4 Codex 并行执行，Claude Haiku 汇总摘要，最终由 Opus 与 Codex 共同完成联合评审。

---

## 快速安装

**第一步 — 在 Claude Code 中注册此插件市场：**

```
/plugin marketplace add https://github.com/Kamisato520/opus-plan-codex-work
```

**第二步 — 安装插件：**

```
/plugin install opus-coding
```

---

## 使用方法

安装完成后，在任何 Claude Code 会话中执行：

```
/opus-coding:opus-coding <你的任务描述>
```

也可以通过自然语言触发：**`opus编码`** · **`多模型编码`** · **`parallel coding`**

---

## 为什么选择 opus-coding？

传统的单模型对话在处理超过约 2000 行的代码变更时容易产生幻觉或上下文断层。`opus-coding` 通过任务分解与并发执行突破了这一瓶颈，让单次会话轻松驾驭 **8000+ 行**的工业级重构任务。

---

## 四阶段协议概览

| 阶段 | 执行模型 | 主要工作 |
|---|---|---|
| **1 — 规划 (Plan)** | Claude Opus | 读取代码库，将任务分解为 3–8 个子任务并建立依赖图 |
| **2 — 编码 (Code)** | GPT-5.4 Codex（最多×8） | 并行实现子任务，始终使用 `reasoning_effort: xhigh` |
| **3 — 归纳 (Read)** | Claude Haiku | 汇总所有输出 → 将代码写入磁盘 → 生成 `CODE_SUMMARY.md` |
| **4 — 评审 (Review)** | Opus + Codex 继电器 | 模拟资深工程师评审循环，最多 3 轮迭代 → 生成 `REVIEW_VERDICT.md` |

查看 [plugins/opus-coding/README_ZH.md](./plugins/opus-coding/README_ZH.md) 获取完整技术说明。

---

## 崩溃恢复

每个阶段结束后，全部状态都会持久化到 `PIPELINE_STATE.json`。如果会话意外中断或触发上下文压缩，协议将在 48 小时内自动从断点续跑，无需手动干预。

---

## 仓库结构

```
opus-plan-codex-work/
├── .claude-plugin/
│   └── marketplace.json              ← 市场清单文件
├── plugins/
│   └── opus-coding/                  ← 插件包本体
│       ├── .claude-plugin/
│       │   └── plugin.json           ← 插件清单文件
│       ├── skills/
│       │   └── opus-coding/
│       │       ├── SKILL.md          ← 完整执行协议（机器可读）
│       │       └── README.md         ← 技能参考文档
│       ├── README.md                 ← 插件完整文档（英文）
│       └── README_ZH.md              ← 插件文档（中文）
├── README.md                         ← 英文说明
└── README_ZH.md                      ← 本文件（中文说明）
```

---

## 依赖要求

- Claude Code ≥ 1.0.33
- `mcp__codex__codex` MCP 工具（需 GPT-5.4 Codex 访问权限）
- `mcp__codex__codex-reply` MCP 工具（用于评审继电器）

## 许可证

MIT © [Kamisato520](https://github.com/Kamisato520)
