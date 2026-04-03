---
name: slim
description: "Context budget auditor and compressor for Claude Code agent systems. Measures token cost of every always-loaded file (CLAUDE.md, MEMORY.md, agent .md files, memory indexes), identifies bloat, and compresses them without losing semantic content. Trigger when: user says 'slim', 'compress context', 'agents cost too many tokens', 'token audit', 'reduce agent spawn cost', 'context is bloated', 'model right-size', or complains that spawning an agent is expensive. Run monthly or any time the system feels sluggish."
---

# Slim — Context Budget Auditor

The mission: reduce the token cost of every agent spawn without losing any semantic content. **Compression means density, not deletion.**

**The core insight**: every agent spawn loads `CLAUDE.md` + `MEMORY.md` + the agent file + its memory index — all verbatim, every time. Like Claude's own conversation compression (collapsing verbose history into dense summaries), we keep only the essential **fast-tier** in always-loaded files and move raw detail to **slow-tier** docs that agents read on demand.

---

## Phase 0: Discover and Audit

Before touching anything, find all always-loaded files and measure them.

### Discover the project structure

1. Look for `CLAUDE.md` in the current working directory — that's the project root.
2. Find agent files: check `.claude/agents/*.md` in the project root.
3. Find the memory directory: check `~/.claude/projects/<project-path>/memory/` — the path is the working directory with `/` replaced by `-`.
4. Find memory indexes: `memory/*/INDEX.md` files.
5. Find `MEMORY.md` at the root of the memory directory.

### Build the audit table

Estimate tokens as **bytes ÷ 4**. Print a table sorted by token cost (highest first):

```
| File                        | Bytes  | Est. Tokens | Loaded by        |
|-----------------------------|--------|-------------|------------------|
| CLAUDE.md                   | 38,519 | 9,630       | Every agent      |
| MEMORY.md                   | 11,070 | 2,768       | Every agent      |
| agents/heavy-agent.md       | 19,477 | 4,869       | That agent only  |
| memory/heavy/INDEX.md       |  8,290 | 2,073       | That agent only  |
| ...                         |        |             |                  |
```

Also compute the **per-agent spawn cost**: `CLAUDE.md + MEMORY.md + agent file + agent memory index`.

After showing the table, ask: **"Run all phases in order, or start with a specific phase?"**

---

## Phase 1: CLAUDE.md Two-Tier Split

`CLAUDE.md` is the single biggest token cost — paid by every agent on every spawn. Typically ~80% of it is reference material rarely needed mid-session: format specs, protocol details, example-heavy sections, integration guides.

### Classify every section

**Fast tier** — stays in `CLAUDE.md` (read every spawn, must be dense):
- Core directory/track structure (terse table only)
- API endpoint list (URLs + methods, no examples)
- Core rules: assignment, escalation, completion — bullets only
- Agent hierarchy (field names + roles, not full explanation)
- File type names with one-line pointers to detail docs

**Slow tier** — move to `docs/claude/*.md` (loaded on demand via Read tool):

Typical candidates:

| Section type | Suggested slow-tier file |
|--------------|--------------------------|
| Full file format specs | `docs/claude/file-formats.md` |
| OKR / goal format + review cadence | `docs/claude/goals-okr.md` |
| Web UI development guide | `docs/claude/web-ui-dev.md` |
| Orchestration / parallel agent protocol | `docs/claude/orchestration.md` |
| Integration guides (external tools, APIs) | `docs/claude/integrations.md` |
| Slide / presentation protocol | `docs/claude/slides-protocol.md` |
| Methodology rules (FLOW, etc.) | `docs/claude/methodology.md` |
| Agent architecture: adding agents, skill registry | `docs/claude/agents-arch.md` |
| Model selection and escalation | `docs/claude/model-escalation.md` |

Each extracted section becomes a one-liner in `CLAUDE.md`:
```
> Detail: `docs/claude/file-formats.md` — task/decision/contact/idea formats
```

**Target**: `CLAUDE.md` shrinks to ≤ 25% of its original size.

### Process

1. Show the proposed fast-tier `CLAUDE.md` in a code block — **wait for approval**.
2. After approval: `mkdir -p docs/claude/` and write each slow-tier file (verbatim extracted content — nothing is deleted, just moved).
3. Write the compressed `CLAUDE.md`.
4. Report new size.

---

## Phase 2: MEMORY.md Compression

**Goal**: compress the memory index without deleting any memory files. Target: under 120 lines.

Rules:
- Remove entries that duplicate what `CLAUDE.md` already covers
- Merge entries covering the same topic into one line
- Each bullet: `- [Title](file.md) — one-hook description` (max one line)
- Keep all links pointing to real files — remove dead links
- Keep structural headers — they orient new conversations

**Show a before/after diff. Wait for approval before writing.**

---

## Phase 3: Agent File Compression

Work through agent files from largest to smallest.

**Cut:**
- Protocols already in `CLAUDE.md` (task completion, escalation rules — agents load it, no need to repeat)
- Rationale and explanation prose — keep the rule, drop the why
- Verbose examples that duplicate `CLAUDE.md`
- Generic filler with no domain-specific value

**Keep:**
- All frontmatter fields — **do not touch frontmatter**
- Agent identity + personality (terse: 3–5 lines max)
- Domain-specific protocols not found in `CLAUDE.md`
- Memory file list
- Session start protocol unique to this agent

**Target**: each agent file under 5KB. Agents over 10KB almost certainly have significant bloat.

For each agent:
1. Show current size + what you'd cut and why
2. Wait for "yes" / "skip" / edits
3. Write and confirm new size

---

## Phase 4: Memory Index Compression

For each `memory/*/INDEX.md`, work largest to smallest.

**Rules:**
- Each entry: `- [File](file.md) — one-line hook` (max one line)
- Verify linked files actually exist — remove dead links
- Remove session log entries older than 2 cycles — replace with pointer to archive file
- Remove duplicates
- Target: under 80 lines per INDEX.md

**Show diff per index. Wait for approval before writing.**

---

## Phase 5: Model Right-Sizing

Read the `model:` field from each agent's frontmatter. Print an audit table:

```
| Agent      | Current | Recommended | Reason                                      |
|------------|---------|-------------|---------------------------------------------|
| Orchestrator | opus  | opus ✓      | Cross-system decisions, strategic reasoning |
| Researcher | opus   | sonnet ↓    | Digestion + structuring — not strategic     |
| Developer  | sonnet | sonnet ✓    | Code tasks — sonnet appropriate             |
| Monitor    | sonnet | haiku ↓     | Polling / scanning — mechanical work        |
```

**Model tiers and cost** (per 1M input tokens, approximate):
- **Haiku** ~$0.25 — mechanical tasks: polling, scanning, simple extraction
- **Sonnet** ~$3 — standard work: coding, research, structured writing
- **Opus** ~$15 — strategic work: complex reasoning, investor-facing, cross-system decisions

**Rule of thumb for downgrades:**
- Opus → Sonnet: agent's primary tasks are research, digestion, or structured output (not decisions)
- Sonnet → Haiku: agent's primary tasks are monitoring, scanning, or simple pattern matching

For each recommended downgrade, explain:
- What tasks this agent handles that don't require the higher model
- What edge cases DO warrant escalation (see Dynamic Escalation below)

**Wait for per-agent approval before editing any frontmatter.**

### Dynamic Model Escalation Pattern

Claude Code sets agent model via `model:` frontmatter. For tasks within an agent that need stronger reasoning, the agent can spawn a sub-agent with a model override:

```
Agent tool: { subagent_type: "general-purpose", model: "claude-opus-4-6", prompt: "..." }
```

This lets a Sonnet-default agent run 95% of its work cheaply and escalate only when needed. Document the escalation triggers in the agent file or in `docs/claude/model-escalation.md`.

---

## Phase 6: Validation

Print the before/after comparison:

```
## Before vs After

| File               | Before tok | After tok | Saved |
|--------------------|-----------|-----------|-------|
| CLAUDE.md          | 9,630      | X         | Y     |
| MEMORY.md          | 2,768      | X         | Y     |
| agents/[name].md   | 4,869      | X         | Y     |
| memory/[x]/INDEX.md| 2,073      | X         | Y     |
| ...                |            |           |       |

Per [heaviest agent] spawn: [before] → [after] tokens ([X]% reduction)
Model changes: [list agent: old → new]
Files changed: [count] files
Slow-tier docs created: [count] files in docs/claude/
```

Optionally, ask the user to spawn the most-used agent with a simple greeting and share the `total_tokens` from the result to verify real-world reduction.

---

## Ground Rules

- **Never delete semantic content** — if something leaves `CLAUDE.md`, it lives in `docs/claude/`.
- **Interactive** — propose, wait, write. Never batch-write files without approval.
- **Slow-tier docs are additive** — `CLAUDE.md` pointers tell agents where to Read detail when they need it.
- **Frontmatter is sacred** — never remove or rename frontmatter fields in agent files.
- **Don't touch generated files** — task JSON, auto-generated markdown, etc. are managed by other systems.
