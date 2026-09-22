# v0.3.2：跨角色通信精简提案

关联 [#19](https://github.com/futouyiba/noos_docs/issues/19)。本提案由用户在
审阅通信成本分析后以“按你的思路修改”明确授权实施。

## 问题

现行流程把授权、唤醒、审计记录和证据传递叠在同一组自然语言消息上。
一次正常 PR 会在直连消息、委派评论、PR body、`IMPLEMENTED`、
`INTEGRATED` 和 issue 验收中重复 head、review 链接、验证结果与范围。
这增加等待和恢复成本，但没有增加独立性；协议也已声明 marker 是审计
协议而非安全边界。

## 决定

保留 implementer、reviewer、integrator、orchestrator 的职责分离，
保留独立 review、exact-head、新提交复审、集成核验和验收关单。只压缩
通信与重复验证：

1. 人类可按仓库／epic、动作、风险和有效期授予持续授权；有效范围内
   后续对象只需指针式唤醒，越界、撤销或找不到授权时才升级人类。
   任务线可以保存授权来源指针供接管恢复，但必须能回读原授权记录。
2. 每项事实只有一个持久维护处；委派和直连消息不复制证据包。
3. DESIGN 记录来源文件路径、Git blob 与裁定射程。等价捷径只接受完整
   文件 blob 相同且射程不变，并记录 source／target；其它情况重新裁定。
4. PR body 是交付清单；`IMPLEMENTED` 仅在 watcher 或关系不易推导时
   作为可选的 `PR + head` 指针。仅当 PR body 明确链接任务 issue 且
   issue 时间线能反查 PR 时，才可省略。
5. Integrator 证明审核对象等于合并对象、主分支正确及适用集成检查
   通过；同树情况下不默认重跑 reviewer 的语义审核与变异探针。
6. 成功只通知 orchestrator；需要修复、重审或 follow-up 时才回流
   implementer／reviewer。
7. 单 PR 完整满足单 issue 时，验收可合并进 `INTEGRATED`；复杂或部分
   验收仍在 issue 单独记录。

## 不变边界

- 评论本身仍不授权敏感动作；持续授权必须先在目标角色会话成立。
- 持续授权不替代 review、exact-head、CI、阻塞裁决或验收。
- Reviewer 与 Integrator 不合并角色；作者验证仍不构成独立 review。
- 设计内容变化、射程扩大或 reviewer 提出语义异议时仍须重新 DESIGN。
- 多 PR、部分完成、残项复杂或实际 merge tree 变化时不得使用精简路径
  隐去证据。

## 验证与回退

以曾出现的四类路径回放：审核通过但反复等待合并授权、缺少
`IMPLEMENTED` 但 PR body／review 完整、准备 DESIGN 与最终内容字节
相同、单 PR 自动关闭后重复写长验收。验证精简后仍能仅凭 issue、PR、
标记及授权会话恢复正确状态，并确认 REQUEST_CHANGES、新 head、部分
验收和授权越界均 fail closed。

若回放发现状态不可恢复，恢复相应持久记录要求；不恢复重复正文、逐 PR
人工确认或 Integrator 对 reviewer 深审的机械重跑。
