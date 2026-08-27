---
name: spec
description: >-
  Turn a feature, bug, or refactor into a reviewable spec (requirements, design,
  tasks) under docs/plans/, with risk-tiered approval gates, hash-pinned approval
  receipts, a decision ledger, and requirement-to-task traceability so a fresh
  session can resume mid-plan. Invoke explicitly with /spec.
user-invocable: true
disable-model-invocation: true
---

# Spec-driven change

Requirements, design, tasks. A risk-tiered number of approval gates, and a
decision ledger that survives compaction. Every task traces to the requirement
that justifies it.

Model invocation is off deliberately: an ordinary "fix this" must never become
three stopped turns because a diff touched four files. The user asks for a spec,
or there is no spec.

Project-specific risk surfaces, deploy steps, hard stops, and specialist
reviewers live in the consuming repo's `AGENTS.md` (or equivalent), not here.
Read that overlay before you pick a tier or name a Verify command.

**Read `references/example-spec.md` before writing your first file.** It is one
small tier-3 spec in full, and it pins the exact shapes, so copy it rather than
inferring them from the rules here.

**Run the validator before presenting any gate, and again before execution.**
The script sits next to this file:

```bash
node <this-skill>/scripts/check-spec.mjs docs/plans/<date>-<slug>
```

It recomputes the approval hashes, cascades a void approval downstream, and
catches uncovered criteria, orphan citations, missing `Verify:` lines, invented
npm scripts, and non-regression assertions whose test neither exists nor is
created by a task. Exit 1 means fix it, not narrate around it. These checks are a
script because by hand they decay into a claim.

## Layout

```
docs/plans/<YYYY-MM-DD>-<slug>/
  requirements.md
  design.md          # feature and bug only, never quick
  tasks.md
  .spec-state.json
```

Flat single-file plans already in `docs/plans/` stay as they are.

## Tier first

Say which tier and why in your first message. Three stops is the top tier, not
the default. File count is not the signal: four locale files are not riskier
than one line in a checkout path.

**Three gates**, each document approved separately. Money, access control,
entitlements, irreversible writes, schema migrations, production data, or when
the user asks for an implementation review. The consuming repo's overlay names
the concrete surfaces. Only here are the three questions independent: right
behaviour, safe architecture, safe decomposition.

**Two gates**, requirements then design; tasks generate and execute without a
further stop. Normal significant work: a new surface, a pipeline change, a
cross-subsystem refactor, an API route that is not money or auth.

**One gate**, criteria and approach together. `quick` specs and obvious
multi-file work. This is most of what gets asked for.

At every tier: write one gated file, then stop, report the path and what it
decided, and wait. Ungated files may follow in the same turn.

Ask clarifying questions only when the answer changes the output, 1 to 3 at a
time, each with your recommended answer. Never ask what the repo can tell you.
Any real either/or goes through a structured choice tool, never prose that
buries the choice in a paragraph: one question per decision, recommended option
first and labelled `(Recommended)`, the trade-off in each option's description.
A structured choice is answered in one click; a prose question costs a full turn
and often comes back half-answered.

## Spec types

**feature**: all three files.
**bug**: `requirements.md` is the investigation, symptoms, reproduction, root
cause, expected behaviour, and what must not change. `tasks.md` is the fix plus
the regression test that would have caught it.
**quick**: short `requirements.md` and `tasks.md`, no `design.md`.

## Approval is a receipt, not a file that exists

Which files exist tells you what was written, not what was approved. An agent
interrupted after writing `design.md` and before review leaves a directory
indistinguishable from an approved one.

So derive the next action from the files, but record approval as a hash-pinned
receipt. Pin the document with SHA-256 so an unreviewed edit turns the next
check red:

```
shasum -a 256 docs/plans/<date>-<slug>/requirements.md
```

Write the receipt on approval. Never verify one by hand afterwards, run the
validator: it applies the downstream cascade, which is the part an eyeball skips.
A mismatch voids that approval and every one below it, so re-open the gate.

Phase is derivable, approval is not. That is why there is no `phase` field and
there is an `approved` array.

## `.spec-state.json`

The decision ledger. Update it whenever you ask a decision, get an answer, or
change phase. It is not a gated document, so update it in the same turn as
whatever it records. Full example in `references/example-spec.md`.

- `tier` is 1, 2 or 3, matching what you declared.
- `approved` is one hash-pinned entry per approved document. Drop an entry when
  its approval is voided. Never edit a hash to clear a mismatch.
- `id` is stable, and must keep carrying the same question. If the question's
  _meaning_ changes, retire the id with its `why` and add a new one: a stable id
  over a changed question is a silent lie to the next session.
- `recommended` is mandatory while a decision is open. A question without one
  offloads the work.
- `answer` is `null` while open. Keep answered entries forever.
- `why` is one sentence, written when the decision is answered. It is the reason
  the file beats the chat log.
- `blocking` is one sentence for a human: what you are waiting on. Never machine
  state, so never branch on it.
- `closure` is `implemented`, `merged`, `deployed` or `verified`, absent while
  open.
- `context.template` names the module you are modelling the work on. There is
  almost always one.

Do not narrate this file or list it as a deliverable. Do show it when the user
asks what state a spec is in: hidden governing state is worse than noisy state.

## requirements.md

Numbered requirements, each a user story then its criteria, numbered `1.1`,
`1.2` so tasks can cite them. Criteria in EARS form:

```
WHEN <event> THE SYSTEM SHALL <response>
IF <precondition> THEN THE SYSTEM SHALL <response>
```

No vague criteria: "should be fast" is not a requirement, "SHALL return within
2s" is. No implementation: "SHALL cache frequently accessed data", not "SHALL
use Redis".

Always include an explicit **Out of scope**, and **Open questions** that is
either empty or mirrored into the ledger before the gate. When presenting the
gate, put the open questions to the user as structured choices (options from the
ledger entry, `recommended` first) instead of leaving them as a list to answer
in prose; write each answer and its `why` into the ledger in the same turn.

For money, access-control, entitlement, or shared-contract work, add
**non-regression assertions**: `WHEN <x> THE SYSTEM SHALL CONTINUE TO <y>`, each
with a `Proven by:` line naming a real test, existing (verified) or created by a
task. An assertion with no test behind it is a wish, and a bug spec without any
is incomplete.

### Analyze before the gate closes

Do not present `requirements.md` the moment it is written. Pass over it once and
report what you found, not that you checked. A contradiction costs a sentence
here and a phase in `design.md`.

- **Contradictions**: two criteria that cannot both hold. Name both numbers.
- **Ambiguity**: any criterion two competent readers would build differently.
  Rewrite it, or make it a decision.
- **Gaps**: empty state, external call failing, unauthenticated caller,
  concurrent writers, the unpaid tier, a non-default locale. These are where
  systems fail silently.
- **Untestable criteria**: no observable outcome.
- **Unstated coupling**: a criterion that holds only if another subsystem
  behaves a certain way. Name it, and say whether you verified it.

Keep it to a few lines for `quick`. For money, access control, entitlements or
migrations it is not optional, and finding nothing must be said out loud.

## design.md

Overview and how it satisfies each requirement, then architecture, data model,
interfaces, error handling, testing strategy. Mermaid only where it helps.

Every non-obvious choice gets a decision block (context, options with trade-offs,
decision, rationale) and the same decision in the ledger. Template in the
example. When the choice is genuinely the user's — taste, product direction,
spend, anything the repo cannot decide — ask it at the design gate as a
structured choice with options mirroring the block's, rather than writing the
decision as settled and hoping the gate review catches it.

Name the co-artifacts here rather than at the end: scoped rules, docs, and
sibling surfaces that must move with the code. Say plainly if merging does not
ship the change, and name the apply step from the overlay.

## tasks.md

Ordered checkboxes, each doable in one focused step, building on the last, with
`Files:`, `Verify:` and `_Requirements:`. Shape in the example.

- No non-coding tasks. "Discuss with the user" is a gate.
- Every criterion cited by at least one task. Report the coverage count.
- No `Verify:` line, not a task. The command must exist in `package.json` or a
  CI job the overlay names. If nothing exercises the criterion, the task is to
  write the test.
- A foundation task implementing no user-visible criterion is
  `_Requirements: enabler_`. A fake citation is worse than an honest gap because
  it makes the coverage count lie.
- Sequence foundation first, or risk first when a piece might not work at all.
  Say which, in one line at the top.

## Resuming

Read in this order: `.spec-state.json`, `ls` the dir, then **`requirements.md`
and `design.md` in full**, then `tasks.md`. Do not skip to the task list. The
constraints, exclusions and error semantics live upstream, and a task list read
alone is how an executor confidently builds the wrong thing.

Then run the validator and report the receipt state.

Then sync: check the codebase for tasks another session already finished, tick
them, say which. An unchecked box is not proof the work is undone.

**One session per spec.** There is no merge model for the checkboxes or the
ledger. Split work into separate specs with non-overlapping files, never two
workers on one `tasks.md`.

## Execution

Work through unchecked tasks in order. Tick the box, run that task's
verification, continue. Stop when all are ticked or on a real blocker, and
summarize what landed and what remains.

Sequentially. `tasks.md` carries no dependency metadata, so nothing here can
tell you which tasks are independent and guessing puts two dispatches in one
file. If the user asks for parallelism, dispatch a subagent with the task text,
`Files:` and `Verify:` as the whole brief, only for tasks whose `Files:`
provably do not intersect, and merge in the main thread.

Unattended: stop and report after 10 consecutive tasks without a human look;
stop on the first failing verification rather than stacking work behind a red
check; leave tool approvals with the user.

Hard stops always: never push without confirmation, never mutate production
data or secrets, never restart a shared local service the user owns. The overlay
may add more.

## Closing: "shipped" is four states

Written is nowhere near live. Merging may not deploy the thing users hit. Record
`closure` and skip nothing:

| `closure`     | Means                                                                              |
| ------------- | ---------------------------------------------------------------------------------- |
| `implemented` | Tasks ticked, verifications green, local only.                                     |
| `merged`      | On the default branch.                                                             |
| `deployed`    | The production apply step from the overlay has run, named in the spec.             |
| `verified`    | Observed working in production.                                                    |

**Archive only at `deployed` or later.** Archiving at `implemented` leaves a spec
that looks finished over a system that never changed. If the deploy is the
user's to run, stop at `merged` and say so.

At closure: report the co-artifact sweep unprompted; dispatch any specialist
reviewers the overlay names for the surfaces you touched; write the decisions
and their reasoning to the project's durable memory if it has one, dropping any
fact a new decision supersedes.

## Lifecycle: a shipped spec is an archive, not a contract

A spec that disagrees with the code is worse than no spec, because readers still
trust it. Three genres, and the split is not optional:

| Genre           | Where                                             | Rule                                                                      |
| --------------- | ------------------------------------------------- | ------------------------------------------------------------------------- |
| In flight       | `docs/plans/<date>-<slug>/`                       | Live. Amend freely, gates apply.                                          |
| Archived        | same dir, frozen apart from the header            | Stale by design, never revised. What was intended on that date.           |
| Living contract | project rules, `docs/`, agent docs                | Present tense, current behaviour, updated in the same commit as the code. |

On archiving, do both:

1. Head each file with `> Archived <date>. Intent as of that date, not current
behaviour. Current: <the rule that owns this area now>.`
2. Migrate anything load-bearing out. An env-flag table, an ordering invariant, a
   gate order, a money path: that does not live in a plan. Move it to the scoped
   rule and link the rule. If nothing is load-bearing, say so.

Never delete an abandoned spec. Head `requirements.md` with
`> Abandoned <date>. <one sentence why.>`. The reason it died stops the idea
being re-proposed next quarter.

## Backward flow

Design proving a requirement wrong is normal. It must flow backwards, not get
patched around. Stop, say which document is now wrong and why in one sentence,
then:

1. **Void the receipts first**, before touching prose, so an interruption cannot
   leave a stale approval over changed text.
2. Amend. Keep requirement numbers stable, renumbering breaks every citation. If
   a number's _meaning_ changes, retire it and add a new one, then fix the citing
   tasks.
3. Record the reversal as a decision with its `why`. An approach tried and
   abandoned is the one thing a later session cannot reconstruct.
4. Re-derive downstream, re-open the voided gates. An amended requirement means
   `design.md` gets a look, not a rubber stamp.
5. Re-run the validator for coverage.

A `design.md` quietly contradicting its own `requirements.md` is the failure this
method exists to prevent.

---

Method adapted from KiroCrew's Spec Builder
(`kirodotdev/KiroCrew`, `src/kiro_crew/apps/builtins/spec_builder/`), keeping the
gates, the ledger and the traceability. Its `.spec-state.json` shape, its
file-derived phase and its stable-id trap are all from there. The approval
receipts, the risk tiers and the four closure states are not: those came from an
adversarial review that found the file-derived phase cannot represent "written
but not approved".
