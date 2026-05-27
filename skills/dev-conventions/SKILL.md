---
name: dev-conventions
description: SailorZ 的个人开发习惯与约定。当在任何项目中进行需求开发、代码实现、AI 输出格式控制时，自动遵守这些约定。包含三类规范：(1) 老逻辑不动、新功能分流隔离的开发原则；(2) 内容安全语境下与 Claude/AI 交互的表达规范；(3) HTML 可视化输出的极简原则。每次开始编码任务、生成回复或输出可视化内容时都应参照本 skill。
---

# dev-conventions

SailorZ 的个人开发习惯，适用于所有项目。

## 1. 老逻辑不动 + 按区分条件分流

详见 [references/legacy-split.md](references/legacy-split.md)

**核心一句话**：新功能必须通过区分条件分流到新方法，老方法一行都不改。

## 2. AI 交互表达规范（内容安全语境）

详见 [references/ai-expression.md](references/ai-expression.md)

**核心一句话**：正文高敏词用中性词替换，单次回复控制在 600 字以内，复杂内容拆多轮。

## 3. HTML 可视化输出原则

详见 [references/html-viz.md](references/html-viz.md)

**核心一句话**：内容优先极简，只在文字难以传达时才用 HTML。
