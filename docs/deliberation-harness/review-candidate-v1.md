# NOOS Deliberation Harness — Review Candidate v1

> Status: **Review Candidate / Frozen for independent review**  
> GitBook review-target revision: `RMBU2hpK6q8oMbJoj0Jt`  
> Scope: Capture / Staging / Working Candidate / Review Return / Promotion Readiness  
> Out of scope: Codex runtime handoff, coding-agent session management, implementation details, full Continuation / Rollover, automatic `/go`, full multi-provider support.

---

## 1. Product Boundary

当前阶段，NOOS Deliberation Harness 只负责把需求、机制、公式、文档结构和方案推敲清楚。

最终交给 Coding Agent 的过程暂不由 NOOS 管理。默认出口是：

- GitHub 中的 Markdown 需求 / Spec；或
- Notion 中的正式需求文档。

随后由 Codex / Claude Code 等 Coding Agent 自行读取并实现。

核心 Job-To-Be-Done：

> 当用户通过 ChatGPT 推敲复杂需求时，让自由讨论可以持续发生，同时消除线程身份、信息沉淀、版本冻结、独立 Review、Review 回流和文档收敛所带来的协作税。

进一步的产品定义：

> NOOS Deliberation Harness 用来把一个模糊、冲突、充满未知的决策空间，通过讨论、沉淀、审查和裁决，逐步收敛成一份可以放心把剩余自由度交给实现 Agent 的规格。

---

## 2. Core Lifecycle

```text
Conversation
  ↓
Capture Proposal
  ↓ human accept
Staged Material
  ↓
Working State
  ├─ Current Draft
  ├─ Design Staging
  ├─ Review Staging
  └─ Open Questions
  ↓ Absorb / Resolve
Coherent Working Candidate
  ↓ Promote to Review
Immutable Review Candidate
  ↓ Review / Revision
Accepted / Implementation-ready Candidate
  ↓
GitHub Markdown / Notion Spec
  ↓
Codex
```

必须始终区分：

- **Raw Conversation**：思考现场，不等于知识，也不等于 Current Truth；
- **Captured / Staged Material**：值得保留但尚未进入正式方案的语义增量；
- **Working Candidate**：当前最能代表“我们现在主张什么”的可变方案 Head；
- **Review Candidate Revision**：被冻结、可被独立 Reviewer 精确攻击的 immutable revision；
- **Committed / Accepted Spec**：最终准备交给外部实现流程的需求文档。

---

## 3. Working Candidate

Working Candidate 不是聊天记录，也不是 Capture 列表。

它表示：

> **如果此刻必须把当前方案交给别人阅读，我们到底主张什么。**

它是一个语义角色，不要求绑定某一种文件格式。

默认一个 Work Item 只有一个 Working Candidate Head。多个备选方案优先作为 Alternatives 共存在一个 Candidate 内，只有确实需要长期独立演化时才考虑 fork。

Working Candidate 可以包含：

- current proposal / proposed model；
- rationale；
- assumptions；
- alternatives / rejected options；
- remaining open questions；
- acceptance criteria；
- supporting artifact references。

性质：

- mutable；
- internally coherent；
- independently readable；
- 不具有 Committed authority；
- 可以保留尚未关闭的 Open Questions。

只有当用户显式执行 **Promote to Review** 时，当前 Working Candidate 才被冻结为 immutable Candidate Revision。

---

## 4. Capture

Capture 的目的不是“把一句话收藏起来”，而是：

> **把 Conversation 中有持久价值、但尚未决定如何进入正式文档的 semantic delta，提升为 durable working material。**

它位于自由讨论与正式文档之间，充当半结构化缓冲层。

典型内容包括：

- requirement；
- constraint；
- principle；
- definition；
- formula；
- diagram；
- evidence；
- counterexample；
- rejected option；
- assumption；
- open question；
- acceptance criterion。

关键边界：

> **Capture ≠ 修改正式文档。**

### Capture vs Absorb

- **Capture**：recall-oriented，重点是“别丢”；
- **Absorb**：coherence-oriented，重点是“把若干材料整合成一致正文”。

两者必须保持分离。

如果把它们合并，用户每次发现一个好想法都必须马上停下来维护正式文档，会严重破坏自由讨论。

---

## 5. Capture Semantic Ownership

当前原则：

> **Semantic importance judgment 主要由正在做 reasoning 的 ChatGPT 提出，浏览器插件 / Shuttle 不自行从 DOM 内容猜测。**

### ChatGPT / Reasoning Worker

负责：

- 判断当前 turn 是否产生 durable semantic delta；
- 提出 Capture Proposal；
- 用自然语言表达其语义；
- 必要时建议大致属于哪个当前文档区域。

### Browser Shuttle

负责：

- 识别 Capture Marker；
- 展示轻量 UI；
- 支持 Accept / Reject / Edit；
- 支持用户手动选中文字补充 Capture；
- 将已确认内容传给 Hub。

不负责：

- 独立判断内容是否“重要”；
- 独立决定其 semantic kind；
- 独立决定正式文档结构。

### Human

保留最终低成本确认权：

- Accept；
- Reject；
- Edit；
- Manual Capture。

### Hub

负责 durable storage、归属关系、状态与后续 Absorb；Capture Proposal 本身不具有正文 mutation authority。

---

## 6. Capture Marker

核心 invariant：

> **如果支持 Capture 会让同一场讨论的回答质量明显下降，那么 Capture 机制就是失败的。**

优先级始终是：

1. 把当前问题想清楚；
2. 给出最好的自然回答；
3. 只有在不损害前两项时，轻量标记少量 semantic delta。

当前倾向使用极轻的 inline / block annotation，例如：

```text
◈ NOOS 不应该替用户做 design judgment，而主要消灭 deliberation coordination tax。
```

或者：

```text
◈ **产品原则：** Capture 应由正在理解当前 Work Item 的 reasoning worker 提出；Shuttle 只负责执行。
```

v0 Markup Contract：

1. ChatGPT **可以**标记 Capture Proposal，但不是每轮必须产生；
2. Capture Proposal 保持在自然正文中，只出现一次；
3. 可见 marker 应极简、可读；
4. 默认 marker 暂定为 `◈`；
5. marker 作用于其前缀的自然 Markdown semantic block；
6. 默认每轮主动 Capture 不超过 3 个，0 个完全正常；
7. Capture semantic delta，而不是重复强调已经存在的重要内容；
8. 不在回答末尾额外重复生成 Captures Summary；
9. 不要求可见正文携带完整 type / target / provenance metadata；
10. Capture Proposal 没有直接修改 Working Candidate 的 authority。

未来可以存在机器侧 richer metadata，但应与人类可见正文解耦。

---

## 7. Staging

Staging 不应该退化成一个全局 Capture 列表。

原则：

> **Capture 尽量靠近最终目标文档和目标 section 暂存。**

三级落点：

1. **Section-local Staging**：知道属于明确章节；
2. **Document-local Staging**：知道属于当前文档，但具体 section 未定；
3. **Unbound Inbox**：连目标文档都暂时不确定，仅作为 fallback。

Staging Item 默认保持轻量，只需要：

- content；
- source；
- target hint；
- status。

不要把每一个 Capture 变成有大量 metadata 的 miniature document。

### Staging Item vs Artifact

- **Staging**：离正文最近的待处理队列；
- **Artifact**：值得独立存在的工作对象。

当某个材料开始需要以下能力中的多项时，应考虑升级为独立 Artifact / 小文档：

- 独立阅读；
- 独立修改；
- 独立引用；
- 独立 Review。

例如：一句新原则继续留在 Staging；一整套 Formula Note / Alternative Analysis / Review Report 升级为独立 Artifact。

---

## 8. Absorb

Absorb 的语义是：

> **基于 Current Draft + selected staged materials，重新形成内部一致的新正文。**

它不是简单 append Capture 原文。

Absorb 应由 reasoning worker 执行，并最好以 **proposed diff / proposed revised section** 的形式让用户确认。

工作节奏：

- Capture：高频、轻量；
- Absorb：低频、综合性较强。

避免“每轮回答 → Capture → 立即 Absorb”的碎片化工作流。

---

## 9. Working State 三轨工作面

针对一个目标 Section：

```text
┌─────────────────────────────┐
│ Current Draft               │
│ 当前我们主张什么             │
├─────────────────────────────┤
│ Design Staging              │
│ 新想法 / Captures            │
├─────────────────────────────┤
│ Review Staging              │
│ Reviewer 发现但尚未处理的问题 │
└─────────────────────────────┘
```

三者分别回答：

- **Current Draft**：现在的方案是什么？
- **Design Staging**：讨论已经产生了哪些尚未吸收的新东西？
- **Review Staging**：当前方案还有哪些未处理的挑战？

目录层可以进一步形成一种“文档健康度”，例如：

```text
1. Product Goal          ✓
2. Core Workflow         ◈ 3
3. Capture Model         ◈ 2   ⚠ 1
4. Review Model                 ⚠ 4
```

---

## 10. Review Return

Reviewer 完成后，不应该：

- 直接覆盖 Working Candidate；
- 把整份 Review prose 无差别塞回 Design Conversation；
- 默认把 Reviewer 建议当作 Designer 必须执行的命令。

Review 应分成两层：

### Review Artifact

保留 Reviewer 完整输出，用于 provenance、审计和回看。

### Review Issues

从 Review 中提取真正需要处理的问题，并挂到对应 Candidate / section 的 Review Staging。

```text
Candidate C5
├─ Review Artifact R12
└─ Review Staging
   ├─ R12-1 blocker
   ├─ R12-2 major
   └─ R12-3 minor
```

> **Artifact 面向审计；Staging 面向工作。**

---

## 11. Review Issue 回到 Design

Review Issue 的权威语义是：

> Reviewer raises challenge; Designer reasons about response.

而不是：

> Reviewer commands; Designer executes.

处理某个 Issue 时，应按需把：

- relevant Current Draft section；
- target Review Issue；
- relevant baseline / captures；

送回 Design reasoning context。

Designer 应重新判断：

1. Issue 是否成立；
2. 如果成立，最小正确修订是什么；
3. 如果不成立，为什么；
4. 是否需要 Human authority choice。

第一版可提供的操作语义：

- Address；
- Dismiss with reason；
- Defer。

重要边界：

> **Designer 修改后，Issue 不应自动视为“已被 Reviewer 验证解决”。**

最多只能表示已经 Addressed；是否真正关闭可以由后续 re-review 判断。

---

## 12. Multi-Agent Review 与 Integration

同一个 frozen Candidate 可以接受不同 review lens，例如：

- Product；
- Architecture；
- Complexity；
- Failure Mode；
- Governance。

真正有用的工作面不应只是“三份 Review Report”，而应把 Issues 尽量挂回相关 section。

Integration 的核心价值是：

> **对同一 Candidate 上来自多个 Reviewer 的 Issues 进行去重、冲突识别、合并和 disposition，而不是简单总结多份 Review。**

第一版不必实现复杂 Integration，但数据与交互语义应避免把 Review Artifact 和 Issue 工作层混为一谈。

---

## 13. Review-ready Gate

一个真实 Working State 可能是：

```text
Working State
=
Current Draft
+ Unabsorbed Design Staging
+ Review Staging
+ Open Questions
```

Promote 时应显式让用户看到未收敛状态：

```text
Before Review
✓ Current Draft exists
⚠ 4 staged materials not absorbed
⚠ 2 open questions
⚠ 1 previous review issue still addressed-but-unverified
```

这些 warning 不一定成为硬阻塞，因为有些 Open Question 本来就值得带给 Reviewer。

因此第一版强调 **promotion readiness visibility**，而不是机械要求“全部清零”。

Review-ready 的含义是：

> **方案已经足够明确，可以被独立 Reviewer 有意义地攻击。**

---

## 14. Implementation-ready

Implementation-ready 不等于“没有未知”。

候选定义：

> **当一个 Candidate 已经明确了要解决的问题、核心用户行为、关键语义和 authority boundary；所有会迫使实现方自行做重要产品决策的问题都已关闭或显式裁决；剩余 Open Questions 被确认是 non-blocking / deferred；并且存在足以验证核心功能和使用体验的 acceptance scenarios 时，它可以被 Promote 为 Implementation-ready Spec。**

更直观的测试是：

> **如果现在把这份 Spec 给一个没有参与前面讨论的 Codex，它是否可以在不做新的产品决策的情况下开始实现？**

如果不可以，说明仍存在 product-blocking ambiguity。

### Readiness dimensions

至少检查：

- **Problem Readiness**：问题、用户工作流和目标价值是否明确；
- **Behavioral Readiness**：核心用户行为和系统反应是否明确；
- **Semantic Readiness**：关键概念、authority 和状态边界是否清楚；
- **Boundary / Failure Readiness**：会改变核心模型的边界条件是否处理；
- **Validation Readiness**：是否存在足够真实的 acceptance scenarios。

---

## 15. Decision Boundary

需求阶段不需要决定所有实现细节。

判断规则：

> **凡是“如果 Codex 做错，会改变产品含义”的东西，应该在 Deliberation 阶段决定。**
>
> **凡是“只影响实现方式，不改变产品含义”的东西，可以留给 Codex。**

进一步分三层：

1. **Product-semantic decision**：必须由 Deliberation 阶段明确；
2. **Interface / contract decision**：用户可见行为或跨边界 contract，应明确；
3. **Implementation decision**：内部实现方式，留给 Codex。

最终 Spec 应显式列出：

### Decided here

哪些行为和语义已经确定。

### Implementation discretion

Codex 可以自行选择哪些技术方案。

### Deferred product questions

哪些产品问题明确不属于当前 scope。

核心目标是：

> **产品语义约束严格，技术实现空间宽松。**

---

## 16. Open Question Classification

Harness 最终真正需要做的，不是消灭未知，而是分类未知。

至少区分：

### Product Blocking Question

若不解决，会迫使 Codex 自己做重要产品选择。

### Implementation Choice

不应该在产品需求阶段继续设计。

### Deferred Product Question

确实是产品问题，但明确不属于当前版本。

只有 Product Blocking Question 影响 Implementation-ready gate。

---

## 17. Acceptance

Acceptance 至少分两类：

### Functional Acceptance

系统行为是否正确。

例如：Accept Capture 后，material 出现在目标 section 的 staging 中。

### Experience / Harness Acceptance

Harness 是否破坏原本想保护的工作方式。

例如：

- 开启 Capture 后，用户仍可以像普通 ChatGPT 一样连续讨论；
- 不要求每轮处理中间状态；
- ChatGPT 不因为 Capture contract 而明显增加重复总结、结构化尾巴或降低深入 reasoning；
- 在一次真实 30–60 分钟需求讨论中，不需要手动把关键段落复制到别处，重要 semantic delta 能被提出、确认并进入正确 Staging。

---

## 18. Current v0 Scope Boundary

当前 v0 重点：

- ChatGPT Conversation ↔ Work Item / Logical Thread 语义绑定；
- lightweight Capture Proposal；
- human confirm / manual capture；
- section/document-local staging；
- Working Candidate；
- Absorb；
- immutable Review Candidate；
- Review Artifact + Review Issues；
- Review Staging；
- Review-ready / Implementation-ready visibility；
- 最终输出到 GitHub Markdown / Notion 文档。

当前明确延后：

- Codex runtime handoff；
- coding-agent session management；
- 自动 `/go`；
- full Continuation / Rollover；
- 自动 multi-review orchestration；
- full Integration workflow；
-复杂 semantic taxonomy；
- hidden rich metadata protocol；
- full issue lineage / verified-in tracking；
- multi-provider adapter parity；
- implementation architecture。

---

## 19. Current Product Principles

1. Harness 首先保护 reasoning quality，而不是最大化结构化程度。
2. 普通讨论不应该被流程频繁打断。
3. Harness 主要在跨越语义边界时出现：Conversation → Capture、Working → Review、Review → Rework、Accepted → Published Spec。
4. ChatGPT 负责语义 proposal；Shuttle 负责执行与 UI；Human 保留 authority；Hub 负责 durable state。
5. Capture 保存 semantic delta，不等于知识库同步。
6. Capture 与 Absorb 分离。
7. Staging 靠近目标文档，而不是形成全局垃圾场。
8. Review Artifact 和 Review Issues 分离。
9. Reviewer 提出挑战，但不自动拥有正文修改 authority。
10. Review-ready 与 Implementation-ready 是两个不同 Gate。
11. 未知问题应被分类，而不是机械清零。
12. 产品语义应收敛，技术实现自由度应尽量保留。
13. 当前阶段不负责 Codex handoff runtime，只负责产出足够成熟的 GitHub / Notion 需求文档。

---

## 20. Review Focus

本 Candidate 当前适合独立 Reviewer 重点攻击：

- Capture 是否真的减少认知负担，还是制造新的 staging debt；
- lightweight marker 是否仍会反向影响 reasoning；
- Section-local Staging 是否过度结构化；
- Staging / Artifact / Current Draft 三层是否必要且边界稳定；
- Review Staging 是否会变成新的行政流程；
- Review-ready 与 Implementation-ready 双 Gate 是否值得保留；
- Product Blocking / Implementation Choice / Deferred Product Question 的分类是否足以支持 handoff；
- 是否能砍掉 50% 机制仍保留核心价值；
- 是否存在更小、更自然的 v0 闭环。
