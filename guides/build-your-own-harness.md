# Build Your Own Harness

### What the scaffolding around a model actually does, why it moves scores more than the model does, and how to build one with Claude that you can point at any model afterwards

*For people who have used AI tools seriously and now want to build the thing underneath them. Assumes you can read Python and have called an API. Assumes nothing else.*

---

## Part I — What a harness is

### 1. The engine and the drivetrain

A language model does exactly one thing: given a sequence of tokens, it predicts the next one. That is the whole product. It has no memory between calls, no ability to act on the world, no clock, no filesystem, no way to check its own work, and no idea whether it has finished.

Everything you think of as "the AI" — the fact that Claude Code can edit your repository, that an agent runs for forty minutes and reports back, that a chatbot remembers your name — is **not the model**. It is the scaffolding around the model. That scaffolding is the harness.

```
┌────────────────────────────────────────────────────────────────┐
│  THE HARNESS                                                   │
│                                                                │
│   ┌────────────┐    ┌────────────┐    ┌──────────────────┐     │
│   │  CONTEXT   │    │   TOOLS    │    │   VERIFICATION   │     │
│   │  assembly  │    │   layer    │    │      gate        │     │
│   └─────┬──────┘    └─────┬──────┘    └────────┬─────────┘     │
│         │                 │                    │               │
│         ▼                 ▼                    ▼               │
│   ╔══════════════════════════════════════════════════════╗     │
│   ║                    THE MODEL                         ║     │
│   ║   stateless · no hands · no clock · no memory        ║     │
│   ║   input: tokens          output: tokens              ║     │
│   ╚══════════════════════════════════════════════════════╝     │
│         ▲                 ▲                    ▲               │
│         │                 │                    │               │
│   ┌─────┴──────┐    ┌─────┴──────┐    ┌────────┴─────────┐     │
│   │   STATE    │    │  BUDGETS   │    │  OBSERVABILITY   │     │
│   │  & memory  │    │  & guards  │    │    & traces      │     │
│   └────────────┘    └────────────┘    └──────────────────┘     │
│                                                                │
│                     ┌──────────────┐                           │
│                     │  THE LOOP    │  ← binds all of it        │
│                     └──────────────┘                           │
└────────────────────────────────────────────────────────────────┘
```

The engine analogy is worth holding onto. A 700-horsepower engine bolted to a go-kart chassis is slower around a track than 400 horsepower in a car with gearing, brakes and steering. Model capability is horsepower. The harness is everything that converts it into motion in a chosen direction.

### 2. Why this is not a minor implementation detail

Here is a measurement that should reframe how you read every benchmark you see.

Claude Opus 4.6, on SWE-bench Pro. One model. One benchmark. Three harnesses:

```
Scale standardized harness       ████████████████████████████  51.9%
Vendor scaffold (Morph baseline) ██████████████████████████████  55.4%
+ WarpGrep search subagent       ███████████████████████████████ 57.5%
                                 └──────┴──────┴──────┴──────┴──┘
                                 0     15     30     45     60%
```

**A 5.6-point spread from scaffolding alone** — and the search-subagent variant also cut cost 15.6% and latency 28%. The model never changed. Only what surrounded it.

Put that next to the gaps people argue about online. Most cross-model comparisons on coding benchmarks are decided by margins smaller than the harness effect. Which means: **a benchmark number without a named harness is not a measurement of a model.** It is a measurement of a pair, reported as if it were a property of one of them.

Three practical consequences:

1. If you are choosing a model for agentic work, the harness is a lever of comparable size to the model choice — and far cheaper to change.
2. If your agent underperforms, the model is usually not the problem. Context assembly and retrieval usually are.
3. If you build your own harness, you can carry it across models. Your harness is the asset that survives the next release; the model underneath it is a swappable component.

### 3. What a harness is *for*

Three jobs, in order of how often they're neglected:

**Turning a stateless function into a process.** The model call is a pure function. Work is not — work has steps, intermediate results, dead ends and revisions. The loop is what converts one into the other.

**Getting the right things into the window, and nothing else.** Every token in the context competes with every other token. The harness decides what earns a place: which files, which prior turns, which tool results, which instructions. This is the single highest-leverage component and it is where nearly all quality differences originate.

**Knowing when the work is done and whether it is right.** Models will happily report success on unfinished work — not from dishonesty but because "done" is not a thing they can observe. The harness supplies the ground truth: tests, validators, checks, a human gate.

---

## Part II — Anatomy

Nine components. You will not build all nine on day one, and you should not try.

```
  ┌───────────────────────────────────────────────────────────┐
  │ 1  CONTEXT ASSEMBLY   what goes in the window, in what     │
  │                       order, under what budget            │
  ├───────────────────────────────────────────────────────────┤
  │ 2  TOOL LAYER         schemas, execution, sandboxing,      │
  │                       result formatting                   │
  ├───────────────────────────────────────────────────────────┤
  │ 3  THE LOOP           assemble → call → route → observe    │
  │                       → check → repeat                    │
  ├───────────────────────────────────────────────────────────┤
  │ 4  STATE & MEMORY     scratchpad, task ledger, artifacts   │
  │                       on disk, history compaction         │
  ├───────────────────────────────────────────────────────────┤
  │ 5  VERIFICATION       tests, validators, critics, the      │
  │                       definition of done                  │
  ├───────────────────────────────────────────────────────────┤
  │ 6  ERROR RECOVERY     retries, repair prompts, fallbacks,  │
  │                       graceful degradation                │
  ├───────────────────────────────────────────────────────────┤
  │ 7  BUDGETS & GUARDS   token/step/cost caps, permission     │
  │                       gates, human checkpoints            │
  ├───────────────────────────────────────────────────────────┤
  │ 8  OBSERVABILITY      traces, logs, replay, cost accounting│
  ├───────────────────────────────────────────────────────────┤
  │ 9  MODEL ADAPTER      the thin layer that makes 1–8        │
  │                       portable across providers           │
  └───────────────────────────────────────────────────────────┘
```

### The loop, in detail

Everything else hangs off this:

```
         ┌─────────────────┐
    ┌───▶│  1. ASSEMBLE    │  system prompt + task + state
    │    │                 │  + retrieved context + tool schemas
    │    └────────┬────────┘
    │             ▼
    │    ┌─────────────────┐
    │    │  2. CALL        │  one model request
    │    └────────┬────────┘
    │             ▼
    │    ┌─────────────────┐
    │    │  3. ROUTE       │  tool calls? → execute
    │    │                 │  final answer? → go to 6
    │    └────────┬────────┘
    │             ▼
    │    ┌─────────────────┐
    │    │  4. OBSERVE     │  append results, update state,
    │    │                 │  write artifacts to disk
    │    └────────┬────────┘
    │             ▼
    │    ┌─────────────────┐
    └────┤  5. CHECK       │  budget blown? step cap hit?
         │                 │  no progress? → abort
         └────────┬────────┘
                  ▼ (final answer)
         ┌─────────────────┐
         │  6. VERIFY      │  ← the step everyone skips
         │                 │  fails? → back to 1 with the failure
         └────────┬────────┘
                  ▼
              ✓ DONE
```

Two arrows matter more than the rest. **Step 5 back to step 1** is what makes it an agent instead of a chatbot. **Step 6 back to step 1** is what makes it trustworthy instead of merely productive — the model gets its own failure as new input, which is the only mechanism by which it reliably self-corrects.

### Context assembly: the budget nobody sets

Your window is finite and every component wants a piece of it. An unmanaged agent's context looks like this by step fifteen:

```
  UNMANAGED                          BUDGETED
  ┌──────────────────────────┐       ┌──────────────────────────┐
  │ system prompt        3%  │       │ system prompt        5%  │
  ├──────────────────────────┤       ├──────────────────────────┤
  │ tool schemas         4%  │       │ tool schemas         6%  │
  ├──────────────────────────┤       ├──────────────────────────┤
  │                          │       │ task + state ledger 12%  │
  │ raw tool output          │       ├──────────────────────────┤
  │ (every file read,        │       │ retrieved context   30%  │
  │  every grep result,      │  vs   │ (ranked, truncated,      │
  │  every stack trace,      │       │  summarised)             │
  │  never pruned)      78%  │       ├──────────────────────────┤
  │                          │       │ recent turns        27%  │
  │                          │       ├──────────────────────────┤
  ├──────────────────────────┤       │ compacted history   12%  │
  │ actual task          8%  │       ├──────────────────────────┤
  ├──────────────────────────┤       │ headroom             8%  │
  │ history (truncated       │       │                          │
  │ mid-sentence)        7%  │       │                          │
  └──────────────────────────┘       └──────────────────────────┘

  symptom: degrades after ~10 steps   symptom: stable to step 50+
```

The rule that fixes most of it: **tool results are not sacred.** A 4,000-line file read is an input to a decision, not a permanent record. Summarise it, keep the decision, drop the raw text. Write the artifact to disk and keep only the path.

---

## Part III — The guide

### Step 0 — Pick a task shape, not a capability

The single most common failure is building "an agent that can do anything." Generality is what platform vendors sell; it is not what makes your harness good.

Pick something narrow enough that you can write down what *done* means:

- "Given a failing test, produce a patch that makes it pass without breaking others"
- "Given a PR, produce a review that flags anything violating our conventions file"
- "Given a support email, draft a reply grounded only in our docs, with citations"

If you cannot state the done-condition in one sentence, you cannot verify it, and without verification the rest of the harness is decoration.

### Step 1 — Write the contract before the code

Four lines. Do this in a file, not in your head:

```
INPUT:      what the harness receives (types, not prose)
OUTPUT:     what it produces (a patch? a file? a decision + reasoning?)
DONE WHEN:  the observable condition — a test passes, a schema validates
FAILS IF:   the conditions under which it should stop and report failure
```

The "fails if" line is the one people skip and the one that prevents an agent burning forty dollars discovering it cannot do something.

### Step 2 — Build the loop skeleton first, with one dumb tool

Not the whole tool suite. One tool, ideally read-only, and the loop around it. Get the cycle turning before you make it capable.

```python
def run(task: str, max_steps: int = 12, max_cost: float = 2.00):
    state = State(task=task, artifacts={}, log=[])
    messages = [user(task)]

    for step in range(max_steps):
        if state.cost > max_cost:
            return state.fail("budget exceeded")

        reply = model.call(
            system=build_system_prompt(state),
            messages=assemble(messages, state),   # ← Step 4 lives here
            tools=TOOL_SCHEMAS,
        )
        state.cost += reply.cost
        state.log.append(trace(step, reply))      # ← Step 7 lives here

        if reply.tool_calls:
            results = [execute(c, state) for c in reply.tool_calls]
            messages += [reply.raw, tool_results(results)]
            continue

        ok, report = verify(reply, state)         # ← Step 5 lives here
        if ok:
            return state.succeed(reply)
        messages += [reply.raw, user(f"Verification failed:\n{report}\nFix it.")]

    return state.fail("step limit reached")
```

That is a complete agent harness in about twenty-five lines. Everything that follows is making each of those calls better — not making the loop more elaborate.

### Step 3 — Tools: narrow surface, rich results

Three rules that matter more than the tool list:

**Fewer tools, sharper.** Every tool schema costs context and every additional option costs decision quality. Five well-chosen tools beat twenty. If two tools overlap, the model will pick wrong roughly half the time.

**Results are messages to a reader who cannot see your screen.** A tool that returns `{"status": 500}` teaches nothing. One that returns `Failed: connection refused on port 5432. Postgres may not be running. Try: docker compose up -db` produces recovery instead of a retry loop.

**Side effects need gates.** Read freely; write, delete, send, deploy and spend behind an explicit approval — either a rule, or a human. This is not paranoia; it is the difference between an agent you can leave running and one you must watch.

### Step 4 — Context assembly: the actual differentiator

This is where the 5.6 points live. Three techniques, in order of return:

**Retrieve, don't preload.** Do not stuff the repository into the window and hope. Give the agent search, let it pull what it needs, twice if necessary. Retrieval twice costs less than preloading once, and it stays correct as the task drifts.

**Compact aggressively.** After a tool result has been acted on, replace it with a one-line summary of what was learned. Keep decisions, drop evidence. Push artifacts to disk and keep paths.

**Order matters.** Instructions early, retrieved material in the middle, the immediate task last. Models attend most reliably to the beginning and end of a long context; burying the actual request in the middle is a well-documented way to be ignored.

### Step 5 — Verification: the step that separates toys from tools

Ask: *what could check this without a human?*

```
  Task type              Verifier
  ─────────────────────────────────────────────────────────────
  code change            run the test suite; typecheck; lint
  structured output      JSON-schema validation
  factual synthesis      every claim carries a source, checked
                         against retrieved text
  refactor               behaviour tests pass + diff size sane
  document / prose       a second model call scoring against an
                         explicit rubric (a "critic")
  anything ambiguous     a human gate — 30 seconds beats a rebuild
```

The critic pattern is worth internalising because it costs one extra call: send the output back with the original requirements and a rubric, ask for pass/fail per criterion with evidence. **Crucially, feed the failure report back into the loop rather than to the user.** A failure the model can see is a failure it can usually fix.

### Step 6 — State on disk, not in the conversation

Conversations get compacted, truncated and lost. Files don't. Anything the agent must not forget goes to disk:

```
run-2026-07-22-1041/
├── task.md          the original ask, unmodified
├── plan.md          the agent's current plan, rewritten as it changes
├── findings.md      what it has learned — the compacted memory
├── artifacts/       what it produced
└── trace.jsonl      every call, tool, cost and result
```

The agent reads `plan.md` and `findings.md` at the top of each assembly. This is what lets a run survive a compaction, a crash, or a restart tomorrow — and it is the same principle whether you are building an agent or organising a knowledge base for one.

### Step 7 — Observability: log the trace, not the vibe

`trace.jsonl` is not optional. One line per step: inputs, tool calls, results, tokens, cost, latency. Without it you are debugging a nondeterministic system by memory, which does not work.

The minimum useful trace answers: *what was in the context when it made that decision?* Nearly every mysterious agent failure resolves the moment you can read the actual window.

### Step 8 — The eval set: 10–20 tasks, written before you tune

This is the step that separates people who build harnesses from people who fiddle with prompts forever.

Write 10–20 real tasks with known-good outcomes. Store them. Run the whole set after every change. Record pass rate, mean cost, mean steps.

```
  BEFORE                              AFTER
  ├─ change a prompt                  ├─ change a prompt
  ├─ try it once                      ├─ run 20 evals
  ├─ "feels better"                   ├─ pass 13/20 → 16/20
  └─ ship                             ├─ cost +4%
                                      └─ ship, with a number
```

Without an eval set you cannot tell an improvement from a coincidence, and you will spend weeks chasing both.

### Step 9 — Portability: the adapter layer

Now make it model-agnostic. This is roughly 150 lines and it is what turns your harness into an asset.

```
                      ┌────────────────────────┐
                      │     YOUR HARNESS       │
                      │  loop · context · tools│
                      │  verify · state · evals│
                      └───────────┬────────────┘
                                  │
                      ┌───────────▼────────────┐
                      │     ADAPTER LAYER      │
                      │  normalise: messages,  │
                      │  tool schemas, stop    │
                      │  reasons, usage, errors│
                      └───────────┬────────────┘
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
        ┌───────────┐       ┌───────────┐       ┌───────────┐
        │  Claude   │       │    GPT    │       │ open-wt.  │
        │           │       │           │       │ (local /  │
        │           │       │           │       │  hosted)  │
        └───────────┘       └───────────┘       └───────────┘
```

What actually differs between providers, and what your adapter must normalise:

| Concern | Varies how | Adapter's job |
|---|---|---|
| Tool-call format | different JSON shapes and field names | one internal `ToolCall` type |
| System prompt | separate parameter vs first message | one `system=` argument |
| Reasoning tokens | some expose them, some bill for hidden ones | normalise usage accounting |
| Stop reasons | `end_turn` / `stop` / `tool_use` / `length` | one enum |
| Structured output | native modes vs prompt-and-pray | one `expect_json()` path with repair |
| Context limits | differ by an order of magnitude | budget expressed as a fraction, not a constant |
| Rate limits & errors | different codes, different retry semantics | one retry policy with backoff |

**Build against Claude first, then port.** Not out of loyalty — for a practical reason. Strong tool-use and instruction-following mean your early failures are harness bugs rather than model limitations, and you learn faster when the failure has one cause. Once the harness passes its evals, swap the adapter and re-run: your eval set now doubles as a model comparison rig, on *your* tasks, on a constant harness. That is a better signal for your workload than any public leaderboard, because it measures the one pairing you actually deploy.

### Step 10 — Iterate against the evals

Change one thing. Run the set. Keep the number. The ranked order of what usually helps:

```
  1. context assembly      ████████████████████████  biggest lever
  2. verification quality  ███████████████████
  3. tool result phrasing  ██████████████
  4. system prompt         █████████
  5. model choice          ███████
  6. sampling params       ██
```

Model choice is fifth. That is the point of this entire guide.

---

## Part IV — Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Great for 10 steps, incoherent after | context never compacted | summarise tool results; move state to disk |
| Confidently reports done, isn't | no verification step | add the gate; feed failures back into the loop |
| Loops on the same failing action | no progress detection | track repeats; force a strategy change or abort |
| Costs explode unpredictably | no budget in the loop | cap steps *and* dollars; check before each call |
| Picks the wrong tool constantly | overlapping tool surface | merge or delete tools; sharpen descriptions |
| Works on your examples, fails on real ones | eval set written after tuning | write evals from real tasks, before tuning |
| Silent wrong answers | errors swallowed by broad exception handling | log every failure even when you recover from it |
| Breaks on every model update | provider specifics leaked into the loop | push all of it behind the adapter |

Two anti-patterns worth naming separately.

**The harness that hides errors.** Wrapping everything in a bare `except: pass` makes an agent that never crashes and never tells you it failed. Silent failure is the dominant failure mode of every retrieval and agent system — absence has no signature. Log even what you recover from.

**Abstracting too early.** Do not build a plugin architecture for your third tool. Write the concrete thing, get evals passing, then abstract the duplication you actually found. Most agent frameworks are someone else's premature abstraction, which is why they feel wrong on your task.

---

## What to build this week

```
  Day 1   contract + loop skeleton + one read-only tool
  Day 2   3–5 tools with rich error messages, side effects gated
  Day 3   context assembly: retrieval, compaction, ordering
  Day 4   verification gate + failure feedback into the loop
  Day 5   state on disk + trace.jsonl
  Day 6   10–20 task eval set, run it, record the baseline
  Day 7   adapter layer; swap the model; re-run the set
```

At the end of it you own something more durable than any prompt: a measured, portable rig that tells you what a model is worth **on your work**, and that keeps its value when the next release lands.

The models will keep changing. The harness is yours.

---

*Corrections and worked examples welcome — especially eval sets, which are the hardest part to write and the most useful thing to share.*
