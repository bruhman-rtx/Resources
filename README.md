# Prompts

A working library of long-form prompts I actually use — built for depth, calibration, and honest output rather than agreeable output.

Most prompt collections are one-liners. These aren't. Each one is a full operating spec: a defined role, a scoring rubric with anchors, a required output structure, and a self-audit step that runs before the model answers. They're written to be pasted into a fresh conversation with any frontier model (Claude, GPT, Gemini) and to produce something you could take into a real meeting.

---

## The Library

| Prompt | What it does | Best for |
|---|---|---|
| [Tech Idea Dissection](./prompts/tech-idea-dissection.md) | Teardown of any product, app, or service idea — overall rating, niche rating, market potency score, ranked strengths and weaknesses, and a failure-mode table with likelihoods and leading indicators | Deciding whether to build, narrow, test, or kill an idea |

*More being added — see [Roadmap](#roadmap).*

---

## How to use these

1. Open a **new** conversation. Context bleed from an earlier chat will skew the output.
2. Copy the prompt file **in full**, from the divider down. The rubric and self-audit sections are what keep the model from flattering you — deleting them to save tokens defeats the purpose.
3. Fill the input block. Blank fields are fine; a good prompt states its own assumptions rather than interrogating you.
4. Read the assumptions block first. If the model inferred the wrong idea, correct it there and re-run rather than arguing downstream.

---

## Design principles

These are the rules every prompt in this repo follows. They're here so you can fork one and write your own in the same shape.

**Calibration beats enthusiasm.** Any prompt that produces a score defines the bands explicitly — what a 3 means, what a 9 means, and roughly how rare each should be. Without anchors, models cluster everything at 7–8 and the number carries no information.

**Force discrimination.** Rules like "don't assign the same score to more than three dimensions" push the model to actually rank rather than smear.

**Mechanism over adjective.** "Strong differentiation" is noise. *What* the differentiation is, *why* a user switches because of it, and *how long* it survives a competitor's response is signal. Prompts should ban the adjective and demand the mechanism.

**Ban the generic.** "Competition" and "execution risk" are non-findings. Naming them as disqualified answers up front forces specificity.

**Make hedging illegal.** Models default to "this could go either way." A single instruction — commit to a position, then state what would change your mind — removes the fence-sitting while keeping intellectual honesty.

**Build in a self-audit.** A short checklist at the end of the prompt, run silently before responding, catches score inflation and unsupported numbers better than any amount of instruction earlier in the prompt.

**Structure the output.** Specify sections and order. Free-form responses drift toward summary; structured ones stay analytical.

---

## Roadmap

Prompts in progress:

- **Architecture review** — teardown of a system design against scale, cost, failure modes, and operational burden
- **Competitive positioning** — where a product sits against a named comparison set, and the axis worth fighting on
- **Hiring scorecard** — role spec, interview loop, and calibrated evaluation rubric from a job description
- **Investment memo** — structured evaluation of a fund, instrument, or allocation decision
- **Document critique** — hostile review of a PRD, strategy doc, or proposal before it goes to a stakeholder

---

## Contributing

Issues and PRs welcome, especially:
- Output samples where a prompt failed or produced flattery — those are the most useful bug reports
- Rubric refinements that improve calibration
- Model-specific variants where behaviour differs meaningfully

---

## License

MIT — use, fork, and adapt freely. Attribution appreciated, not required.
