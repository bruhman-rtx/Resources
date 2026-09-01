# Claude Code Investigation Prompt — Context Decay & Compaction Thresholds

Paste everything below the line into a fresh Claude Code session, run from your
home directory (not inside a project repo, so it can see all projects).

Give it a fresh session with a high context budget — this task reads a lot of
files, and you don't want it compacting mid-analysis.

---

## TASK

I want to test a hypothesis about my own Claude Code usage using real data from
my local session transcripts. Do not guess, estimate, or fabricate any number.
If data isn't available, say so explicitly and move on.

### The hypothesis

1. Output quality degrades as the context window fills.
2. That degradation is severe enough that compacting early (at ~20% of the
   window) beats letting autocompact fire at its default (~84%).
3. My baseline context floor — what's loaded before I type anything — is
   substantial, and is driven mostly by MCP tool definitions rather than by
   project files.

All three might be wrong. Your job is to find out, not to confirm them.

---

## PHASE 0 — Establish ground truth about my configuration

Before touching transcripts, report the following. For each, say where you got
it and flag anything you could not determine:

- Claude Code version (`claude --version`)
- Contents of `~/.claude/settings.json` and any project-level
  `.claude/settings.json` files
- Whether `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` or
  `CLAUDE_CODE_AUTO_COMPACT_WINDOW` are set in my environment or shell rc files
  (`~/.bashrc`, `~/.zshrc`, `~/.profile`, PowerShell `$PROFILE`)
- Every MCP server configured, from `~/.claude.json` and any
  `.mcp.json` files
- The model I'm using and its context window size
- The actual documented default autocompact threshold — check
  https://docs.claude.com/en/docs/claude-code/overview and the docs map at
  https://docs.anthropic.com/en/docs/claude-code/claude_code_docs_map.md
  If it isn't officially documented, say so plainly. Do not substitute a
  number from a blog post without labelling it as unofficial.

---

## PHASE 1 — Measure the context floor

I want to know what a "fresh" session actually costs before any work happens.

- Find my session transcripts. They're normally JSONL under
  `~/.claude/projects/<encoded-project-path>/*.jsonl`. Confirm the location on
  this machine before assuming it.
- For a sample of at least 20 sessions across different projects, find the
  **first assistant message** in each and pull its `usage` block —
  specifically `input_tokens`, `cache_read_input_tokens`, and
  `cache_creation_input_tokens`.
- That first-turn input total is my floor. Report it as an absolute token
  count and as a percentage of the window, with median / min / max / spread.

Then decompose the floor. Estimate how much comes from each of:

- MCP tool definitions (count servers, count tools per server, and estimate
  the token cost of their JSON schemas — you can measure this by comparing a
  session started with MCP servers against one started with `--strict-mcp-config`
  and an empty config, if that's feasible)
- `CLAUDE.md` files (global at `~/.claude/CLAUDE.md`, plus project-level) —
  measure these directly with a tokenizer or a chars/4 approximation, and say
  which method you used
- System prompt and built-in tools (the remainder)

State clearly which parts are measured and which are inferred by subtraction.

---

## PHASE 2 — Reconstruct context growth over time

For each session with more than 10 turns:

- Build a time series: for every assistant turn, record the cumulative input
  token count and the wall-clock timestamp.
- Convert tokens to percentage of the context window.
- Identify compaction events. These usually show as a sharp drop in input
  tokens between consecutive turns, or as an explicit summary/compact marker in
  the transcript. Determine how compaction is actually represented in my
  transcript format before writing the detection logic, and tell me what
  marker you used.
- For each session compute: number of compactions, wall-clock time to first
  compaction, mean context % across the session, and the fraction of turns
  spent in each band — under 20%, 20–50%, 50–75%, over 75%.

Output this as `context-timeseries.csv` with one row per turn:
`session_id, project, turn_index, timestamp, input_tokens, context_pct, compaction_flag`

---

## PHASE 3 — Build a quality proxy (the hard part)

There is no ground-truth quality score in the transcripts, so we need proxies.
Compute each of these **as a function of context %**, bucketed into 10% bands.
This is the actual test of the hypothesis: if quality degrades with context,
these should get worse as context % rises.

1. **Tool error rate** — fraction of `tool_result` blocks with `is_error: true`
2. **Bash failure rate** — non-zero exit codes on Bash tool calls
3. **Edit retry rate** — how often the same file is edited again within the
   next 3 turns (a proxy for "got it wrong the first time")
4. **User correction rate** — fraction of user turns that look like corrections
   rather than new instructions. Build a keyword list (no, wrong, actually,
   revert, undo, that's not, I said, stop) but also report how many user turns
   you classified and spot-check 20 of them by hand so I can see the false
   positive rate
5. **Turn verbosity drift** — mean assistant output tokens per turn by band
6. **Repetition** — whether the assistant re-reads files it already read in the
   same session, by band

For each proxy report the sample size per band. Any band with fewer than 30
observations should be marked as underpowered rather than reported as a result.

Run a simple correlation between each proxy and context %, report the
coefficient and a p-value, and **state plainly if there is no significant
relationship.** A null result is a valid and useful outcome here.

Output as `quality-proxies.csv`.

---

## PHASE 4 — Compare compaction strategies

Using the reconstructed data, compare sessions that compacted early against
sessions that ran to the default threshold:

- Group sessions by their observed compaction threshold
- For each group, compute the quality proxies from Phase 3
- Compute total tokens spent on compaction summaries per hour of work — early
  compaction should cost more here, and I want to know how much more
- Compute how often work is repeated after a compaction (files re-read,
  context re-established), which is the real cost of aggressive compaction

The key question: does the quality gain from staying under 20% outweigh the
overhead of compacting 4× more often? Answer it with numbers, and if the data
can't support an answer, say that instead.

---

## PHASE 5 — Report

Write `findings.md` containing:

- A one-paragraph plain answer: is the hypothesis supported, partly supported,
  or not supported?
- A table of every number you measured, with its source
- A separate, clearly-labelled section listing everything you could NOT measure
  and why
- Every assumption you made, listed explicitly
- The three strongest arguments against my hypothesis that this data supports

### Rules

- Never fill a gap with a plausible-sounding number. Write UNKNOWN.
- Distinguish measured / derived / assumed on every figure.
- If a phase is impossible given my transcript format, say so and explain what
  data would be needed instead.
- Do not tell me the hypothesis is correct unless the numbers say so.
