# Independent Reviewer Packet — NOOS Deliberation Harness v1

> Review Target: `docs/deliberation-harness/review-candidate-v1.md`  
> GitHub commit: `fa2173b60bd3fb96b87d80611f3d7347b794da06`  
> GitBook historical review-target revision: `RMBU2hpK6q8oMbJoj0Jt`  
> Role: Independent Reviewer  
> Scope: Product / workflow / semantic-contract audit only.

## Reviewer Mission

请独立审计本 Candidate 是否真的形成了一个轻量、可自举的 Deliberation Harness 需求闭环，而不是把复杂 Coding Harness 的治理机制机械搬到需求讨论里。

不要默认 Candidate 的术语或层级是必要的。重点寻找：

- 不必要对象；
- 不必要状态；
- 用户认知负担；
- workflow friction；
- authority 混淆；
- 能被更简单机制替代的部分；
- 会降低 ChatGPT reasoning quality 的 Harness 行为；
- 会使 v0 过重、延迟自举价值的设计。

## Baseline / Intent

当前产品目标不是“重新实现 Claude Code / Codex Harness”。

ChatGPT Web 被视为高质量 reasoning runtime；NOOS 只在外部提供 durable state、artifact、review 和 coordination。

第一阶段的重点是让复杂需求讨论：

```text
自由讨论
→ 有价值内容沉淀
→ Working Candidate
→ immutable Review Candidate
→ independent Review
→ rework
→ Implementation-ready Spec
```

最终 Codex handoff 暂不经过 NOOS runtime；Accepted Spec 通过 GitHub Markdown / Notion 提供给 Coding Agent。

## Explicit Review Questions

### R1 — Capture 是否必要？

Candidate 把 Capture 定义为 Conversation 与正式文档之间的半结构化缓冲层。

请攻击：

- 是否真的需要独立 Capture / Staging 概念？
- 能否直接通过 document suggestion / comment / edit model 达到同样效果？
- Capture 是否容易退化为另一个 inbox / knowledge dump？

### R2 — ChatGPT-owned semantic capture 是否合理？

当前设计让 reasoning worker 提出 Capture，Shuttle 不自行做 semantic importance classification。

请攻击：

- 这是否会使 ChatGPT 的主 reasoning 被 secondary task 干扰？
- 轻 marker 是否真的足以避免污染？
- 是否应完全改成人工选择，或采用其他触发方式？

### R3 — Section-local Staging 是否过重？

请攻击三轨工作面：

```text
Current Draft
Design Staging
Review Staging
```

它是否帮助用户理解“当前状态”，还是制造项目管理负担？

### R4 — Capture / Artifact 边界是否稳定？

当前原则：短小临时材料留在 Staging；需要独立阅读/修改/引用/Review 的材料升级为 Artifact。

请找 counterexample：

- 是否会频繁来回升级？
- 用户是否需要理解太多对象？
- Artifact 是否其实可以完全由外部 Notion/GitHub 文档承担？

### R5 — Review 回流模型是否过度治理？

当前设计把完整 Review Artifact 与 Review Issues 分开，并把 Issues 放入 Review Staging。

请攻击：

- 对单人 + AI 的需求设计，这是否太像企业 issue tracker？
- 是否直接把 Review 送回 Design Thread 反而更自然？
- 如果只保留一种机制，应该保留什么？

### R6 — Review-ready / Implementation-ready 双 Gate 是否必要？

请检查：

- 两者是否真的表达不同 semantic boundary；
- 是否会制造 gate bureaucracy；
- 是否有更轻的 readiness 表达。

### R7 — Implementation-ready 定义是否正确？

当前定义核心是：

> 实现方不再需要替产品方做重要产品决策；剩余自由度属于 implementation discretion、explicit deferred scope 或 non-blocking uncertainty。

请寻找：

- 反例；
- 模糊边界；
- 会错误阻塞实现的情况；
- 会过早放行的情况。

### R8 — v0 是否还能砍掉 50%？

这是本轮最重要的问题之一。

请提出一个你认为更小的最小闭环，并说明：

- 删除什么；
- 保留什么；
- 为什么价值仍成立；
- 哪些删除会真正破坏核心 invariant。

## Do Not Do

- 不要进入具体 DOM selector / database / React component 等实现设计；
- 不要重新设计 Codex handoff；
- 不要假设所有 Coding Harness 模式都应该复制到文档工作；
- 不要因为 Candidate 已经写得完整就偏向认可；
- 不要只做总结。

## Preferred Output

请输出：

```yaml
review:
  target:
  overall_assessment:
  promotion_recommendation:

  issues:
    - id:
      severity: blocker | major | minor
      target_section:
      finding:
      why_it_matters:
      evidence_or_counterexample:
      minimal_fix_direction:

  simplification_challenge:
    removable_mechanisms:
    irreducible_core:
    proposed_smaller_v0:

  strengths_that_should_not_be_lost:
```

不要替 Designer 直接重写整个 Candidate。Reviewer 的职责是攻击、给证据和最小修正方向。
