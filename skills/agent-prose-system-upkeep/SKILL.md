---
name: agent-prose-system-upkeep
description: Weekly upkeep for systems whose behavior is mostly natural-language instructions (prompts, rule docs, decision tables, per-market/per-flow "slices") continuously edited by coding agents. Use when such a system keeps growing messy or the user wants to rewrite or mass-delete it ("rewrite from scratch"); split work between prose, workflow tools, decision tables and experts; expert sign-off on prompts; replay historical sessions to estimate a new check's blast radius; triage feedback signals (single vs clustered, sibling slices); ledger/file-based memory; delete old prompt variants with evidence. NOT ordinary code-repo maintenance (for repeated agent mistakes, guardrail layers, codebase cleanup, use agent-environment-design; for anti-gaming gates, mutation testing, CI gates, SPEC, ratchets, use agent-proof-codebase).
---

# Prose-first agent systems: weekly sweeps and feedback loops

## Scope (read first)

- Applies to: systems where **most behavior is defined by natural-language instructions** (prompts, rule docs, decision tables, skills) and which are **continuously changed by coding agents**. Typical shape: work is cut into many slices by "market/country x flow", a runtime agent follows the instructions, another kind of coding agent maintains those instructions, and business experts judge right and wrong.
- Does not apply to: ordinary code-repo maintenance. Correcting mistakes agents keep repeating, deciding which layer a guardrail belongs in, and codebase cleanup -> **agent-environment-design**. Anti-gaming, mutation testing, CI gates, SPEC, and ratchets -> **agent-proof-codebase**.
- Label convention: **Speaker practice** = what the talk describes doing; **Inferred (not stated by speaker)** = judgment added to make it actionable; unlabeled text is general writing.

## Hard rule: every PR needs human approval

All changes produced by this skill (sweeps, deletions, prose rewrites, decision-table changes, new checks) are submitted as PRs, and **every PR must be approved by a designated person before it can merge. Auto-merge is not allowed.** Even if the platform has auto-merge enabled, it may only take effect after the designated person approves. Reason: for prose, "what must survive" can only be judged by a human, and tests cannot judge it; auto-merging hands that judgment back to an agent that can overlook things.

## Default cadence: small weekly sweeps, no big rewrites

**Speaker practice**: a big rewrite loses things other people are using, and because agents keep writing, the mess grows back. Use "small weekly sweeps + a built-in feedback loop" to keep the volume of change below the line of "no one can check each change".

**Weekly sweep procedure** (Inferred (not stated by speaker): the speaker's individual practices arranged into a weekly order):

1. **Check the line**: has this week's volume of change exceeded what a person can check one by one? Look at your own gauge (see "Treat yourself as the gauge") and the open ledger entries. If over the line, this week only fix the process and add no new features.
2. **Read the ledger**: entries added, recurring, and closed this week; whether each has a root-cause link.
3. **Triage signals**: handle them per the "Signal triage" table; wait for single signals to recur, fix the concept behind clustered ones.
4. **Sweep**: touch only what has evidence this week: merge drifted copies, solidify repeated walks, move expressions experts cannot read back into prose. Each sweep item is its own PR, scoped to one slice or one concept.
5. **Delete**: follow "Deletions need a name and evidence".
6. **Replay**: replay any new check or rule change on historical sessions first (see "Replay and blast radius").
7. **Update memory**: write the ledger and the "what is true now" file.
8. **Merge after human approval**, never auto-merge.

Sweep checklist:
- [ ] Every change this week falls inside its declared slice, or states why it crosses slices
- [ ] Every deletion has a name + evidence
- [ ] Every new check has been replayed and has been made to fail on purpose once
- [ ] Every issue links to a root cause
- [ ] No condition was added solely to make that day's task pass
- [ ] Every PR has human approval (no auto-merge)

## Cut slices, and by default stay inside the slice

**Speaker practice**: every task lands in one slice (for example, one flow in one country). A change **stays inside its own slice by default**; sibling slices may **differ on purpose** because of market or local regulation, so a fix in one slice cannot be copied wholesale to the others.

- Every PR description states: the slice, the changed files, and whether it crosses a boundary and why.
- Cross-slice changes are made only when they hold at the "concept level" (see Signal triage).
- How to cut your system: cut by "who signs off on correctness" and "which differences are intentional"; the unit of cutting need not copy country x flow. (Inferred (not stated by speaker))

## The prose ledger

**Speaker practice**: every prose file that was changed goes through checks and is recorded in the ledger, treated the same as code; every feedback loop is also written to the ledger, so nothing waits on one person's memory. The speaker built a prose compiler and lint; the tools are not public, so no concrete tool steps are given here: if you have existing checks, run every changed file through them; if not, record the check result and date by hand in the ledger.

Entry format (Inferred (not stated by speaker): fields designed by purpose):

```
- id: L-0042
  date: YYYY-MM-DD
  slice: <slice name>
  files: [<changed prose/table/tool>]
  kind: change | signal | deletion | eval | incident
  what_happened: <one sentence, facts only>
  root_cause: <link to a root-cause entry; if unknown write "unknown", and do not act on it>
  evidence: <replay result / expert annotation / session id>
  check_result: <whether lint/checks passed>
  status: open | confirmed | fixed | wont-fix
  approved_by: <PR approver>
```

- After a batch-type task (an agent changing many places at once) ends, use the ledger to check the tail of the batch: an agent may finish most of it and quietly skip the rest, especially when the context fills up. A backlog that looks "covered" may have far fewer items that can actually be closed, so accept only evidence that can be counted. (Speaker practice)

## One document, two kinds of readers

**Speaker practice**: in the same document, the **blocks are for the agent to read** (they must be precise), and the **explanatory text around the blocks is for experts to read and sign off**. Keep only one copy; a second copy will drift. Conversely, code also needs a plain-language description that experts can understand.

- Do not write a separate "explanatory version" for experts; put the explanation next to the block in the same file.
- Move hundreds of lines of expressions that experts cannot read back into prose (or a decision table); keep regular expressions only for recognizing a single key signal. (Speaker practice)

**Experts review a single task**: have each expert review one task **as the agent saw it at the time** (give them the agent's actual input, not the internal implementation). Expert annotation -> the coding agent's next change -> retest on the **same task**. Record "accepted / rejected" for each result, and use the acceptance rate to see where people still need to keep watch. (Speaker practice)

Questions about system internals that business experts cannot answer are answered by developers, using an agent to dig through traces and logs.

## Division of labor among prose, code, models, and people

**Decision-table division of labor**: code prepares the facts and executes the decision; the expert owns the table; the model only picks a row; if no row fits, hand off to a human. (Speaker practice)

**Solidify into workflow tools** (Speaker practice, thresholds are parameters): only walks (the paths the runtime agent takes through the prose) that are both "repeated" and "consistent across slices" get solidified; **after solidifying, the prose keeps only the decision and does not write the tool's internal step names**, otherwise the agent will execute those steps again (a pitfall the speaker hit). Include this rule whenever you answer a question about division of labor.

1. **Repeated**: the same path occurs at least `repeat_min` times (parameter; set by your traffic and session volume; start with a value that leaves you only a few candidates to look at each week)
2. **Consistent across slices**: the walk is the same in at least `slice_min` slices (parameter; slices with intentional differences do not count as consistent)
3. **Delete the prose after solidifying**: prose now covered by the tool must be deleted, keeping only the decision.

Parts that still differ stay in prose inside the framework.

Sample (column names are Inferred (not stated by speaker); the example entries are fictional, for demonstration):

```
| Condition facts (computed by code) | Action | Owner | When no match |
|---|---|---|---|
| amount <= limit | approve | Finance expert A | - |
| amount > limit and role=lead | route_to_lead | Finance expert A | - |
| any other case | handoff_human | Process owner | Hand off to a human, and write a ledger signal entry |
```

Rules:
- The model's output can only be a row number or `NO_MATCH`; it does not freely generate actions; `NO_MATCH` must have a corresponding human queue.
- Table changes are signed off by the owner; code does not change the table on the expert's behalf.
- "The model makes small decisions, then a workflow chains them together" is a next step the speaker said has not been built yet; do not treat it as an existing practice.

## Minimal context and handles

**Speaker practice**: the **runtime** decides what the model sees; do not let the model choose what to load (if you let it choose, it loads everything). Anything put into the context becomes a reason for it to act: seeing a price may lead it to invent a price check out of thin air. "Do not call X" also tells the model that X exists.

- For each decision point, list the minimum facts it needs and give only those; the full record stays in the runtime.
- Pass large datasets as a **handle** (a short name for data the runtime holds), and the runtime validates that the handle matches the context before writing.
- When writing prose, describe positively what to do, and avoid listing what not to do as prohibitions. (Inferred (not stated by speaker): a writing style derived from the two points above)

## The write path is opened only by code

**Speaker practice**: the model's decisions become actions, so the write path is opened only by code, and prose being edited several times a day does not widen write permissions. Control it twice: once at development time, once at the moment of the runtime write.

- Pre-write checks: list them according to your system's risks, each executed by code.
- **Post-write verification**: confirm the write actually landed; do not trust "looks successful".
- **Journal idempotency**: an agent that sees a failure will retry; use a journal to guarantee the same write executes only once.
- For the layering judgment "code layer takes priority over prose" -> agent-environment-design.

## Treat yourself as the gauge

**Speaker practice**: a dashboard can be all green, and "I am too tired to say clearly what changed this week" is the first signal. When it appears, stop first and fix the process (shrink the volume of change, add loops, cut the steps that pass only through you), without waiting for some number to turn red.

- Your job is to build the loop (see "The user as evaluator"), not to be the message bus between users and the agent.
- If you find yourself reading every diff, the loop is not built yet. (Inferred (not stated by speaker))

## Signal triage

**Speaker practice**: wait for a single signal to recur, and fix the concept behind it only when signals cluster; an expert annotation becomes an eval, an agent searches sibling slices for the same kind of failure, and the fix lands only in slices where the failure is confirmed.

| Signal shape | Action |
|---|---|
| A single signal (including one from an expert) | Record in the ledger with `status: open`, wait for it to recur, do not change prose |
| The same signal recurs in at least `cluster_k` slices (parameter) | Find the concept behind it and fix the concept once; go through a PR, human approves |
| Expert annotation | Turn it into an eval (input = what the agent saw at the time, expectation = the expert's judgment) |
| A slice that turned red | Use that eval to search **every sibling slice** for the same kind of failure, confirm one by one; fix only those confirmed failing; do not change intentional differences |
| A number jumps | A human goes down and reads the concrete cases, rather than judging from the aggregate |

Set `cluster_k` and "how long to wait" by your signal volume: start with a value at which a few clusters surface within a week, then tune by false positives/negatives.

## Replay and blast radius

**Speaker practice**: before a new check goes live, replay its rules on a batch of historical sessions to see how many it triggers and where; a human reads the places where numbers jump.

Acceptance checklist (the replay set size `replay_n` is a parameter, set by your system: use the largest amount that covers the main slices; the speaker's specific replay volume is not a standard and is not relayed):
- [ ] The replay set is **frozen real tasks** covering the slices, not newly invented examples
- [ ] The report gives: trigger rate, distribution by slice, and change versus the previous version
- [ ] Trigger rate far above expectation -> read the triggered samples first to judge whether the rule is too broad or the system truly has a problem
- [ ] Any branch/decision-table row that no real task has walked is marked **no evidence** and does not count as verified
- [ ] The new check is **made to fail once on purpose** first, proving it can go red (method: see agent-proof-codebase, "a gate that has never failed proves nothing")
- [ ] Write down in advance "what result would prove this change wrong"

## Anti-overfitting

**Speaker practice**: left alone, agents keep adding conditions until that day's task passes. Periodically strip such conditions out and see how much the eval drops; the part that drops is the honest list of gaps.

Method:
1. Find conditions recently added to make a particular task pass (specific values, specific names, "if this customer").
2. Remove them on a branch and rerun the eval.
3. For whatever drops in score, record it in the ledger as a gap and fix it at the concept level, rather than putting the condition back.

Boundary: anti-gaming for ordinary code repos, where "agents hardcode expected values or special-case test inputs" -> agent-proof-codebase; this section covers only the periodic stripping of conditions added in a prose system just to pass that day's task.

## The user as evaluator

**Speaker practice**: once scale grows, every user (including business experts) needs a feedback entry point that does not go through you. The entry point writes directly to the ledger, using the entry format above, with the slice and session id. Without an entry point, you become the message bus between users and the agent.

## File-based memory

**Speaker practice**: an agent forgets when its session ends, so memory goes in files, in two kinds; a new session starts from "what is true now".

```
memory/
  events.md      # what has happened: appended in time order, never rewritten (may point to ledger ids)
  current.md     # what is true now: current rules, confirmed facts, open questions; updated after every change
```

- When changing `current.md`, state the basis (ledger id), and move the old conclusion into `events.md`.

## Verification discipline (specific to this skill)

- After refactoring prose or a decision table, check that **every branch lands exactly once**: a lost branch is silent (for example, if the "over limit" branch is lost, orders will bypass human approval).
- Have a second agent re-check against the checklist without giving it the author's reasoning; it can also be persuaded into agreeing, so key items still need countable evidence. For the general rule "the author agent must not grade itself" -> agent-proof-codebase.
- For the general method of mutation testing on prose -> agent-proof-codebase; the conclusion for this system: if you deliberately break one sentence of prose and almost no test turns red, that part has to be guarded by humans, so do not pretend there is coverage.

## Deletions need a name and evidence

**Speaker practice**: for anything no real task has touched, write the name and evidence before deleting; each week, real usage decides what to keep. When a variant is retired, its tests are retired with it.

```
deletion:
  name: <name of the variant/branch/paragraph>
  evidence: <zero hits in replay / expert confirmation / last-used time>
  retired_tests: [<tests retired at the same time>]
  approved_by: <person>
```

Count-type metrics (number of variants, number of drifted copies) may only go down (for the general ratchet technique -> agent-proof-codebase).

## Look back: are you only copying one corrector's biases?

**Speaker practice**: skills and memory carry the habits of the main corrector. Check periodically: do recent corrections all come from the same person? Make expert and user annotations a source of corrections too, and pick one question your system still cannot answer to start filling in.

## Boundaries and referrals

- When asked for a statement that can be relayed: **this method is supported by only one case; extending it to code-first systems is Inferred (not stated by speaker) and has not been validated against the original case**; ordinary code repos get referred out first (agents weakening tests or adding skips -> agent-proof-codebase).
- When asked for the speaker's specific numbers (replay volume, trigger rates), the content of the visuals (flowcharts, the "five walls"), or prose compiler details: answer only that there are no verifiable specific numbers/content, the tool is not public, and do not fabricate or give an order of magnitude from impression.
- Outer loops that auto-open and auto-merge PRs: first confirm the inner loop is trustworthy -> agent-environment-design; and this skill requires human approval of every PR.
- When you need to build automated check gates, use agent-proof-codebase.

## Source

Speaker Dmitry Buykin, talk "Agents Made Code Free. Maintaining It Didn't Get Cheaper." (title labeled Maersk), https://www.youtube.com/watch?v=oeknLUsN_GE . Based on auto-generated captions; slides/visuals not verified.
