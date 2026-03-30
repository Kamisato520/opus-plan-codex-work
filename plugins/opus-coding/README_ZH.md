# opus-coding 插件

> 🇬🇧 [English Documentation](./README.md)

`opus-coding` 实现了 **Opus 多模型协作编码协议**——一套四阶段工作流，将编码过程的每个环节都交由最擅长的模型来处理。

---

## 设计理念

单模型会话处理大型任务时面临两大瓶颈：**上下文耗尽**和**逻辑断层**。`opus-coding` 的解法是：

1. 由 **Claude Opus**（逻辑推理最强）负责全局架构规划
2. 将实现工作分发给**最多 8 个并行的 GPT-5.4 Codex 线程**
3. 用轻量级的 **Claude Haiku** 汇总输出，大幅压缩上下文占用
4. 通过 Opus 与 Codex 的**继电器循环评审**确保代码质量

最终效果：单次会话可处理 **8,000+ 行**的工业级重构任务，并支持自动断点续跑。

---

## 四阶段协议详解

### 阶段一 — 规划 (Plan) · Claude Opus 执行

- 使用 `Glob` + `Read` 全量扫描代码库上下文
- 将任务分解为 **3–8 个子任务**，同时建立明确的依赖关系图
- 确定集成验证指令 `trial_run_command`（如 `pytest`、`npm test`）
- 将全部规划写入 `PIPELINE_STATE.json` 以支持崩溃恢复

**分解原则**：优先选择数量少、粒度大的子任务而非数量多、粒度小的。没有相互依赖关系的子任务将在下一阶段并发执行。

### 阶段二 — 编码 (Code) · GPT-5.4 Codex 并行执行

- 在**同一次响应**中并发调用 `mcp__codex__codex`，处理所有无依赖的子任务
- 所有调用必须使用 `config: { "model_reasoning_effort": "xhigh" }`，无一例外
- 最多同时运行 **8 个** Codex 线程
- 每次调用后立即将 `threadId` 存入 `PIPELINE_STATE.json`

独立子任务完成后，依赖它们的子任务随即被解锁并启动。

### 阶段三 — 归纳 (Read) · Claude Haiku 执行

对每个完成的 Codex 线程，启动一个 `claude-haiku-4-5-20251001` 子代理，提取：
- 创建/修改的文件列表
- 公开函数和类的签名
- 错误信息、异常和 TODO 注释
- 关键代码片段（5–15 行核心逻辑）
- 供下一阶段使用的集成注意事项

所有提取内容汇编到 `CODE_SUMMARY.md`。随后，Codex 输出的代码通过 `Write` / `Edit` 写入磁盘。

### 阶段四 — 评审 (Review) · Opus + Codex 继电器模式

由于子代理无法直接调用 MCP 工具，Opus 充当中继器，将代码和摘要转发给 GPT-5.4 Codex 进行评审，应用修复后再次提交。

评审循环流程：
1. Codex 对实现代码打分 **1–10 分**，输出 `APPROVED` 或 `NEEDS_FIXES`
2. Opus 使用 `Write` / `Edit` 应用所有关键修复项
3. Opus 调用 `mcp__codex__codex-reply` 提交更新后的代码进行再评估
4. 最多循环 **3 轮**（`MAX_REVIEW_ROUNDS`）
5. 运行 `trial_run_command` 并捕获输出
6. 将最终结果写入 `REVIEW_VERDICT.md`

---

## 崩溃恢复机制

| `PIPELINE_STATE.json` 状态 | 行为 |
|---|---|
| 文件不存在 | 全新开始 |
| `status: "completed"` | 全新开始 |
| `status: "in_progress"` 且时间戳超过 48 小时 | 全新开始（视为陈旧） |
| `status: "in_progress"` 且时间戳在 48 小时内 | **从保存的阶段续跑** |

---

## 生成的输出文件

| 文件 | 说明 |
|---|---|
| `PIPELINE_STATE.json` | 实时状态文件，支持崩溃恢复 |
| `CODE_SUMMARY.md` | Haiku 提取的代码变更全量摘要 |
| `REVIEW_VERDICT.md` | 最终评审分数、结论及试运行结果 |

---

## 关键常量

| 常量 | 值 | 说明 |
|---|---|---|
| `MAX_CODEX_THREADS` | 8 | 最大并行实现线程数 |
| `MAX_REVIEW_ROUNDS` | 3 | 最大评审/修复迭代轮数 |
| `REASONING_EFFORT` | `"xhigh"` | 适用于所有 Codex 调用 |

---

## 必要工具

| 工具 | 用途 |
|---|---|
| `Bash(*)`, `Read`, `Write`, `Edit`, `Grep`, `Glob` | 标准文件和 Shell 操作 |
| `Agent` | 启动 Claude Haiku 子代理进行摘要提取 |
| `mcp__codex__codex` | 发起新的 GPT-5.4 Codex 线程 |
| `mcp__codex__codex-reply` | 继续现有的 Codex 线程（用于继电器评审） |

---

## 核心约束

- 🔴 **始终使用 `model_reasoning_effort: "xhigh"`** — 所有 Codex 调用强制要求，无一例外
- 🔴 **立即保存所有 threadId** — 丢失 threadId 将导致继电器评审和崩溃恢复双双失效
- 🟡 **并发调用写在同一次响应中** — 无相互依赖的子任务必须同时启动，不得顺序执行
- 🟡 **Haiku 用于読取，不用于编码** — Haiku 子代理只做摘要，代码写入统一走 Codex 或 Write/Edit
- 🟢 **如实报告错误** — 所有失败、错误和 TODO 都必须载入 CODE_SUMMARY.md 和 REVIEW_VERDICT.md
- 🟢 **严守范围纪律** — 不得私自修复范围外的 Bug；将其记录在 ERRORS/TODOS 供用户后续处理

---

## 示例场景

```
用户：opus编码 — 为现有的 FastAPI 服务添加 JWT 认证、权限中间件和单元测试
```

协议将会：
1. 读取现有服务结构，识别与认证相关的文件
2. 分解为：`[jwt_module]`、`[middleware]`（依赖 jwt_module）、`[test_suite]`
3. 并发启动 `jwt_module` 和 `test_suite` 的 Codex 线程；完成后启动 `middleware`
4. Haiku 汇总每个输出；代码写入磁盘
5. Codex 对合并后的实现进行评审；Opus 应用修复（最多 3 轮）
6. 运行 `pytest tests/` → 捕获输出 → 写入 `REVIEW_VERDICT.md`

---

## 触发方式

加载插件后，通过以下任意方式启动：

```
/opus-coding:opus-coding <任务描述>
```

或在消息中自然包含：`opus编码` · `多模型编码` · `parallel coding`

---

## 技术背景

本插件起源于处理超大规模重构任务的工程需求。传统的单模型交互在代码变更量超过约 2000 行时容易出现幻觉或逻辑断层，而 `opus-coding` 通过任务分解与并发执行，将这一瓶颈推向了 8000+ 行以上的工业级规模。

---

## 文件结构

```
opus-coding/
├── SKILL.md          ← 完整执行协议（机器可读）
├── README.md         ← 英文文档
└── README_ZH.md      ← 本文件（中文文档）
```

---

*隶属于 [opus-plan-codex-work](https://github.com/Kamisato520/opus-plan-codex-work) 插件市场。*
