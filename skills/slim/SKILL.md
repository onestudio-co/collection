---
name: slim
description: "CLI-native context budget auditor and compressor for Claude Code agent systems — no MCP dependencies. Measures token cost of every always-loaded file (CLAUDE.md chain, rules, MEMORY.md, agent .md files, memory indexes) using cc-token or built-in tools. Identifies bloat and compresses without losing semantic content. Trigger when: user says 'slim', 'compress context', 'agents cost too many tokens', 'token audit', 'reduce agent spawn cost', 'context is bloated', 'model right-size', or complains that spawning an agent is expensive. Run monthly or any time the system feels sluggish."
---

# Slim — Context Budget Auditor

The mission: reduce the token cost of every agent spawn without losing any semantic content. **Compression means density, not deletion.**

**The core insight**: every agent spawn loads the full `CLAUDE.md` chain + rules + `MEMORY.md` (first 200 lines) + the agent file + its memory index — all verbatim, every time. Like Claude's own conversation compression (collapsing verbose history into dense summaries), we keep only the essential **fast-tier** in always-loaded files and move raw detail to **slow-tier** docs that agents read on demand.

**Implementation constraint**: This skill uses only CLI tools and Claude Code built-in tools (Read, Glob, Grep, Bash). No MCP servers are required.

### Optional CLI tools (recommended)

Check which tools are available before starting. The skill works without any of them but produces better results when they're installed.

- **`cc-token`** — accurate Claude token counting via Anthropic's official API (free). Install: `go install github.com/iota-uz/cc-token@latest`. Requires `ANTHROPIC_API_KEY`.
- **`yq`** — YAML/frontmatter processor. Install: `brew install yq` or `apt install yq`.
- **`markdown-link-check`** — validates markdown links point to real files. Install: `npm install -g markdown-link-check`.

Run these checks:
```
which cc-token && echo "cc-token: available" || echo "cc-token: not found — will use bytes/4 estimation"
which yq && echo "yq: available" || echo "yq: not found — will use Grep for frontmatter"
which markdown-link-check && echo "markdown-link-check: available" || echo "markdown-link-check: not found — will use Glob for link validation"
```

---

## Phase 0: Discover and Audit

Before touching anything, find all always-loaded files and measure them.

### Discover the project structure

Enumerate every file that Claude Code loads into context. Check each location — skip any that don't exist.

**CLAUDE.md chain** (all loaded every agent spawn, in order):
1. `/etc/claude-code/CLAUDE.md` — system-level (managed installs)
2. `~/.claude/CLAUDE.md` — user-level
3. Ancestor directories — walk from cwd up to `/`, check each for `CLAUDE.md`
4. `<project-root>/CLAUDE.md` — project-level
5. `<project-root>/.claude/CLAUDE.md` — project .claude directory
6. `<project-root>/CLAUDE.local.md` — local overrides (gitignored)

**Rules** (always loaded):
7. `.claude/rules/*.md` — all rule files in the project

**Agents** (loaded per-spawn):
8. `/etc/claude-code/agents/*.md` — system-level agents
9. `~/.claude/agents/*.md` — user-level agents
10. `.claude/agents/*.md` — project-level agents

**Memory** (loaded per-spawn):
11. `~/.claude/projects/<project-path>/memory/MEMORY.md` — **only the first 200 lines are loaded**. The `<project-path>` is the working directory path with `/` replaced by `-`.
12. Per-agent memory index files under the memory directory.

Use `Glob` to discover files at each location. Use `Bash(wc -c <file>)` to get byte counts.

### Build the audit table

**If `cc-token` is available**, run it for accurate Claude token counts:
```bash
cc-token count <project-root> --ext .md --json
```
This gives exact token counts, cost estimates, and optimization analysis. Optionally generate an HTML report:
```bash
cc-token count <project-root> --ext .md --output audit-report.html
```

**If `cc-token` is not available**, estimate tokens as **bytes ÷ 4** (approximate — ~15-25% margin of error). Use `Bash(wc -c <file>)` for byte counts.

**Important**: For `MEMORY.md`, measure only the first 200 lines since that's all Claude Code loads:
```bash
head -200 <memory-dir>/MEMORY.md | wc -c
```

Print a table sorted by token cost (highest first):

```
| File                        | Bytes  | Tokens (est) | Loaded by        |
|-----------------------------|--------|--------------|------------------|
| CLAUDE.md                   | 38,519 | 9,630        | Every agent      |
| ~/.claude/CLAUDE.md         |  4,200 | 1,050        | Every agent      |
| .claude/rules/coding.md     |  2,100 |   525        | Every agent      |
| MEMORY.md (first 200 lines) |  8,000 | 2,000        | Every agent      |
| agents/heavy-agent.md       | 19,477 | 4,869        | That agent only  |
| memory/heavy/INDEX.md       |  8,290 | 2,073        | That agent only  |
| ...                         |        |              |                  |
```

Compute the **per-agent spawn cost**: `Sum(CLAUDE.md chain) + Sum(rules) + MEMORY.md (first 200 lines) + agent file + agent memory index`.

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

### Find dead links first

**If `markdown-link-check` is available**, run it to identify broken links automatically:
```bash
markdown-link-check <memory-dir>/MEMORY.md
```

**If not**, use `Glob` to verify each linked file exists manually.

### Compression rules
- Remove entries that duplicate what `CLAUDE.md` already covers
- Merge entries covering the same topic into one line
- Each bullet: `- [Title](file.md) — one-hook description` (max one line)
- Keep all links pointing to real files — remove dead links identified above
- Keep structural headers — they orient new conversations

**Show a before/after diff. Wait for approval before writing.**

---

## Phase 3: Agent and Rules File Compression

Work through agent files and rules files from largest to smallest.

### Extract frontmatter metadata

**If `yq` is available**, batch-extract model and key fields from all agents:
```bash
for f in .claude/agents/*.md; do echo "=== $f ==="; yq -f=extract '.' "$f"; done
```

**If not**, use `Grep` to find `model:` lines in frontmatter across all agent files.

### Compression rules for agent files

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

### Compression rules for `.claude/rules/*.md`

Rules files are always-loaded for every agent spawn — bloat here multiplies across all agents.

**Cut:**
- Verbose examples — keep the rule, drop the illustration
- Rules that duplicate `CLAUDE.md` content
- Explanatory prose — one-liner per rule is sufficient

**Keep:**
- Path-scoped frontmatter (e.g., `paths: ["src/**/*.ts"]`)
- Domain-specific constraints not in `CLAUDE.md`

For each file:
1. Show current size + what you'd cut and why
2. Wait for "yes" / "skip" / edits
3. Write and confirm new size

---

## Phase 4: Memory Index Compression

For each `memory/*/INDEX.md`, work largest to smallest.

### Find dead links

**If `markdown-link-check` is available**:
```bash
markdown-link-check <index-file>
```

**If not**, use `Glob` to verify each linked file exists.

### Compression rules
- Each entry: `- [File](file.md) — one-line hook` (max one line)
- Remove dead links identified above
- Remove session log entries older than 2 cycles — replace with pointer to archive file
- Remove duplicates
- Target: under 80 lines per INDEX.md

**Show diff per index. Wait for approval before writing.**

---

## Phase 5: Model Right-Sizing

**If `yq` is available**, extract all model assignments in one pass:
```bash
for f in .claude/agents/*.md; do echo "$(basename "$f" .md): $(yq -f=extract '.model' "$f")"; done
```

**If not**, use `Grep` to find `model:` in each agent file's frontmatter.

Print an audit table:

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

**If `cc-token` is available**, re-run the audit for accurate before/after comparison:
```bash
cc-token count <project-root> --ext .md --json --show-cost
```
Optionally generate a shareable HTML report:
```bash
cc-token count <project-root> --ext .md --output slim-report.html
```

Print the before/after comparison (include all file categories):

```
## Before vs After

| File                  | Before tok | After tok | Saved   |
|-----------------------|-----------|-----------|---------|
| CLAUDE.md chain       | 10,680     | X         | Y       |
| .claude/rules/*.md    | 1,200      | X         | Y       |
| MEMORY.md (200 lines) | 2,000      | X         | Y       |
| agents/[name].md      | 4,869      | X         | Y       |
| memory/[x]/INDEX.md   | 2,073      | X         | Y       |
| ...                   |            |           |         |

Per [heaviest agent] spawn: [before] → [after] tokens ([X]% reduction)
Model changes: [list agent: old → new]
Files changed: [count] files
Slow-tier docs created: [count] files in docs/claude/
```

If `cc-token` was used, also include estimated cost savings per 1,000 agent spawns.

Optionally, ask the user to spawn the most-used agent with a simple greeting and share the `total_tokens` from the result to verify real-world reduction.

---

## Ground Rules

- **Never delete semantic content** — if something leaves `CLAUDE.md`, it lives in `docs/claude/`.
- **Interactive** — propose, wait, write. Never batch-write files without approval.
- **Slow-tier docs are additive** — `CLAUDE.md` pointers tell agents where to Read detail when they need it.
- **Frontmatter is sacred** — never remove or rename frontmatter fields in agent files.
- **Don't touch generated files** — task JSON, auto-generated markdown, etc. are managed by other systems.
- **No MCP dependencies** — all discovery and measurement uses CLI tools (`cc-token`, `yq`, `markdown-link-check`) and Claude Code built-in tools. The skill degrades gracefully when optional CLI tools are not installed.
