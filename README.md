# Claude Playbooks

Working guides and prompts for getting real output out of Claude — the kind you'd hand a colleague, not the kind you'd screenshot.

Everything here comes out of actual use. Each file is a full operating spec rather than a clever one-liner: a defined role, a structure the output has to follow, and the specific constraints that stop a model from producing something fluent and useless. If a technique made it into this repo, it's because it survived a real project.

---

## Guides

Workflows and playbooks. Read these.

| Guide | What it covers |
|---|---|
| [The Autonomous App Build Stack](./guides/autonomous-app-build-stack.md) | Running Cowork, Claude Design, and Claude Code as one pipeline — Cowork orchestrates, Design owns the interface, Code implements. Folder structure, a `CLAUDE.md` template, copy-paste prompts per stage, and where to keep a human in the loop |

## Prompts

Paste into a fresh conversation and run.

| Prompt | What it does |
|---|---|
| [Tech Idea Dissection](./prompts/tech-idea-dissection.md) | Full teardown of a product, app, or service idea — calibrated overall/niche/market-potency scores, ranked strengths and weaknesses, a failure-mode table with likelihoods and leading indicators, and a 30-day plan to falsify the riskiest assumption |

*[Sample output](./examples/) for the dissection prompt, so you can see what it actually produces before running it.*

---

## How to use these

**Guides** — read once, then keep open while you work. Most of the value is in the file structure and hand-off discipline, not the prompt text.

**Prompts** — open a *new* conversation; context bleed from an earlier chat skews the output. Copy the file in full, from the divider down. The rubric and self-audit sections are what keep the model from flattering you, so trimming them to save tokens defeats the point. Read the assumptions block first: if the model inferred the wrong version of your idea, fix it there and re-run rather than arguing downstream.

Model-agnostic in principle. Written and tested against Claude.

---

## Design principles

The rules everything here follows. They're documented so you can fork one and write your own in the same shape.

**Calibration beats enthusiasm.** Anything that produces a score defines the bands explicitly — what a 3 means, what a 9 means, how rare each should be. Without anchors, models cluster everything at 7–8 and the number carries no information.

**Force discrimination.** Rules like "don't assign the same score to more than three dimensions" push the model to rank rather than smear.

**Mechanism over adjective.** "Strong differentiation" is noise. *What* the differentiation is, *why* a user switches because of it, and *how long* it survives a competitor's response is signal. Ban the adjective, demand the mechanism.

**Ban the generic.** "Competition" and "execution risk" are non-findings. Naming them as disqualified answers up front forces specificity.

**Make hedging illegal.** Models default to "this could go either way." One instruction — commit to a position, then state what would change your mind — removes the fence-sitting while keeping the honesty.

**Artefacts over conversation.** For anything long-running, state belongs in files, not chat history. Context windows fill; a spec on disk doesn't drift.

**Keep the human at the expensive decisions.** Automate the typing, not the judgement. Agents don't fail loudly — they lock in a misread requirement early and build faithfully on top of it. A few checkpoints in the right places are what make the rest genuinely autonomous.

**Build in a self-audit.** A short checklist at the end, run before responding, catches score inflation and unsupported numbers better than any amount of instruction earlier in the prompt.

---

## Repo layout

```
guides/     workflows and playbooks
prompts/    paste-and-run prompts
examples/   real outputs, unedited
```

## Roadmap

- **Architecture review prompt** — system design against scale, cost, failure modes, operational burden
- **Spec-writing guide** — turning a vague brief into something an agent can build from without guessing
- **Document critique prompt** — hostile review of a PRD or strategy doc before a stakeholder sees it
- **Competitive positioning prompt** — where a product sits against a named comparison set
- **Long-running agent guide** — keeping context coherent across multi-day work

## Contributing

Issues and PRs welcome, especially:
- Outputs where something failed or produced flattery — the most useful bug reports here
- Rubric refinements that improve calibration
- Worked examples where a guide's workflow broke down in practice

## License

MIT — use, fork, and adapt freely. Attribution appreciated, not required.
