---
name: agent-environment-design
description: Decide where guardrails for AI coding agents belong and fix the environment they work in. Use when the user keeps correcting the same agent mistake (代理反复犯同一个错, "next time don't do X", another line in agents.md/CLAUDE.md), asks whether a fix belongs in deterministic code (API/architecture/lint/types/tests), runtime or formal verification, or prose, wants to clean up a codebase agents copy bad patterns from, wants agents to run and verify the app themselves (Chrome DevTools Protocol, feature map, recordings/screenshots), worries agents justify workarounds with comments, wants lint messages that point to the root cause, is about to build a "software factory"/outer agent loop (自动开 PR、自动合并) before checking the inner coding loop, or cites Lauren Tan's "All Code Is Slop" talk. Hardening a repo against gaming tests, mutation testing, CI quality gates, SPEC invariants, ratchets, protected referee files, banned-API checks, go to agent-proof-codebase instead.
---

# 代理环境设计：先改环境，再指望可合并的产出

目标不是接受 slop，而是让代理「默认就做对」：约束写进代码和确定性检查，不只写进散文。

## 来源与标注约定（先读）

- 来源：Lauren Tan（昵称 poteto，自动字幕作 "Potato"）在 AI Engineer 的演讲 *All Code Is Slop (Except Mine)*，https://www.youtube.com/watch?v=O_zcc6Xl12g 。
- **依据仅为英文自动字幕，未核验画面。** 字幕会听错专有名词：Grok Bot 常被写成 Graph Bot/Growth Bot；**Grok Bot** 和 **Dune** 两个名字已用视频简介核对（简介写 "the Grok Bot team at SpaceXSI" 和章节 "Dune and enforceable architecture"）；**Bombadil** 只出现在字幕里，拼写未核验，不当作已确认的库名；pstack 的 state chart skill 名称同样未核验。引用时说明这一点，不声称看过演示画面。
- **讲者实践**：讲者对其团队做法的描述，是单个团队的取舍。移植时按当前仓库取舍，极端规则（禁止约 99% 的注释、自动合并大多数 PR）只作示例。
- **整理推论**：为了让方法可执行而补充的顺序或判断，演讲没有逐步规定。
- 讲者的数字（个人约 3,000 PR、团队约 11,000 PR、约 0.5% 回滚、约每天一个稳定版、10 月约 227 位贡献者、「多维度比上线时更快」、Cindy「约 70% token 用于验证」）是口径未给出的现场主张。引用必带「讲者称」，不当作目标或已核实事实，个人数与团队数不混用。

## 与 agent-proof-codebase 的分工

本 skill 决定护栏放哪一层、纠正怎么沉淀、代码库作为输入、运行时验证、注释逃逸口、lint 文案、软件工厂前的地基。**需要搭建自动检查关卡时，用 agent-proof-codebase**：目录/依赖规则的具体配置、禁用 API 检查、变异测试、CI 闸门、SPEC、棘轮、受保护文件、红队语料、fail-to-pass、形式化/属性测试的搭建。先在这里判层级，再去那里实施。

**请求本身就是「搭闸门」时，转介后收手**（「直接帮我搭」也一样）。只交付三样：
1. 一句话说明这属于 agent-proof-codebase；
2. 一两句层级判断（这类问题落在倒金字塔哪一层，为什么）；
3. 需要用户补充的信息（语言/测试框架、现有 CI、要保护的路径、已有的 SPEC/GATES 文件）。

SPEC 模板、变异测试工具清单、棘轮基线、CI 配置留给那边：它的设计完整且互相配套，这里补一份通用版会重复、还可能冲突。例：「这套是搭闸门，该用 agent-proof-codebase 落地。层级上，『测试不能被删/跳过』属于确定性层，不该靠散文。要开工我需要：测试框架与语言、CI 平台、哪些文件应被保护。」

**找不到 agent-proof-codebase 时**：如实说落地需要它但当前不可用，只给高层路径（先判层级、再建关卡、再验证关卡能拦住坏改动），不编造它的文件或模板，建议安装后再做。安装来源：https://github.com/cassandre60/skills 里的 `skills/agent-proof-codebase`。

## 1. 核心原则

- **信任，但不是盲目信任。** 投资于代理的环境，使你有理由相信它们做对；不假设「生成出来就能合」。讲者把「软件工厂」收束成这件事：信任。
- **每次纠正都优先变成可复现的检查**，不只加一句「下次别这样」。散文不可强制、不可复现；注释会过时、人也不太读；代理还可能拿它当「这次没做对」的理由，或把一次小纠正过度推广成永不能破的硬规则（over-rotate）。

## 2. 倒金字塔：护栏放哪一层

投入顺序，越靠前越优先，后面的层不能代替前面的层：散文代替不了检查（讲者实践）：

1. **确定性层 A — 代码本身：** API 设计、架构、重构。把正确用法做成默认路径，错误用法写不出来。
2. **确定性层 B — 机器检查：** lint、类型、编译器、测试。与 A 一起迫使代理默认做对。
3. **验证层 — runtime 与 formal 验证：** 找 bug，检查应始终成立的性质，检查代理是否理解了你的意图。叠加的工具，不是银弹。
4. **散文层 — rules、skills、`agents.md`：** 只放需要判断、写不成检查的部分。最后一层。

**选层速查**（整理推论）：

| 问题的样子 | 放哪层 | 例 |
|---|---|---|
| 某种写法「根本不该存在」 | 确定性 A/B（封装、禁止、lint） | 禁用易误用 API |
| 某两类代码不能混 | 确定性 A（目录 + 依赖规则） | main/renderer 分离 |
| 「行为对不对」要看跑起来的效果 | 验证层（运行时） | UI 流程、卡顿 |
| 某性质必须在所有状态下成立 | 验证层（形式化/属性） | 流程、算法不变量 |
| 需要品味/取舍，无法判真伪 | 散文层 | 命名风格、产品语气 |

## 3. 纠正决策流程

代理犯错、或你想纠正一种行为时，按此顺序，先于改 `agents.md`（整理推论）：

1. **描述错误：** 代理做了什么？期望什么？是一次性走神，还是会重复的模式？只有重复的才往下走。
2. **能否做成 API/架构改动？** 更好的封装、目录边界、默认 API，把这一类问题整类消灭。
3. **不能，则能否做成确定性检查？** lint、类型、编译器、测试。**拦新的，也清旧的**（见 §4）。
4. **再问能否用验证接住：** 让代理自己跑程序、用调试工具、对照 feature map、看录像/截图；关键流程或算法考虑形式化检查。
5. **都做不到、确实需要判断，才写散文。** 写成判断标准，不写成不可强制的禁令清单，也不留「可以用文字解释为什么这次例外」的口子。
6. **留一条记录：** 这条约束在哪一层、覆盖哪种错误、怎么确认它真的拦住了（例如先写一个会触发它的反例）。

**反例：** 代理第三次在组件里滥用某个易误用的 hook → 在 `agents.md` 加一行「请谨慎使用 X」。这一行没有任何东西强制它；应走步骤 2/3：收窄 API 或加 lint。

## 4. 代码库是最大输入（花园比喻）

代理的输出取决于输入，代码库往往是最大的输入。权宜之计、反模式、废弃代码、技术债务，会被下一个代理当成「这里就是这么写的」复制，像杂草扩散；约束与护栏像混凝土地基，新代码绕着它生长（讲者实践；比喻不是可执行规范）。

- **同时处理「不再新增」和「清理已有」。** 先加检查拦新实例，再清旧实例。
- **审查代理的变通写法时问：它会不会成为下一轮的输入？**
- **保留或写出一个清晰的正确范例**（整理推论）：否则代理只学到「不要那样」，不知道「该怎样」。放在代理搜索时容易碰到的位置，且自己通过全部检查。
- 清理顺序（整理推论）：删死代码和废弃用法 → 把仍在用的坏模式迁到正确范例 → 最后才加严检查，避免一上来就红满屏。

## 5. 具体模式（均为讲者实践，当示例）

### 5.1 禁掉需要分寸的 API

要用好需要 nuance 的东西，往往导致 slop，代理不擅长。例：React `useEffect`（讲者称团队在产品代码里禁止，只允许在 Dune 框架部分 API 内部使用；是他们的选择，不是 React 官方禁令）。Dune 是 Grok Bot 的客户端框架，讲者比作「给 Electron 应用的 Next.js」，价值在内建约束。

**选取标准**（整理推论）：代理反复用错、人类也常误用、且有受控替代方案的 API。

### 5.2 用目录结构表达边界

代理常把 Electron main 进程代码导入 renderer，造成长任务和卡顿；对策是 main / renderer 两个顶层目录，加依赖图分析与 lint 禁止互相导入。这属于确定性 A：用结构而非口头提醒。

### 5.3 注释与文档会成为逃逸口；lint 指向根因

**讲者实践：** 代理会用 Markdown（注释、文档、rules、skills）为「没做对的事」写理由。例：用 `setTimeout` / `requestAnimationFrame` / `requestIdleCallback` 快速修补，掩盖命令式与声明式架构的冲突，还写注释辩护；计时器随后扩散，大量创建销毁加重 GC 与卡顿。他们因此禁止约 99% 的注释，改用带明确报错的 lint，要求修根因。

**整理推论：** 这是他们较极端的取舍。更稳妥的移植：禁的是「为例外辩护的注释」，不是所有注释；先约定仍允许的注释（如公开 API 的必要说明），让任何抑制/例外都走可检查的机制，而不是自由文本。

**lint 报错文案**同时做到三点：点名被禁的写法、说明为何有害（含「会被后续代理复制」）、给出正确方向。示例（取自演讲主题，可改写）：

```
Do not use setTimeout, requestAnimationFrame, or requestIdleCallback to patch over a timing or architecture problem.
This usually hides a clash between an imperative update style and a declarative UI, and the workaround will get copied by later agents.
Fix the root cause in the data or update flow so the UI can render correctly without a timer.
```

### 5.4 运行时验证：CDP、feature map、可复用 CLI、验证产物

讲者实践：

- 给代理自己跑代码的途径，并用人类开发者也用的调试工具，例如 Chrome DevTools Protocol（CDP）。
- 在运行时验证的 skill 里放 **feature map**：从用户视角列出应用全部功能的「站点地图」，提供共享词汇，免得「左边栏那个东西」让代理猜。
- 把自定义 CLI/driver 放进 skill，让代理每次复用同一套命令，减少 token，且可测试。
- 代码审查更多转向看 **verification artifacts**：录像、截图、图示、形式化结果。**前提是验证体系足够强，不能由此推出可以完全不看代码。**
- pstack 里有个 state chart skill，从用户视角把状态机可视化，便于与视频/截图一起审（名称未核验）。

整理推论：feature map 可从路由/菜单/入口逐项列，每项写「用户看到什么叫名字」「如何到达」「怎么判断它工作」；与 CLI 一起放进 skill 目录，并让 CLI 自己有测试。

### 5.5 形式化验证与 UI 属性测试：只作附加层

讲者用 TLA+ 对重要用户流程（如跨设备通知）做模型检测，用 Lean 检查自研虚拟化库无闪烁/抖动，并刚开始用属性测试做 UI（库名 Bombadil，字幕所述、未核验）。她强调仍在实验，已找到不少 bug，**但不是银弹，只是环境里的又一层**。

整理推论：用在「必须成立」的流程和算法不变量上，不试图证明整个应用；它们还能检验代理是否真正理解了意图。

### 5.6 把 token 花在验证上

讲者转述（Cindy，全名与原推未在口播给出）：补上验证后，约 70% 的日常 token 用于验证；讲者称自己类似。要点：**信任缺口要用验证补，不是生成更多代码**。这些技法多不新，只是以前难写难维护而不划算；代理降低了成本，测试可以做到以前经济上不合理的深度。70% 是转述，不是指标。

## 6. 先打地基，再谈软件工厂

用户想搭自动化外层流程（编排 agents 自动开 PR、自动合并、发版后 fuzz 并修 bug）时，先检查：

1. **内层编码循环的产出是否已可信？** inner loop 产 slop，自动化只会产更多 slop 并烧更多 token（讲者原话方向）。
2. **每次纠正是否都已沉淀为检查？** 先改代码库：加 lint 拦新反模式、清旧模式。
3. **验证体系是否扎实？** 静态分析 + 运行时验证（+ 视需要形式化）能否在没人盯着时抓住回归。
4. **以上成立，再设计外层。** 讲者的例子：每切一个新版本，自动拉起 Cursor project 对应用 fuzz、像真实用户一样操作找 bug，为每个 bug 开修复 PR 并自动合并。

**不照搬自动合并。** 「产品代码不再需要深入人工审查」是讲者对其当前代码库质量的判断，不是安全保证。未通过 1–3 时，明确建议先做地基并列出缺什么，不直接给工厂搭建方案。

## 7. 仓库审计清单

- [ ] `agents.md`/rules/skills 里每条「不要再犯 X」都有强制检查；没有的，下沉到确定性层。
- [ ] 已识别代理可能复制的坏输入（权宜之计、反模式、废弃 API、技术债务、辩解性注释），每类都有拦新增和清旧例的计划。
- [ ] 仓库里至少有一处清晰的正确范例（整理推论）。
- [ ] 需要分寸、代理反复用错的 API 已收窄；不可混用的边界由目录和依赖规则强制。
- [ ] 用计时器掩盖架构问题的补丁已修根因并拦回归；关键约束的报错告诉代理修根因，注释/Markdown 没成为「这次可以例外」的口子。
- [ ] 代理能自己运行程序并使用关键调试途径；有 feature map 或等价共享词汇；重复验证步骤已收成可复用、可测试的 CLI/skill。
- [ ] 审查时能看到录像/截图/图示等验证产物；关键流程/算法有对应检查，形式化方法没被当唯一保障。
- [ ] 搭软件工厂/自动合并之前，已确认内层循环不稳定产出 slop；没把讲者的团队数字当安全依据。
