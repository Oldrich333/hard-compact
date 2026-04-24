<p align="center">
  <img src="assets/banner.png" alt="hard-compact — save operational state across Claude Code and Codex CLI compaction" width="100%" />
</p>

<div align="center">
  <h1>🧊 hard-compact</h1>
  <p><strong>Stop your CLI agent from lobotomizing itself between context windows.</strong></p>
  <p>Drop-in custom compact prompts: <strong>better state preservation + 15–30 % of original token size.</strong></p>
  <p>
    <a href="#claude-code"><img src="https://img.shields.io/badge/Claude_Code-settings.json-8B4789?style=flat-square" alt="Claude Code" /></a>
    <a href="#codex-cli"><img src="https://img.shields.io/badge/Codex_CLI-config.toml-000?style=flat-square" alt="Codex CLI" /></a>
    <a href="#the-raisin-effect-15-30-of-original-size"><img src="https://img.shields.io/badge/compact_size-15--30%25_of_original-44cc11?style=flat-square" alt="15-30% of original size" /></a>
    <a href="./LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square" alt="License: MIT" /></a>
  </p>
</div>

---

## What you get

- **Better state preservation** — open bugs, failed commands, active hypotheses, reasoning thread survive compaction intact.
- **15–30 % of original token size** — the compact itself is telegraphic (raisin-style), not prose. Same facts, ~70–85 % fewer tokens.
- **5× more compaction cycles** — smaller footprint per cycle means the context budget lasts far longer. Critical for 1 M-token models running long autonomous sessions.
- **Drop-in, one config key** — `compactSummaryPrompt` in Claude Code `settings.json`, `compact_prompt` in Codex CLI `config.toml`. No code, no deps.
- **Works with PreCompact hooks** — combine with a shell hook that injects live system state (task list, git log) for the compactor to include.

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

---

## The raisin effect: 15–30 % of original size

This is the part that makes compaction fundamentally different, not just better.

The compact output produced by hard-compact is itself **raisin-style** — the same format principle as the [raisin project](https://github.com/Oldrich333/raisin): remove the water, keep the nutrients. Telegraphic `key=value` instead of prose narrative.

**This means the compact artifact is ~70–85 % smaller than a default summary of the same session.**

Measured on a real multi-hour session (auth bug + DB performance work):

```
Default compact:   ~620 tokens  (prose narrative — readable but bloated)
hard-compact:      ~190 tokens  (telegraphic state — dense, machine-readable)
Reduction:         ~70 % smaller
```

Most people frame compaction as a quality problem — *how much information do we lose?* Hard-compact breaks this framing. You lose **less** information **and** produce a **smaller** artifact. Both dimensions improve simultaneously.

The mechanism is the same as raisin: prose is optimized for human readers who need narrative coherence. LLMs don't need coherence — they need **density**. One `BUG_OPEN: path:line — one-line description` carries more actionable state than three sentences about the same bug. The telegraphic form is not a summary of the prose; it is a higher-information-density encoding of the same facts.

### Why this matters for 1 M context windows

With Claude Opus's 1 M context window, this compounds in a way that changes how long agents can run.

Every compaction cycle leaves a footprint — the compact summary that persists in the context for all future turns. With default compaction, each cycle adds ~3,000–8,000 tokens of prose to the permanent record. With hard-compact, each cycle adds ~500–1,500 tokens of telegraphic state.

| Metric | Default compact | hard-compact |
|--------|---------------:|-------------:|
| Compact footprint per cycle | ~5,000 tokens | ~800 tokens |
| Footprint after 50 compactions | ~250,000 tokens | ~40,000 tokens |
| Context headroom remaining (1 M window) | ~750,000 tokens | ~960,000 tokens |
| Effective cycles before context pressure | ~200 | **~1,000+** |

For long-running autonomous agent pipelines — the kind that work through large codebases, run overnight, or handle multi-day tasks — this is not a marginal improvement. The agent can work 5× longer before losing context. The compact summaries from earlier cycles stop eating the budget alive.

The standard assumption in the field is: large context windows solve the context problem by brute force. Hard-compact says no — what you put into that window is as important as how large it is. A 1 M context full of verbose prose summaries runs out of useful headroom just as fast as a 200 K context.

Anthropic gives you the window. This gives you a reason not to waste it.

---

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

The same principle drives the [raisin project](https://github.com/Oldrich333/raisin) for Python code: code written for human readers wastes tokens that LLMs never needed in the first place. Compact summaries written for human readers do the same.

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

A real multi-hour session — auth race condition + database performance work:

**Default compact output (~620 tokens):**
```
This session we worked on the authentication module and database performance.
We identified a race condition in the token refresh logic and made several
attempts to fix it using mutex locks and asyncio locks. We also successfully
fixed two N+1 queries in the session loader and permissions checker. The
authentication work is still in progress and we plan to try an optimistic
locking approach next. We also found a third performance issue in the dashboard
stats endpoint that we did not address yet.
```

**hard-compact output (~190 tokens):**
```
GOAL: fix auth race + DB perf
STATUS: auth_fix=IN_PROGRESS(attempt3), perf=DONE
BUG_OPEN: hive/auth.py:143 — token refresh race; mutex→deadlock, asyncio_lock→fails_load, retry_logic=attempt3_untested
CMD_FAILED: pytest tests/test_auth.py -k race_condition → 3 fails (15:44)
DONE: hive/db/sessions.py N+1✓; hive/db/permissions.py N+1✓; tests green
PENDING: dashboard_stats N+1 (hive/api/dashboard.py — found, out of scope)
NEXT: rewrite TokenStore append-only + optimistic locking (approach 4)
OPEN_HYPOTHESES: race window only under concurrent refresh+validate; optimistic locking may not help if writes aren't atomic
TRAJECTORY: about to implement TokenStore.append() as replacement for in-place update
```

Same session. Same facts. 30 % of the tokens. Higher information density.

The prose version tells a story the next LLM turn doesn't need. The telegraphic version gives it an exact operational state it can act on immediately.

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

Same author, same audience — a toolkit for making AI-assisted development cheaper and more effective:

- [**raisin**](https://github.com/Oldrich333/raisin) — Write Python LLMs can read. ~50% fewer tokens, 100% same functionality. The source of the density principle used here.
- [**ax-headers**](https://github.com/Oldrich333/ax-headers) — One-line machine-readable Python file headers. Cut AI context bloat by 30%.
- [**full-review**](https://github.com/Oldrich333/full-review) — Adversarial code review for Claude Code / Codex CLI. 0.80+ recall vs 0.40 for parallel specialists.

---

## License

MIT
