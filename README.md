<p align="center">
  <img src="assets/banner.png" alt="hard-compact — save operational state across Claude Code and Codex CLI compaction" width="100%" />
</p>

<div align="center">
  <h1>🧊 hard-compact</h1>
  <p><strong>Stop your CLI agent from lobotomizing itself between context windows.</strong></p>
  <p>Drop-in custom compact prompts that preserve operational state — not narrative prose.</p>
  <p>
    <a href="#claude-code"><img src="https://img.shields.io/badge/Claude_Code-settings.json-8B4789?style=flat-square" alt="Claude Code" /></a>
    <a href="#codex-cli"><img src="https://img.shields.io/badge/Codex_CLI-config.toml-000?style=flat-square" alt="Codex CLI" /></a>
    <a href="./LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square" alt="License: MIT" /></a>
  </p>
</div>

---

## The problem

When Claude Code or Codex CLI hit their context limit, they auto-compact the session history into a summary. The default behavior produces **narrative prose** — "This session we worked on the authentication module and fixed a bug..."

Narrative prose destroys operational state:

- **Open bugs vanish.** The compact logs "resolved issue with auth" but drops the `file:line` ref and the unverified fix.
- **Active hypotheses disappear.** "Might be a race condition" is gone. Next turn, the agent re-discovers it from scratch.
- **Failed commands evaporate.** The exact command that produced the error — gone. The agent tries it again.
- **The reasoning thread breaks.** The agent loses *why* it was about to do the next thing.

The agent wakes up in the next context window like someone who just had a blackout — technically alive, functionally amnesiac.

## The insight

Compaction is not a retrospective. It is an **operational handoff**.

The next-turn LLM does not need to read a story about what happened. It needs to know: what is currently broken, what was tried and failed, what the next concrete action is, and which assumptions are still unverified. Everything else can be compressed to a path and a single line.

The prompt controls what survives the compaction. Default prompts optimize for readability. These prompts optimize for **agent continuity**.

## Two engines, two prompts

Claude Code and Codex CLI both support custom compact prompts, but they run under different constraints:

| | Claude Code | Codex CLI |
|---|---|---|
| Config key | `compactSummaryPrompt` in `settings.json` | `compact_prompt` in `config.toml` |
| Compactor model | fast one-shot, no extended thinking | GPT-5.4, reasoning budget available |
| Prompt style | rigid `key=value` straightjacket | structured shift-change briefing |
| Thinking | ✗ | ✓ (non-interactive sessions) |

---

## Claude Code

**The Strict Enforcer.** Because Claude Code's compactor runs as a one-shot instruction without extended thinking, the prompt must act as a rigid format constraint — no prose allowed, no narrative, just telegraphic machine-readable state.

### Install

Add to `~/.claude/settings.json`:

```json
{
  "compactSummaryPrompt": "STYLE: telegraphic key=value, not prose. FORBIDDEN: narrative sentences, 'this session we...', self-reported completeness, repeating facts. IMMORTAL (never compress): open bugs with file:line refs, pending decisions with rationale, failed commands with error, active hypotheses. COMPRESS AGGRESSIVELY: completed tasks (→ file path + 1 line), resolved bugs, historical background. PRIORITIZE: sections 7-9 (pending/current/next) OVER sections 1-6 (history). ADD at end: OPEN_HYPOTHESES — unverified assumptions that could be false negatives. ADD: TRAJECTORY — 1 sentence reasoning thread from this session."
}
```

### What it does

- **IMMORTAL items** are never compressed: open bugs (with `file:line`), pending decisions with rationale, failed commands with exact error, active hypotheses.
- **Aggressive compression** of completed work: a finished task becomes `→ path/to/file.py (one line)`.
- **Explicit section weighting**: sections about pending/current/next work survive; sections about history get squeezed.
- **OPEN_HYPOTHESES** section added at the end: unverified assumptions that could be wrong.
- **TRAJECTORY**: one sentence capturing the reasoning thread — why the agent was about to do the next thing.

### Why `key=value`, not prose

Natural language summaries drift toward telling a coherent story. Coherent stories omit contradictions, failed branches, and open questions — exactly the things an agent needs to continue work. Telegraphic format forces the compactor to inventory state rather than narrate it.

---

## Codex CLI

**The Reasoning Handoff.** Because Codex CLI uses GPT-5.4 with a reasoning budget, the compact can be more sophisticated — preserving structured schemas, weighing what actually matters for continuation, and separating verified facts from hypotheses with more nuance.

### Install

Add to `~/.codex/config.toml`:

```toml
compact_prompt = """
Compact the Codex session history into a continuation brief for the next turn.

Preserve:
- the user's current goal and the newest instruction;
- active plan state, blockers, and the next concrete action;
- files changed or inspected, exact paths, config keys, command names, model/backend choices, and important IDs;
- commands already run and their pass/fail result;
- unresolved bugs, risks, TODOs, and assumptions;
- dirty-worktree constraints and any user changes that must not be reverted;
- any structured state blocks present in the session (SESSION_COMPACT, etc.) — preserve their schema and meaning, do not rewrite into prose.

Separate verified facts from hypotheses. Keep dates/times concrete when present.
Do not preserve long logs, repeated grep output, dead exploration branches,
conversation filler, or generic reasoning narration.

Output a concise but complete operational handoff that lets Codex continue work
without re-reading the whole prior transcript.
"""
```

### What it does

- **Goal + latest instruction** are always the first thing preserved.
- **Exact paths, IDs, model choices, config keys** — not paraphrased, not summarized, verbatim.
- **Pass/fail on every command** already run — Codex won't re-run what already failed.
- **Dirty-worktree constraints** — user changes not yet committed are listed explicitly so the agent doesn't overwrite them.
- **Structured state blocks** (e.g. `SESSION_COMPACT` in Hive/agent orchestration setups) are preserved with their schema intact.
- **Facts vs. hypotheses** clearly separated — "the DB connection is broken" vs. "might be a race condition in the auth middleware".

### The reasoning advantage

In non-interactive Codex sessions, the compactor model can use its thinking budget before producing the handoff. This means it can weigh tradeoffs ("this failed command is probably environmental, not a bug — exclude it"), resolve apparent contradictions, and produce a denser handoff than a one-shot compactor can.

---

## Before / after

**Default compact output (Claude Code):**
```
This session we worked on the authentication module. We fixed a bug where 
users were being logged out unexpectedly. We also investigated some performance 
issues and made improvements to the database queries. Next we plan to look at 
the notification system.
```

**hard-compact output:**
```
GOAL: fix auth logout bug + investigate perf
STATUS: auth_fix=MERGED, perf=IN_PROGRESS
BUG_OPEN: hive/auth.py:143 — token refresh race, not yet tested under load
CMD_FAILED: pytest tests/test_auth.py -k race_condition → AttributeError: 'NoneType' object has no attribute 'token' (last: 2026-04-24 14:32)
OPEN_HYPOTHESES: perf issue may be N+1 in user.sessions loader (unverified), retry logic may mask the failure
TRAJECTORY: was about to add sleep(0.1) probe in token refresh to confirm race
```

One paragraph of prose. Zero actionable state.  
Ten lines of telegraphic data. Agent continues immediately.

---

## Combine with a PreCompact hook

Claude Code also supports a `PreCompact` hook — a shell command that runs just before compaction and whose stdout is injected into the context. Useful for surfacing live system state (current task list, recent git commits, open alerts) that the compactor then includes in the summary.

```json
{
  "hooks": {
    "PreCompact": [{
      "matcher": "",
      "hooks": [{"type": "command", "command": "your-context-prime-script"}]
    }]
  }
}
```

The custom `compactSummaryPrompt` tells the compactor *how* to summarize. The `PreCompact` hook tells it *what extra context to include*. Used together they give the compactor the best possible inputs.

---

## Sister projects

Same author, same audience:

- [**raisin**](https://github.com/Oldrich333/raisin) — Write Python LLMs can read. ~50% fewer tokens, 100% same functionality.
- [**ax-headers**](https://github.com/Oldrich333/ax-headers) — One-line machine-readable Python file headers. Cut AI context bloat by 30%.
- [**full-review**](https://github.com/Oldrich333/full-review) — Adversarial code review for Claude Code / Codex CLI. 0.80+ recall vs 0.40 for parallel specialists.

---

## License

MIT
