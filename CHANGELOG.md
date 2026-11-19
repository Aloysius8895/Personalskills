# Changelog

## 2026-11-19

- Added `skills/agent-proof-codebase` from the talk *All Code Is Slop (Except Mine)* by Lauren Tan (AI Engineer, video `O_zcc6Xl12g`).
- Tested the skill on a tiny Python task-list helper: a custom AST lint failed on bare `except:` and passed after the root-cause fix.
- Added `skills/agent-unit-test-backfill-pipeline` from the talk *225,000 Lines of Tests, Written and Merged by Agents* by Kush Agrawal, Duolingo (AI Engineer, video `D_4Rn6uLdQM`). Kept as a separate skill from `agent-proof-codebase`: this one is a test-backfill pipeline (selector, quality gates, PR state machine), not general agent-environment guardrails.
- Tested the skill on a tiny untested Python shop (pricing / inventory / order repository). A selector picked the slice files; quality gates closed or rejected shortcut PRs; good tests raised slice coverage from 0% to 100%; a PR state machine took the expected action on eight fixture PRs.
