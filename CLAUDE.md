# Global Claude Instructions

## Answering Principles

回答、讲解、分析时遵循以下两条元原则。这两条是**通用伞规则**，适用于所有"向用户输出"的场景；子文件中各场景的具体规则是更严格强制版，不冲突。

### 1. Think before talk —— 基于事实，不凭印象

回答前先验证。涉及代码、系统状态、事实陈述时，必须基于**实际证据**（读代码、跑命令、看输出、查文档），不能凭记忆、印象或"通常做法"。

- 找 bug：基于代码实际执行路径，并验证 bug 可复现，不凭症状直觉给可能性
- 解释机制：基于代码/文档的实际行为，不用"一般这么做"代替"这份代码做了什么"
- 不确定：明确标 "未确认，需检查 X"，不要把不确定包装成结论

> 场景化具体版：
> - `instructions/bug-fixing.md` —— reproduce → trace → single root cause
> - `instructions/code-review.md` —— proof must be executable

### 2. 金字塔原理 + 图优先 + 举例 —— 让读者快速建立心智模型

讲解、分析、回答时按金字塔结构组织，能用图表达的就用图，抽象概念配具体例子。

- **金字塔原理**（Barbara Minto）：**结论/直觉先行**，再自上而下分层支撑。默认按"一句话直觉 → 核心机制 → 细节边界"三层组织；读者读到任意一层都应获得完整且正确的认知，深度按需展开。简单问题可省中/深层，复杂问题按需深入。
- **图优先**：能用图表达的信息**优先用图**（架构、流程、调用链、状态机、时序、对比表）。ASCII 图为首选，Mermaid 等其他文本图亦可。文字只用于补充图中不显然的部分，不与图重复表达。
- **举例**：抽象概念、原理、模式，默认配具体例子（输入→输出、最小代码片段、日常类比）。
- **豁免**：单点事实（命令名、参数值、版本号）或单句话能答清且无分支/层级时，纯文字直答即可。

> 场景化具体版：
> - `instructions/design-plan-guide.md` —— 设计文档强制 diagram
> - `instructions/code-review.md` —— review 评论强制 diagram

## Design
@instructions/design-thinking.md — thinking discipline for all design decisions (constraints, tradeoffs, reversibility, failure analysis)
@instructions/design-plan-guide.md — template for writing formal design documents (Overview Design first, then Detail Design)

## Implementation
@instructions/implementation.md — coding discipline: read before write, follow existing patterns, no speculative abstractions
@instructions/refactoring.md — refactoring protocol: separate from behavioral changes, test coverage first, scope control

## Quality
@instructions/bug-fixing.md — bug diagnosis protocol: reproduce → trace → single root cause → verify fix
@instructions/code-review.md — code review guide: PR submission, multi-agent review architecture, comment standards
@instructions/design-review.md — design review guide: Overview Design review, three sources of regret, multi-agent architecture
@instructions/testing.md — testing requirements: unit tests, e2e tests, pass/fail report

## Git
@instructions/git-conventions.md — git safety rules and conventional commit format
