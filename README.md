# collection

A Claude Code plugin built by [One Studio Co](https://one-studio.co).

## Origin

We built a multi-agent system in Claude Code — a command center with 10+ specialized agents, each with their own memory, tools, and domain. It worked well. Then we noticed something: spawning a simple agent just to answer a one-line question cost **~32,000 input tokens**. Every single time.

We traced it. The agent wasn't doing anything special — it just loaded `CLAUDE.md` (~9,600 tokens) + `MEMORY.md` (~2,800 tokens) + its own agent file (~4,900 tokens) + its memory index (~2,000 tokens) before saying a word. Files that had grown organically over months, full of detail that was only occasionally needed.

The fix was obvious in hindsight: the same thing Claude Code already does to its own conversation history — compress. Keep the essential summary always in context, move the raw detail somewhere agents can Read on demand.

We ran a full audit and compression pass across the whole system:

| What | Before | After | Reduction |
|------|--------|-------|-----------|
| `CLAUDE.md` | 38,519 bytes | 5,630 bytes | **-85%** |
| Average agent file | ~10,000 bytes | ~4,800 bytes | **-52%** |
| Average memory index | ~5,000 bytes | ~1,700 bytes | **-66%** |
| Heaviest agent spawn | ~15,700 tokens | ~4,700 tokens | **-70%** |

One agent was running on Opus when Sonnet was sufficient — fixing that alone cut its cost **5×** per session.

We turned the process into `/slim` so any Claude Code agent system can run the same pass.

---

## What are Claude Code skills?

Skills are reusable prompts that extend Claude Code with domain-specific workflows. Drop a skill file into `.claude/commands/` and invoke it with `/skill-name` in any Claude Code session.

## Skills

### `/slim` — Context Budget Auditor

Measures and compresses the token cost of your Claude Code agent system. **CLI-native — no MCP servers required.**

Every agent spawn loads the full `CLAUDE.md` chain + rules + `MEMORY.md` (first 200 lines) + the agent file + its memory index — all verbatim. As projects grow, these files accumulate detail that's only needed occasionally. `slim` audits all always-loaded files across 12+ locations, proposes a two-tier split (fast-tier always loaded, slow-tier read on demand), and compresses agent files, rules, and memory indexes — without losing any content.

**Optional CLI tools** (recommended for better results):
- [`cc-token`](https://github.com/iota-uz/cc-token) — accurate Claude token counting via Anthropic's official API. Install: `go install github.com/iota-uz/cc-token@latest`
- [`yq`](https://github.com/mikefarah/yq) — YAML/frontmatter processor. Install: `brew install yq`
- [`markdown-link-check`](https://github.com/tcort/markdown-link-check) — validates markdown links. Install: `npm install -g markdown-link-check`

The skill works without any of these — it falls back to built-in tools (`wc -c`, `Grep`, `Glob`).

**Run it when:**
- Spawning an agent costs more tokens than expected
- `CLAUDE.md` has grown beyond ~5KB
- Agent files have accumulated protocol details that duplicate `CLAUDE.md`
- You want a monthly context hygiene pass

**What it does (6 phases):**
1. **Audit** — discovers all always-loaded files (CLAUDE.md chain, rules, agents, memory), measures token cost using `cc-token` or byte estimation
2. **CLAUDE.md split** — extracts reference material to `docs/claude/*.md`, keeps only the fast-tier in `CLAUDE.md`
3. **MEMORY.md compression** — collapses verbose entries to one-liners, validates links with `markdown-link-check`
4. **Agent & rules file compression** — removes duplication with `CLAUDE.md`, cuts prose rationale, keeps domain-specific rules
5. **Memory index compression** — each entry to one line, under 80 lines per index, validates links
6. **Model right-sizing** — audits `model:` frontmatter per agent using `yq`, recommends Haiku/Sonnet/Opus based on task type, includes cost estimates

Interactive — proposes changes, waits for approval before writing each phase.

## Installation

This plugin is distributed via the **One Studio** marketplace on GitHub.

### 1. Add the marketplace

From within Claude Code:

```
/plugin marketplace add onestudio-co/claude-code
```

Or from the CLI:

```bash
claude plugin marketplace add onestudio-co/claude-code
```

### 2. Install the plugin

From within Claude Code:

```
/plugin install collection@one-studio
```

### 3. Restart the session

After installing, **restart your Claude Code session** (`/exit` then relaunch) to load the new plugins.

> **Note:** The built-in `/reload-plugins` command _should_ reload newly added plugins without a restart, but it currently does not. This is a [known bug](https://github.com/anthropics/claude-code/issues/35641). Once the Anthropic team fixes it, you can use `/reload-plugins` instead of restarting.

### 4. Use the skills

```
/slim
```

## Update

To pull the latest version of the plugin, update the marketplace:

From within Claude Code:

```
/plugin marketplace update one-studio
```

Or from the CLI:

```bash
claude plugin marketplace update one-studio
```

## License

MIT
