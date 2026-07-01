---
name: idea-to-implementation
description: Use AFTER a deep-research session has produced a set of ideas/hypotheses/approaches (a list of candidate directions, "思路", "想法", "改进方案", or a research report with multiple recommendations). This skill turns each idea into its OWN isolated, minimal-change code implementation by copying reference source files as editable copies and dispatching one parallel subagent per idea — so the user can independently verify whether each idea actually has an effect. Each idea lands in its own folder with a brief markdown rationale. CRITICALLY, no code is written until each idea has been planned first: the main agent runs a short brainstorming pass to clarify intent, then each dispatched subagent MUST write a PLAN.md (following superpowers:writing-plans) BEFORE touching any code, then executes that plan. Use this skill whenever the user says things like "把这些想法实现出来", "逐个验证这些思路", "针对 deep research 的结论做实验", "每个想法给我一个独立实现", or whenever a research/brainstorming output contains multiple distinct ideas that should be tested separately rather than merged into one change. Do NOT merge ideas — one idea = one folder = one implementation. Plan first, execute second — always.
---

# Idea → Implementation

This skill bridges **research output** (a set of ideas) to **isolated, individually-verifiable code implementations**. It is the natural next step after `deep-research` (or any brainstorming/research that yields multiple candidate directions).

## Why this skill exists

After deep research, the user typically gets a *list* of ideas/hypotheses. The temptation is to combine them into one big change. That's exactly wrong for validation: if a combined change "works," you can't tell which idea did it; if it fails, you can't tell which idea broke it. 

The core discipline of this skill is **one idea = one folder = one minimal-change implementation**, so each idea can be judged on its own merits. To make that safe, reference source files are **never edited in place** — they are copied, and the copy is what gets modified.

## The four invariants (never violate these)

1. **Plan before code.** No subagent writes a single line of implementation until it has produced a `PLAN.md` for its idea (following `superpowers:writing-plans`). The plan names the exact files to copy, the precise lines to change, and the verification step. Code is the execution of an already-written plan — never a simultaneous act. A subagent that starts editing before its plan exists has broken the discipline.
2. **Minimal change.** Each idea is implemented with the smallest diff that can plausibly produce the hypothesized effect. The goal is *observability of effect*, not completeness. If an idea has multiple sub-approaches, that's multiple ideas — give each its own folder.
3. **One folder per idea.** Every idea gets its own directory containing: a `PLAN.md` (the plan, written before code), the modified copy/copies, and a brief `APPROACH.md` describing the idea and what it changes. Nothing from idea A bleeds into idea B's folder.
4. **Copy, never overwrite the original.** Reference source files are copied before any modification. The user's original files stay byte-identical. If you find yourself about to edit a file the user pointed at as reference, stop — copy first.

## Inputs

Read ideas from **both** sources, whichever the user provides:

- **Conversation context**: the most recent deep-research / brainstorming output. Look for a bulleted/numbered list of ideas, hypotheses, recommendations, or "思路"/"想法"/"approaches".
- **User-provided file path**: if the user names an `.md` (e.g., a research report) or points at a directory, parse it for the idea list.

The user will also indicate **reference source files** — the existing project code these ideas are meant to modify. This can be a path, a directory, or a list. If it's ambiguous which files are "reference," ask once, briefly, before proceeding. Do not guess and start copying the wrong tree.

## Workflow

### 1. Extract & normalize the idea list

Pull every distinct idea out of the input. De-duplicate near-identical ones, but **err toward splitting**, not merging — if an idea has two plausible readings, treat them as two ideas. For each idea, capture in one line:
- a short slug (kebab-case, e.g. `lr-warmup-on-loss-plateau`)
- what it claims to change / what effect it expects
- which reference file(s) it touches

Present this list to the user for confirmation before spawning anything. This is the first checkpoint — getting the idea set right matters more than speed.

### 2. Brainstorming pass (main agent, before any dispatch)

Before any code or plan is written, run a **short brainstorming pass** on the confirmed idea set, in the main agent, with the user. The point is to resolve ambiguity *once, up front*, so the parallel subagents don't each have to stop and ask questions (which would serialize them).

Invoke `superpowers:brainstorming` for this. Focus the brainstorm narrowly on what each idea is actually testing — the hypothesis, the expected direction of effect, and what "it worked" / "it didn't" looks like. Do NOT design full solutions here; that is the subagent's planning job. The output of this stage is a clarified one-paragraph-per-idea statement of intent that gets folded into each subagent's brief.

Keep this lightweight. If an idea's intent is already crystal-clear from the research output, don't manufacture questions for it — note "intent clear, no clarification needed" and move on. The brainstorming exists to serve verification clarity, not as ceremony.

If the user's research output already went through brainstorming, you may skip a redundant re-brainstorm — but only if the intent behind every idea is unambiguous. When in doubt, do the pass.

### 3. Lay out the workspace

Create a parent directory to hold all implementations (default: `ideas-impl/` next to the reference code, or wherever the user wants). Inside it, one folder per idea, named by slug:

```
ideas-impl/
├── README.md                # index: each idea, its folder, status
├── lr-warmup-on-loss-plateau/
│   ├── PLAN.md              # the plan, written BEFORE any code
│   ├── APPROACH.md          # brief rationale: what, why, what to look for
│   └── <copied source files with minimal edits>
├── gradient-noise-injection/
│   ├── PLAN.md
│   ├── APPROACH.md
│   └── ...
```

The `README.md` at the root is the map — list every idea, its folder, and a one-line status. The user uses this to pick what to verify.

### 4. Dispatch one parallel subagent per idea — each MUST plan before coding

This is the core mechanism. Use the **`dispatching-parallel-agents`** superpowers skill to fan out — one subagent per idea, running concurrently. Each subagent is self-contained: it gets exactly one idea, the clarified intent from the brainstorming pass, the list of reference files to copy, and the target folder. Subagents are independent by construction (separate folders, separate file copies), so there is no shared state and no ordering.

Invoke `superpowers:dispatching-parallel-agents` to set this up properly — it governs how to dispatch independent agents without shared state.

Each subagent's task brief must contain, in this order:

- **The idea + clarified intent** (the one-liner, the hypothesized effect, and the brainstorming-pass clarification of what "worked" looks like).
- **The reference files to copy** (absolute paths), and explicit instruction: *"Copy these into your folder FIRST. Modify only the copies. Never edit the originals."*
- **The target folder** (absolute path).
- **The plan-first mandate (non-negotiable)**: the subagent MUST invoke `superpowers:writing-plans` and write a `PLAN.md` into its folder BEFORE writing any implementation code. The plan must name: which copied files get changed, the precise change (down to the function/line/flag), and the verification step. The subagent executes the plan; it does not improvise alongside it. If the subagent finds the plan wrong mid-execution, it updates `PLAN.md` first, then continues.
- **The minimal-change mandate**: smallest diff that plausibly exhibits the effect. Resist scope creep. The plan should reflect this — a plan that grows beyond the idea's hypothesis is a signal to shrink the idea, not expand the code.
- **TDD where it applies**: if the idea is testable, the plan should include the failing test as the first step, and the subagent should follow `superpowers:test-driven-development` — write the failing test first, then the minimal change. For ideas that aren't unit-testable (e.g. training hyperparameters, prompts), the verification recipe in `APPROACH.md` substitutes.
- **Write `APPROACH.md`** in the folder: the idea in 2–4 sentences, exactly which lines/files were changed and why, and the verification recipe (command to run, metric to watch, expected direction).
- **Do not touch anything outside your folder.**

The sequence inside each subagent is therefore: **copy → plan (PLAN.md) → test/verify-design → minimal edit → APPROACH.md**. Code never precedes the plan.

### 5. Verify completion, then report back

When subagents finish, confirm (don't just trust the report):
- Each idea folder exists and contains both `PLAN.md` (written before code) and `APPROACH.md`.
- The original reference files are unchanged (a quick check — e.g. compare against what you copied; originals must be byte-identical).
- Each folder's edits are localized to the copied files inside it, and the edits match what `PLAN.md` said would change (a plan that doesn't match the code is a red flag — flag it to the user).

Then update the root `README.md` with status and hand control to the user with a short summary: which ideas got which folders, and the one command or step per idea needed to verify it.

## What "minimal change" really means

A useful framing: if the user runs idea X's folder and sees *no effect*, that should be informative — it should mean the idea itself didn't pan out, not that the implementation was too tangled to tell. So:

- Change the fewest lines that could plausibly produce the effect.
- Don't refactor "while you're in there." Refactors pollute the signal.
- Don't add conveniences, logging frameworks, or abstractions the idea didn't ask for.
- If the idea needs a config flag, add exactly that flag and the one code path that reads it — nothing else.

## What to put in APPROACH.md

Keep it brief — this is a lab notebook entry, not documentation:

```
# <idea slug>

## Idea
<2–4 sentences: the hypothesis and the expected effect>

## What changed
- <file>: <what was modified, in plain terms>
- ...

## How to verify
<the exact command or step, and what result would confirm the idea works>
```

## Anti-patterns (do not do these)

- **Writing code before the plan.** A subagent that starts editing copied files before `PLAN.md` exists has broken the discipline. Plan first, always — this is the whole point of the iteration that added planning.
- **Merging ideas** into one implementation. If two ideas both touch the same file, they're still two folders with two separate copies — never one combined diff.
- **Editing reference files in place.** Always copy first.
- **One big subagent doing all ideas sequentially.** That defeats independent verification and loses the parallelism. One subagent per idea.
- **Skipping the idea-list confirmation or the brainstorming pass.** Spawning on the wrong idea set, or with ambiguous intent, wastes a lot of work — and forces subagents to serialize by asking questions.
- **Over-implementing.** A "complete" feature that hides the effect is worse than a tiny change that reveals it. The plan should stay minimal; if it balloons, the idea is too big and should be split.
- **A plan that doesn't match the code.** If the final edits diverge from `PLAN.md`, that's a red flag — either the plan was wrong (and should have been updated first) or the subagent went off-script. Flag it to the user rather than silently shipping the mismatch.

## Relationship to other skills

- **Upstream**: `deep-research`, `superpowers:brainstorming` — these produce the idea list this skill consumes. This skill *also* runs its own short brainstorming pass (step 2) to clarify intent before dispatch.
- **Planning**: `superpowers:writing-plans` — invoked inside each subagent to produce `PLAN.md` before any code. This is now mandatory, not optional.
- **Mechanism**: `superpowers:dispatching-parallel-agents` (fan-out), `superpowers:test-driven-development` (per-idea, when applicable — usually the first step of the plan).
- **Execution reference**: `superpowers:executing-plans` / `superpowers:subagent-driven-development` — a subagent executing its own `PLAN.md` is essentially running a single-task plan; these skills describe how to do that disciplinedly, and a subagent may consult them.
