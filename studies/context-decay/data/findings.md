# Context Decay & Compaction Thresholds — findings

Run 2026-09-01 against 52 main-thread session transcripts (2026-07-19 → 2026-09-01),
20,702 assistant turns, 156 compaction events, ~790 MB of JSONL.

Every figure below is tagged **[M] measured**, **[D] derived** (computed from measured
values), or **[A] assumed**. Anything unavailable is written **UNKNOWN**.

---

## 1. Plain answer

**Not supported, on all three limbs — and one limb is arithmetically impossible.**

Limb 1 (quality degrades as context fills): **not supported.** Six independent proxies
were computed over 20,702 turns. Not one degrades as context grows. Five improve
slightly and one (output length) rises slightly; all effect sizes are tiny
(|r| = 0.014–0.145) and survive a within-session fixed-effects test that removes every
between-session difference. If context decay exists in these sessions, none of these
proxies can see it.

Limb 2 (compact at ~20% beats the default): **not supported, and not reachable.** A
fresh session's floor is **49,553 tokens** median — already 24.8% of a 200 K window
before you type. The lowest context ever observed immediately after a compaction is
**46,470 tokens**; the median is **72,297**. Nothing in 20,668 turns with a non-zero
usage block ever sat below 20% of a 200 K window for more than 36 turns (0.2%). So
"stay under 20%" cannot be executed at 200 K. Read as 20% of the *1 M* window, it is
what you already run: your `--autocompact 200k` wrapper fires at 16.8% of 1 M. Against
that, sessions that ran deep without compacting scored *better* on every proxy.

Limb 3 (floor is substantial, driven mostly by MCP): **half right.** The floor is
substantial — 49.5 K tokens, 25% of a 200 K window. But MCP is **not** the driver: an
A/B measurement puts all MCP tool definitions at **1,305 tokens, 2.6% of the floor**,
because Claude Code defers MCP schemas by default and loads only tool *names*. The
floor is dominated by the ~43 K of system prompt, built-in tool schemas, and skill/agent
listings.

There is one clearly measured cost in the data, and it points *against* compacting more:
**re-reading of already-read files rises from 35.9% to 53.9% in the 10 turns after a
compaction** (+18.0 pp, z = 5.20, p = 2.0 × 10⁻⁷).

---

## 2. Every number measured

### Phase 0 — configuration

| Item | Value | Tag | Source |
|---|---|---|---|
| Claude Code version | 2.1.252 | M | `claude --version` |
| Model setting | `opus[1m]` | M | `~/.claude/settings.json` → `model` |
| Effort | `effortLevel: high`; per-model `claude-opus-5: xhigh` | M | same file |
| Model context window | 1,000,000 tokens | M | `[1m]` variant; corroborated by an observed 990,563-token turn |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | not set | M | `env`; no match in any rc file |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | not set | M | `env`; no match in any rc file |
| `~/.bashrc`, `~/.zshrc`, `~/.profile`, `~/.bash_profile` | all absent | M | `ls` |
| PowerShell profile | `Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1` defines a `claude` function injecting `--autocompact 200k` unless one is passed | M | file lines 6–15 |
| `autoCompactEnabled` / `autoCompactWindow` in settings | **not set** in `settings.json` or `settings.local.json` | M | both files read in full |
| MCP — global | `mobbin` (http, `https://api.mobbin.com/mcp`) | M | `~/.claude.json` → `mcpServers` |
| MCP — per-project in `.claude.json` | none | M | `projects[*].mcpServers` all empty |
| MCP — `.mcp.json` files | `project-01` → a local node MCP bridge (node bridge/server.mjs); `project-17` → `claude-flow` (`autoStart: false`) | M | both files read |
| MCP — managed connectors active this session | claude-in-chrome, Gmail, Google Drive, Google Calendar, Superhuman Mail, Clay | M | session tool listing |
| Project `.claude/settings.json` files | 8 found; 5 contain only `enabledPlugins`; `project-01` = hooks+permissions; `project-17` = env/hooks/permissions/statusLine; none sets a model, window, or compaction key | M | each file parsed |
| Plugins enabled | 6 (typescript-lsp, pyright-lsp, prompt-improver, frontend-design-audit, ui-ux-pro-max, token-optimizer) | M | `settings.json` |
| Local skills / agents | 87 SKILL.md files / 10 agent files | M | glob |

**Documented default auto-compact threshold.** Officially documented, quoted verbatim
from `code.claude.com/docs/en/model-config` § *Default auto-compact thresholds*:

> If you don't set an auto-compact window, Claude Code compacts when the conversation
> reaches the model's context limit

with listed exceptions (Sonnet 4.6 / Opus 4.6 without extended context, and Opus 4.8 /
Opus 5 on a 200 K window, compact at the 200 K boundary; Sonnet 5 "auto-compact before
the window fills, at about 967K tokens by default").

**There is no documented "~84%" default.** [M] No percentage threshold appears anywhere
in the official docs. The 84% figure in the hypothesis is not a documented default — but
it *is* an accurate description of your observed behaviour, for a reason given in Phase 2.

> **Correction to an existing note.** `~/.claude/projects/<USER>/memory/autocompact-threshold.md`
> states "There is no settings.json key for it — CLI flag only, so persistence needs a
> wrapper." That is now wrong: the docs document an `autoCompactWindow` settings key and
> an `/autocompact <value>` command that persists to user settings. The PowerShell wrapper
> is no longer the only persistence route, and unlike the wrapper a settings key would also
> cover Git Bash launches. [M, docs]

### Phase 1 — the context floor

Transcripts confirmed at `~/.claude/projects/<encoded-path>/*.jsonl` [M]: 52 main-thread
files (depth 2) plus 1,974 subagent files under `<session-uuid>/subagents/`, 1.8 GB total.
Only main-thread files were analysed.

Floor = first assistant turn with a non-zero usage block; context =
`input_tokens + cache_creation_input_tokens + cache_read_input_tokens`.

| Statistic | Tokens | % of 200 K | % of 1 M | Tag |
|---|---|---|---|---|
| n sessions | 45 | | | M |
| min | 31,688 | 15.8% | 3.17% | M |
| p25 | 38,829 | 19.4% | 3.88% | M |
| **median** | **49,553** | **24.8%** | **4.96%** | **M** |
| p75 | 54,987 | 27.5% | 5.50% | M |
| max | 67,614 | 33.8% | 6.76% | M |
| mean / sd | 48,169 / 10,708 | | | M |

**Floor decomposition.**

| Component | Tokens | Tag | Method |
|---|---|---|---|
| MCP tool definitions (all servers) | **1,305** | **M** | A/B: identical `claude -p` run, all MCP (29,292 total input) vs `--strict-mcp-config` + empty config (27,987). Same model, same prompt. |
| `~/.claude/CLAUDE.md` + `RTK.md` + `ROUTING.md` | ~583 | D | chars/4 on 2,333 bytes |
| `MEMORY.md` (auto-memory) | ~1,660 | D | chars/4 on 6,640 bytes |
| SessionStart vault hook injection | ~2,746 | D | chars/4 on 10,982 bytes of hook stdout |
| **Remainder** — system prompt, built-in tool schemas, skill + agent listings | **~43,259** | **D (by subtraction)** | 49,553 − 4,989 − 1,305 |

Method note: chars/4 is an approximation, not a tokenizer. No Claude tokenizer was
available locally (`tiktoken` absent, and it would be the wrong vocabulary anyway). The
MCP figure is the only component measured directly rather than approximated.

### Phase 2 — context growth and compaction

Compaction is represented **explicitly** in this transcript format — no heuristic drop
detection was needed. Two paired markers, 156 of each, perfectly 1:1 [M]:

1. `{"type":"system","subtype":"compact_boundary","compactMetadata":{trigger, preTokens, postTokens, cumulativeDroppedTokens, durationMs, preservedMessages…}}`
2. a following `{"type":"user","isCompactSummary":true}` carrying the summary text

`compactMetadata.trigger` is `"auto"` or `"manual"` — that field is the marker used.

`context-timeseries.csv` — 20,702 rows, one per turn:
`session_id, project, turn_index, timestamp, input_tokens, context_pct, compaction_flag, assigned_window, model, output_tokens`.

| Metric | Value | Tag |
|---|---|---|
| Assistant turns (deduplicated by `requestId`) | 20,702 | M |
| Compaction events | 156 (82 auto, 74 manual) | M |
| **Auto-compaction trigger point, tight cluster (80 of 82)** | **165,690 – 182,900 tokens; median 167,879** | **M** |
| — expressed against a 200 K window | **82.8% – 91.5%; median 83.9%** | D |
| Auto-compactions *before* 2026-08-29 in the entire corpus | **1** | M |
| — that one event | 997,170 tokens = **99.7% of 1 M**, session e0497c67, 2026-08-22 | M |
| Unexplained outlier | 570,654 tokens, session 29da694d, 2026-08-30 | M |
| Median turns between consecutive compactions | 68 (min 16, max 555) | M |
| Compaction wall-clock duration | median 138 s, max 298 s | M |
| Summary text size | median 20,103 chars ≈ 5,025 tokens (chars/4), max 77,336 chars | M/D |
| Context immediately **before** auto-compaction | median 165,907 | M |
| Context immediately **after** auto-compaction | **median 72,297; min 52,766; max 96,628** | M |
| Context immediately after *manual* compaction | median 61,250; **min 46,470** | M |
| Headroom reclaimed per auto-compaction | median 92,097 tokens | D |
| Median context across all 20,668 turns | **193,050 tokens** | M |
| Turns below 40,000 tokens (= 20% of 200 K) | **36 of 20,668 (0.2%)** | M |

**This resolves the "84%" question.** [D] Your auto-compactions fire at 84% of *200,000*,
not 84% of the model window, and only because your PowerShell wrapper sets
`--autocompact 200k`. Before that wrapper existed (2026-08-29), the corpus contains
exactly one auto-compaction, at 99.7% of the 1 M window — precisely the documented
default behaviour of "compacts when the conversation reaches the model's context limit."
The data and the docs agree completely.

Per-session band fractions (fraction of turns under 20% / 20–50% / 50–75% / >75% of each
session's assigned window) are in the terminal output and reproducible from
`context-timeseries.csv`; turn-weighted across 38 sessions with >10 turns:
0.358 / 0.436 / 0.161 / 0.045 [D].

### Phase 3 — quality proxies

`quality-proxies.csv` (126 rows) holds both banding schemes with per-band n and an
`UNDERPOWERED` flag on every band with n < 30. Bands are on **absolute context tokens**;
20 K bands are exactly 10% bands of a 200 K window, 100 K bands are 10% bands of 1 M.

20 K bands (10% of a 200 K window). The 0–20 K band is empty — no turn in the corpus
ever ran that low:

| Band | % of 200 K | n | tool err | bash err | edit retry | user corr | out tok | re-read |
|---|---|---|---|---|---|---|---|---|
| 20–40 K | 10–20% | 44 | 9.09% | 5.00%* | — | 0.00%* | 732 | 0.00%* |
| 40–60 K | 20–30% | 352 | 3.98% | 4.38% | 30.77%* | 13.64%* | 835 | 24.14% |
| 60–80 K | 30–40% | 1,247 | 3.53% | 4.54% | 44.59% | 21.19% | 885 | 51.43% |
| 80–100 K | 40–50% | 1,859 | 4.20% | 5.09% | 44.25% | 16.48% | 1,019 | 43.37% |
| 100–120 K | 50–60% | 1,878 | 3.35% | 4.15% | 33.49% | 16.25% | 1,215 | 47.03% |
| 120–140 K | 60–70% | 1,830 | 2.79% | 3.85% | 35.52% | 22.95% | 1,047 | 43.65% |
| 140–160 K | 70–80% | 1,904 | 2.99% | 4.84% | 37.97% | 16.47% | 1,114 | 47.80% |
| 160–180 K | 80–90% | 1,141 | 2.54% | 3.25% | 32.27% | 17.31% | 1,146 | 37.04% |
| 180–200 K | 90–100% | 805 | 4.35% | 6.03% | 31.78% | 20.00% | 1,245 | 44.23% |
| 200 K+ | >100% | 10,345 | 2.71% | 3.90% | 28.03% | 11.12% | 1,154 | 30.24% |

`*` = that proxy had n < 30 in that band (underpowered, not a result).

100 K bands (10% of the 1 M window), which is where the real dynamic range lives:

| Band | n (tool) | tool err | bash err | edit retry | user corr | out tok | re-read |
|---|---|---|---|---|---|---|---|
| 0–100 K | 3,502 | 4.00% | 4.84% | 43.98% | 17.92% | 951 | 43.45% |
| 100–200 K | 7,558 | 3.11% | 4.29% | 34.77% | 19.09% | 1,141 | 44.61% |
| 200–300 K | 3,765 | 2.60% | 4.20% | 29.41% | 15.34% | 1,120 | 35.31% |
| 300–400 K | 2,655 | 2.37% | 3.57% | 28.91% | 9.60% | 1,072 | 33.09% |
| 400–500 K | 1,838 | 4.03% | 5.01% | 27.50% | 9.72% | 1,157 | 26.80% |
| 500–600 K | 959 | 2.29% | 3.31% | 23.92% | 7.10% | 1,217 | 21.93% |
| 600–700 K | 484 | 2.89% | 3.80% | 19.35% | 11.36% | 1,297 | 3.77% |
| 700–800 K | 330 | 1.52% | 0.74% | 41.11% | 17.78% | 1,485 | 30.00% |
| 800–900 K | 235 | 1.70% | 2.74% | 19.05% | 0.00%* | 1,344 | 59.09%* |
| 900 K–1 M | 79 | 0.00% | 0.00% | 0.00%* | 6.25%* | 2,202 | 0.00%* |

**Correlations** (`correlations.csv`, `within-session-correlations.csv`). Pearson on
per-observation values against absolute context tokens; binary outcomes make this a
point-biserial correlation, which is the same statistic. p-values are two-sided from the
t-distribution (regularized incomplete beta, implemented locally — scipy is not installed).

| Proxy | n | r | p | partial r (control turn index) | within-session r | within p | verdict |
|---|---|---|---|---|---|---|---|
| Tool error rate | 21,405 | −0.0202 | 3.1e−03 | −0.0207 | −0.0165 | 1.6e−02 | improves |
| Bash error rate | 9,862 | −0.0223 | 2.7e−02 | −0.0228 | −0.0242 | 1.7e−02 | improves |
| Edit retry rate | 5,294 | −0.0928 | 1.4e−11 | −0.0905 | −0.0545 | 8.9e−05 | improves |
| User correction rate | 1,909 → 1,075 | −0.1071 | 2.8e−06 | −0.1132 | −0.0695 | 4.3e−02 | improves |
| Output tokens/turn | 20,668 | +0.0385 | 3.0e−08 | +0.0379 | +0.0140 | 4.6e−02 | gets longer |
| Re-read rate | 2,171 | −0.1451 | 1.1e−11 | −0.1402 | −0.0652 | 3.8e−03 | improves |

**Stated plainly: there is no proxy in this dataset that gets worse as context fills.**
Every coefficient is statistically significant only because n is in the thousands — the
largest |r| is 0.145, which explains 2.1% of variance. The honest reading is *flat*, not
*improving*: these proxies are essentially independent of context size, with a weak
negative tilt.

The within-session column is the strongest available test: it demeans both variables
inside each session, so it cannot be explained by long sessions being different sessions.
The result does not change.

**Proxy-by-proxy caveats:**

- **Bash exit codes are not stored.** [M] A Bash `toolUseResult` contains only
  `stdout`, `stderr`, `interrupted`, `isImage`, `noOutputExpected` — there is no
  `exitCode` field. "Bash failure rate" is therefore the `is_error: true` flag on Bash
  tool results, not an exit code. Only 375 results carried a parseable `Exit code N`
  string in their error text.
- **User corrections.** 2,107 raw user records reduce to **1,075 genuine typed turns**
  after removing 784 system-only records (`<task-notification>`, `<local-command-stdout>`,
  `<command-name>`, `[Request interrupted…]`, hook injections), 76 slash commands and
  108 pastes >3,000 chars. 79 (7.3%) flagged as corrections. **A first attempt without
  this filter was wrong** — it classified `<task-notification>` blocks as user
  corrections. Hand spot-check of 20 random classified turns: **0 false positives**, but
  ~4 false negatives ("forget this im using claude design", "nvmmm just open in
  localhost i dont want you controlling the tab", "ahh it looks terrible", "yk you could
  have created diff agents for all of those changes"). So precision is high and recall is
  roughly 20–25% — the keyword list under-counts corrections but does not invent them.
- **Re-read rate is confounded with compaction.** The "already read this file" set is
  never reset at a compaction boundary, so a session that compacts often will re-read
  more. That inflates the re-read rate for exactly the strategy the hypothesis favours.

### Phase 4 — compaction strategies compared

Sessions grouped by observed regime [D]:

| Regime | Sessions | Turns | tool err | bash err | edit retry | user corr | out tok | re-read |
|---|---|---|---|---|---|---|---|---|
| **A** auto-compacts at ~167 K (the 200 K budget) | 7 | 10,704 | 3.25% | 4.64% | 39.36% | 9.92% | 1,046 | 47.61% |
| **B** ran deep (>300 K), rarely compacted | 17 | 9,477 | **2.70%** | **3.41%** | **24.92%** | **5.39%** | 1,189 | **26.68%** |
| **C** manual compaction only | 2 | 206 | 5.88% | 3.51% | n<30 | n<30 | 1,078 | n<30 |
| **D** short, never compacted | 19 | 315 | 6.03% | 6.91% | 38.24% | n<30 | 1,407 | 6.25% |

**Regime B — running deep without compacting — is better on every single proxy.** This is
the direct opposite of the hypothesis. See the caveats section: A and B are not randomly
assigned, and this comparison is confounded.

**Overhead of compacting more often:**

| Regime | Hours | Compactions | per hour | Summarizer input tokens/hour | Compaction minutes/hour |
|---|---|---|---|---|---|
| A (auto @167 K) | 1,771.6 | 115 | 0.065 | **15,669** | 0.16 |
| B (ran deep) | 2,063.6 | 39 | 0.019 | 8,737 | 0.04 |
| C (manual only) | 25.0 | 2 | 0.080 | 22,557 | 0.17 |

Regime A compacts **3.4× as often** and spends **1.8× the summarizer input tokens per
hour** as regime B. Each compaction feeds the full pre-compaction conversation (median
165,907 tokens) back through the model to produce a ~5,000-token summary, and blocks for
a median of 138 seconds.

**Rework after compaction** — the real cost:

| Measure | Value | Tag |
|---|---|---|
| Re-read rate within 10 turns after a compaction | **53.9%** (n = 217) | M |
| Re-read rate for all other reads | **35.9%** (n = 1,954) | M |
| Difference | **+18.0 pp**, z = 5.20, **p = 2.0 × 10⁻⁷** | D |

**Answer to the key question.** The data can support an answer, and it is: **no.** There
is no measurable quality gain to weigh against the overhead, because no proxy degrades
with context in the first place. There *is* a measured overhead — 1.8× summarizer tokens
per hour, 3.4× as many 138-second stalls, and a statistically strong 18-point jump in
redundant file re-reading immediately after each compaction. A cost with no measured
benefit.

Separately, "stay under 20%" is not executable at a 200 K window: the floor is 24.8% and
the post-compaction minimum is 23.2% of 200 K.

---

## 3. What could NOT be measured, and why

1. **Actual output quality.** There is no ground-truth quality signal in the transcripts.
   Everything in Phase 3 is a proxy for quality, and proxies can be blind. A model that
   is subtly worse at reasoning — shallower analysis, weaker architecture calls, missed
   edge cases — while still issuing syntactically valid tool calls would be **invisible**
   to all six of these. This is the single largest limitation of the study and it cuts
   both ways: it cannot confirm the hypothesis, and it cannot rule it out.
2. **Bash exit codes.** Not recorded in the transcript format (see above). Would need
   `exitCode` persisted in `toolUseResult`.
3. **A controlled comparison.** Compaction strategy was never randomised. Regime A is
   almost entirely post-2026-08-29 sessions (many of them autonomous overnight runs
   with heavy subagent traffic); regime B is older interactive sessions. Task mix, model
   version, and era all differ alongside the strategy. **The A-vs-B table cannot support
   a causal claim.** A real answer needs an A/B test: alternate `--autocompact` values
   across matched sessions on similar work.
4. **Exact token counts for CLAUDE.md / memory / hook output.** chars/4 approximation
   only; no Claude tokenizer available locally. Error is plausibly ±20%.
5. **The split of the ~43 K remainder** between system prompt, built-in tool schemas,
   skill listing, and agent listing. Derived by subtraction as one lump. Separating it
   would need per-component A/B runs (e.g. launching with skills disabled).
6. **Subagent context.** 1,974 subagent transcripts were excluded entirely. Subagent work
   runs in its own window and never enters the main context, so it does not affect the
   floor or growth — but if quality degrades inside subagents, this study cannot see it.
7. **Two compaction outliers.** The 570,654-token auto-compaction in session 29da694d
   (2026-08-30) is unexplained — it fits neither the 200 K nor the 1 M regime. UNKNOWN.
8. **Whether the wrapper was active per session.** Inferred from date and from observed
   compaction behaviour, not recorded in the transcript. The launch command line is not
   persisted. [A]
9. **Cache-hit economics.** `cache_read` vs `cache_creation` split was collected but not
   analysed for cost; compaction invalidates the prompt cache, which is an additional
   real cost of compacting that this study did not quantify.

---

## 4. Assumptions, stated explicitly

1. Context size at a turn = `input_tokens + cache_creation_input_tokens +
   cache_read_input_tokens`. This is total prompt size regardless of cache state.
2. One "turn" = one unique `requestId`. Multiple assistant records sharing a `requestId`
   are one API call; the maximum usage value across them is taken.
3. `compactMetadata.trigger` is authoritative for auto vs manual.
4. `compactMetadata.postTokens` (median 20,966) is **not** on the same scale as API usage
   and was not used as a context measure — the first real post-compaction turn's usage
   (median 72,297) was used instead. The discrepancy is noted, not explained.
5. A session's model context window is 1,000,000 if any turn exceeded 200,000 tokens,
   else 200,000. Used only for the `assigned_window` / `context_pct` columns; all Phase 3
   analysis uses absolute tokens instead.
6. Edit retry = the same `file_path` edited again within the next 3 turns. Legitimate
   multi-step edits to one file are counted as retries; this inflates the absolute rate
   but should not bias its relationship with context.
7. Re-read = a `Read` on a `file_path` already read earlier in the same session, not
   reset at compaction boundaries.
8. Correction keywords: no, nope, wrong, incorrect, actually, revert, undo, stop, that's
   not, i said, doesn't/didn't work, still broken/failing/wrong, not what, you
   broke/missed/forgot, but you, why did you.
9. chars/4 for token approximation of files.
10. The 52 main-thread JSONL files are the complete session record. `cleanupPeriodDays`
    is 99999, so nothing should have been pruned.
11. Wall-clock session duration = last timestamp − first timestamp, which counts
    overnight idle time as "work". This inflates hours and therefore *understates*
    per-hour compaction overhead in both regimes.

---

## 5. The three strongest arguments against the hypothesis

**1. The floor makes the 20% target arithmetically impossible.** A fresh session costs
49,553 tokens (median), which is 24.8% of a 200 K window before a single word is typed.
The lowest context ever reached after a compaction across 151 measured events is 46,470
tokens. Only 36 turns out of 20,668 (0.2%) ever sat below 40 K. The proposed operating
band is below the machine's own floor. Any policy built on "stay under 20% of 200 K"
would compact continuously and never reach its target.

**2. Every proxy points the other way, and the strongest test doesn't rescue it.** Six
proxies, 20,702 turns, and not one degrades with context — including under a
within-session fixed-effects test that removes all between-session confounding. Two of
these proxies are *mechanically biased toward* the hypothesis and still contradict it:
re-read rate should rise with context simply because more files have been read by then,
and edit-retry rate should rise because more edits have accumulated. Both fall.

**3. The measured cost of compacting is real while the benefit is not.** Compaction is
not free: +18.0 pp of redundant file re-reading in the 10 turns afterwards
(p = 2.0 × 10⁻⁷), a 138-second median stall, ~165,907 tokens fed to the summarizer each
time, and prompt-cache invalidation. The aggressive regime already spends 1.8× the
summarizer tokens per hour of the deep regime. Quadrupling the compaction rate multiplies
a demonstrated cost in exchange for a benefit that six independent measures cannot detect.

**A fourth point, in the hypothesis's favour, for fairness:** the null result rests
entirely on mechanical proxies. Reasoning quality is exactly the thing they cannot see,
and it is exactly the thing you noticed when you set the 200 K window on 2026-08-29. This
study shows your *tooling* does not get worse in long context. It cannot show that your
*thinking* doesn't. If you want that answered, it needs a deliberate A/B — same task,
matched sessions, different `--autocompact` values, and a human judging the output.

---

## 6. Files produced

All in `project-06\context-decay-study\`:

- `context-timeseries.csv` — 20,702 rows, one per turn (Phase 2)
- `quality-proxies.csv` — 126 rows, both banding schemes with per-band n and power flags (Phase 3)
- `correlations.csv` — pooled + partial correlations per proxy
- `within-session-correlations.csv` — fixed-effects correlations per proxy
- `findings.md` — this file

Analysis scripts are in the session scratchpad (`scan.py`, `analyze1.py`, `analyze2.py`,
`analyze3.py`, `statsutil.py`) and re-run from the raw JSONL without network access.
