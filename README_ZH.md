# opus-plan-codex-work — Claude Code 插件市场

> 一个托管 **opus-coding** 技能的 Claude Code **插件市场**。`opus-coding` 协议实现了四阶段多模型协作编码：由 Claude Opus 规划，GPT-5.4 Codex 并行执行，Claude Haiku 总结，以及 Opus/Codex 联合评审。

---

## 安装方法

**第一步：添加此市场**

```
/plugin marketplace add https://github.com/Kamisato520/opus-plan-codex-work
```

**第二步：安装插件**

```
/plugin install opus-coding
```

---

## 使用方法

安装完成后，在任何 Claude Code 会话中触发：

```
/opus-coding:opus-coding <任务描述>
```

或者直接说：**`opus编码`**、**`多模型编码`** 或 **`parallel coding`**

---

## 包含的插件

| 插件 | 描述 |
|---|---|
| `opus-coding` | 四阶段多模型编码协议：规划 → 并行 Codex → Haiku 总结 → 评审 |

---

## opus-coding 工作原理

```
阶段 1 — 规划 (Plan)    分析代码库 → 分解为 3-8 个子任务
阶段 2 — 编码 (Code)    并行启动 GPT-5.4 Codex 线程 (最多 8 个)
阶段 3 — 读取 (Read)    Claude Haiku 总结输出 → 写入文件到磁盘
阶段 4 — 评审 (Review)  Opus + Codex 继电器式评审，最多 3 轮迭代 → REVIEW_VERDICT.md
```

查看 [plugins/opus-coding/README_ZH.md](./plugins/opus-coding/README_ZH.md) 获取详细文档。

---

## 仓库结构

```
opus-plan-codex-work/
├── .claude-plugin/
│   └── marketplace.json          ← 市场清单文件 (Marketplace manifest)
├── plugins/
│   └── opus-coding/              ← 插件包本体
│       ├── .claude-plugin/
│       │   └── plugin.json       ← 插件清单文件
│       ├── skills/
│       │   └── opus-coding/
│       │       ├── SKILL.md      ← 完整执行协议 (机器可读)
│       │       └── README_ZH.md  ← 技能中文文档
│       └── README_ZH.md          ← 插件中文说明
└── README_ZH.md                  ← 本文件 (市场中文说明)
```

---

## 依赖要求

- Claude Code ≥ 1.0.33
- `mcp__codex__codex` MCP 工具 (需 GPT-5.4 Codex 访问权限)
- `mcp__codex__codex-reply` MCP 工具 (用于评审反馈)

## 许可证

MIT © [Kamisato520](https://github.com/Kamisato520)
