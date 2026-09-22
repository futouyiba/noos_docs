# v0.3.3：最小核心提案

关联 [#21](https://github.com/futouyiba/noos_docs/issues/21)。用户要求删除
“a 也可、b 也可”及模型可自行判断的内容。

## 决定

- canonical 只保留角色边界、独立 exact-head review、设计裁定、授权、
  集成后验收和无状态恢复。
- 删除 `IMPLEMENTED`、强制 `intg:` 委派记录、DESIGN 等价专用格式、
  解析别名清单、逐项通知分支和重复流程展开。
- 风险相称的审核、验证、通知对象、传输通道和记录正文交给执行模型判断。
- PR body、结论 marker 和 `INTEGRATED` 仍提供恢复所需的唯一真源。

## 不变边界

独立 review、exact head、新提交复审、可核验授权、CI／阻塞项、实际合并
核验及集成后验收不得省略。
