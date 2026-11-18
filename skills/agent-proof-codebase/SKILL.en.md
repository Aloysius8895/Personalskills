---
name: agent-proof-codebase
description: Use when setting up or auditing a codebase so AI coding agents produce trustworthy, mergeable code; when correcting a recurring agent mistake; or when deciding where to invest in guardrails (API and architecture, lint rules, types, compilers, tests, runtime or formal verification) versus prose such as rules, skills, or agents.md. Also use before automating an outer "software factory" loop, to check whether the inner coding loop already produces quality.
---

# Make the agent environment a trustworthy codebase

When using AI coding agents in a repository, follow this skill: change the environment the agents work in before you expect mergeable output. The goal is not to accept slop (low-quality or filler code), but to use enforceable constraints so agents do the right thing by default.

The method in this skill comes from practices described in a talk. It is not a universally proven guarantee. Items labeled **Speaker practice** are Lauren Tan's descriptions of what her team does. Adapt them to the current repository; do not copy extreme rules wholesale (for example, banning about 99% of comments, or auto-merging most PRs). Steps labeled **(inference)** are an operational order added so the method can be followed; the talk itself did not spell them out step by step.

## 1. Core principles

- **Trust, but not blind trust.** Invest in the agents' environment so you can trust that they will do the right thing. Do not assume that generated code is ready to merge when there are no constraints.
- **Encode constraints in code and deterministic checks, not only in prose.** Prefer API design, architecture, refactoring, lint rules, type systems, compilers, and tests. `rules`, `skills`, and `agents.md` should cover only what actually requires judgment.
- **Every time you correct an agent, prefer turning that correction into a reproducible check.** Do not just add "don't do this next time" in a comment or in `agents.md`. Comments go stale, often go unread, and are not enforceable. Agents may also use comments to justify not doing the right thing, or over-rotate on a small correction and treat it as a hard rule that can never be broken.
- **The meta-work of a software engineer:** maintain or even improve code quality while massively increasing productivity. Some people call a high-throughput pipeline a software factory. Get inner-loop quality solid before talking about outer-loop automation.

## 2. The inverted pyramid: where to invest first

Spend time according to this inverse pyramid. Lower layers should be funded first; upper layers cannot replace lower ones.

1. **Deterministic layer A — constrain with the code itself: APIs, architecture, refactoring.**  
   Make the correct usage the default path, and make the wrong usage hard or impossible to write. For APIs that require nuance to use well, consider narrowing them or banning them in product code, leaving them only inside a controlled wrapper.

2. **Deterministic layer B — lint, type systems, compilers, and tests.**  
   Use this together with the layer above to force agents to do the right thing by default. Lint errors should point at the root cause, not leave room for another patch.

3. **Verification layer — runtime verification and formal verification.**  
   Use these to find bugs, check properties and invariants you want to hold true, and judge whether the agent understood your goals and intent.

4. **A thin prose layer — rules, skills, and `agents.md`.**  
   Put only guidance that requires judgment and cannot be written as a deterministic check. This is the last layer, not the first response.

The two deterministic layers are used together to constrain what agents can do and force them to do the right thing by default. Verification is a layer you add on top; it is not a silver bullet. Prose rules cannot replace the layers below.

## 3. Decision procedure for corrections

Whenever an agent makes a mistake, or you want to correct a behavior, follow this order. Do not start by editing `agents.md`.

1. **Describe the mistake.** What did the agent do? What did you want it to do? Was this a one-off slip, or a pattern that will recur?
2. **First ask whether it can become an API or architecture change.** Can a better wrapper, a directory boundary, or a default API eliminate this class of problem so that product code can no longer be written this way?
3. **If not, ask whether it can become a deterministic check:** a lint rule, a type, a compiler check, or a test (including property-based tests). Block new instances of the anti-pattern, and also clean up existing ones in the repo. Otherwise the next agent will copy the old code as the established pattern.
4. **Then ask whether verification can catch it:** let the agent run the program itself, use debugging tools, consult a feature map, look at recordings, screenshots, or diagrams, or formally check critical flows and algorithms.
5. **Only if the steps above cannot be done, or judgment is genuinely required, write it into rules, skills, or `agents.md`.** Avoid turning that prose into an unenforceable ban list. Do not give the agent an escape hatch to explain in prose why this time can be an exception.
6. **(inference):** Record which inverted-pyramid layer the new constraint belongs to, which class of mistake it covers, and how you will verify that it actually catches the problem. Do not turn a one-off case into a hard rule that can never be broken.

## 4. The codebase is the largest input (the garden metaphor)

Agent output depends on input, and the codebase is often one of the largest inputs. Workarounds, anti-patterns, deprecated code, and tech debt are easily copied by the next agent as "this is how we do it here," like weeds spreading through a garden.

Constraints and guardrails are like a concrete foundation: new code grows around them. Keep the codebase as clean as possible so agents are always copying good patterns.

Therefore:

- When you find a bad pattern, handle both "no new instances" and "clean up existing ones."
- When review finds a workaround an agent wrote, do not stop at fixing that one PR. Ask whether it will become input for the next round.
- **(inference):** When cleaning up, keep or write a clear correct example. Otherwise agents may only learn "don't do that" and still not know what to do instead.

## 5. Concrete patterns (all speaker practice; treat as examples)

These are practices from the speaker's team, used to show how the inverted pyramid can land. Do not treat any one of them as a command that must be applied in every repository.

### 5.1 Ban APIs that require nuance, via a framework

Things that require nuance to use well often lead to slop; agents are not good at those APIs. Example: React's `useEffect` has legitimate uses, but it is often misused.

**Speaker practice:** Ban `useEffect` in product code and allow it only inside some of the Dune framework's APIs, eliminating this class of review burden. Dune is a client framework they built for Grok Bot. The speaker compares it to "Next.js for Electron apps." Its value is mainly in built-in constraints and lint, not in technical novelty.

**(inference):** In other stacks, first find APIs that agents misuse repeatedly and that humans also often get wrong. Consider narrowing them into a controlled wrapper, rather than reminding people in prose to "please use this carefully."

### 5.2 Express hard boundaries with directories and dependency rules

**Speaker practice:** They observed that agents often imported Electron main-thread code into the renderer thread, causing long tasks and jank. The approach was two top-level directories (described by the speaker as Electron renderer and Electron main), with a dependency graph analyzer and lint rules forbidding imports between them.

**(inference):** For any boundary where "these two kinds of code must not mix" (process, layer, permission, sync versus async), prefer a directory convention plus enforceable dependency rules, not a verbal reminder.

### 5.3 Do not let comments justify patches; use lint to point at the root cause

**Speaker practice:** Agents use Markdown (docs, comments, rules, skills) to write justifications for not doing the right thing. One example was heavy use of `setTimeout`, `requestAnimationFrame`, and `requestIdleCallback` as quick fixes, papering over an architecture clash between VS Code's imperative style and React's declarative style. Agents also wrote comments defending this approach. Timers then spread, and rapidly creating and destroying them can worsen garbage collection and jank.

They therefore banned about 99% of comments (comments go stale, people often do not read them, they are not enforceable, and they give agents an escape hatch) and replaced them with lint rules that have clear, helpful error messages telling the agent to fix the root cause instead of working around it.

This is a fairly extreme tradeoff, not an industry standard. If you adopt a similar policy, first agree which comments are still allowed (for example, necessary documentation of a public API), then ban the defensive text used to justify "this time can be an exception."

### 5.4 Runtime verification: feature maps, reusable CLIs, and verification artifacts

**Speaker practice:**

- Give agents a way to run the code themselves, and use the same debugging tools human developers use, such as the Chrome DevTools Protocol.
- Put a feature map in the runtime-verification skills: a "site map" of the application's features from the user's point of view. This provides a shared vocabulary so agents do not have to guess what "the thingy in the left sidebar" means.
- Put a custom CLI and driver in the skill so agents reuse the same interface every time, which cuts token usage and is testable.
- Code review shifts more toward verification artifacts: recordings, screenshots, diagrams, and formal results. This assumes a strong verification story. It does not follow that you can stop looking at code entirely.

They also mentioned a skill called state chart in pstack (the speaker also called it potato stack), which visualizes state machines from the user's point of view and makes them easier to review alongside videos and screenshots.

### 5.5 Formal verification and UI property-based testing (not a silver bullet)

**Speaker practice:**

- Use **TLA+** for model checking of major user flows, such as notifications across multiple devices and platforms.
- Use **Lean** to prove properties of certain algorithms, for example that a custom virtualization library should not flicker or jitter.
- Use property-based testing on UIs. Auto captions say they used a library called **Bombadil** (the name appears only in auto-generated captions and is unverified; do not treat that spelling as a confirmed official name).
- Formal verification has already helped them find bugs and looks promising, but it is "certainly not a silver bullet"—just another layer in the agents' environment, and overall still experimental.

**(inference):** Prefer formal methods for flows and algorithmic invariants that must hold, rather than trying to prove the entire application.

### 5.6 Spend tokens on verification

The speaker recounted that someone (Cindy), after not being able to trust her agents because subtle bugs were getting merged, invested in verification; later, most of her daily tokens went into verification. The speaker said that tracks with her own observation. The specific ratio is a second-hand report or self-observation, not a target you must hit.

The point is: close a trust gap with verification, not by generating more code alone. Many verification techniques are not new; they were just previously expensive to write and maintain. Agents lower that cost, so you can test more deeply than used to be economical. Formal verification can also help you reason about how the product should work and whether agents really understand your intent.

## 6. Warning: foundations first, then the software factory

- **Do not automate the outer loop while the inner coding loop is still producing slop.** If the inner coding loop itself produces slop, automating it only produces more slop and burns more and more tokens. First improve the codebase, static analysis, and verification. That work is a prerequisite to automation anyway.
- **Formal verification is not a silver bullet.** Treat it as an additional layer, not the entire safety net.
- **Auto-merging, and the claim that most product code no longer needs deep human review, are the speaker's judgments about her current codebase quality, not a safety guarantee you can copy.** Do not imitate auto-merge when the foundations are not in place.
- Throughput, revert rate, release cadence, and performance comparisons mentioned by the speaker are on-stage claims. The talk does not give the measurement definitions. **Do not treat them as target numbers or verified facts for this skill.**

An outer-loop example from the speaker (only to show what comes after the foundations): once you are ready, orchestration tools can launch Cursor, cloud agents, and similar work, for example fuzzing the app after a new release, using it like a real user to find bugs, then opening fix PRs. Do not start with that outer loop before constraints and verification are reliable.

## 7. Repository audit checklist

Walk the current repository item by item. If something can be a deterministic check, do not stop at writing it into a document.

**Inverted pyramid and priorities**

- [ ] The default response to correcting an agent is to change an API or architecture, or add lint, types, or tests—not to edit `agents.md` or add a comment first.
- [ ] `agents.md`, rules, and skills keep only what requires judgment. If they contain "don't do X again" with no enforceable check, push that down into a deterministic layer.

**The codebase as input**

- [ ] Bad inputs that agents might copy have been identified: workarounds, anti-patterns, deprecated APIs, tech debt, and defensive comments.
- [ ] For each class of bad pattern: lint or tests block new instances, and there is a plan to clean up or isolate existing ones.
- [ ] **(inference):** The repo has at least one clear correct example that can be copied.

**Architecture and boundaries**

- [ ] APIs that require nuance to use well, and that agents repeatedly misuse, have been narrowed or banned in product code and left only inside a controlled wrapper.
- [ ] Boundaries that must not be crossed (for example, process or layer) are enforced by directory structure and dependency or lint rules, not only written in documentation.
- [ ] There are no patches that hide architecture problems with timers such as `setTimeout`, `requestAnimationFrame`, or `requestIdleCallback`. If there are, fix the root cause and prevent regressions.

**Lint and comments**

- [ ] Error messages for important constraints tell the agent to fix the root cause, rather than implying that another patch is acceptable.
- [ ] Comments and Markdown are not an escape hatch for "this time we don't have to follow the rule." (Whether to ban the vast majority of comments, as the speaker's team did, is the team's own decision.)

**Verification**

- [ ] Agents can run the program themselves and, where applicable, use the same key debugging paths as human developers (for example, the Chrome DevTools Protocol).
- [ ] There is a user-facing feature map, or an equivalent shared vocabulary, that reduces vague instructions.
- [ ] Repeated verification steps are packaged as a reusable, testable CLI or skill, rather than having the agent write a script from scratch each time.
- [ ] Review can look at verification artifacts such as recordings, screenshots, diagrams, or formal results. Invariants of critical user flows and algorithms have corresponding checks (tests, model checking, or theorem proving, chosen by risk).
- [ ] Formal methods are treated as an additional layer, not as the only safeguard.

**Automation gate**

- [ ] Before building a software factory, or auto-opening or auto-merging PRs, it has been confirmed that the inner loop will not reliably produce slop.
- [ ] The speaker's team metrics have not been used as evidence that automation is already "safe enough."

## 8. Template for a lint rule error message

An error message should say what is forbidden, why, and what root cause to fix. Do not write a full lint plugin. The sample string below is themed on the timer workaround from the talk and can be rewritten for the actual rule:

```
Do not use setTimeout, requestAnimationFrame, or requestIdleCallback to patch over a timing or architecture problem.
This usually hides a clash between an imperative update style and a declarative UI, and the workaround will get copied by later agents.
Fix the root cause in the data or update flow so the UI can render correctly without a timer.
```

**(inference):** A good error message does three things at once: names the forbidden pattern, explains why it is harmful (including that later agents will copy it), and points in the right direction. Avoid writing only `Don't do this`.

## 9. Source

This skill is based on the talk *All Code Is Slop (Except Mine)* by Lauren Tan (poteto): https://www.youtube.com/watch?v=O_zcc6Xl12g. It was prepared only from notes on that talk and from its English auto-generated captions (no manual track). **Based on captions; visuals were not verified.** Auto captions misheard some proper nouns. The tool name **Bombadil** appears only in the auto-generated captions and is unverified.
