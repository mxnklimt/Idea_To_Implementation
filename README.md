# Idea → Implementation

一个 [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills)，用于在 **deep research 之后** 把调研得到的一组想法/思路，逐个转成**相互独立、最小改动、可单独验证**的代码实现。

> 一句话：**一个想法 = 一个文件夹 = 一个实现**，绝不合并，绝不改原件，**先计划再写代码**。

---

## 为什么需要这个 skill

做完深度调研，你通常会拿到一**串**候选想法。常见的错误是把它们揉进一个大改动——这样验证时：有效了分不清是哪个想法起作用，失效了分不清是哪个想法搞坏的。

本 skill 的核心纪律就是**隔离**：每个想法单独成文件夹、单独复制参考代码做最小修改，让你能逐个判断每个想法是否真的有效。为安全起见，参考源码**永不在原地被修改**，一律先复制副本再改。

更进一步，**任何代码都不能先于计划写出**：主代理先做一次集中的意图澄清，每个子代理在动手前必须先为自己的想法写一份 `PLAN.md`，再按计划执行——代码是对已写好计划的执行，不是边想边写。

## 四个不变量

1. **先计划再写代码** — 子代理在为自己的想法写出 `PLAN.md` 之前，不得碰任何实现代码。计划要写清：复制哪些文件、改哪些行、验证怎么做。代码是"执行已写好的计划"。若执行中发现计划错了，先改 `PLAN.md` 再继续，不能脱离计划。
2. **最小改动** — 每个想法用最小的 diff 来体现预期效果。目标是"效果可观测"，不是"功能完整"。一个想法有多种读法 → 拆成多个想法。
3. **一想法一文件夹** — 每个想法一个目录：内含 `PLAN.md`（计划，先于代码）、修改后的副本、一份 `APPROACH.md`（思路、改了什么、怎么验证）。想法之间互不串味。
4. **复制而非改原件** — 参考源码先复制再改，原件保持字节不变。

## 工作流

```
deep-research / brainstorming 产出想法列表
                │
                ▼
   ① 提取 & 归一化想法（拆分而非合并）→ 与用户确认
                │
                ▼
   ② brainstorming 澄清意图（主代理集中做一次，明确每个想法在测什么）
                │
                ▼
   ③ 建立工作区（每想法一个文件夹 + 根 README 索引）
                │
                ▼
   ④ dispatching-parallel-agents：每想法派一个子代理，并行执行
      复制参考文件 → 写 PLAN.md → TDD（若可测）→ 最小改动 → 写 APPROACH.md
      （代码永远晚于 PLAN.md 出现）
                │
                ▼
   ⑤ 校验：每文件夹同时有 PLAN.md + APPROACH.md；原件未变；代码与计划一致
                │
                ▼
   ⑥ 更新索引 → 交还用户逐个验证
```

### 产出目录结构示例

```
ideas-impl/
├── README.md                          # 索引：每个想法、其文件夹、状态
├── lr-warmup-on-loss-plateau/
│   ├── PLAN.md                        # 实现计划，先于代码写出
│   ├── APPROACH.md                    # 简要思路：是什么、为什么、看什么
│   └── <复制后做最小修改的源文件>
└── gradient-noise-injection/
    ├── PLAN.md
    ├── APPROACH.md
    └── ...
```

## 它如何编排 superpowers

| 环节 | 调用的 superpowers skill | 作用 |
|------|--------------------------|------|
| 意图澄清 | `superpowers:brainstorming` | 派 subagent 前，主代理集中做一次，明确每个想法的假设与"成功"标准 |
| 计划 | `superpowers:writing-plans` | 每个子代理在写代码前必须用它产出 `PLAN.md`——**强制，非可选** |
| 并行实现 | `superpowers:dispatching-parallel-agents` | 每个想法一个独立子代理，无共享状态 |
| 单想法实现 | `superpowers:test-driven-development` | 可测的想法先写失败测试（通常是计划的第一步），再最小改动 |
| 执行参考 | `superpowers:executing-plans` / `subagent-driven-development` | 子代理执行自己的 `PLAN.md` 时可参考 |

上游消费：`deep-research`、`superpowers:brainstorming` 的产出。

## 关于 brainstorming 放在哪里的设计取舍

`superpowers:brainstorming` 本质是参与式的（会反问用户）。如果每个想法各自独立跑 brainstorming，N 个想法就会变成 N 轮串行问答，毁掉并行性。因此本 skill 把 **brainstorming 留在主代理集中做一次**（澄清意图），把 **writing-plans 下沉到每个子代理内部并行做**（写各自计划）。这样既满足"先计划再执行"，又不丧失并行性。

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
