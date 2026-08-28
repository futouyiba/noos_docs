# Review Candidate: Continuation × Multi-Conversation Role Model Cross-Contract Tightening

- status: review_candidate
- date: 2026-08-28
- source_role: Design Thread
- target_repo: futouyiba/noos_docs
- target_branch: review/harness-continuation-role-cross-contract-v2
- review_rule: Review this exact candidate revision/commit; do not review a mutable live head by implication.

## Work Item

在不推翻 Runtime Object / Authority / State Delta + Reducer 已提交 Baseline 的前提下，收紧 Continuation 与多 Conversation Role Model 的接口，使其能够进入下一轮独立 Review。

Scope:

- Continuation authority boundary
- Provider Conversation active-binding / rollover semantics
- continuation operation identity
- compaction authority boundary
- completion / human-gate semantics
- Review Snapshot immutable target semantics
- reasoning role 与 mutation authority 分离
- stale / revalidation semantics

Explicitly out of scope:

- Execution Journal planned/sent/observed/committed lifecycle
- 完整 crash recovery algorithm
- 重新设计 Runtime Object Model
- 重新设计 State Delta 全部 operation vocabulary
- Reviewer issue adjudication policy

## Proposed Model

### 1. Core split

Harness 中必须明确拆开三种不同控制：semantic orchestration、canonical mutation、runtime/carrier continuation。

#### Semantic orchestration

Owner: Control Role + orchestration policy.

负责回答：

- 下一步做哪个 Work Item？
- 应该进入 Design、Review 还是 Integration？
- 是否真正需要 Human Authority？
- Run 是否具备结束条件？

输出 `Orchestration Proposal`，只有 proposal authority。

#### Canonical mutation

Owner: Policy + Reducer.

继续坚持：

`Proposal → Policy → Reducer`

所有 canonical mutation 必须走这条路径。

#### Execution continuity

Owner: Continuation Decision Machine.

负责回答：

- 已被选择的 Logical Thread 现在怎样继续运行？
- 留在当前 Conversation、compact、refresh，还是 rollover？

它没有 canonical mutation authority，也不拥有独立 durable continuation state。

### 2. Continuation Machine

Continuation Machine 是一个 control-plane decision function，而不是第二个 authoritative state machine。

Inputs:

- canonical Run State snapshot
- Logical Thread identity
- current active Provider Conversation binding
- runtime observations
- browser / adapter health
- latest stable turn/checkpoint
- applicable policy

Result:

- `ContinuationProposal`
- 或 `NoRuntimeAction`

禁止把以下状态作为 Continuation 自己的 durable authoritative state：

- RUNNING
- WAITING_HUMAN
- COMPLETE
- ROLLOVER_PENDING

如果这些概念具有业务语义，必须存在于 canonical state 或 proposal 中；如果只是瞬时执行状态，则属于 runtime / future Execution Journal。

### 3. Continuation actions

Continuation action enum 收窄为：

- CONTINUE_FOCUSED
- COMPACT
- REFRESH
- ROLLOVER

从 Continuation action taxonomy 中移除：

- ASK_HUMAN
- COMPLETE

理由：二者是 semantic workflow claims，而不是 carrier continuation action。

NoRuntimeAction 的典型情况：

- awaiting authorized human input
- no semantic work selected
- thread not runnable
- run not runnable

### 4. Continuation Proposal / operation identity

v0 采用：**一份 Continuation Proposal = 一个 logical continuation operation**。

因此 `proposal_id` 同时作为未来 Execution Journal 的 top-level operation / idempotency key。

Minimum fields:

```yaml
proposal_id: stable across retries of the same logical operation
logical_thread_id: required
base_state_version: required
action: required
expected_active_carrier:
  provider_conversation_id: required for carrier-touching actions
action_specific_inputs:
  checkpoint_id: optional
  context_projection_id: optional
  browser_session_or_attachment_ref: optional
```

Retry semantics:

- same logical operation retry → reuse `proposal_id`
- new continuation decision → allocate new `proposal_id`

本 Contract 不定义 `planned / sent / observed / committed`，也不定义低层 step retry；这些留给 Execution Journal。

### 5. Provider Conversation active binding

不新增 `Continuation Session`。

Logical Thread 拥有至多一个 active Provider Conversation binding：

```text
LogicalThread 1 ---- 0..1 active ----> ProviderConversation
```

Provider Conversation 被替换后仍然存在；replacement 不删除其 identity 或 provenance。

Lineage 可表达：

- predecessor
- supersedes

关键 invariant：

> 创建了一个新 Provider Conversation，不等于它已经成为 active carrier。

### 6. Rollover semantics

Rollover 的语义是：保持 Run 与 Logical Thread identity 不变，替换 Provider Conversation carrier。

#### Safe boundary

Rollover 前要求：

- predecessor assistant turn stable
- no active provider streaming
- relevant State Delta already reduced or explicitly discarded
- checkpoint captured
- Context Projection compiled against a known `base_state_version`
- per-thread continuation executor is single-flight

#### Proposal preconditions

- current state version == `base_state_version`
- current active carrier == `expected_predecessor`
- target Logical Thread still runnable

#### v0 concurrency choice

v0 不新增独立 `binding_version`。

采用：

`base_state_version + expected_predecessor`

作为 compare-and-swap 条件。

理由：这已经足以防止两个基于同一旧 carrier 的 rollover 同时成为 active。独立 binding version 可以减少无关 State update 导致的 conservative stale rejection，但会引入第二套 revision token；v0 优先 correctness。

#### Reducer operation candidate

```yaml
op: replace_active_provider_conversation
fields:
  logical_thread_id: required
  base_state_version: required
  expected_predecessor_id: required
  new_provider_conversation_id: required
atomic_preconditions:
  - base_state_version exactly matches current canonical state
  - active binding still equals expected_predecessor_id
atomic_effect:
  - active binding becomes new_provider_conversation_id
  - record predecessor/supersedes lineage
  - retain old Provider Conversation
  - produce one new canonical state version
failure:
  result: STALE_PRECONDITION
  guarantee: active binding remains unchanged; no partial rebinding
```

#### Concurrent example

Initial:

`State v31, active=C1`

Two proposals:

- A: expected=C1, base=v31, wants C2
- B: expected=C1, base=v31, wants C3

First success performs `C1 → C2` and advances State. The second attempt then fails base-version and/or expected-predecessor preconditions, so C3 cannot become a second active carrier.

### 7. Unbound bootstrap Conversation

Provider UI may require sending a first prompt before a Conversation ID exists。

因此新 Conversation 可能在 active rebinding 之前就已经被创建。

规则：

- 在 CAS 成功之前，它没有 Logical Thread execution authority。
- CAS 失败时，它只是 external execution residue，不得继续产生 canonical work。
- 对 residue 的 crash/recovery 清理属于后续 Execution Journal。

如果 provider 必须通过发送首个 prompt 才能获得 Conversation ID：

- rollover bootstrap turn 只用于建立 carrier + 注入 projection
- bootstrap turn 不视为 normal authoritative worker turn
- bootstrap turn 不允许 canonical state mutation
- discover Provider Conversation identity
- attempt atomic active-binding CAS
- only after CAS success allow normal worker continuation

### 8. Compaction authority

Compaction 只有 projection/artifact authority。

可输出：

- Compaction Artifact
- Carry Context / Context Projection
- optional separate State Delta Proposal

Compactor 绝不能输出“新的 Current State”并直接覆盖 canonical State。

Compaction Artifact 至少记录：

- base_state_version
- logical_thread_id
- source checkpoint / source range
- content hash

如果 Compaction Artifact 与 canonical State 冲突，canonical State 始终优先。

如果 compaction 过程中发现某个 Open Question 应关闭、某个 Decision 应新增或 supersede，必须另外生成 State Delta Proposal，继续经过 Policy + Reducer。

### 9. Completion and Human Gate

`COMPLETE` 与 `ASK_HUMAN` 从 Continuation action taxonomy 中移除。

#### Work completion

表现为 semantic completion claim / orchestration proposal。

Possible targets:

- work_item
- logical_thread
- run

Provider Conversation 自身不使用 semantic COMPLETE；rollover 后旧 carrier 只是 `superseded / inactive`。

#### Run complete

单个 Design / Review / Integration thread 不得把自身完成解释为 Run completion。

Path:

`role output → Control semantic evaluation → Run-completion proposal → Policy → Reducer`

#### Human Gate

Human Gate 表现为 Human Gate request/proposal。

Path:

`role identifies authority dependency → semantic proposal → policy evaluation → canonical waiting/human-gate state if authorized`

授权进入等待后，Continuation 返回 `NoRuntimeAction`，不再自动向 provider 发送“继续”。

### 10. Role Model

Control / Design / Review / Integration 是 reasoning / orchestration roles，不是 mutation authorities，也不是新的 Runtime Object types。

Runtime identity 继续由 `Logical Thread` 提供。

Role representation 倾向：

- Logical Thread 上的 extensible role/kind metadata
- 或 versioned orchestration role assignment

禁止在 Runtime Object Baseline 中硬编码四种新的 durable object class。

Output contract:

```yaml
Control:
  primary: OrchestrationProposal
Design:
  primary: DesignCandidateArtifact
Review:
  primary: ReviewIssueArtifact
Integration:
  primary:
    - IntegratedCandidateArtifact
    - optional StateDeltaProposal
```

Universal rule：角色产生的任何 State Delta 都只是 proposal；没有角色因为名字叫 Control 或 Integration 就获得 Reducer bypass。

### 11. Control vs Continuation

Control Role owns semantic workflow：

- semantic phase selection
- work-item selection
- review/integration scheduling
- human authority escalation
- run completion proposal

Continuation Machine owns runtime/carrier continuity：

- same-carrier continuation
- compaction execution
- browser refresh
- carrier rollover

两层不再共享 generic `next_action` 字段。

Example:

```yaml
Control:
  proposal: "让 Runtime Reviewer 审查 Candidate C8"
Continuation:
  proposal: "该 Reviewer Logical Thread 当前 active carrier 已 degraded，在安全窗口执行 REFRESH"
```

### 12. Review Snapshot

Review Snapshot 是 immutable review-target artifact。

Minimum identity:

```yaml
review_snapshot_id: required
base_state_version: required
target:
  candidate_id: required
  candidate_revision_id: required
  candidate_content_hash: required
```

Candidate 每次 substantive edit 产生新 immutable revision；不在原 revision 上原地修改。

Review Issue 必须引用 `review_snapshot_id`；禁止只引用“当前 Candidate”或 mutable candidate head。

原因：即使 Run State 一直停留在 v37，Candidate 仍可能已经从 C5 演化到 C8；`state_version` 无法表达这种变化。

### 13. Review freshness / stale semantics

Freshness 有两个独立变化轴。

#### State relation

- exact: `current_state_version == snapshot.base_state_version`
- advanced: `current_state_version != snapshot.base_state_version`

#### Target relation

- exact: current candidate content hash == snapshot target hash
- superseded: current candidate content hash != snapshot target hash

Combined classification:

- EXACT
- STATE_ADVANCED
- TARGET_SUPERSEDED
- BOTH_CHANGED

Invariant：freshness relation 只是 revalidation input，不是 Review Issue disposition。

明确禁止：

`stale => OBSOLETE`

Integration 必须重新检查 issue 指向的问题在新 Candidate 中是否仍存在，再产生 disposition。

## Rationale

1. 已提交 Baseline 的核心安全边界是统一 mutation authority；Continuation 若拥有自己的 RUNNING / COMPLETE / ROLLOVER_PENDING durable truth，会形成第二个 reducer。
2. Refresh 与 Rollover 应继续分离：前者解决 Browser runtime，后者改变 Provider Conversation carrier / explicit-history boundary。
3. rollover 的真正一致性问题不是“有没有新 Conversation”，而是谁有资格成为 Logical Thread 当前 active carrier；activation 因此必须成为 reducer-managed atomic relation change。
4. `base_state_version + expected predecessor` 是 v0 足够强的 CAS token。它可能 conservative stale，但不会造成 split brain。
5. 将 COMPLETE / ASK_HUMAN 移出 Continuation 后，completion scope、Human Gate authority、Control/Continuation controller overlap 可以由同一个分层原则解决。
6. Candidate revision 与 Run State version 是两个独立变化轴。Review Snapshot 必须同时冻结两者。
7. Compaction 本质上是 loss-controlled projection，没有理由拥有比 Reducer 更高的 authority。

## Alternatives Considered

### Independent continuation state machine

让 Continuation 自己维护 RUNNING / WAITING_HUMAN / COMPLETE / ROLLOVER_PENDING durable state。

Rejected because it creates ownership overlap with Run State and effectively a second reducer.

### Generic next_action

Control Role 和 Continuation Machine 都输出同一个 next_action。

Rejected because it conflates semantic workflow selection with carrier/runtime execution.

### COMPLETE with scope field

保留 COMPLETE action，只增加 `scope=conversation/thread/work_item/run`。

Better than the original version, but still rejected because semantic completion is not a continuation operation.

### Dedicated binding_version

给每个 Logical Thread 的 carrier relation 增加 binding_version。

Potentially useful later, but not required in v0. `expected predecessor + canonical base version` already provides correctness.

### state_version-only Review Snapshot

Rejected because it cannot identify Candidate revision changes under the same Run State version.

### Role as Runtime Object

Rejected because it hard-codes current orchestration workflow into the Runtime Object Baseline and causes type proliferation.

## Rejected Options

- Continuation 拥有第二套 canonical durable state
- Continuation Session 作为新 durable object
- last-created Provider Conversation 自动成为 active
- COMPLETE 继续作为无 scope 的 continuation action
- ASK_HUMAN 由 Continuation 直接改变 Run State
- Compactor 可以直接覆盖 Current State
- Review Snapshot 只绑定 state_version
- stale 自动转成 OBSOLETE
- 四个 reasoning role 成为 mutation authority
- Control 与 Continuation 共用 generic next_action

## Assumptions

- canonical State 具有可用于 optimistic concurrency 的 `state_version`
- `replace_active_provider_conversation` 可以作为 reducer-managed atomic mutation；底层实现可以不同，但必须保持相同 CAS 语义
- Provider Conversation identity 一旦被 NOOS 观察并登记，不因失去 active binding 而消失
- Candidate / Review Snapshot / Review Issue 可作为 immutable artifact 保存
- Execution Journal 后续可使用 continuation `proposal_id` 作为 logical-operation correlation key

## Remaining Open Questions

1. Provider 实际创建新 Conversation 时，Conversation ID 在 send 前还是 send 后可获得。它影响 rollover bootstrap executor 的具体步骤，但不改变“bootstrap 无 semantic authority + CAS 后才 active”的 contract。
2. Candidate revision ID 最终采用 monotonic revision、UUID 还是 content-addressed ID。Contract 只要求 immutable revision + content hash。
3. 是否未来因高并发需要把 carrier CAS 从 global state_version 优化成 thread-scoped binding revision；当前不提前引入。

## Proposed State Changes

- 对 Runtime Object Baseline 做一个加法式关系补充：Logical Thread 存在 0..1 active Provider Conversation binding，replacement 保留 lineage/provenance。
- 对 Reducer contract 增加窄操作：`replace_active_provider_conversation` with `base_state_version + expected_predecessor` CAS。
- Continuation Candidate 删除 authoritative continuation state；Action taxonomy 收窄为 CONTINUE_FOCUSED / COMPACT / REFRESH / ROLLOVER。
- ASK_HUMAN / COMPLETE 移入 semantic orchestration proposal。
- Multi-Conversation Candidate 将 Review Snapshot 升级为 immutable state-version + candidate-revision snapshot。
- Role Model 明确 reasoning role output type，不新增 mutation authority 或 durable role object type。
- stale 改为 freshness relation / revalidation input，不再隐含任何 disposition。

## Independent Review Targets

Reviewer 应重点攻击以下问题，而不是做文案式 review：

1. Rollover bootstrap → CAS activation → normal worker continuation 是否仍存在 split-brain 窗口。
2. `base_state_version + expected_predecessor` 是否足以作为 v0 carrier CAS，还是存在必须现在处理的 ABA 场景。
3. 将 COMPLETE / ASK_HUMAN 完全移出 Continuation，是否造成必要 execution-continuity signal 丢失。
4. unbound bootstrap Conversation 的“无 semantic authority”是否在 ChatGPT / Claude 等实际 adapter 上可 enforce。
5. Review Snapshot 的 candidate revision/content hash 是否足以覆盖 candidate stale，是否还必须冻结 reviewer-specific Context Projection identity。
6. Compaction Artifact 与 Context Projection 的 authority priority 是否已经足够明确，确保任何摘要都不能覆盖 canonical State。

## Requested Reviewer Output

请输出 Structured Issues。每个 issue 至少包含：

```yaml
id:
severity: blocker | major | minor
claim:
reason:
evidence:
suggested_action:
blocks_candidate_promotion: true | false
```

Review 时请把此 PR 的当前 head commit 视为 immutable candidate revision；如果 head 后续变化，应明确标记旧 Review 的 target revision，而不是自动把旧 issue 视为 obsolete。
