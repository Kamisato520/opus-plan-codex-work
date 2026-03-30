# opus-coding 插件详细介绍

`opus-coding` 是一个专为 Claude Code 打造的高级编码插件，它封装了 **Opus 多模型协作编码协议 (Opus Multi-Model Coding Protocol)**。

该插件的核心理念是将复杂的编码任务交由最擅长的模型处理：由逻辑推理能力最强的 **Claude Opus** 进行顶层设计，由生成效率最高的 **GPT-5.4 Codex** 进行大规模并发编码，再由轻量级的 **Claude Haiku** 进行信息归纳。

---

## 核心特性

1.  **并行任务调度**：自动将大型需求拆解为 1-8 个相互独立或存在依赖关系的子任务，并利用多线程技术同时调用 GPT-5.4 Codex 进行实现。
2.  **状态韧性 (Resilience)**：所有执行状态实时持久化到 `PIPELINE_STATE.json`。即使会话中断或达到上下文上限被压缩，也能在 48 小时内随时恢复进度。
3.  **四阶段严密流程**：
    *   **Phase 1 - 规划 (Opus)**：全量阅读上下文，定义子任务、依赖图和验证指令。
    *   **Phase 2 - 并发 (Codex)**：利用 `model_reasoning_effort: "xhigh"` 模式进行高质量的代码生成。
    *   **Phase 3 - 归纳 (Haiku)**：对海量生成的代码进行自动化审计和摘要，提取关键变更和风险点。
    *   **Phase 4 - 联审 (Relay Mode)**：模拟资深架构师评审，通过 Opus 与 Codex 的往复沟通对代码进行持续打磨，最高支持 3 轮迭代。
4.  **自动执行验证**：在交付前自动运行指定的 `trial_run_command`（如 pytest、npm test），并根据运行结果自动修复潜在 Bug。

---

## 触发关键词

插件加载后，你可以通过以下方式启动：
*   `/opus-coding:opus-coding [任务描述]`
*   或者在自然语言中包含：`opus编码`、`多模型编码`、`parallel coding`。

---

## 安装方法 (Marketplace 模式)

1.  添加市场源：
    ```
    /plugin marketplace add https://github.com/Kamisato520/opus-plan-codex-work
    ```
2.  安装插件：
    ```
    /plugin install opus-coding
    ```

---

## 文件结构

*   `SKILL.md`: 存放最核心的执行逻辑指令。
*   `CODE_SUMMARY.md`: (执行后生成) 包含全量代码变更的摘要。
*   `REVIEW_VERDICT.md`: (执行后生成) 包含最终的评审打分和测试结果。

---

## 技术背景

本插件起源于处理超大规模重构任务的需求。传统的单模型交互往往在处理超过 2000 行代码变更时会出现幻觉或逻辑断层，而 `opus-coding` 通过任务分解与并发执行，将这一瓶颈推向了 8000+ 行以上的工业级规模。

---

*由 [Kamisato520](https://github.com/Kamisato520) 维护。*
