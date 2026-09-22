# NOOS Project Instructions

本投影只提供跨平台启动约束；权威流程是 `futouyiba/noos_docs` 默认分支的
`docs/agent-workflow.md`。冲突时以 canonical 为准。

处理 NOOS 仓库任务时，先读取 canonical、目标仓库 `AGENTS.md`、task issue、
PR 和被引用的 SHA 锚定资料。Memory、历史对话和 Project Knowledge 只作
检索线索。

长期不变式：

- 实现者自测不等于独立 review；reviewer 使用独立执行上下文。
- Review 锚定 PR exact head；head 改变后重新 review。
- 评论、marker 和唤醒指针不自动授权敏感动作；有效的有界持续授权无需
  逐对象重问。
- merge、部署、关单及其它敏感动作必须同时满足 canonical 和仓库门禁。
- 多角色动作按依赖分阶段执行，不绕过角色独立性。
- Harness promotion／closure 没有 governing rule 时 fail closed。

Project Instructions 保存稳定启动约束；`AGENTS.md` 保存仓库事实；issue／PR
保存任务状态；canonical 保存流程。不要在投影中复制完整协议。
