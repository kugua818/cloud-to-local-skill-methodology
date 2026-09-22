# Cloud-to-Local Skill Translation Methodology

**云端技能 → 本地弱模型转译方法论**

[English](#english) · [中文](#中文)

---

## 中文

### 这是什么

一套**开源编译方法论**：把依赖云端强模型的复杂 Prompt / Skill / Agent 工作流，编译成 **7B–35B 本地弱模型**可稳定执行、且上下文峰值恒定（目标 < 3.8K tokens）的 Local Skill。

经验可无限累积，**运行代价不增长**。

### 解决什么问题

| 强模型能扛 | 弱模型扛不住 |
|------------|--------------|
| 长上下文硬吃复杂技能 | 上下文小，关键注意力被挤爆 |
| 隐式纠错、模糊指令 | 易跑偏、自创流程、乱输出格式 |
| 隐式跨会话学习 | 无记忆，同一坑反复踩 |
| 读写文件自由发挥 | 一旦可写就会创建未定义文件 |

### 核心机制（一句话版）

1. **五条公理**：运行时信息最小化 / 绝对读写隔离 / 独立证据驱动 / 编译运行解耦 / 最小充分编译
2. **PCLP 五层信息分级（L0–L4）**：Token 硬预算 + 物理级上下文隔离
3. **双向经验门禁**：失败坑 ≥3 Session + 恢复率 ≥80% 才固化；成功路径 ≥5 Session + 成功率 ≥95% 才硬化为代码
4. **零写权限护栏**：弱模型只文本回报经验，一切落盘由宿主执行
5. **C0–C4 编译分级**：命中最小充分条件即停，禁止跳级
6. **嵌套全量内联（克迁移）**：跨文件/跨技能依赖先自包含，再隔离
7. **P0 参数前置门**：参数不对，后面全是猜测
8. **可选自愈扩展**：未知错误 → 强模型定向修复 → 弱模型重验 → 经验固化

### 文档

| 文件 | 内容 |
|------|------|
| [METHODOLOGY.md](./METHODOLOGY.md) | 完整方法论正文（公理 / 分级 / 门禁 / 流水线 / 记分卡） |
| [LICENSE](./LICENSE) | MIT |

### 快速上手

见 [METHODOLOGY.md · 附录 B：最小可落地清单](./METHODOLOGY.md#附录-b最小可落地清单quick-start)。

### 适用对象

- 要把云端 Skill 降级跑在 Ollama / llama.cpp / LM Studio 等本地模型上的工程师
- 希望 Agent 工作流 Context 恒定、可审计、可积累经验的开发者
- 研究「强模型 → 弱模型」编译 / 蒸馏式 Prompt 工程的人

### License

MIT

---

## English

### What is this

An **open-source compilation methodology** that translates complex cloud / strong-model Prompts, Skills, and Agent workflows into **Local Skills** that run reliably on 7B–35B weak models with a **constant context peak** (target < 3.8K tokens).

Experience can grow without bound — **runtime cost does not**.

### The problem

| Strong models cope | Weak models fail |
|--------------------|------------------|
| Long context swallows complexity | Small window; key attention gets crowded out |
| Implicit correction, fuzzy instructions | Drift, invented procedures, broken formats |
| Implicit cross-session learning | No memory; same pitfalls again and again |
| Free file I/O | Zero-write required; free form invents files |

### Core mechanisms (one-liners)

1. **Five axioms**: minimize runtime info · absolute read–execute separation · independent-evidence evolution · compile/run decoupling · least-sufficient compilation
2. **PCLP five-layer info tiers (L0–L4)**: hard token budgets + physical context isolation
3. **Bidirectional experience gates**: failure traps need ≥3 sessions & ≥80% recovery; success shortcuts need ≥5 sessions, ≥95% success & regression pass before hardening into code
4. **Zero-write guardrail**: weak model only reports ``[EXPERIENCE]`` text; the host persists everything
5. **C0–C4 compile levels**: stop at the least-sufficient level; no skipping
6. **Full nested inlining (clone-migration)**: self-contain dependencies first, then isolate
7. **P0 parameter gate**: wrong params make every downstream guess invalid
8. **Optional self-heal extension**: unknown error → strong-model targeted fix → weak-model re-verify → experience solidify

### Docs

| File | Contents |
|------|----------|
| [METHODOLOGY.md](./METHODOLOGY.md) | Full methodology (axioms / tiers / gates / pipeline / scorecard) |
| [LICENSE](./LICENSE) | MIT |

### Quick start

See [METHODOLOGY.md · Appendix B: Minimal Checklist](./METHODOLOGY.md#附录-b最小可落地清单quick-start).

### Who is this for

- Engineers porting cloud Skills onto local runtimes (Ollama / llama.cpp / LM Studio, etc.)
- Developers who want constant-context, auditable, experience-accumulating agent workflows
- Researchers of strong→weak compilation / distillation-style prompt engineering

### License

MIT

