# Changelog

## 2026-11-20 (split of agent-code-verification-and-maintenance)

- Added `skills/agent-test-backfill-pipeline` from the talk *225,000 Lines of Tests, Written and Merged by Agents* by Kush Agrawal, Duolingo (AI Engineer, video `D_4Rn6uLdQM`). It is the test-backfill part of `agent-code-verification-and-maintenance`, split out and rebuilt with Claude skill-creator, reviewed with writing-for-agents.
- Added `skills/agent-prose-system-upkeep` from the talk *Agents Made Code Free. Maintaining It Didn't Get Cheaper.* by Dmitry Buykin (AI Engineer, video `oeknLUsN_GE`). It is the long-term maintenance part of `agent-code-verification-and-maintenance`, split out and rebuilt the same way.
- `agent-code-verification-and-maintenance` is kept for now; whether to remove it is still open.
- Tested `agent-test-backfill-pipeline` on a tiny untested Python shop in a local git repo (64/64 checks). The first trial batch failed the inner-loop gate (2 of 5 files reviewable); after a lint and a codebase change, the second batch was 6/6 green. The selector's three filters, idempotent parallel children, the table-driven PR state machine (stale, conflict, fix cap of 5 with a 30-day cooldown, script-only format fixes), git-blame reviewer routing with a cap, judge rejections for J1-J8, the human-approval merge gate and the label trigger all behaved as described. Slice coverage went from 0.82% to 15.05% (view-model layer 10.91% to 96.36%, UI layer still 0). A mutation spot-check killed 4 of 5 mutants. Found one wording issue: read literally, row 5 of the action table ("already has a reviewer -> skip") hides row 8 ("reviewer rejected -> close").
- Tested `agent-prose-system-upkeep` on a tiny four-slice refund/amend prose system with 48 frozen sessions (47/47 checks): slice guard and human-only merges, prose lint, minimal context and handles, row-or-NO_MATCH decisions, a code-only write path with an idempotent journal, signal triage with a sibling-slice search, walk solidifying, replay blast radius, dropped-branch detection, anti-overfitting strip, batch-tail check, evidence-backed deletion, ratchet and file memory.
- Not tested (accounts or paid services needed): real Git platform settings, hosted orchestration and storage, real LLM agents/judges/review bots, dashboards, and the speaker's unpublished prose compiler.

## 2026-11-20

- Replaced `skills/agent-proof-codebase` with `skills/agent-environment-design`, a rebuild of the same talk (*All Code Is Slop (Except Mine)*, Lauren Tan, video `O_zcc6Xl12g`) made with Claude skill-creator and reviewed with writing-for-agents. The old `agent-proof-codebase` directory is removed. The new skill points to the separate upstream skill `agent-proof-codebase` at https://github.com/cassandre60/skills for gate building, and keeping a different skill with the same name here would clash with that.
- Tested the new skill on a tiny Electron-style JS project: a root-cause lint caught 5 timer patches, cross-boundary imports, and justification comments; cleanup made it pass; a new counterexample was blocked; a runtime test passed without timers.

## 2026-11-19

- Added `skills/agent-proof-codebase` from the talk *All Code Is Slop (Except Mine)* by Lauren Tan (AI Engineer, video `O_zcc6Xl12g`).
- Tested the skill on a tiny Python task-list helper: a custom AST lint failed on bare `except:` and passed after the root-cause fix.
- Added `skills/agent-unit-test-backfill-pipeline` from the talk *225,000 Lines of Tests, Written and Merged by Agents* by Kush Agrawal, Duolingo (AI Engineer, video `D_4Rn6uLdQM`). Kept as a separate skill from `agent-proof-codebase`: this one is a test-backfill pipeline (selector, quality gates, PR state machine), not general agent-environment guardrails.
- Tested the skill on a tiny untested Python shop (pricing / inventory / order repository). A selector picked the slice files; quality gates closed or rejected shortcut PRs; good tests raised slice coverage from 0% to 100%; a PR state machine took the expected action on eight fixture PRs.
- Replaced `skills/agent-unit-test-backfill-pipeline` with `skills/agent-code-verification-and-maintenance`, merging the Duolingo talk (`D_4Rn6uLdQM`) and the Maersk/Dmitry Buykin talk (`oeknLUsN_GE`) into one skill as requested. The old skill name is retired; its directory should be deleted from the repo.
- Kept it separate from `agent-proof-codebase`: that skill is about general agent-environment guardrails, while this one covers a test-backfill pipeline plus long-term maintenance (slices, prose-as-code, weekly cleanup).
- Re-ran the earlier shop-repo pipeline test for Part A (still passing). Tested Part B on a tiny two-slice order-amendment system: slice guards, prose mutation checks, limit drift, write-path journal, blast-radius replay, dropped-branch detection, and a ratchet cleanup all behaved as the skill describes.
