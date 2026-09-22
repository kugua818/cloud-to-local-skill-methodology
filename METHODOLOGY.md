# Cloud-to-Local Skill Translation Methodology

**云端技能 → 本地弱模型转译方法论**

> 版本 1.0 · 开源协议：MIT
> 适用对象：任何需要把「强模型才能稳定跑的 Prompt / Skill / Agent 工作流」降级到 7B–35B 本地弱模型的工程场景。

---

## 0. 这份方法论解决什么问题

云端强模型（GPT/Claude/Gemini 级）能靠「大上下文 + 强推理」硬吃复杂技能：多阶段流水线、长文档引用、模糊指令、隐式纠错。本地弱模型不行——它们：

- 上下文窗口小，长 Prompt 会挤爆关键注意力；
- 容易跑偏、自创流程、乱输出格式；
- 没有跨会话记忆，同一坑反复踩；
- 一旦允许写文件，会自由发挥创建未定义的文件。

**本方法论给出一套可复用的编译协议**：把依赖强模型的复杂 Skill，编译成弱模型可单次闭环稳定执行、且上下文峰值恒定的 Local Skill。经验可无限累积，运行代价不增长。

---

## 1. 五条核心公理（Core Axioms）

所有机制均从这五条推导，不得互相矛盾：

| # | 公理 | 含义 |
|---|------|------|
| A1 | **运行时信息最小化，离线经验最大化** | 经验可无限累积（离线侧），但模型运行时的上下文峰值**绝对恒定**（建议 < 3.8K tokens）。 |
| A2 | **绝对读写隔离（Read-Execute Separation）** | 弱模型严禁读脚本源码、经验库、历史全量日志；只由宿主机 Runtime 消化脚本并喂入 JSON 机器事实。弱模型对技能目录**零写权限**。 |
| A3 | **独立证据驱动演进** | 单次或同 Session 内的 Retry 拒绝认定为经验；唯有跨独立 Session 的证据链才能触发经验固化。 |
| A4 | **编译与运行解耦** | 存在一份 Compiler 独占的权威真相源（CANONICAL_SPEC）；弱模型运行时只读自动编译出的极简核心（L0）。 |
| A5 | **最小充分编译（Least-Sufficient Compilation）** | 严禁为了「工程化」强加结构。新增机制必须证明能降低明确的失败风险（收益/成本比 > 1.0）。能直接跑，就不动。 |

> **编译器第一价值判断**：只有当某个机制能明确降低本地弱模型的失败或误跑风险时，才允许加入。

---

## 2. 信息分级：PCLP 五层模型

弱模型的上下文通过宿主机 Wrapper 实现**物理级硬隔离**，禁止主动遍历文件系统。

| 层级 | 名称 | 谁可见 | 加载时机 | Token 硬上限 | 载体 |
|------|------|--------|----------|--------------|------|
| **L0** | Core Rule | 弱模型 | 任何任务启动时固定注入 | ≤ 800 | `L0.md`（从 Spec 编译） |
| **L1** | Stage Contract | 弱模型 | 进入指定 Stage 时加载，退出即 Reset | ≤ 3,000 | `stages/*.md` + `schemas/` |
| **L2** | Error Slice | 弱模型 | 校验失败且 Hash 匹配时按需注入**单条** | ≤ 1,500 | `experience/known_traps.json` |
| **L3** | Compiler Data | 仅强模型/Compiler | 离线重编译时加载 | 0（弱模型不可见） | `regression_tests/` + `trace_delta.json` |
| **L4** | Raw Logs | 仅人类审计 | 归档 | 0（弱模型不可见） | `logs/` |

**预算公式**：
```
运行时注入总峰值 = L0 + L1(当前) + L2(匹配1条)
硬预算上限     = 800 + 3000 + 1500 = 5.3K tokens
设计目标       < 3.8K tokens（L0 压到最小 + Stage 契约精简 + L2 只取 1 条）
```

### 2.1 运行权限配置（context_policy.yaml）

编译产物根目录生成唯一配置，Runtime 据此执行：

```yaml
context_policy:
  enforce_strict_budgets: true
  budgets:
    L0_max_tokens: 800
    L1_stage_max_tokens: 3000
    L2_slice_max_tokens: 1500
    L3_compiler_tokens: 0
    L4_logs_tokens: 0

  permissions:
    llm_context_access:
      allow: ["L0.md", "stages/${current_stage}.md", "schemas/*", "L2_matched_slice_only"]
      deny:  ["CANONICAL_SPEC.yaml", "scripts/*", "experience/*", "regression_tests/*", "logs/*"]
    host_runtime_execution:
      deliver_to_llm: "JSON_MACHINE_FACTS_ONLY"   # 脚本输出只以 JSON 机器事实喂回

  runtime_flow:
    default_init: { load: ["L0.md"], precheck: "scripts/precheck.py" }
    on_validation_failure: { action: "O1_FETCH_MATCHED_L2", max_entries: 1, collision_verification: true }
    on_stage_pass: { action: "CONTEXT_RESET" }     # 物理抹除 L1 与已加载 L2
```

---

## 3. 双向经验收敛门禁（Experience Gates）

经验引擎是本方法论的核心：所有运行期经验必须通过**证据门禁**才能固化，防止单次运气/单次失败污染知识库。

```
              【 宿主机 Runtime 执行 】
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
   [ FAIL ]                    [ PASS ]
        │                         │
 Failure Delta                Success Delta
        │                         │
 Failure Trap Gate            Success Shortcut Gate
        │                         │
 known_traps.json (L2)       success_shortcuts.json
                                      │
                              离线硬化 → scripts/sanitize_auto.py
```

### 3.1 失败避坑门禁（Failure Trap Gate）

| 条件 | 阈值 |
|------|------|
| 独立 Session 数 | ≥ 3（同 Session Retry 不计） |
| 不同输入数 | ≥ 2 |
| 恢复成功率 | ≥ 80%（该切片注入后恢复通过的比例） |

**失效机制（Staleness Rule）**：每条 L2 绑定 `canonical_spec_hash`。主版本升级导致 Hash 不匹配 → 自动标记 `DEPRECATED`，由离线 Compiler 清理。

### 3.2 成功快道门禁（Success Shortcut Gate）

| 条件 | 阈值 |
|------|------|
| 独立 Session 数 | ≥ 5 |
| 不同输入数 | ≥ 5 |
| 成功率 | ≥ 95% |
| 回归测试 | 必须通过离线回归集 |

**硬化下沉（Hardening）**：满足条件的成功路径**直接重构为确定性代码**（如 Python 脚本），彻底免除弱模型推理开销——从「经验照抄」升级为「零推理」。

### 3.3 O(1) 索引与原子更新

- 索引结构：`SHA256_PREFIX_32BIT → Byte Offset + Full Signature + Spec Hash`
- **原子发布**：修改经验库必须同步重建索引，并用「写临时文件 → 重命名替换」防并发脏读。

### 3.4 经验的三层承接

| 经验类型 | 承接位置 | 加载时机 |
|----------|----------|----------|
| P0 关键参数 | `L0.md` 第一条铁律 | 恒定注入 |
| 走通链路（成功序列） | `L0.md` 走通链路段 | 恒定注入 |
| 已踩坑清单 | `experience/known_traps.json` | 校验失败且 Hash 匹配才注入单条 |
| 提速经验 | 门禁通过后硬化为确定性脚本 | 离线重编译后零运行时开销 |

**经验总账（GOTCHAS.md）**：人可读、可审计的总账，**不注入弱模型上下文**。

**零写权限红线**：弱模型对技能目录无任何写权限。新坑/新提速 → 弱模型以 `[EXPERIENCE]` **纯文本回报宿主**，由宿主追加到总账 PENDING 区。**弱模型永不落盘。**

**知识分类**：
- 坑（失败恢复）→ known_traps（L2）
- 操作流程（首次怎么做）→ stages 模板（L1，任务命中时注入）
- 弱模型禁止自创流程 / 自建文件

---

## 4. 弱模型运行护栏（Runtime Guard）

编译产物必须包含一份护栏文本，核心条款：

```markdown
# Local Skill Runtime Guard Policy
ROLE: YOU ARE AN EXECUTOR (LEVEL 0 PERMISSION).

1. READ BOUNDARY: 只读 L0.md 与宿主注入的当前 Stage 契约。
2. FILE DENY: 禁止搜索/读取/解析/执行 scripts、experience、regression_tests、
   CANONICAL_SPEC、GOTCHAS、logs。
3. WRITE DENY (ZERO WRITE): 禁止创建/修改/删除技能目录内任何文件。
   新经验以 [EXPERIENCE] 纯文本回报宿主，绝不落盘。
4. EXECUTION: 确定性逻辑由宿主静默执行，只接收 JSON 机器事实。
5. ERROR HANDLING: 校验失败立即 STOP，等待宿主注入精确的 L2 恢复指令。
   禁止猜测、臆测、无限重试。
6. EVIDENCE: 完成后回报 [RUN_SUMMARY]（run_id / app / hit_trap_ids / recovery_result）。
```

### 4.1 三级权限模型

| 级别 | 角色 | 允许 | 禁止 |
|------|------|------|------|
| **L0 Executor**（运行默认） | 弱模型 | 读 L0 + 当前 Stage + L2 切片；处理 JSON；文本回报经验 | 写任何文件；读脚本/经验/Spec；无限重试 |
| **L1 Workspace Worker**（宿主侧） | 宿主 Runtime | 写运行时 Artifact；落经验草稿（PENDING） | 直接修改 known_traps（固化仅限门禁） |
| **L2 Upgrade**（强模型重编译） | 强模型/Compiler | 生成变更提案；重建 L0 | 打补丁改运行时技能（一律 Clean Build） |

> **弱模型自身永不持有 L1 写权限。** L1 的动作由宿主 Runtime 执行，弱模型只做文本回报。
> （实跑教训：弱模型一旦拥有「可写技能文件」的暗示，哪怕语义层，就会自由发挥创建未定义文件并落空处。）

---

## 5. 标准目录结构

```
compiled-skill/
├── SKILL.md                  # 主入口 Prompt（极简意图识别）
├── LOCAL_SKILL_RUNTIME.md    # 弱模型运行护栏
├── CANONICAL_SPEC.yaml       # [Compiler 独占] 全量规范唯一真相源
├── L0.md                     # [LLM 可读] 从 Spec 编译的 ≤800t 运行核心
├── context_policy.yaml       # PCLP 加载与 Token 预算配置
├── COMPILE_REPORT.md         # 编译记分卡
├── GOTCHAS.md                # [经验总账] 人可读，不注入弱模型
│
├── schemas/                  # L1: 输入输出契约
├── stages/                   # L1: Stage 契约（进入任务时注入，≤3000t）
├── scripts/                  # 宿主静默执行（LLM 只收 JSON 结果）
│   ├── precheck.py           #   连通性/前置断言
│   ├── validate_stage.py     #   机器校验
│   └── sanitize_auto.py      #   高门槛成功路径硬化产物
│
├── experience/               # 经验数据库（按需单条切片注入）
│   ├── index.hash
│   ├── known_traps.json
│   └── success_shortcuts.json
│
├── regression_tests/         # [Compiler 独占] 离线回归测试
├── extensions/               # 可选运行时插件（如自愈扩展）
└── logs/                     # [L4] 原始日志（弱模型不可见）
```

**按编译等级裁剪**：C2 级只出核心四件套 + schemas/stages；C3/C4 级才追加脚本、经验库、回归集。**不做无差别全套。**

---

## 6. 编译分级 C0–C4（最小充分防线）

| 等级 | 名称 | 触发特征 | 结构增量 |
|------|------|----------|----------|
| **C0** | No Compile | 原技能无模糊修饰词，弱模型可无干预稳定跑通 | 零修改 |
| **C1** | Prompt Compile | 存在「合适/适当」等模糊词及死规则，无复杂格式 | 精简入口 Prompt，删除无消费者文件 |
| **C2** | Contract Compile | 输入格式易错，或模型易输出乱格式 | + schemas/ + stages/ + L0.md + context_policy.yaml |
| **C3** | Runtime Compile | Context Peak 超预算、数据量大或步骤 ≥ 3 | + Stage 拆分 + Context Reset + Artifact 隔离 + experience/ |
| **C4** | Full Compile | 环境依赖强、需工具自动降级与断言 | + scripts/ + regression_tests/ + CANONICAL_SPEC.yaml |

**决策规则**：从 C0 向上探测，**命中最小充分条件即停，禁止跳级**。C1 能解决绝不升 C2。

---

## 7. 嵌套识别与内联整合（克迁移）

> **与 PCLP 互补**：嵌套内联管**输入侧**依赖自包含（可迁移），PCLP 管**输出侧**运行时隔离（恒定 Context）。任何转译动作开始前，先扫描嵌套依赖。

### 7.1 嵌套依赖类型

| 类型 | 形态 |
|------|------|
| 跨文件引用 | 相对路径引用 `见 references/members.md` |
| 文档内指引 | 正文/代码块中的文件路径提示 `详见 example.md` |
| 结构目录依赖 | 同目录其他文件被入口 Prompt 消费 |
| 跨技能调用 | 引用其他 Skill / Agent |
| 环境依赖 | 绝对路径、远端 URL、环境变量 |

### 7.2 内联规则

1. 被引用内容 → **全量**并入对应位置，**禁止摘要化**（摘要 = 丢信息 = 平移后缺失）。
2. 合并处标注来源 `（已内联：references/xxx.md）`。
3. 跨技能调用 → 降级为「用已内联内容直接推断」，删除跨 skill 引用。
4. 环境依赖 → 显式声明前置条件或提供降级路径。
5. 内联完成后删除原引用语句；无消费者文件直接删（死依赖）。

### 7.3 可迁移验证（产物强制自检）

- [ ] **平移测试**：整目录复制到空目录，仅凭入口 Prompt + 同目录契约文件即可运行。
- [ ] **引用清零**：0 条跨文件引用、0 条跨技能调用、0 条绝对路径/远端依赖。
- [ ] **内容核对**：原技能所有嵌套内容逐一确认内联（非摘要）。
- [ ] **消费者核对**：无死依赖残留。

---

## 8. P0 参数前置确认门

所有 Stage 之前的第 0 门。**参数不正确 → 后边所有步骤的猜测都不成立。**

```yaml
parameter_gate:
  stage: "P0_PARAMETER_GATE"           # 高于一切 Stage，编入 L0.md 第一条
  action: "ECHO_PARAMETERS_AND_CONFIRM" # 弱模型向用户弹出参数清单要求确认（Y/N）
  parameters: [server_ip, server_port, token, target_id, working_dir]  # 编译时从 Spec 冻结
  on_mismatch: "STOP_NO_DOWNSTREAM"     # 用户给新值 → 按新值走 + 回流更新 Spec
  on_confirm: "HOST_EXEC_PRECHECK"      # 确认后宿主执行连通性探测
  on_connect_fail: "STOP_AND_REPORT"    # 不通 → 立即 STOP，禁止「换参数再试」猜测循环
```

**规则**：
- 编译时把关键参数显式写入 `L0.md` 的「参数前置确认」区。
- 修改默认值属 L2 强模型动作；弱模型无权私自改，只能报告新值。
- 预检失败 → 报告 + 结束，不进入任何 Stage。

---

## 9. 阶段执行骨架

### 9.1 门禁三步法

```
INPUT（校验上一级 Artifact，FAIL 即 Fail-Fast）
  → EXECUTE（宿主执行确定性逻辑；弱模型仅在 decision_contract 内响应）
  → VALIDATE & COMMIT（机器校验，Pass 才写入固化 Artifact）
```

### 9.2 Artifact 依赖 DAG

- Stage 边界 = Artifact 边界。**无独立产物的步骤不得单独成 Stage。**
- 多输入 Stage 仅加载所需 Artifacts，Stage 间执行 Context Reset（物理抹除 L1 与已加载 L2）。

---

## 10. 编译流水线（Pipeline）

```
1.  定位目标技能（读入口 Prompt 全文；读取历史经验素材）
2.  嵌套扫描与内联（§7：扫依赖 → 全量内联 → 克迁移四项自检）
3.  Patch 冲突消解（展开旧规则与 Patch，收敛冲突分支，行为等价性断言）
4.  评估（对照触发标准 + 记分卡基线估算）
5.  定级（C0–C4，命中最小充分等级即停）
6.  ≥C2：生成 CANONICAL_SPEC.yaml（权威真相源）+ 提取关键参数（供 L0 P0 区）
7.  ≥C2：从 Spec 编译 L0.md（≤800t：P0 参数确认 + 走通链路 + 最高频坑 Top-N）
8.  ≥C3：分析 Artifact DAG → 拆分 stages/ + schemas/，配置 Stage 隔离流式契约
9.  编译脚本骨架（precheck + validate + sanitize）与 context_policy.yaml
10. 生成运行护栏 + 经验总账（PENDING 空）+ experience/ 空骨架
11. 产出 COMPILE_REPORT.md（记分卡：Context Peak 验证 + PCLP 隔离检查 + 经验引擎状态 + 终审）
12. 落位（`<原名>-local/`，与云端原技能**并存不覆盖**，保留对比与回退能力）
13. 自检（对照 §11 Permissions：是否过度编译 / 残留未内联引用 / 弱模型越权 / 缺 L0 / 缺 context_policy）
```

### 10.1 什么技能值得转译

命中以下任一即进入评估，否则默认 C0 不动：

0. **嵌套依赖（最高优先）**：含跨文件引用、跨技能调用、绝对路径/远端依赖 → 必须内联；
1. **云端/内置插件技能**：结构庞大（多 Stage、多脚本、多 references）；
2. **多工具联动**：依赖宿主机命令、外部 API、多轮工具调用；
3. **长链路任务**：步骤 ≥ 3 或需跨轮记忆（触发 C3 + PCLP 预算校验）；
4. **模糊指令密集 / Patch 堆叠**：含「适当/自行判断」等无界修饰词，或版本历史存在冲突 Patch；
5. **经验复用诉求**：技能将反复运行、环境易变（触发经验引擎 + P0 参数门）；
6. **目标环境是本地弱模型**：7B–35B 级、上下文窗口小、易跑偏/易乱答。

---

## 11. 运行期双通道运营

### 通道 A · 运行期即时（弱模型 + 宿主机，秒级）

```
1. P0：L0.md 弹出参数确认 → 用户确认 → 宿主 precheck 连通性
2. Stage 执行：宿主静默跑脚本 → JSON Machine Facts → 弱模型只做有界语义决策
3. 失败 → Validator FAIL → 宿主按 Hash 取 L2 单条切片注入 → 弱模型按切片恢复
4. 新坑/新成功 → 弱模型 [EXPERIENCE] 纯文本回报 → 宿主追加 GOTCHAS PENDING 区
   （弱模型零写权限，不落盘）
```

### 通道 B · 离线经验固化（需强模型，分钟级）

```
5.  跨 ≥3 Session 且恢复率 ≥80% 的 PENDING 坑 → 固化进 known_traps.json + 重建 index.hash
6.  跨 ≥5 Session 且成功率 ≥95% 且回归通过的成功路径 → 硬化为确定性脚本
7.  捕获 trace_delta.json（失败现场）→ 更新 CANONICAL_SPEC → Recompile Clean Build
8.  同步重编译 L0.md 与 COMPILE_REPORT.md
```

---

## 12. 可选扩展：运行时自愈插件（Runtime Self-Correction Extension）

独立插件，**不推翻主协议任何结构**，作为宿主 Wrapper 的异常处理钩子接入。

### 12.1 触发条件（唯一入口）

```
ON_UNKNOWN_ERROR_ESCALATION:
  条件: L2 查找为空（known_traps 无匹配 Hash）且 Validator FAIL
  动作: 挂起当前 Stage → 打包 Evidence Package → 走自愈闭环
  注意: L2 有匹配 → 仍走主协议通道 A，本扩展不介入
```

### 12.2 自愈闭环

```
【弱模型失败】→ [L2 为空] → [挂起 + Evidence Package]
                                │
                                ▼
              【强模型诊断修复】（结合 Context/Trace 分析）
                     │
                     ├─ 生成临时 L2 Patch（≤1,500 tokens）
                     ▼
              【弱模型重验】宿主注入 Patch + 原位 Re-run
                     ├─ PASS → 记 independent_sessions: 1 → 写入 L2 候选
                     └─ FAIL → 标记 Patch 无效 → 止损（MAX_REPAIR_ROUNDS=3）→ 报人工
```

### 12.3 Evidence Package

```json
{
  "error_signature": "stage_B:UNKNOWN_FORMAT_ERROR",
  "failed_stage": "stage_B",
  "input_data_hash": "a1b2c3...",
  "weak_model_raw_output": "...",
  "validator_failure_message": "Expected JSON object, got markdown table.",
  "elapsed_time": "12m30s",
  "retry_count": 3,
  "evidence_level": "LEVEL-1/2/3/4"
}
```

### 12.4 与主协议的兼容

| 主协议机制 | 本扩展的承接 | 兼容性 |
|------------|--------------|--------|
| L2 单条切片注入 | 命中 L2 走主协议；L2 为空升级为本扩展 | 互补 |
| Failure Trap Gate | 重验 PASS 仅记 `independent_sessions: 1`；跨 Session 再验证达标才固化 | 完全一致 |
| 通道 B 离线固化 | 本扩展是通道 B 的在线化前移（当场修复+当场重验） | 增强 |
| L2 Clean Build | 只产生临时 Patch，结构性改进仍走 Clean Build | 分层无越权 |
| trace_delta | Failure Snapshot 即标准化 trace_delta | 格式升级 |

**编译注入要求**：C2+ 技能默认具备——可观察 / 可诊断 / 可修复 / 可验证 / 可积累 五项能力（Runtime Logger / Timeout Monitor / Loop Detector / Failure Snapshot / Repair & Verification Interface / Experience Writer & Loader）。

---

## 13. 编译记分卡模板（COMPILE_REPORT.md）

```markdown
# Local Skill Compilation & PCLP Report

## 1. Context Peak & Budget Verification
- L0.md Token Count: ___ (Budget: ≤800) ──▶ PASS/FAIL
- L1 Current Stage Peak: ___ (Budget: ≤3000) ──▶ PASS/FAIL
- L2 Matched Slice Peak: ___ (Budget: ≤1500) ──▶ PASS/FAIL
- Total Injected Context Peak: ___ ──▶ PASS (Context Bloat Rate: 0%)

## 2. PCLP Security & Isolation Check
- LLM Script Source Access Denied: TRUE
- LLM Experience Store Read Denied: TRUE
- LLM Zero Write Permission: TRUE
- Machine Fact Formatting: JSON STRICT

## 3. Experience Engine Status
- Active L2 Failure Traps: N items (Spec Hash Verified)
- Index Hash Verification: OK
- Hardened Shortcuts: N items
- Deprecated Traps (Spec Mismatch): 0

## 4. 嵌套内联与克迁移明细
- 扫描发现嵌套引用: N 处；已全量内联: N；已删除原引用: N；死依赖清除: N
- 克迁移验证: 平移[PASS] 引用清零[PASS] 内容核对[PASS] 消费者核对[PASS]

## 5. Final Verdict
Protocol compliant. Ready for deployment on local SLMs (7B–35B).
```

---

## 14. 权限与禁区（Permissions & Restrictions）

### You MAY

- 输出 `C0: No Compile` 跳过编译；
- 在数据量大时强行插入 Stage 间的 Context Reset；
- 下沉确定性逻辑至脚本（含把高门槛成功路径硬化）；
- 对嵌套技能执行全量内联整合，并在报告中标明验证结果；
- 生成 PCLP 全套结构，并为每个编译产物建立经验引擎骨架。

### You MUST NOT

- 允许弱模型读取脚本源码、经验库、CANONICAL_SPEC 或历史日志；
- 允许弱模型自修改 Skill 或流程；
- 允许单次 Session 的 Retry 经验直接固化进 known_traps；
- 在 C1/C2 可解决时强行引入全套经验引擎（违反最小充分）；
- 保留原技能的跨文件引用/跨技能调用而不内联；
- 对被引用内容做摘要式内联——必须全量原样并入；
- 合并 Patch 时删去原有的 Bug 修复逻辑（行为等价性断言必须 Pass）；
- 让编译产物缺少 L0 就进入运行期（无 L0 = 弱模型没有恒定核心，必脑补）；
- 允许弱模型持有技能目录任何写权限。

---

## 15. 边界（Scope）

### 本方法论不做

- 为本地模型重新设计业务流程；
- 注入原 Skill 没有的新功能；
- 无脑压缩 Token。

### 本方法论只做

审核 → 嵌套识别与内联（可迁移）→ Patch 冲突消解（行为保留）→ 定级 → 生成 Spec + L0 → PCLP 结构 + context_policy → 权限护栏 + 经验引擎（门禁固化）+ P0 参数门 → 双通道运营 → 出报告。

---

## 附录 A：术语表

| 术语 | 定义 |
|------|------|
| **CANONICAL_SPEC** | Compiler 独占的全量规范唯一真相源 |
| **L0** | 从 Spec 编译出的 ≤800 token 运行核心，恒定注入 |
| **Stage** | 以独立 Artifact 为边界的任务阶段 |
| **Artifact** | Stage 的输入/输出产物，Stage 边界的唯一依据 |
| **PCLP** | 本方法论的五层信息分级模型（L0–L4） |
| **Context Reset** | Stage 通过后物理抹除 L1 与已加载 L2 片段 |
| **JSON Machine Facts** | 宿主执行脚本后喂回弱模型的结构化事实，替代原始输出 |
| **Failure Trap Gate** | 失败经验固化所需的跨 Session 双重证据门禁 |
| **Success Shortcut Gate** | 成功路径硬化为确定性代码所需的高门槛门禁 |
| **克迁移** | 嵌套依赖全量内联后的自包含平移能力 |
| **P0 Parameter Gate** | 一切 Stage 之前的参数前置确认门 |
| **Evidence Package** | 失败现场的标准化打包，供强模型诊断 |
| **Clean Build** | 从 Spec 全量重编译，禁止打补丁改运行时产物 |

---

## 附录 B：最小可落地清单（Quick Start）

如果你只想快速跑通一个 Local Skill，按此顺序：

- [ ] 写 `CANONICAL_SPEC.yaml`（哪怕只有 20 行）
- [ ] 编译 `L0.md`（≤800t：P0 参数 + 走通链路 + Top 高频坑）
- [ ] 写 `context_policy.yaml`（预算 + 权限 + 流程）
- [ ] 写运行护栏（零写权限 + 只读边界）
- [ ] 建 `stages/` 空骨架 + `experience/` 空骨架 + `GOTCHAS.md` PENDING 空区
- [ ] 自检：弱模型是否缺 L0 / 是否被授予越权 / 是否还有未内联引用
- [ ] 跑一个真实输入，产出第一份 `COMPILE_REPORT.md`

C0–C2 场景到此为止。只有当 Context 超预算或步骤 ≥3 时，才继续拆 Stage、加脚本、开经验引擎。

---

*Methodology version 1.0 · Fork freely under MIT · PRs welcome.*
