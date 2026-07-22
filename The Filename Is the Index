# The Filename Is the Index

### What a 359,438-word knowledge base built for a machine reveals about writing for readers that cannot remember

---

## I. An artifact with the wrong reader

There is a folder on a Windows machine called *Claude's Brain*. It contains 281 markdown notes and 359,438 words — roughly four novels — organised into a dozen directories, cross-linked by 1,318 wikilinks, and indexed by a 22-megabyte vector store. By every surface measure it is an ordinary Obsidian vault, the kind tens of thousands of people maintain.

It is not ordinary in one respect, and that respect changes everything about it: **its primary reader is not a human being.**

A person writes into it occasionally, by hand. A language model reads from it on nearly every prompt and writes back to it automatically whenever its context window compacts. The human is the minority contributor and a rare reader. The machine is the daily consumer. And once you accept that inversion, a long series of conventions that look like personal-knowledge-management taste turn out to be load-bearing engineering — while a second series of conventions that look like best practice turn out to be doing precisely nothing.

What makes this vault worth writing about is not that it exists. Plenty of people are now pointing coding agents at their notes. What makes it worth writing about is that it was **measured**. Someone ran the scripts: counted the notes, profiled the word distribution, parsed the frontmatter, resolved the link graph, read the retrieval hook line by line, validated the embedding index, and grepped every stale claim to see whether it was actually stale. The result is an unusually honest picture of the gap between what a knowledge system is documented to do and what it actually does — and that gap, it turns out, is where all the interesting information lives.

This essay is about what the measurement found. Some of it is specific to one setup. Most of it is not.

---

## II. The founding constraint: a hard drive for a machine with no RAM

Every coherent system has one sentence at the bottom of it, and this vault's is written down inside itself, in a note titled — in the vault's own house style — *claude's context window should be a budgeted working set pulled from the vault*:

> The vault is the hard drive, Smart Connections is the RAM/retrieval layer, Claude is the CPU. A session is a disposable scratchpad.

That is the whole design compressed into an analogy, and it is a good analogy because it identifies the correct scarce resource. The constraint on an AI collaborator working across many sessions is not storage — storage is free and enormous. It is not intelligence either. It is **working memory**: how much can be in front of the model at once, and how well what's in front of it was chosen.

So the system's job statement becomes precise: *keep a long-running, multi-session collaborator oriented without ever loading the corpus.* Not "store everything." Not "make everything searchable." Keep oriented, on a budget.

The budget is literal — the vault's own rules cap a task at five to fifteen notes and ten to twenty thousand tokens, with an explicit instruction to retrieve twice rather than preload once. That single number, the budget, is the parent of every other decision downstream. Notes should be atomic because a budget slot should return an answer, not a document. Code should never enter the main context because forty-two repositories would consume the budget on the first task — so the vault stores forty-two *navigation maps* instead, averaging 503 words per repository, holding not code but the coordinates of code. Sessions should be disposable because anything that survives a session and matters should have been promoted to a real note, and anything that didn't wasn't worth the tokens.

Notice what kind of design this is. It is not a filing system that happens to be read by a machine. It is a **memory hierarchy**, borrowed wholesale from computer architecture, applied to a knowledge base. Once you see it that way, the vault's pathologies become legible as cache-management problems: what's in the index, what's evicted, what's stale, what's addressable and by what key.

---

## III. The filename is the index

Here is the finding that reorganises everything else.

Every prompt typed into this system passes first through a Python script — a `UserPromptSubmit` hook — before the model sees a word of it. The script tokenises the prompt, strips a ninety-word stoplist, and scores candidates from three sources: the bullets of a memory file, the bullets of one index note, and **the filename of every markdown file in the vault**. It scores each candidate as filename-token overlap, double-weights title matches, adds a bonus if the note belongs to the app the prompt seems to be about, sorts into four buckets, takes the top fifteen, and injects them into the prompt as a shortlist.

Two properties of that script dominate the entire system.

**It never reads a note body.** Only filenames, plus the one-line descriptions in two index files. Not the first paragraph, not the headings, not the frontmatter. Filenames.

**It never fails loudly.** The whole thing is wrapped in a bare `except: pass`. This is correct engineering — a retrieval hook must never block a prompt — and it is also the reason that everything downstream fails in silence.

The consequence is stark: *a note is reachable by the dominant mechanism only through the words in its own filename.* A brilliant, exhaustive, perfectly accurate note about photo storage architecture, filed as `codemap-skylight.md`, is invisible to a prompt about photo storage. It doesn't rank low. It doesn't appear at all. And nothing anywhere reports that it was missed.

This is why the vault's most distinctive convention exists. Notes in the curated tree are supposed to be named as **claims, not categories**: `memory graphs beat giant memory files.md`, not `memory-systems.md`. `launch framework is the evidence-based definition of production ready.md`, not `launch-framework.md`.

Read as writing advice, that convention is charming and slightly precious — the kind of thing that shows up in Zettelkasten forums. Read against the retrieval mechanism, it is something else entirely. A filename that asserts something is *a retrievable answer*: it can be surfaced by the hook, printed as a one-line candidate, and be useful **without ever being opened** — which, on a fifteen-note budget, is the difference between costing nothing and costing a slot. A filename that names a category is a request for further reading, and a request for further reading is a bill.

**The naming convention is load-bearing retrieval infrastructure disguised as a writing style.**

And here is where measurement earns its keep. Of 281 filenames in this vault, 240 are lowercase kebab-case and only 24 contain spaces — and those 24 are almost exactly the claim-titled notes. The convention that matters most for retrieval is followed by fewer than one note in ten, and only inside a single subdirectory.

Nobody noticed, because nothing complained. There is no error for a note that could have been found and wasn't.

---

## IV. Four mechanisms wearing one word

The system is described, casually, as RAG. That word is doing an enormous amount of concealing, and pulling it apart is the most useful thing an audit of a system like this can do. Four distinct mechanisms operate over this vault. They have different triggers, different indexes, different blind spots, and — critically — **different sets of notes they can see at all.**

| | Mechanism | Sees | Blind to |
|---|---|---|---|
| 1 | Lexical prefilter (hook, every prompt) | All 281 filenames | Every word inside every note |
| 2 | Vector embeddings (on request) | 206 notes, block-level | The 75 excluded notes; anything under ~35 words |
| 3 | Agentic grep/glob | All 281 notes, full text | Meaning; synonyms; anything not literally spelled |
| 4 | Graph traversal via links | Whatever the current note links to | Everything not linked; anything linked wrongly |

Mechanism one runs unconditionally on every prompt and decides the shortlist. Mechanism two runs when asked, refines semantically, and is the only one that understands that "blob storage" and "photo bucket" might be the same thing. Mechanism three is the fallback with no index and no exclusions — the only one that can find a phrase buried in the middle of a nineteen-thousand-word file. Mechanism four is how an agent walks from a landing note to its neighbours.

The failure modes do not overlap, which is the good news: what one misses another can catch. But they are not equal partners. Mechanism one fires first, always, and everything after it is **recovery from a bad shortlist**. If the right note is not in those fifteen, the model doesn't know it exists, and whether it eventually finds it depends on whether the person happens to say "run a semantic lookup" or "grep the vault."

The practical implication is one that generalises far beyond this setup: **you must be able to name your retrieval mechanism, in the singular, at each step.** "It uses RAG" is not an architecture description. It's a shrug in technical clothing. The moment you say instead "a lexical filename prefilter, then optionally a block-level embedding search over 73% of the corpus," you can immediately predict what will fail — and every one of this vault's real defects becomes obvious rather than mysterious.

---

## V. Every failure here is silent

Now the theme that runs under all of it.

Catalogue the ways this system can fail to produce a note that exists and is correct:

- The note's filename doesn't lexically overlap the prompt. **No error.** It simply isn't in the fifteen.
- The note lives in one of the two excluded directories, so semantic search cannot see it. **No error.** An empty result, indistinguishable from the note not existing.
- The note is shorter than about thirty-five words and fell below the embedding threshold. **No error.**
- The note is registered in the vector index but has no embedded content. **No error** — unless you run `validate` and read the warnings.
- The note is one of seven files named `index.md`, and the hook deduplicates candidates by lowercase title, keeping the first and silently dropping the rest. **No error.**
- The prompt happened to contain a word that the hook's alias table maps to a product — and the entire retrieval scope narrowed to that product, discarding everything else. **No error.**

That last one deserves its own paragraph, because it is the most instructive bug in the whole system. The alias table maps the ordinary English word *"reference"* to a product called Easel. Any prompt containing "reference" — a word that appears in roughly every request to look something up — scopes the entire retrieval to Easel and drops all other candidates.

The audit notes, deadpan, that this fired **on the prompt that generated the architecture document itself**. The injected block reads *"App scope detected: easel."* The system's most consequential silent failure demonstrated itself while being documented, and would never have been discovered by use — only by reading the source.

This is the deep property of retrieval systems that distinguishes them from almost all other software: **their failure mode is absence, and absence has no signature.** A web server that can't find a page returns 404. A database that can't find a row returns zero results, and you know your query ran. A retrieval layer that misses returns a *plausible, well-formatted, confidently-delivered answer built from the second-best notes*. The output looks identical whether retrieval worked perfectly or catastrophically.

The consequence for anyone building one of these is uncomfortable but simple: **you cannot evaluate a retrieval system by using it.** Using it tells you about the queries that worked. The failures are invisible by construction — that's what "silent" means. You evaluate it by instrumenting it, by validating the index, by reading the scoring function, and by deliberately searching for notes you already know exist and observing whether they surface. Every real defect in this vault was found by measurement. Not one was found by noticing something wrong.

---

## VI. The keystone paradox

The vault has a constitution. One note — `canonical-sources-of-truth.md` — holds the precedence rule that resolves every factual conflict in the corpus: *live code beats this table, this table beats the memory file, the memory file beats everything else.* When two notes disagree about which storage provider a product uses, this note decides. It is the single most important file in the system.

The audit found it registered in the vector index **with no embedded text**. Present. Named. Reachable by link and by grep. Semantically invisible.

Follow the consequence through, because it is worse than it first sounds. Thirteen notes in the vault mention the superseded storage provider. One note states the correct one. A semantic query — *"does this product use R2 or S3?"* — searches only the embedded corpus, which contains all thirteen discussants and not the arbiter. **The system returns the thirteen notes that are wrong and never the one note that is right.**

The fix takes five minutes: touch the file, force re-embedding, re-run validate. It's a triviality. But the shape of the failure is worth keeping.

A hierarchy of trust is only as good as the retrievability of the thing at the top of it. Every knowledge system eventually grows a keystone — a canonical index, a decisions log, a source-of-truth table — precisely because deduplication proved impossible and arbitration was the fallback. **A source-of-truth table is what you build when you have decided not to deduplicate.** And the moment that keystone is less findable than the noise it exists to arbitrate, the hierarchy inverts silently, and the system becomes *more* confidently wrong than it would have been with no keystone at all — because the wrong notes are now numerous, mutually corroborating, and unopposed.

If you build one of these, the keystone should be the single most instrumented object in the system. It should be checked on a schedule, not on suspicion.

---

## VII. Inoculation: the best idea in this vault

Now the finding I'd most like to see travel, because it inverts a piece of received wisdom about knowledge bases.

A grep across the vault turned up thirteen notes mentioning a storage provider that had been replaced, and eleven mentioning a deployment platform that had been replaced. Twenty-four apparent instances of stale knowledge. A prior audit had reported exactly this and flagged it as widespread drift.

Then someone read the lines.

Twelve of the thirteen and all eleven were **correct**. They were dated correction banners opening with *"Written when the product used R2. Storage moved to S3…"*. They were explicit gotcha annotations reading *"the README is WRONG"*. They were decision records documenting the migration. One was a different product that genuinely does use that provider.

Exactly one note asserted the stale claim as present fact, without inoculation, in a folder index — a note other notes link into. Twenty lines below the error, the same file correctly stated the *other* half of the migration, meaning the deploy half had been patched and the storage half missed.

Two lessons come out of this, and they point in different directions.

**The first is methodological, and it is brutal about a habit almost everyone has.** Counting occurrences of a stale term measures nothing. It produced a false positive in the previous audit and reproduced the identical false positive here. A term-frequency sweep cannot distinguish *asserting* a claim from *refuting* it — they are lexically identical. This generalises well past vaults: any audit that counts mentions rather than reading them will systematically misclassify the corrections as the disease. And the better a corpus is at documenting its own corrections, the worse a naive count makes it look. **The vault's healthiest convention is the one that makes it appear broken to automated inspection.**

**The second is architectural, and it is the transferable idea.** The convention at work here — attach a dated correction banner to the stale note rather than deleting or silently editing it — solves a problem that is unique to writing for machine readers.

For a human reader, a stale claim is mildly dangerous: you'll usually notice the note is from last year, or check something, or remember the migration. For a retrieval system, a stale claim is *acutely* dangerous, because retrieval strips context. A note surfaces alone, without its date, without its neighbours, without the correction that lives three notes away in the graph. The model sees a confident assertion and has no way to know it was superseded.

The banner convention fixes this by making **every note carry its own refutation**. The correction travels with the claim, inside the retrieval unit, so an agent that reads the stale note gets the correction *in the same breath*. The note becomes safe to retrieve in isolation — which is the only way it will ever be retrieved.

Deleting the stale note would also have worked, but it destroys the history of why the decision changed, which is often the most valuable thing in the file. Silently editing it is worse: it produces a corpus that looks consistent and quietly loses the record of what was believed when.

So: **don't delete stale knowledge, inoculate it.** Date the correction, put it at the top, keep the original. It costs one line and it makes a note that would otherwise be a landmine into a note that teaches.

That the vault does this almost perfectly — twenty-three of twenty-four instances — and that the one uninoculated exception sits in an index note where retrieval lands first, is the tidiest illustration you could ask for of why the convention matters.

---

## VIII. Length is not a cost problem, it is a labelling problem

Everyone building these systems knows long notes are expensive. Fewer people know they are also **wrong**.

The embedding model here has a 512-token window. A note of four thousand words does not embed as a vector meaning "this note is about X." It embeds as a vector computed from its **truncated head** — the first few hundred words — plus a scatter of per-heading block vectors.

The whole-note vector is therefore not a summary. It is a *sample of the introduction*, presented to the system as a summary. The note is **semantically mislabelled**. It will surface for queries that match its opening and fail to surface for its own content — content it genuinely contains, at length, correctly.

This is a categorically different problem from cost. An expensive note is one you'd rather not open. A mislabelled note is one you will be confidently told is irrelevant. The first wastes budget; the second produces wrong answers with no symptom.

The vault's stated rule is 800 words per note. The measured reality: 65 of 142 notes in the curated tree exceed it, 53 notes vault-wide exceed 2,000 words, and the largest single file runs 19,528 words — roughly twice the *entire* per-task context budget, in one file, in a system whose founding principle is budgeted retrieval.

Atomicity is not fussiness. In a system with a fixed embedding window, **the atomic note is the unit that can be honestly labelled.** Above that size, your index is describing something other than what's in the file, and it is doing so with total confidence.

---

## IX. What is inert is not what is broken

Now for the most philosophically interesting finding, and the one most likely to change how you audit your own systems.

The vault has, by conventional standards, a frontmatter disaster. Only 36% of notes carry any frontmatter at all. Coverage is wildly bimodal — 41 of 42 in one directory, 16 of 142 in another. Two incompatible schemas coexist, one machine-shaped and one human-shaped. A `status` field has twenty-five distinct values across thirty-six uses, including free-prose entries like `v1 CO-DESIGNED 2026-07-19 — master-spec…`. It is prose in a field shaped like an enum.

Every PKM guide on the internet would call this technical debt requiring immediate remediation.

The audit's verdict: *"today, nothing."* Because there are zero Dataview blocks, zero Bases queries, zero Templater expressions, and zero embeds in the entire vault. **Nothing reads the frontmatter.** It is inert — documentation of intent, not machinery. Backfilling 179 notes would be pure ritual.

The recommendation that follows is correspondingly precise: don't backfill. Collapse `status` to a four-value enum in the thirty-six notes that use it, add `description` opportunistically as you touch notes, and understand that this becomes a genuine P1 the instant someone writes the first query against it.

The same logic exposes an entire dimension as fictional. A sweep for tags returns hex colour codes inside CSS blocks and five real-looking tags that each appear exactly once inside example snippets. The tag pane is enabled and displays essentially nothing. There is no tag sprawl to fix and no synonym collision to consolidate, **because there is no tag system**. As the audit puts it: any advice about tagging here is advice to *build* a system, not maintain one — and it shouldn't be built without a query layer that consumes it.

Generalise the principle:

> **A schema is not a defect relative to a standard. It is a defect relative to a consumer.**

Inconsistent frontmatter with no query layer is not debt; it's unused capacity. Inconsistent frontmatter *with* a Dataview dashboard is a broken dashboard. Identical artifact, opposite verdicts, and the only thing that distinguishes them is whether anything downstream depends on it.

This is why so much knowledge-management advice fails on contact with a real vault. It prescribes structure in the abstract — tag consistently, fill your frontmatter, maintain your MOCs — without asking the only question that determines whether the structure does anything: *what reads this?* Structure that nothing consumes is cost with no return, and the discipline it demands is drawn from the same finite budget as the discipline that actually matters — which, in this vault, is naming notes as claims and keeping them short.

The dead folders make the same point from the other direction. A `voice-notes/` directory, empty since the day it was created. A `daily/` folder, empty, despite the daily-notes plugin being enabled and the front page instructing "start each day with one." An `inbox/` with three notes, none added in three months, and a `queue-generated/` subfolder documented as "where Claude drops auto-captures," containing nothing.

The documented capture pipeline — *raw thought → inbox → process → knowledge* — has not run in a quarter. Knowledge actually arrives two ways: the model writes directly into the curated tree, or the compaction hook writes automatically into sessions. The empty folders are not a hygiene problem. They are **evidence**, and what they're evidence of is a design that was aspirational at creation and never tested against how the system came to be used.

Every vault has these. They're the most honest thing in it. The folders that filled up tell you what the system is for; the folders that stayed empty tell you what you thought it was for.

---

## X. The system cannot describe itself

There is a recursive failure here worth pausing on.

The vault's own self-description — the note explaining how the machinery works — says session capture runs on a `PreCompact` hook. It runs on `PostCompact`. It gives the session filename format as `YYYY-MM-DD-session.md`. The actual format is `YYYY-MM-DD-compact-HHMM.md`. The banner printed at the start of every session announces "100 connected notes" to a vault containing 281. The front page still advertises a daily-notes folder and an inbox that haven't been used in months.

Individually, trivia. Together, something specific: **the vault's self-description cannot be trusted for operational facts.** And that is exactly the failure the keystone note was built to prevent — now applied to the vault itself. The system has a rigorous protocol for arbitrating conflicting claims about its *products*, and no protocol at all for claims about its *own machinery*.

This is the general case, not a quirk. Documentation drifts because the code changes and the prose doesn't, and the prose has no test. In a system whose self-description is *read by an agent at the start of every session* — which is precisely what a SessionStart banner is — that drift isn't cosmetic. An agent trying to debug a capture failure will go looking for a hook that does not exist, find nothing, and report that the capture system is missing.

The fix is not "write better documentation." It's to notice which claims are *checkable* and check them. Hook names, file formats, note counts, folder inventories — these are all script-verifiable in seconds. The parts of documentation that can be tested should be generated, not written.

---

## XI. What breaks first

Ask the scaling question — what fails at ten times the size, at 2,800 notes — and the intuitive answers are all wrong.

Not disk. Not the vector index; a small embedding model over 3,700 blocks isn't near any limit. Not Obsidian.

What breaks is **mechanism one** — the dumbest component in the stack.

Two ways. First, mechanically: the hook walks the entire vault on every prompt, which is comfortable at 281 files and borderline at 2,800 inside a ten-second timeout. Second, and more seriously, its top-fifteen cut degrades from a ranking into a lottery. The scoring function is raw token overlap, double-weighted for title matches, with no length normalisation, no IDF weighting, and no tie-breaker. With 2,800 filenames competing on scores of two to four points, dozens of notes tie exactly, and which fifteen survive is determined by directory-walk order.

**The system does not fail at scale by getting slower. It fails by becoming arbitrary while continuing to return exactly fifteen confident-looking results.** Silent, again.

The fix isn't a bigger cap — that just moves more noise into the context. It's scoring against note *descriptions* rather than filenames alone, which requires the `description` field to be populated in far more than the current 26% of notes. Which means the frontmatter that is inert today (§IX) is the exact thing that would need to have been maintained for the system to survive its own growth.

That's a nice illustration of why "is this inert?" is a question with a time index. Today the frontmatter does nothing. The single change that would fix the scaling ceiling is the one that makes it do something. **The inert convention isn't worthless — it's a call option on a future consumer.** Which argues not for backfilling all of it now, but for the audit's actual recommendation: populate it opportunistically, on the notes you touch anyway, so the option stays alive at near-zero cost.

---

## XII. What transfers

Strip out the specifics and this is what remains for anyone building a knowledge layer for an AI collaborator.

**1. Name your retrieval mechanism, in the singular, at every step.** "It uses RAG" is not a description. Lexical prefilter, embedding search, grep, graph walk — each has a different blind spot, and you cannot reason about failures until you know which one is running when.

**2. Whatever the first-stage filter reads is your highest-value real estate.** Here it's filenames, so filenames must carry meaning — hence claim-titles. If your first stage reads descriptions, descriptions are the asset. Find out what it reads, then put your effort exactly there.

**3. Assume every retrieval failure is silent, and instrument accordingly.** Absence has no signature. Validate the index on a schedule. Search for notes you know exist and check they surface. You cannot find these defects by using the system, only by testing it.

**4. Inoculate stale claims rather than deleting them.** Retrieval strips context; a note must carry its own correction because it will be read alone. A dated banner at the top costs one line and converts a landmine into a lesson.

**5. Never audit staleness by counting terms.** Corrections and errors are lexically identical. Read the lines. A corpus that documents its own corrections well will look maximally broken to a naive count.

**6. Keep notes under the embedding window.** Not for cost — for honesty. A note longer than the window is mislabelled in the index, and mislabelled beats expensive as a source of wrong answers.

**7. Ask "what consumes this?" before enforcing any convention.** Structure with no consumer is discipline spent for nothing, drawn from the same budget as the discipline that matters.

**8. The keystone must be the most findable object in the system.** If the note that arbitrates conflicts is less retrievable than the notes that conflict, the hierarchy has inverted and the system is now confidently wrong.

**9. Empty folders are data.** They record the gap between the workflow you designed and the workflow you have. Read them before you delete them.

**10. The parts of your documentation that are checkable should be generated, not written.** Note counts, hook names, file formats, folder inventories — prose has no test, and an agent reads your self-description first.

---

## XIII. You do not design these systems. You measure them.

The most valuable property of this vault is not any of its conventions. It's that somebody profiled it and wrote down what was actually there, including — especially — the parts that contradicted the documentation.

That exercise produced findings no amount of design thinking would have: that the convention doing the most work was followed by fewer than one note in ten; that a single ordinary English word in the alias table silently narrowed retrieval to the wrong product, and did so while the documentation was being written; that the one file the entire trust hierarchy depends on was invisible to half the retrieval stack; that the "widespread staleness" reported by a previous audit was a measurement artifact and the underlying convention was working almost perfectly; that 47% of the corpus had been made semantically unreachable by a single configuration string, correctly, and that nothing anywhere told an agent what to do about it.

None of that was knowable from the outside. All of it was cheap to find. The three highest-leverage fixes on the list total **half an hour of work** — five minutes to re-embed the keystone, four to correct one stale line and add one missing index entry, twenty to script a link conversion — and between them they repair the trust hierarchy, the only uninoculated false claim in the corpus, and graph traversal at the hubs where traversal starts.

That ratio is the real lesson. In retrieval systems the defects are cheap to fix and expensive to find, because they are silent by construction. Which means the discipline that pays is not better design at the outset. It's periodic, script-assisted, unsentimental **measurement** — of a system that will otherwise keep telling you, in fluent and well-formatted prose, that everything is fine.

---

*Written from a script-measured audit of a live 281-note vault, July 2026.*
