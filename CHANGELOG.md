# Changelog

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
