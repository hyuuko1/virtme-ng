# Issue tracker: Local Markdown

本仓库的 Issue 与需求规格保存在 `.scratch/` 目录下的 Markdown 文件中。

## Conventions

- 每个特性/功能独立目录：`.scratch/<feature-slug>/`
- 特性规格说明文件：`.scratch/<feature-slug>/spec.md`
- 实现工单每个 ticket 单独建档：`.scratch/<feature-slug>/issues/<NN>-<slug>.md`，编号从 `01` 开始，禁止合并为单个工单文件
- Triage 状态记录在每个 issue 文件顶部的 `Status:` 行（角色字符串参见 `triage-labels.md`）
- 评论与讨论记录追加在文件底部的 `## Comments` 标题下

## When a skill says "publish to the issue tracker"

在 `.scratch/<feature-slug>/` 目录下创建新文件（若目录不存在则创建）。

## When a skill says "fetch the relevant ticket"

读取引用路径对应的文件。用户通常会直接传入文件路径或 issue 编号。

## Wayfinding operations

供 `/wayfinder` 技能使用。**Map** 是一个文件，每个 ticket 对应一个 **child** 文件。

- **Map**：`.scratch/<effort>/map.md`（包含 Notes / Decisions-so-far / Fog 正文）。
- **Child ticket**：`.scratch/<effort>/issues/NN-<slug>.md`，从 `01` 开始编号，并在正文中包含问题。`Type:` 行记录工单类型（`research`/`prototype`/`grilling`/`task`）；`Status:` 行记录 `claimed`/`resolved`。
- **Blocking**：文件顶部包含 `Blocked by: NN, NN` 行。当列出的所有文件状态均为 `resolved` 时，该工单即视为解除阻塞。
- **Frontier**：扫描 `.scratch/<effort>/issues/` 中处于打开、未阻塞且未被认领的文件；按编号顺序最先匹配的胜出。
- **Claim**：在执行任何工作前设置 `Status: claimed` 并保存。
- **Resolve**：在 `## Answer` 标题下追加解答内容，设置 `Status: resolved`，随后在 `map.md` 的 Decisions-so-far 中追加上下文指针（要点摘要 + 链接）。
