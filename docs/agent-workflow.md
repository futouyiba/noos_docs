# NOOS 多对话 Agent 工作流规范

> 多平台 / 多 harness 的对话任务协作开发同一批仓库时的最小约束。
> 本规范存于权威仓库默认分支；对本规范的修改走第 1 节的 review 流程。
> v0.2（2026-09-16）：经两轮推敲与独立 reviewer 审核后采纳。

## 0. 术语与范围

- **对话任务（conversation task）**：一个会话内的开发 / 设计 / 审核 / 集成单元。
- **orchestrator**：编排、委派、跟踪的主对话。
- **reviewer**：执行独立审核的一方。
- **epic designer**：拥有设计裁定权的一方（人类或指定的设计 agent）。
- **integrator**：常驻的集成会话，拥有合并与部署循环。

本规范只约束协作流程，不约束任何仓库的技术设计；技术契约以各仓库
自身文档及其 SHA 锚定的权威基线为准。

## 1. 硬门禁：先审核，后合并

1.1 门禁在 **merge**：PR 可以先建（标 draft，亦可作为 review 评论的
    载体），但请求合并前必须获得独立 reviewer 的 APPROVE；
    REQUEST_CHANGES 必须修复后重审，直至 APPROVE。

1.2 reviewer 的独立性由**行为**定义，不由会话或模型身份定义：
    (a) 任务书证伪导向，不预述预期结论；
    (b) 亲跑关键命令并引用实际输出，不采信实现者转述；
    (c) 对核心不变量做退化（变异）验证。

1.3 PR 描述必须引用 review 证据链接与被审 **exact head** commit SHA，
    integrator 合并前核对二者一致。reviewed head 之后的任何 commit
    都触发增量复审（内容不限，补测试同样算）。实现者若认为
    REQUEST_CHANGES 的 finding 有误，可携证据请求重审，但未撤销前
    不得合并。

1.4 审核深度分级，按 diff **触及路径**判定而非实现者自述：触及
    运行代码的按 1.2 全项；docs-only 可降级为引用核对。reviewer 与
    integrator 均可升格深度。

1.5 实现者自身的测试结果不构成独立审核结论。

## 2. 裁定与确认的传递

2.1 需要 epic designer 审核、确认或裁定的内容：先写成 proposal
    文档（含冲突描述、待裁定问题、建议方案、影响面），存放于
    待实现仓库的 docs/，以 GitHub issue 或 PR 评论的形式传递。

2.2 流转提示词保持极简：verdict + 文档引用 + 具体请求；不复述
    契约原则与边界。

2.3 裁定记录必须注明交付来源（谁、在何处交付）；**决定性表述
    原文引用**，禁止转述改义或代拟 designer 结论。

2.4 proposal 的作者不得裁定自己的 proposal。

2.5 冲突优先级：语义与契约归 epic designer，实现正确性归
    reviewer。reviewer 若因技术异议拒绝 designer 的要求，异议
    连同证据回流 designer 重裁，不得僵持也不得互相覆盖。

## 3. 角色分工：orchestrator 委派

3.1 orchestrator 负责编排、任务拆分、状态跟踪与对外汇报；
    实现与具体设计交给对话任务完成，harness 支持时可自动启动
    新任务。

3.2 例外：机械、无设计含量、逐次声明的改动（如笔误、注释、
    十行以内的机械修复）orchestrator 可直接完成并在提交信息
    注明。同文件同主题反复使用该例外视为违规。

3.3 每个任务保留 provenance：来源请求、裁定引用、review 证据。

3.4 工作区规约：每任务在独立 branch / worktree 上工作；
    主 checkout 保留给 integrator。

## 4. Integrator

4.1 集成由常驻的专门 integrator 会话执行。integrator 必须
    **无状态可恢复**：一切真源在仓库文档与项目记忆，任何人都
    可重启新会话接管。

4.2 合并纪律：核对 reviewer APPROVE 与 exact head → review
    intake 留档 → 合并 → 验证（typecheck / 测试 / 发布脚本）→
    构建与部署 → 在 PR 回帖记录 merge commit 与构建时间戳。

4.3 integrator 不顺手实现：集成中被阻塞的修复回流给任务，
    或按 3.2 机械例外处理。

4.4 寻址：本机直连会话为主（登记于各仓库 AGENTS.md / 项目
    记忆）；跨平台兜底通道为约定标签的 issue 收件箱（任何平台
    的任务都可评论，integrator 轮询或被通知）。

## 5. 环境注记

各仓库在自身 AGENTS.md 中追加仓库特定事实（构建命令、部署循环、
已知环境坑），不写入本规范。

## 6. 版本与引用

本规范存于权威仓库默认分支，变更走第 1 节流程；引用规范时按需
以 commit SHA / tag 锚定特定版本。裁定与合同类文件始终 SHA 锚定。

## 7. Bootstrap

v0.2 由人类操作者授权、经独立 reviewer 审核后生效；epic
designer 可按第 2 节对后续修订作出裁定。

## 附录 A：可选 CI 门禁

第 1.3 节可由 CI 强制：PR body 必须匹配"review 证据链接 +
head SHA"模式（如包含 `review` 链接与 7–40 位十六进制 SHA），
否则阻止合并。各仓库自行决定是否启用。

---

*维护约定：本规范的每次修改本身就是一个对话任务——走 branch、
review、merge，不以直接提交的方式改动。*
