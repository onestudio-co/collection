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

Measures and compresses the token cost of your Claude Code agent system.

Every agent spawn loads `CLAUDE.md` + `MEMORY.md` + the agent file + its memory index — all verbatim. As projects grow, these files accumulate detail that's only needed occasionally. `slim` audits all always-loaded files, proposes a two-tier split (fast-tier always loaded, slow-tier read on demand), and compresses agent files and memory indexes — without losing any content.

**Run it when:**
- Spawning an agent costs more tokens than expected
- `CLAUDE.md` has grown beyond ~5KB
- Agent files have accumulated protocol details that duplicate `CLAUDE.md`
- You want a monthly context hygiene pass

**What it does (6 phases):**
1. **Audit** — measures every always-loaded file, prints a token cost table sorted by impact
2. **CLAUDE.md split** — extracts reference material to `docs/claude/*.md`, keeps only the fast-tier in `CLAUDE.md`
3. **MEMORY.md compression** — collapses verbose entries to one-liners, removes dead links
4. **Agent file compression** — removes duplication with `CLAUDE.md`, cuts prose rationale, keeps domain-specific rules
5. **Memory index compression** — each entry to one line, under 80 lines per index
6. **Model right-sizing** — audits `model:` frontmatter per agent, recommends Haiku/Sonnet/Opus based on task type, documents a dynamic escalation pattern for edge cases

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

### 3. Use the skills

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
