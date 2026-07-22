# The Autonomous App Build Stack

**Cowork orchestrates. Claude Code builds. Claude Design draws.**

A practical guide to running all three Claude surfaces as one pipeline — brief in, working application out, with you supervising rather than typing.

*Verified against product behaviour as of July 2026. Capabilities move fast; check [docs.claude.com](https://docs.claude.com) and [support.claude.com](https://support.claude.com) before relying on any specific detail.*

---

## The idea in one paragraph

Most people use one Claude surface for everything and hit a ceiling. They ask Claude Code to design a UI and get competent-but-generic markup. They ask chat to plan a product and end up re-explaining the plan to a coding agent an hour later. The ceiling isn't the model — it's that each hand-off between stages loses fidelity, and you're the one carrying it. The fix is to stop being the courier. Give each surface the job it's actually built for, make them write to the same filesystem, and let the artefacts carry the context instead of your memory.

| Surface | What it is | Its job in this pipeline |
|---|---|---|
| **Claude Cowork** | Agentic knowledge-work app in Claude Desktop. Reads and writes local files, uses subagents, runs long tasks, and schedules work. GA on all paid plans since April 2026; also on web and mobile for Max | **The conductor.** Turns a vague brief into a spec, keeps the plan current, sequences the work, chases loose ends, reports status |
| **Claude Design** | Canvas with real design tools, iterated through chat | **The eye.** Screens, states, layout, hierarchy, visual system — before a line of UI code exists |
| **Claude Code** | Agentic coding agent in terminal, IDE, desktop, and mobile | **The hands.** Implements, tests, refactors, runs the build, wires the deploy |

The unlock is not that these are three models. It's that **Cowork and Claude Code operate on the same local filesystem.** Files are the integration layer. Everything below is built on that one fact.

---

## The pipeline

```
   BRIEF
     │
     ▼
┌─────────────┐   spec.md
│   COWORK    │──────────────┐
│  (conduct)  │              │
└─────────────┘              ▼
     │                ┌─────────────┐   /design/*.png
     │                │   DESIGN    │──────────────┐
     │                │   (draw)    │   design.md  │
     │                └─────────────┘              ▼
     │                                      ┌─────────────┐
     │                                      │    CODE     │
     │◀─────────────────────────────────────│   (build)   │
     │       status, gaps, next tasks       └─────────────┘
     ▼                                             │
  ── GATE 1 ──  scope approved                     ▼
  ── GATE 2 ──  design approved                 WORKING APP
  ── GATE 3 ──  before deploy
```

Three human gates. That's deliberate — see [The gates](#the-gates-where-you-stay-in-the-loop). Everything between them runs unattended.

---

## Set up the workspace first

This takes five minutes and it is the difference between a pipeline and a mess. One folder, granted to Cowork, opened by Claude Code.

```
myapp/
├── CLAUDE.md              ← the spine. Read by Claude Code every session
├── docs/
│   ├── brief.md           ← your original ask, unedited
│   ├── spec.md            ← Cowork's output. The contract
│   ├── decisions.md       ← running log of choices + why
│   └── status.md          ← Cowork keeps this current
├── design/
│   ├── design-system.md   ← tokens, type scale, spacing, components
│   └── screens/           ← exports from Claude Design
├── src/                   ← Claude Code's territory
└── tests/
```

**Why this shape works:** Cowork owns `docs/`, Design fills `design/`, Claude Code owns `src/` and `tests/`. Nobody overwrites anybody. Every stage reads what the previous stage left on disk, so no context has to survive in a chat window — which matters, because Cowork sessions don't carry memory across sessions by default.

Grant Cowork access to the `myapp/` folder. Open the same folder in Claude Code. They're now working on the same thing.

---

## Stage 1 — Cowork writes the spec

**Don't start with code. Start by making the vague thing specific.**

Open Cowork, point it at the folder, and give it the brief. Ask for interrogation, not agreement:

> Read `docs/brief.md`. Before writing anything, list the 10 decisions this brief leaves undecided, ranked by how expensive they'd be to change after we build. Ask me only the 5 that matter most. Then write `docs/spec.md` containing: the user and the job they're hiring this for, the 5–8 screens, the data model with field types, every user flow as numbered steps, what is explicitly out of scope for v1, and the acceptance criteria for each flow — written so a test could check them. Record every choice and its reasoning in `docs/decisions.md`.

Two details that make this work:

**Ask for the out-of-scope list.** It's the single highest-leverage line in the spec. Without it, scope grows silently during the build and you discover it at deploy time.

**Demand testable acceptance criteria.** "Users can log in" is not testable. "A user with a valid email receives a magic link within 30s and lands on `/dashboard`" is. This is what Claude Code will verify against later, so vagueness here becomes bugs there.

📍 **GATE 1 — read `spec.md` yourself.** Ten minutes now saves a rebuild. You are checking one thing: *did it understand what I meant, or what I said?*

---

## Stage 2 — Claude Design does the interface

Skipping this stage is the most common mistake. If you go straight from spec to Claude Code, you get an app that works and looks like every other AI-built app — because a coding agent optimises for correctness, not for hierarchy, rhythm, or restraint.

Give Design the spec, not a vibe:

> Here is `docs/spec.md`. Design the 6 screens it describes for a [desktop-first / mobile-first] product. Establish a design system first: type scale, spacing scale, colour roles, and states for buttons, inputs, and cards. Then lay out each screen, including empty, loading, and error states. Keep it restrained — a strong type hierarchy and generous spacing rather than decoration.

Iterate on the canvas until it's right. Then — the step people miss — **export it as instructions, not just pictures:**

> Write `design/design-system.md` as implementation-ready specification: exact tokens for colour, type, spacing, radius, and shadow; component anatomy; and per-screen layout notes with the responsive behaviour at each breakpoint. Write it for an engineer who cannot see the designs.

That file is what makes design survive the hand-off. Screenshots alone get approximated; a token table gets implemented.

📍 **GATE 2 — approve the designs.** Cheapest change you'll ever make is here.

---

## Stage 3 — Claude Code builds it

Before the first build command, write `CLAUDE.md`. Claude Code reads it every session, so this is your standing instruction set, not something you repeat:

```markdown
# Project: [name]

## Sources of truth
- Requirements: docs/spec.md — never contradict it
- Visual system: design/design-system.md — implement exactly, no substitutions
- Decisions: docs/decisions.md — append every non-obvious choice

## Stack
[framework, database, auth, hosting, package manager]

## Rules
- Every feature ships with tests covering the acceptance criteria in spec.md
- Run typecheck, lint, and tests before saying anything is done
- Never invent a UI pattern not present in design-system.md
- Anything ambiguous: append it to docs/questions.md and continue with the
  most reversible option. Do not guess silently
- Never commit secrets. Environment variables via .env.local only

## Definition of done
Typecheck passes · tests pass · matches the design tokens · acceptance
criteria in spec.md verified · docs/status.md updated
```

Then work in vertical slices, not layers:

> Read CLAUDE.md, docs/spec.md, and design/design-system.md. Implement flow 1 end-to-end — data model, API, UI, and tests — exactly as specified. Verify it against the acceptance criteria before reporting done. Then update docs/status.md.

One flow per session, front to back. A working slice teaches you something; a half-finished data layer teaches you nothing. And each completed slice gives Cowork something real to track.

**Two things to hand back to Cowork mid-build:**
- `docs/questions.md` — every ambiguity Claude Code hit. Cowork resolves these against the spec instead of you fielding them one at a time.
- `docs/status.md` — what's done, what's next.

---

## Stage 4 — Cowork closes the loop

This is the stage that makes it a pipeline rather than three tools in a row. Back in Cowork:

> Read `docs/spec.md` and `docs/status.md`, then inspect `src/` and `tests/`. Produce `docs/gap-report.md`: every acceptance criterion in the spec, marked implemented / partial / missing, with the file and line as evidence. Then resolve everything in `docs/questions.md` against the spec, and write the next three tasks for Claude Code in priority order.

Take that gap report straight back to Claude Code. Repeat until the report is clean.

**Cowork is also where the non-code work happens** — the part people forget is on the critical path. It can draft the README, the landing copy, the privacy policy, the onboarding emails, and the launch post, all from the same spec, with the same understanding of the product. It can pull from your connected Drive or Slack for context. And it can run on a schedule: a nightly task that re-runs the gap report and leaves it waiting for you in the morning.

📍 **GATE 3 — before deploy.** Read the gap report. Check secrets aren't committed. Check destructive operations are reversible. Then ship.

---

## The gates: where you stay in the loop

The pitch for this stack is "autonomous," and the stages genuinely are. The pipeline is not. That's the design, not a limitation.

An agent without a human checkpoint doesn't fail loudly — it fails by locking in a misread requirement in minute two and building faithfully on top of it for the next forty. It has no way to detect the divergence, because self-written tests verify the system's own interpretation of the goal, not yours. Cowork itself is built on this principle: it completes the work, but consequential decisions stay with you.

So put the gates where errors are cheapest to catch and most expensive to miss:

| Gate | What you're checking | Time | Cost of skipping |
|---|---|---|---|
| **1 — Spec** | Did it understand my intent? | 10 min | Rebuild |
| **2 — Design** | Do I want to use this? | 10 min | Refactor of every screen |
| **3 — Deploy** | Is anything here irreversible or exposed? | 15 min | Leaked keys, lost data |

35 minutes of attention across a build that otherwise runs itself. Everything between gates is genuinely hands-off — go do something else.

---

## Common failure modes

| What goes wrong | Why | Fix |
|---|---|---|
| Output drifts from the spec over a long build | Chat context fills; earlier constraints fall out | Sources of truth live in files, not conversation. Re-read `spec.md` at the top of each session |
| The UI is generic despite good designs | Screenshots were shared, but not tokens | `design-system.md` with exact values. Coding agents implement tables, not impressions |
| It reports done, but flows are broken | "Done" meant "compiles" | Definition of done in `CLAUDE.md`, plus the gap report in Stage 4 |
| Scope quietly triples | No out-of-scope list | It's in `spec.md` for a reason. Point at it |
| Second and third features break the first | Layer-first building, no regression tests | Vertical slices with tests per slice |
| You're re-explaining the project constantly | Context is in your head, not on disk | `CLAUDE.md` + `docs/`. Sessions are disposable; files aren't |
| Agent silently guessed wrong | No route for ambiguity | `docs/questions.md`, resolved by Cowork against the spec |

---

## Quick start

1. Create the folder structure. Grant Cowork access to it, open it in Claude Code.
2. Write `docs/brief.md` — one page, plain language, no formatting effort required.
3. **Cowork:** interrogate the brief → `spec.md` + `decisions.md`. **← Gate 1**
4. **Design:** design system → screens → `design-system.md`. **← Gate 2**
5. Write `CLAUDE.md`.
6. **Claude Code:** one vertical slice at a time, tests included.
7. **Cowork:** gap report → resolve questions → next tasks. Loop until clean.
8. **← Gate 3**, then deploy.
9. **Cowork:** README, landing copy, launch post — from the same spec.

---

## What this is not

It is not a replacement for knowing what good software looks like. The pipeline compresses the distance between intent and artefact; it does not supply the intent, and it will build a well-tested, beautifully designed version of a bad idea without hesitating.

It is also not zero-touch. Anyone selling fully unattended app generation is selling the demo, not the second project. The honest claim is better: **35 minutes of judgement replaces three weeks of typing.** That's the actual win, and it's a large one.

---

*Corrections and improvements welcome — particularly worked examples where a stage broke down. Those are the most useful contributions.*
