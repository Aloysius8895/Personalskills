---
name: agent-test-backfill-pipeline
description: 'Pipeline for coding agents to batch-write unit tests for existing low-coverage code and push them to human-approved merge - selector scoring with coverage baseline, local small-batch trial, scheduled backfill with parallel child workflows, PR lifecycle state machine, CI-fix tiers with retry caps, git-blame reviewer routing, AI-review P0 handling, dedicated judge agent, label-triggered test PRs for new code, shared state and cooldowns, metrics. Use for the test backfill pipeline / coverage backfill, "have agents write tests for our untested files", "auto-open test PRs on a schedule". Do NOT use for hardening a repo against agents gaming tests, mutation testing, CI quality gates, SPEC invariants or ratchets (use agent-proof-codebase), or for fixing recurring agent mistakes, deciding which layer a guardrail belongs to, or checking the inner coding loop before building a software factory (use agent-environment-design).'
---

# Agent Batch Test Backfill Pipeline

Labels: [Speaker practice] is what the talk described; [Inferred (not stated by speaker)] is judgment added by the note-taker; [From the Duolingo blog] is content in the Duolingo engineering blog (early version) that the talk did not cover. Every threshold is a **tunable parameter**; the "values from speaker practice" are only starting points.

## 0. First confirm the inner loop is trustworthy (do this first)

This pipeline is the outer loop of "auto-open PRs". If the inner loop (a single agent writing a test in your repo that passes CI) is not trustworthy, the pipeline will only mass-produce junk PRs. [Inferred (not stated by speaker), based on agent-environment-design section 6]

Before building any orchestration, do the following and record the results:

1. Run the agent by hand or locally on 5-10 files to write tests, and check whether most of the output is reviewable and passes CI (see section 3).
2. Check whether recurring mistakes have been turned into checks or code changes, rather than just one more line in the prompt. **For how to decide the layer and how to codify the fix, use agent-environment-design.**
3. Check whether the agent can build, run tests, and run lint in the same environment as CI (see section 10).

If any one of the three fails: report what is missing first, and do not give a full pipeline plan.

When asked "what merge rate or coverage gain should we benchmark against": do not offer the speaker's result numbers as targets. Explain that these depend on the repo and the selector and are not comparable; what to provide is the user's own baseline (section 3).

## 1. Hard rule: every PR must be approved by a human before merging

- The pipeline **must not auto-merge**. Every agent PR can be merged only after a designated person approves it.
- If you use the platform's auto-merge, enable it only **after** a reviewer has been assigned, and the effect must be equivalent to "merge only after a human approves". Branch protection requires at least one human approval, and the agent account must have no bypass permissions.
- PR descriptions use a standard template that states "AI-generated" explicitly and carry a standard label (such as `generated-test`).
- The reviewer has four options: approve, push changes directly, reassign, or close with a stated reason. [Speaker practice]

When asked to "just auto-merge everything": refuse this point and explain why (agents will game CI to turn it green, see section 9), and offer an alternative: reduce review cost (routing, judge pre-screening, small PRs) rather than removing the human.

## 2. Division of labor: what to refer elsewhere

| Topic | Where to go |
|---|---|
| Hard rules against agents changing tests / changing production code / deleting assertions, protected files, test_weakening_check, author agents must not self-grade | agent-proof-codebase |
| Using mutation testing to check whether generated tests can really fail; ratchets; "a gate that has never failed proves nothing" | agent-proof-codebase |
| How to codify recurring failures (which layer: codebase change / lint / prose), writing feedback back into prompts and skills | agent-environment-design |

**Refer the same way when the user asks you to "write out the full set of anti-gaming gates"**: this skill delivers only the pipeline-specific parts (section 9: close the PR if non-test files were changed, handle AI review P0 first, the judge criteria list and an independent judge, writing rejection reasons back). For protected files, weakening detection, mutation testing, and ratchets, point to the ready-made scripts in agent-proof-codebase (`protected_paths_check.py`, `test_weakening_check.py`, `manual_mutation_runner.py`, `ratchet_check.py`) and do not rewrite them here.

After the pipeline goes live, schedule mutation-testing spot checks of merged generated tests; use agent-proof-codebase for the specific method. [Inferred (not stated by speaker)]

## 3. Step 1: baseline, slice, local small-batch trial

1. **Record the coverage baseline** (overall and per layer) and write it into a scope document; the baseline and the selector both use only the post-merge CI report (see the "single source of truth" in section 4).
2. **Define the slice**: start with the parts that are architecturally easiest to test (view model / repository / data source, layers with clear dependency injection). Exclude UI and tightly coupled legacy code at first. [Speaker practice]
3. **Local small-batch trial**: one orchestration command -> generate the coverage report -> a script picks files -> one subagent per file writes the tests, and one subagent opens the PR. Run 2-3 batches of 10-40 files each.
   - The goal is to **collect failure modes**, not to chase merge counts. For each PR, record the CI result and the failure reason, and tally by category. [Speaker practice]
   - Batch size and number of batches are starting parameters; if new failure categories keep appearing, add another batch.
4. **For each failure category, pick the lowest-level fix** (use agent-environment-design for the layer decision). Common categories in this pipeline [Speaker practice]:
   - Two test frameworks mixed in one file -> state in the testing skill when to use which;
   - Concrete-class singletons that are hard to mock -> change the codebase to depend on protocols/interfaces, then lock it in with lint;
   - Concurrency and time-based tests causing flakiness -> add rules and examples to the testing skill;
   - Compile-type errors such as access control and missing try/throws [From the Duolingo blog] -> add a pitfall list to the prompt.
5. Exit condition: every major failure category has a corresponding fix, one more batch produces no new failure categories, and the merge rate / first-pass CI rate no longer rises noticeably.

Keep a testing skill in the repo (how to write good tests, how to generate mocks/fakes), and have the agent read it on every run. [Speaker practice]

## 4. Step 2: single source of truth for coverage + selector

**Single source of truth**: on every merge to the main branch, a CI job generates a coverage report and uploads it to shared storage; backfill reads only this report and does not re-run coverage itself. [Speaker practice]

**Selector scoring**: the score is "confidence that the agent can write a good test", not "what most needs testing first". Write it as configuration, not hard-coded:

```yaml
# selector.yaml - all values are tunable parameters
include_types: [view_model, repository, data_source]   # your testable slice
skip_types: [ui_view]            # speaker practice: skip UI for now (better suited to snapshot tests)
weights:
  file_type_match: 2.5           # speaker practice: target types x2.5; start with this, add the rest as needed
  coverage_gap: 0.5              # from the Duolingo blog: more uncovered lines means a higher score
  size: -0.3                     # speaker practice: penalize large files; coefficient given in the blog
  module_priority: 0.5           # from the Duolingo blog: priority by module
legacy_penalty: 0.5              # down-weight legacy / loosely typed files; choose your own coefficient
batch_size: 5                    # from the Duolingo blog: start by picking the top N each round
```

How to set your own values: regress on the results of the section 3 small-batch trial - weight up the file types with high merge rates, and down-weight the types whose CI is often red. After changing coefficients, re-run a batch and compare; do not go by feel. [Inferred (not stated by speaker)]

**Three filters** (after scoring, before starting work) [From the Duolingo blog]:

1. The file already has an open PR -> skip;
2. A corresponding test file already exists -> skip;
3. Within the cooldown period (the state machine gave up on it or a reviewer rejected it) -> skip; for the cooldown length see section 6.

Note: this scoring systematically bypasses the hardest-to-test and possibly highest-risk legacy code and UI, so coverage growth concentrates in the testable slice and overall coverage will not rise evenly; report it by layer (section 12). Covering those areas is a separate project; do not force it onto the agent by turning up the weights. [Inferred (not stated by speaker)]

## 5. Step 3: orchestration and parallel child workflows

The orchestration engine must provide the following (not tied to a particular engine; choose according to your team's existing infrastructure):

- **Scheduling**: backfill runs every few hours (the interval is a parameter);
- **Child workflows**: one child workflow per selected file, run in parallel;
- **Retry + timeout**: agent sessions last anywhere from seconds to hours, and external calls (Git platform, CI, storage) fail;
- **Idempotency**: the same file is never started twice. Use "file path -> in-progress record" as the idempotency key, and write the record before starting work.

Each child workflow: clone the repo -> read the testing skill -> write only tests and necessary mocks/fakes -> open a PR with the standard label -> write the PR record to shared state.

## 6. Step 4: PR lifecycle state machine

Patrol on a schedule (every few hours), iterate over all open PRs with the `generated-test` label, and **take only one action per PR per round**, taking the first matching row from the table below, top to bottom. [Speaker practice: overall structure]

| # | Condition | Action | Cooldown |
|---|---|---|---|
| 1 | Merge conflict | Close and mark | None (just regenerate from a clean main branch) [From the Duolingo blog]; the speaker handles it the same as stale, tunable |
| 2 | Open longer than `stale_days` | Close, mark stale | Yes |
| 3 | CI red and fix attempts >= `max_fix` | Close, mark failed | Yes [From the Duolingo blog] |
| 4 | CI running | Skip, look again next round | - |
| 5 | Already has a reviewer | Skip (waiting on a human) | - |
| 6 | CI green, no reviewer | Route a reviewer (section 8) and assign | - |
| 7 | CI red | Enter fix tiers (see below) | - |
| 8 | Reviewer rejected | Close | Yes |

Starting parameters: `stale_days`: speaker practice is about 3 weeks, while the early version of the blog says 14 days (the two disagree, which shows this value should be set by your reviewers' throughput: loosen it if the review queue is chronically backed up, tighten it if PRs go stale quickly); `max_fix`: speaker practice is 5 attempts; how to tune: look at the success rate of the 4th and 5th fixes, and lower it if they almost never succeed, raise it if the last attempt often succeeds; `cooldown_days`: start at 30 days; how to tune: look at the merge rate of PRs reopened on the same file after the cooldown ends, and lengthen it if still very low, shorten it if very high. [Set per your own repo: look at the share of closed PRs that "would have been merged with a few more days"]

Implementation notes:

- For checks done first, write state before acting, so that if the patrol re-enters it does not take two actions on the same PR (idempotency).
- Write the action table as data/configuration, not scattered across if statements, so thresholds are easy to tune later.

### CI fix tiers

1. **Deterministic errors** (formatting, pre-commit, auto-fixable lint): fixed and pushed directly by a script or CI job, without dispatching an agent. [Speaker practice] The blog's approach: add an `apply-pre-commit` label to trigger an Action that runs pre-commit and commits. [From the Duolingo blog]
2. **Real test/compile failures**: dispatch another agent to read the CI logs and push the fix to that PR; this counts toward the fix attempts. [Speaker practice]
3. **Attempt cap**: once `max_fix` is reached, close the PR and set a cooldown; do not fix forever.
4. **The fix agent is bound by the section 9 hard rules too**: the blog records an incident where a fix agent changed production code. [From the Duolingo blog]

When unsure which category a failure belongs to, treat it as a "real failure" first - misjudging it as a deterministic error makes the script spin idly again and again. [Inferred (not stated by speaker)]

## 7. Cooldowns and shared state

After the state machine gives up on a file or it is rejected, backfill must not immediately pick the same file again, so the two processes must share state. [Speaker practice: shared storage; From the Duolingo blog: one JSON record per file]

Choice of store: anything with atomic writes and conditional updates will do; for a lightweight start, use one JSON file per file.

Sample per-file state record [From the Duolingo blog: field ideas; the other fields are Inferred (not stated by speaker)]:

```json
{
  "file_path": "src/sessions/SessionViewModel.swift",
  "prs": [
    {
      "pr_number": 1234,
      "pr_url": "https://host/org/repo/pull/1234",
      "created_at": "2026-03-15T08:00:00+00:00",
      "fix_attempts": 2,
      "actions": [
        {"at": "2026-03-16T08:00:00+00:00", "action": "fix_ci"},
        {"at": "2026-03-22T14:30:00+00:00", "action": "close_failed"}
      ],
      "gave_up_at": "2026-03-22T14:30:00+00:00",
      "cooldown_until": "2026-04-21T14:30:00+00:00"
    }
  ]
}
```

Rules: backfill reads it before selecting files; the state machine writes it after every action; `cooldown_until` is computed from `gave_up_at + cooldown_days`, not filled in by hand.

## 8. Reviewer routing

After CI goes green, run `git blame` on the **source file** under test, take the main contributor, and assign them as reviewer; then (optionally) enable auto-merge - only as a switch for "merge only after that reviewer approves", see section 1. [Speaker practice]

Additional rules [Inferred (not stated by speaker)]: exclude departed or bot accounts; when the main contributor is unavailable, fall back to CODEOWNERS or the module owner; set a concurrency cap per reviewer so they are not buried under dozens of PRs a day. Reviewer bandwidth is often the real bottleneck [From the Duolingo blog], so prioritize rate limiting and improving per-PR quality over opening more PRs.

## 9. Quality defenses (pipeline-specific parts)

Agents take the shortest path to turn CI green: changing production code to make tests pass, changing/deleting/silencing existing tests, adding long waits in tests. [Speaker practice] The defenses, in order:

1. **Deterministic rules**: close the PR outright if it touches any file other than tests, mocks, fakes, or stubs. For how to make such rules impossible to bypass (protected paths, test-weakening detection, author agents must not self-grade), use agent-proof-codebase. In the pipeline, run it at least at the first step of the state machine.
2. **Handle the AI review's P0 comments first**: if the repo already has an AI code review bot, have an agent resolve all highest-priority (P0) comments before a human sees the PR. [Speaker practice]
3. **Dedicated judge**: a review agent built specifically for this kind of test PR, which reads the diff and rejects on any matched criterion, stating the reason. The criteria list only grows, never shrinks; starting point:

```
judge criteria (any match -> reject, and write out the file, line, and matched criterion number)
J1 Modified production code (paths other than test/mock/fake/stub)
J2 Modified or deleted existing tests (including deleting assertions or loosening assertions)
J3 Disabled/skipped tests (skip, disabled, compiled out conditionally)
J4 Added a long wait / sleep / fixed delay to wait for a result
J5 The test has no assertions, or only asserts "no exception thrown"           <- Inferred (not stated by speaker)
J6 The expected value in an assertion is taken directly from the function under test's own output   <- Inferred (not stated by speaker)
J7 Mocks the object under test itself                                          <- Inferred (not stated by speaker)
J8 Mixes two test frameworks (per your testing skill)                          <- Inferred (not stated by speaker)
```

   Add and remove criteria according to your own failure-mode statistics. The judge must be an independent session / independent agent that does not share context with the agent that wrote the tests; for how to guarantee independence, see agent-proof-codebase.

Every judge rejection **must carry a written reason** (file, line, criterion number); the reasons must be saved, read regularly, and written back into the prompt / testing skill (see section 12).

## 10. The agent's runtime environment matches CI

Before opening a PR, the agent must be able to build, run tests, and run lint locally. A mismatched environment means the PR is opened and only then fails on CI, wasting agent time and (scarce) CI. [Speaker practice]

Fix lint failures in tiers, from cheap to root-cause [Speaker practice]:

1. Give better examples and pitfalls to avoid in the prompt;
2. Add an auto-fix job in CI;
3. Audit the lint rules that cannot be auto-fixed and delete those with weak signal (which also helps humans);
4. The root fix: make the agent's runtime environment match CI's build environment (the same OS/toolchain image).

Diagnostic method [Inferred (not stated by speaker)]: tally the failing rules across failed PRs; usually a few rules account for most failures, so fix those first. If the agent platform temporarily cannot align environments because of cost (the blog records that the early Linux worker could not compile Swift), do 1-3 thoroughly and note "not built locally" in the PR description. [From the Duolingo blog]

## 11. Label trigger: get tests for new code

- An engineer adds a label to their own PR (such as `trigger:write-tests`; name it as you like) -> a webhook triggers the same process.
- The only difference from backfill: **skip the selector** and directly take the testable files changed by that PR.
- Produce a **draft PR** based on the developer's feature branch, with a comment linking back to the original PR; after the developer reviews it and merges it into their own branch, they then merge the original PR. [Speaker practice; branch baseline from the Duolingo blog]
- Sections 1 and 9 still apply.

## 12. Feedback write-back and metrics

- **Feedback write-back**: periodically (such as weekly), read the judge rejection reasons and reviewer close reasons, group them by failure category, pick a fix layer for each category (use agent-environment-design), and update the prompt / testing skill / lint. The criteria list is extended accordingly. [Speaker practice]
- **Metrics** (define them precisely so they stay comparable with each other):
  - Coverage: report **per-layer** and **overall** separately, and state the scope (within the slice vs the whole repo); do not present slice coverage as whole-repo coverage.
  - Pipeline: PRs opened, PRs merged, PRs closed (by reason), judge rejections, first-pass CI rate, average fix attempts, number of files in cooldown.
  - Share of new tests produced by agents: a decline is not necessarily bad (engineers may be writing more themselves); look at the total volume before interpreting. [Speaker practice]
  - Cost: time and spend per agent run, connected to an observability platform.

## 13. Pre-launch acceptance checklist

- [ ] All three items in section 0 have been checked and recorded
- [ ] Every agent PR carries an AI-generated declaration and the standard label; merging requires human approval, and the agent account has no bypass permissions (deliberately try it once: an unapproved PR cannot be merged)
- [ ] A PR that changes non-test files is closed automatically (construct a counterexample to verify)
- [ ] For each of J1-J4, the judge rejects one counterexample PR, with a reason attached
- [ ] The state machine takes only one action per PR per round (re-entry test: run twice in a row without repeating actions)
- [ ] The same file is never started twice; files in cooldown are not picked by the selector
- [ ] The fix attempt cap works: construct a PR that is always red, and confirm it is closed and put in cooldown after `max_fix`
- [ ] The agent environment matches CI, or the differences are noted in the PR
- [ ] The metrics dashboard has per-layer coverage and the distribution of close reasons
- [ ] Spot-check a batch of merged tests with mutation testing (agent-proof-codebase)

## 14. Deliverables

Scope and baseline document | in-repo testing skill | `selector.yaml` and the selector script | scheduled backfill workflow | state machine action table and implementation | per-file state record schema | label-triggered workflow | deterministic rules + judge prompt (including the criteria list) | failure-mode ledger | metrics dashboard.

## Source

Speaker Kush Agrawal (Duolingo), talk "225,000 Lines of Tests, Written and Merged by Agents", https://www.youtube.com/watch?v=D_4Rn6uLdQM . Based on auto-generated captions; slides/visuals not verified. Content marked [From the Duolingo blog] comes from the reference blog https://blog.duolingo.com/ai-ios-unit-test-generation-pipeline/ (early version).
