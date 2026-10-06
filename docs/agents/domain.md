# Domain Docs

工程技能在探索代码库时如何使用本仓库的领域文档。

## Before exploring, read these

- 仓库根目录下的 **`GLOSSARY.md`**，或
- 仓库根目录下的 **`GLOSSARY-MAP.md`**（若存在）：指向每个上下文对应的 `GLOSSARY.md`。阅读与主题相关的每个词汇表。
- **`docs/adr/`**：阅读涉及你即将进行修改的区域的 ADR。在多上下文仓库中，还应检查 `src/<context>/docs/adr/` 以获取上下文范围的架构决策。

若上述文件不存在，**请静默继续**。不要标记其缺失，也不要提前建议创建它们。`/domain-modeling` 技能（通过 `/grill-with-docs` 与 `/improve-codebase-architecture` 触发）会在术语或决策真正确定时延迟创建它们。

## File structure

单上下文仓库（绝大多数仓库适用）：

```
/
├── GLOSSARY.md
├── docs/adr/
│   ├── 0001-event-sourced-orders.md
│   └── 0002-postgres-for-write-model.md
└── src/
```

多上下文仓库（根目录下存在 `GLOSSARY-MAP.md`）：

```
/
├── GLOSSARY-MAP.md
├── docs/adr/                          ← 系统级决策
└── src/
    ├── ordering/
    │   ├── GLOSSARY.md
    │   └── docs/adr/                  ← 上下文特定决策
    └── billing/
        ├── GLOSSARY.md
        └── docs/adr/
```

## Use the glossary's vocabulary

当你的输出命名领域概念时（在 issue 标题、重构方案、假设、测试名称中），请使用 `GLOSSARY.md` 中定义的术语。不要随意使用词汇表中明确避免的同义词。

若所需的概念尚未收录于词汇表中，这是一个信号：要么你在创造项目中未使用的语言（需重新审视），要么确实存在术语缺失（记录给 `/domain-modeling`）。

## Flag ADR conflicts

若你的输出与现有 ADR 相冲突，请明确指出而不是静默覆盖：

> _Contradicts ADR-0007 (event-sourced orders), but worth reopening because…_
