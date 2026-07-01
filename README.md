# Idea → Implementation

一个 [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills)，用于在 **deep research 之后** 把调研得到的一组想法/思路，逐个转成**相互独立、最小改动、可单独验证**的代码实现。

> 一句话：**一个想法 = 一个文件夹 = 一个实现**，绝不合并，绝不改原件。

---

## 为什么需要这个 skill

做完深度调研，你通常会拿到一**串**候选想法。常见的错误是把它们揉进一个大改动——这样验证时：有效了分不清是哪个想法起作用，失效了分不清是哪个想法搞坏的。

本 skill 的核心纪律就是**隔离**：每个想法单独成文件夹、单独复制参考代码做最小修改，让你能逐个判断每个想法是否真的有效。为安全起见，参考源码**永不在原地被修改**，一律先复制副本再改。

## 三个不变量

1. **最小改动** — 每个想法用最小的 diff 来体现预期效果。目标是"效果可观测"，不是"功能完整"。一个想法有多种读法 → 拆成多个想法。
2. **一想法一文件夹** — 每个想法一个目录：内含修改后的副本 + 一份 `APPROACH.md`（思路、改了什么、怎么验证）。想法之间互不串味。
3. **复制而非改原件** — 参考源码先复制再改，原件保持字节不变。

## 工作流

```
deep-research / brainstorming 产出想法列表
                │
                ▼
   ① 提取 & 归一化想法（拆分而非合并）→ 与用户确认
                │
                ▼
   ② 建立工作区（每想法一个文件夹 + 根 README 索引）
                │
                ▼
   ③ dispatching-parallel-agents：每想法派一个子代理
      （复制参考文件 → 最小改动 → TDD（若可测）→ 写 APPROACH.md）
                │
                ▼
   ④ 校验原件未变、各文件夹自洽 → 更新索引 → 交还用户逐个验证
```

### 产出目录结构示例

```
ideas-impl/
├── README.md                          # 索引：每个想法、其文件夹、状态
├── lr-warmup-on-loss-plateau/
│   ├── APPROACH.md                    # 简要思路：是什么、为什么、看什么
│   └── <复制后做最小修改的源文件>
└── gradient-noise-injection/
    ├── APPROACH.md
    └── ...
```

## 它如何编排 superpowers

| 环节 | 调用的 superpowers skill | 作用 |
|------|--------------------------|------|
| 并行实现 | `superpowers:dispatching-parallel-agents` | 每个想法一个独立子代理，无共享状态 |
| 单想法实现 | `superpowers:test-driven-development` | 可测的想法先写失败测试，再最小改动 |
| 复杂想法（少见） | `superpowers:writing-plans` | 仅当某想法复杂到需规划时，子代理可参考 |

上游消费：`deep-research`、`superpowers:brainstorming` 的产出。

## 安装

把 `idea-to-implementation/` 文件夹放到 Claude Code 的 skills 目录：

- **Windows**: `C:\Users\<你>\.claude\skills\`
- **macOS / Linux**: `~/.claude/skills/`

安装后重启 Claude Code，skill 会自动出现在可用列表中。

## 触发方式

skill 会在以下场景被触发（也可手动 `/idea-to-implementation` 调用）：

- 紧跟在 `deep-research` 之后，用户说"把这些想法实现出来""逐个验证这些思路""每个想法给我一个独立实现"
- 任何 research / brainstorming 产出包含多个**应当分开测试**的想法时

## 仓库结构

```
Idea_To_Implementation/
├── README.md                          # 本文件
└── idea-to-implementation/
    └── SKILL.md                       # skill 本体
```

## 适用与不适用

**适用**：调研后得到多个待验证的改进思路，想低成本逐个试错——典型场景如 ML 训练超参/技巧、prompt 优化、性能优化方案对比。

**不适用**：单一明确的功能需求（直接开发即可）、需要合并多个改动才能评估的场景。

## 许可

仅供个人 / 团队 Claude Code 工作流使用。
