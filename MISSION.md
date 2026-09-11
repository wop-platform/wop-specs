# wop-specs 工厂使命（MISSION）

> 本仓：万联易达开放平台（WOP）对外公开规格文档集——协议契约、SDK 统一规格与跨语言测试向量的唯一真源。

## 使命范围

本仓一切变更均为治理事件：规格版本演进、决策记录、索引面同步、黄金向量与互操作样本变更——正道全部是人工 PR（分支保护 + CODEOWNERS + 评审），纪律见 README「规格治理」六条。

AI 工厂链在本仓的定位是**分诊与把关**：triage 对 issue 做二值裁决，只有可机械验证且零语义风险的文档一致性修复才可入链；规格语义变更一律 reject 走人工 PR。

## Triage 判据

1. **使命一致**：属于规格文档的质量维护（typo、断链修复、格式一致），且不改变任何条款语义、版本号、状态列、决策编号、样本计数或字节级合同（crypto-vectors.json / interop-cases.json）；
2. **可机械判定**：完成与否能由可执行验证客观判定（grep 断言、链接检查、字节比对）——doc-only 变更在默认测试门零投影，必须自带可执行验收载体，否则 reject；
3. **不触周界**：不需要修改 PERIMETER 中任何路径。

任一判据不满足 → `reject`（重投协议：补充上下文后可重开，工厂全新评估）。

## 周界（PERIMETER）

`MISSION.md`
`README.md`
`LICENSE`
`.factory/`
`.github/`
`.gitignore`
`.sourcery.yaml`
`crypto/`
`docs/specs/`
`docs/`
`interop/`
