# 确定性重构工作流（v1.1，已修复第二轮六个阻塞项）

这份文档设计一套用于「未经严格审查的 Python 移植仓库」的重构机制。典型对象是按 S4 C/C++ 源码移植的 Python 版，再加上后来按零散需求打上的局部补丁。目标是在数值行为保持不变的前提下，清掉 Agent 式过度封装、缺输入却静默继续、双向依赖、重复计算，以及照搬 C 的低效双循环等问题。

机制放在**被重构的那个 Python 仓库**里，检查器本身放在仓库之外（见 1.1）。本文档只规定要落地的 skill、command、脚本、日志和单测，不在本 C 仓库里实现它们。S4 自身的模块边界见 `Summary.md`，下文用它作为 Python 移植的目标结构样例。

确定性不来自更长的提示词。Agent 不挑选下一处要改的文件，不手写「已完成」，也不靠自觉遵守风格。下一处改什么由队列脚本给出；能不能结束由检查脚本的退出码决定；结论由当前代码树上可重算的证明决定，而不是由 Agent 可编辑的状态文件决定。

## 修订说明（v0 → v1）

v1 依据 `RefactorWorkflowReview.md` 修改。正文里每一处改动都用下面的标注块写明「原方案 / 现方案 / 原因」，没有标注的段落与 v0 一致：

> **[Rxx 变更｜评审项｜处置]** 原方案：…　现方案：…　原因：…

处置取值：`采纳`（按评审意见改）、`修正后采纳`（方向接受，做法与评审建议不同）、`暂缓`（认可问题，v1 不实现，写明触发条件）。

### 变更总表

| 编号 | 位置 | 原方案 | 现方案 | 原因 | 评审项 | 处置 |
| --- | --- | --- | --- | --- | --- | --- |
| R01 | 1.1、3、9 | 检查脚本、基线、allowlist 与业务代码同在仓库，由 Agent 一起改 | 检查器做成固定版本的独立工具，放在 Agent 工作区之外；权威文件只能经修订提交修改 | 同一个 Agent 能放宽检查器再让 CI 变绿，「硬编码」就退化成可编辑的提示词 | S0-1 | 采纳 |
| R02 | 3、5A、9 | 调用图只由 `ast` 生成，一律当事实 | 调用边分 `static` / `declared` / `observed` / `dynamic` 四级；死代码要求多条证据同时成立 | `ast` 解析不了别名、多态、回调、注册表，漏边会被当成死代码或漏任务 | S0-2 | 采纳 |
| R03 | 3 | 状态只绑定符号自身的 `ast_hash` | 状态绑定 `proof_hash`，含直接依赖的契约、总纲版本、工具哈希、夹具与环境 | 函数体没变，依赖契约变了，旧验收仍会被当成有效 | S0-3 | 采纳 |
| R04 | 3、10.1 | `mark.py` 是唯一写状态入口，事件日志用哈希链保证不可篡改 | 事实来源改为 CI 在当前代码树上重算；哈希链降为防误改的辅助；证明对象用不含自身的 `source_digest` | Agent 能重写日志和哈希；报告与提交互相引用还会循环定义 | S0-4 | 修正后采纳 |
| R05 | 4.1、4.3、4.4 | 每步只改一个调用点；一律 W1 | 任务分 `implement`（签名冻结）与 `migrate`（一个符号加全部直接调用者的原子迁移）等类型 | 多调用者时只改一个调用点会弄坏其余调用者；逐个迁移又要适配层，与「禁止新增包装」冲突 | S0-5 | 采纳 |
| R06 | 4.2、5A | 跨模块环阶段 A 失败；模块内递归手工声明后拓扑排序 | 先压缩强连通分量再排序；现存非法依赖进入只减不增的 `architecture_debt.toml` | 相互递归无法拓扑排序；总纲禁止的边又必须靠阶段 B 才能消除，会死锁 | S0-6 | 采纳 |
| R07 | 5.0、5D、10.2 | 数值门槛是逐样例 `deviation_growth ≤ 0` | 容差、物理不变量、本征分解的等价比较、固定环境；已知偏差设上限而非逐样例只降不升 | 浮点在换 BLAS、线程或循环顺序时会变；偏差为零也不等于正确 | S0-7 | 修正后采纳 |
| R08 | 5A、5B | 阶段 0 只有成功算例 | 增加负向用例，但由 schema 和契约**规定**期望异常，不从现有代码行为冻结 | 现有代码正是靠静默默认值继续运行，冻结它的行为等于把 P2 固化 | S0-7 | 修正后采纳 |
| R09 | 5.0、5D、9、10.2 | 没有性能验收，只有循环 AST 扫描 | 增加固定 runner 上的基准：中位数、MAD、峰值内存 | 向量化可能更慢或更占内存；`np.vectorize`、`apply` 等 AST 扫不出 | S0-8 | 采纳 |
| R10 | 2 的 P1/P20、4.5、9、10.2 | class 不增、def 不净增、注释行不净增均为硬门禁 | 纯透传仍硬失败；新增 class/def 必须写枚举理由；不再限制注释数量 | 计数指标会诱导塞进大函数、误杀合理抽象，还会禁掉必要的数学说明 | S1-1 | 采纳 |
| R11 | 2 的 P2、5B | 靠变量名判断 `.get` 默认值是否非法 | 核心模块的公开函数不接收裸 `dict` / `Mapping[str, Any]` / `**kwargs`，配置在 API 边界解析成带类型对象 | 按变量名扫描误报漏报都多；把 dict 挡在边界外更确定 | S1-2 | 采纳 |
| R12 | 2 的 P7、5B | 契约 `raises` 必须非空；链上函数不得返回 `None` | `raises` 只对有前置条件的边界必填；`None` 区分 `optional_value` 与 `failure_sentinel` | 纯数值内核可能无业务异常；`None` 有时是合法领域值 | S1-2 | 采纳 |
| R13 | 5B、9 | 契约是文本字段，检查「函数前部引用 shape」 | 契约用受限表达式，由测试期运行时代理实际执行；`mutates` 用数组哈希验证 | 引用一次 `.shape` 就能通过，并没有验证形状 | S1-3 | 采纳 |
| R14 | 2 的 P4/P12、5B、5C | 重复 AST 聚类强制合并；规范量只查变量名赋值 | AST 聚类只出候选，结论记 `merge` 或 `independent`；规范量走带类型载体，构造点唯一；缓存用生产者调用次数验证 | 相同 AST 可以是合理的独立公式；换个变量名就能绕过赋值检查 | S1-4 | 采纳 |
| R15 | 5A | 阶段 A 只做总纲，直接进入自底向上 | 阶段 A 拆成顶向下发现（每条链先有端到端用例）和依赖序实施 | 只从叶子开始，可能把旧 C 数据布局当成正确契约冻结 | S1-5 | 采纳 |
| R16 | 3、5.0、5D | `public_api` 只检查符号存在 | 阶段 0 生成 API 快照，阶段 D 逐项比较 | 导入路径、参数、默认值、异常、返回 dtype 变了也不会被发现 | S1-6 | 采纳 |
| R17 | 5B、10.2 | 只看数值夹具是否通过 | 增加被改行的分支覆盖、声明调用边是否被观察到、契约异常路径有负向测试 | 夹具通过不代表改动的分支和声明的边真的被执行 | S1-7 | 采纳 |
| R18 | 4.5 | 未说明并发；只防同一工作区开两张卡 | 明确 v1 为单 Agent 串行；任务卡带 `base_commit` 与总纲版本，过期即拒发 | 不同分支的两张卡合并后调用图与契约可能已变 | S1-8 | 修正后采纳（并行租约暂缓） |
| R19 | 3 | 清单含「全部函数、变量」 | 清单只含模块、类、函数、模块级状态、配置字段、规范数据载体和缓存；局部变量由任务内 AST 临时分析 | 局部变量、闭包、视图数量大且不稳定，长期登记只制造噪声 | S2-1 | 修正后采纳 |
| R20 | 2 的 P17、3 | 扇入超阈值的模块即失败 | 每个模块必须写一句 `responsibility`；命名为 utils/common 类或缺职责说明的模块才硬失败，扇入与内聚度量只出报告 | 稳定的底层数学模块扇入高是正常的；无环单向也不能证明高内聚 | S2-2 | 采纳 |
| R21 | 3、5.0 | 未规定环境复现 | 环境锁进入基线与 `proof_hash` | 换 numpy/BLAS/线程数会改变数值和性能 | S2-3 | 采纳 |
| R22 | 3、10.1 | 每步的完整报告提交进仓库 | 完整报告放 CI 制品；仓库只提交小型证明清单 | 大量生成文件会淹没业务 diff | S2-4 | 采纳 |
| R23 | 5C | 只被测试调用的代码直接删除 | 先核对 API 快照、文档和发布历史 | 测试是唯一使用者不等于无价值 | S2-5 | 采纳 |
| R24 | 13 | 直接实现全部脚本 | 先做三个原型，通过后才写完整门禁 | 先验证信任边界、调用图漏边率、任务原子性这三个最关键假设 | 评审 9 | 采纳 |
| R25 | 3、9 | 权威文件（总纲、schema、基线）与业务改动可在同一提交中修改 | 修改权威文件的提交不得同时改业务代码，且必须带修订编号 | 评审要求的「独立审批」在本地模式下需要可脚本判定的形式；这是为 R01 补的落地机制，不是评审直接提出的 | S0-1 延伸 | 本次新增 |

### 评审项处置一览

| 评审项 | 处置 | 对应变更 | 与评审建议不同之处 |
| --- | --- | --- | --- |
| S0-1 控制面未隔离 | 采纳 | R01、R25 | 分「团队 CI」和「单人本地」两档，本地档用只读安装加提交历史检查，不强求发布独立包 |
| S0-2 调用图不完整 | 采纳 | R02 | 无 |
| S0-3 状态只绑 ast_hash | 采纳 | R03 | 无 |
| S0-4 提交与哈希循环 | 修正后采纳 | R04 | 不用 `git write-tree`：它包含证明清单自身，仍会循环。改为对「排除证明目录的全部受版本控制文件」求 `source_digest` |
| S0-5 W1 处理不了多调用者 | 采纳 | R05 | 增加调用者数量阈值与临时适配层登记，避免与「禁止包装」冲突 |
| S0-6 SCC 与死锁 | 采纳 | R06 | 无 |
| S0-7 数值门禁不足 | 修正后采纳 | R07、R08 | 「冻结负向行为」改为「按 schema 规定负向行为」；已知偏差用固定上限，不做逐样例 Pareto |
| S0-8 缺性能验收 | 采纳 | R09 | 硬失败只发生在固定 runner，普通机器只出报告 |
| S1-1 计数指标诱导 | 采纳 | R10 | 新增抽象要求登记理由，理由为 `shared_invariant` 时还要求仓库内至少两处调用 |
| S1-2 .get 与 raises | 采纳 | R11、R12 | 以名称到具体值类型的查找表（如材料表）合法，不一刀切禁止所有 `Mapping` |
| S1-3 契约只是文本 | 采纳 | R13 | 运行时检查由测试期代理注入，不在生产代码里加装饰器 |
| S1-4 重复与唯一生产者 | 采纳 | R14 | 无 |
| S1-5 自底向上冻结错误契约 | 采纳 | R15 | 无 |
| S1-6 API 兼容 | 采纳 | R16 | 无 |
| S1-7 覆盖 | 部分采纳 | R17 | 变异测试暂缓：见文末暂缓清单 |
| S1-8 并发 | 修正后采纳 | R18 | v1 只声明串行，不实现租约 |
| S2-1 清单范围 | 修正后采纳 | R19 | 原需求要「全部变量清单」，折中为跨函数可见的状态；局部变量任务内分析 |
| S2-2 内聚度量 | 采纳 | R20 | 无 |
| S2-3 环境锁 | 采纳 | R21 | 无 |
| S2-4 报告体积 | 采纳 | R22 | 无 |
| S2-5 删除测试 | 采纳 | R23 | 无 |
| §9 三个原型 | 采纳 | R24 | 无 |

## 修订说明（v1 → v1.1）

v1.1 只修 `RefactorWorkflowReview.md` 第二轮的六个阻塞项（V1-S0-1 至 V1-S0-6）。第二轮的高优先级与中优先级意见本版不改。正文里用 `R26` 至 `R31` 标注；没有这些标注的段落仍是 v1。

| 编号 | 位置 | 原方案（v1） | 现方案（v1.1） | 原因 | 评审项 | 处置 |
| --- | --- | --- | --- | --- | --- | --- |
| R26 | 3 | `proof_hash` 只含直接依赖的契约哈希，并用手工维护的 `constitution_version` | `proof_hash` 含排序后的直接依赖 `proof_hash`，以及全部权威文件自动算出的 `authority_digest`；强连通分量共用一个组证明 | 被调用方只改实现、不改契约时，调用方仍显示 `reviewed`；漏改版本号就绑不上总纲 | V1-S0-1 | 采纳评审的方案 A |
| R27 | 3、10.1 | 一个 `source_digest` 既记录当时的全仓树，又被写成当前证明对象 | 拆成 `run_tree_digest`（只追溯，不参与状态）和 `proof_hash`（当前是否有效） | 下一步一改别的文件，全仓摘要就变，旧证明会全部失效；若因此不比较它，它又不能证明当前状态 | V1-S0-2 | 采纳 |
| R28 | 1、9、10.2、11 | `Any`、`type: ignore`、架构债务和反模式按数量棘轮 | 按稳定 issue ID 的集合包含关系棘轮；数量只给人看 | 删掉一处、换一处加上，数量不变，门禁仍通过 | V1-S0-3 | 采纳 |
| R29 | 4.5、5B、9、11 | 单方法类一律失败；`new_optional` 仍要求缺键抛错；上下游按 Python 参数名对接 | 单方法类只有「纯委托且无自身不变量、又没登记策略理由」才失败；可选键在边界套用默认值；跨函数按 `canonical_id` 对接 | 这三处与 v1 已有的策略对象、可选配置、`canonical_id` 互相矛盾，实现时只能任选一条 | V1-S0-4 | 采纳 |
| R30 | 3、4.2、5A、9 | SCC 和队列直接用 static、declared、observed 的并集；dynamic 不进这张图 | 另建 `dependency_graph.json`，只有目标唯一的边才进入 SCC；已确认的 dynamic 边进入；说不清目标的边阻断相关队列 | 一条误解析会把大量函数压成一个分量；一条没进图的回调环又会让排序是错的 | V1-S0-5 | 采纳 |
| R31 | 3、5D、10.1 | 阶段 D 的报告只引用会过期的 CI 制品 | 每步报告仍可过期；终局另存长期的 `final-attestation.json`，审计引用这个地址 | 制品过期后，审计里的哈希无法再核对 | V1-S0-6 | 采纳 |

## 1. 设计原则

1. **事实与判断分开。** 符号清单、调用图、import 图、AST 反模式由脚本生成。Agent 只在脚本留出的枚举项里做判断，并把判断写成可校验的记录。
2. **判断尽量前移到一次总纲。** 某个量是必填还是可选、某条依赖允不允许、某个函数是不是模块的公开入口，都在阶段 A 写进 schema。阶段 B 只对照 schema 改代码，不再临时决定。
3. **棘轮。** 防御性默认、`type: ignore`、`Any`、透传包装、未登记双循环、架构债务只允许减少，不允许换一处加上。比较的是稳定 issue ID 的集合，不是个数。个数可以显示，不能当通过条件。

> **[R28 变更｜V1-S0-3｜采纳]** 原方案（v1）：上述各项的数量不得高于基线。现方案：`current_issue_ids ⊆ baseline_issue_ids`。一个 ID 是 `sha256(rule_id + module + qualname + normalized_ast_path)`。行号不进 ID，否则一次格式化就会变成「旧问题消失、新问题出现」。同一符号内部、归一化 AST 路径不变的移动仍是同一个 ID；换到另一个符号就是新 ID，门禁失败。原因：删掉 A 处的 `type: ignore`、在 B 处加一个，数量不变，v1 的棘轮发现不了。
4. **功能锁在外部参考上。** S4 移植的数值基准是 C 版 S4 在固定算例上的输出，不是脏 Python 自己的输出。没有 C 参考时才退化为冻结当前 Python 输出，并在总纲里写明。
5. **一次只走一步。** 一步是任务卡列出的符号（或预先分组的原子阶段）。窗口之外的文件出现在 diff 里，任务失败。
6. **失败即停。** 门禁非零就留在当前任务里修。需要扩大窗口时停止改代码，走修订流程，不在任务里偷偷多改。

> **[R01 变更｜S0-1、S1-4、S0-2｜采纳]** 原方案：原则只有以上六条。现方案：新增下面两条。原因：v0 默认检查器可信、所有图都是事实，评审指出这两点不成立。

7. **检查器不受被检查者控制。** 谁写业务代码，谁就不能改检查器、基线、容差和总纲。这些改动只走修订提交（第 11 节），并由历史检查脚本核对。
8. **证据分级。** import 图是硬事实。调用图的每条边标可信度，只有多条证据同时成立才能得出「死代码」「入口已覆盖」这类否定性结论。

### 1.1 信任模型与五个平面

> **[R01 变更｜S0-1｜采纳，R25 补充]** 原方案：无此节。现方案：把工件按谁能改分成五个平面，并给出团队和单人两档落地方式。原因：v0 里 `scripts/refactor/`、`baseline.json`、allowlist 与业务代码同仓同权限，哈希链也只是 Agent 自己写、自己校验。

| 平面 | 内容 | 谁能改 |
| --- | --- | --- |
| 控制面 | `refactor-gate` 检查器（含其自测）、CI 配置、`refactor/tool.lock` | 只经修订提交；检查器版本变更单独审批 |
| 行为基线面 | `golden/`、`tolerances.toml`、`invariants.toml`、`known_deviations.json`、`api_snapshot.json`、`bench/baseline.json`、`env.lock`、`negative_cases.toml` | 只经修订提交 |
| 架构事实面 | `constitution.md`、`modules.toml`、`options_schema.toml`、`canonical_vars.toml`、`chains.json`、`architecture_debt.toml`、`allowlists/*` | 只经修订提交（阶段 A 首次建立） |
| 任务执行面 | 业务源码、契约、任务对应测试、`decisions/*` | 任务卡限定的文件 |
| 验收面 | 门禁报告、证明清单、审计 | 由检查器生成；Agent 只能引用 |

两档落地：

- **团队 CI 档。** 检查器发布成带版本号的包，`tool.lock` 记录版本与制品 SHA256。CI 从锁文件安装，不使用 PR 分支里的同名脚本。控制面、基线面、架构事实面走 `CODEOWNERS` 独立审批。业务 PR 不得修改这三类文件。
- **单人本地档。** 检查器安装在 Agent 工作区之外的只读目录，命令用绝对路径调用；`tool.lock` 记录该目录的提交哈希。三类权威文件的修改由 `check_authority_commits.py` 按提交历史核对（见第 9 节）。本地档能防误改和顺手放宽，防不了有权限的人主动绕过；需要更强保证就用团队档。

## 2. 要消除的问题

前五项对应本次要求。后面是同一类仓库里会反复出现、并且能用同一套门禁挡住的问题。标 `▲` 的行相对 v0 有改动。

| 编号 | 问题 | 判定 | 何时钉死 |
| --- | --- | --- | --- |
| P1 ▲ | 过度封装：函数只转发给另一个函数，单方法类，为了一处调用新加一层 | 纯透传由脚本硬判；合法透传只能是 `modules.toml` 里该模块的 `public_api`。新增 class/def 必须在任务卡登记理由枚举（4.5） | 阶段 A 登记公开入口；阶段 B 每步核对 |
| P2 ▲ | `dict.get(k, default)`、`getattr(obj, k, default)`、`pop(k, default)`、`x or default` 让必填输入缺失时继续跑 | 核心模块公开函数禁止裸 `dict` / `Mapping[str, Any]` / `**kwargs` 形参，配置在 API 边界解析为带类型对象；AST 再扫默认值取法作为辅助 | 阶段 A 的 options schema 列出仅有的合法默认值 |
| P3 | 局部修改造成的跨模块调用和双向 import | import 图相对 `allowed_imports` 的差集；调用图上的跨模块边同样检查 | 阶段 A 的依赖总纲 |
| P4 ▲ | 同一物理量沿调用链算了多次 | 总纲指定每个规范量的唯一生产者和带类型载体；载体只能在生产者内构造；缓存用例统计生产者调用次数 | 阶段 A 登记；阶段 B 每步核对 |
| P5 ▲ | 直接双层 `for i in range` / `for j in range` 做逐点算术，把 C 循环原样搬进 Python | AST 候选发现（嵌套 `range`、`np.vectorize`、逐行 `apply`、嵌套推导式）；核心模块未登记即失败；是否真的更快由基准判定（P24） | 豁免名单，且理由只能取枚举 |
| P6 | 吞掉异常后返回默认值或 `None`：裸 `except`、`except Exception: return/pass` | 脚本 | 全程禁止，无默认豁免 |
| P7 ▲ | 用 `None` 表示失败，调用方再判空 | 契约区分 `optional_value` 与 `failure_sentinel`；后者禁止，失败必须抛异常 | 契约 schema |
| P8 | `**kwargs` 把签名藏起来，上下游对不齐 | 内部函数禁止 `**kwargs` / `*args`；公开 API 若保留，必须在契约里逐键展开 | 阶段 B 的签名检查 |
| P9 | 函数内部 import，用来掩盖环 | 脚本统计；棘轮下降，终局除总纲点名的延迟 import 外必须为 0 | 阶段 A 点名，之后只减不增 |
| P10 | 可变默认参数、星号 import、模块级可变全局量 | 脚本，终局为 0 | 全程 |
| P11 | 热循环里 `list.append` 再转数组；对标量反复调 BLAS | 并入 P5 的循环检查 | 阶段 B |
| P12 ▲ | 两段 AST 归一化后相同的实现并存 | 脚本聚类只产出候选；每簇结论记为 `merge` 或 `independent` | 阶段 C 清残留时必须处理完 |
| P13 | 0-based / 1-based 混用、复数用实部虚部交错数组和 `complex` dtype 混用 | 契约写 `index_base` 和 `dtype`；脚本核对链上边界的契约字段是否填了 | 不能证明哪一次 `+1` 是错的，所以做成必填字段，而不是猜 |
| P14 ▲ | 广播碰巧成功，形状错了也不报 | 契约里的形状表达式在测试期由运行时代理在入口和出口实际求值 | 每步契约 |
| P15 ▲ | 就地改写调用方还要复用的数组 | 契约 `mutates` 列出被改的参数；运行时代理比较调用前后的数组哈希，未声明的改动失败 | 每步契约 |
| P16 ▲ | 几何或频率变了，模式缓存仍在 | 编排层契约 `invalidates` 必须覆盖总纲里的缓存名；缓存用例断言改区域、频率、材料、G 之后结果与冷启动一致，且生产者被重新调用 | 阶段 B 编排步 |
| P17 ▲ | `utils` / `common` / `helpers` 成为谁都 import 的枢纽 | `modules.toml` 每个模块必须有一句 `responsibility`；名字属于 utils/common/helpers/misc/base 类、或缺职责说明的模块失败；扇入和模块内调用占比只出报告 | 阶段 A |
| P18 | 调试 `print`、`breakpoint`、无单号的 TODO | 脚本，终局为 0 | 全程 |
| P19 | `Any` 与 `# type: ignore` 扩散 | 计数棘轮 | 全程 |
| P20 ▲ | 注释和 docstring 代替结构，重构后与代码漂移 | 步骤 skill 禁止新增只复述代码的叙述性注释；数学来源、单位、数值稳定性说明允许。不设自然语言质量分，也不设注释行数门槛 | 步骤 skill 的硬禁止 |
| P21 ▲ | 死代码、只有测试引用的代码、字符串动态调用 | 静态不可达、运行未观察到、不在 API 快照与动态引用名单、字符串与注册表检索为空，四者同时成立才成为删除候选 | 阶段 C |
| P22 | 为通过测试而放宽数值容差 | 容差只在基线文件里；变松必须走修订，且修订脚本拒绝松于基线 | 门禁 |
| P23 新增 | 公开 API 的导入路径、签名、默认值、异常、返回 dtype 与形状被悄悄改变 | 与 `api_snapshot.json` 比较 | 阶段 0 生成，阶段 D 比较 |
| P24 新增 | 向量化后更慢、峰值内存暴涨，或 AST 扫不到的低效写法 | 固定 runner 上的基准：中位数、MAD、峰值内存 | 阶段 0 基线，每步与终局 |

P23、P24 是评审后新增的两类问题（R16、R09）。P13、P14、P16 脚本不能独自判断物理对错，它们被收成必填字段和运行时检查之后，Agent 的自由只剩「填哪一个枚举值」。

`▲` 行对应的变更：P1→R10；P2→R11；P4→R14；P5→R09；P7→R12；P12→R14；P14、P15→R13；P16→R14；P17→R20；P20→R10；P21→R02、R23。

> **[R20 变更｜S2-2｜采纳]** 原方案：P17 用扇入阈值判定，超过即失败。现方案：`modules.toml` 每个模块必须写一句 `responsibility`；只有名字属于 utils/common/helpers/misc/base 类、或缺职责说明的模块才硬失败；扇入、模块内调用占比、跨模块边数只出报告（`module_metrics`）。原因：稳定的底层数学模块扇入高是正常的，阈值会误伤；无环、单向的依赖图也不能证明模块内高内聚，边界最终由总纲决定，脚本只核对总纲是否被遵守。

## 3. 目标仓库里的工件

路径都相对于 Python 仓库根目录。`★` 表示权威文件，只能经修订提交修改；`☆` 表示由检查器生成，不手改；`◇` 表示派生视图，不入库。

> **[R01、R02、R03、R04、R16、R19、R21、R22、R25 变更]** 原方案：脚本、基线、状态、报告都在同一棵树里，状态由 `mark.py` 写。现方案：检查器移出仓库；新增 API 快照、容差、不变量、基准、环境锁、负向用例、架构债务等权威文件；报告只留小型证明清单。原因：见变更总表对应各行。

```
refactor/
  tool.lock                    ★ 检查器版本与制品哈希
  constitution.md              ★ 人读的总纲，章节标题固定
  modules.toml                 ★ 模块、responsibility、允许的 import 边、public_api、阈值
  options_schema.toml          ★ 仅有的合法默认值
  canonical_vars.toml          ★ 规范物理量 → 唯一生产者 + 带类型载体
  chains.json                  ★ 业务链、原子组；初稿由脚本生成
  architecture_debt.toml       ★ 现存非法依赖与临时适配层，只减不增
  allowlists/                  ★ loops / delayed_import / dynamic_ref
  baseline.json                ★ 反模式计数
  tolerances.toml              ★ 每个输出的 atol / rtol / 比较方式
  invariants.toml              ★ 物理不变量清单
  known_deviations.json        ★ 当前 Python 相对参考的已知偏差与上限
  api_snapshot.json            ★ 公开 API 快照
  negative_cases.toml          ★ 由 schema 规定的负向用例
  env.lock                     ★ Python、numpy、scipy、BLAS、线程数、种子
  bench/cases.toml             ★ 基准算例
  bench/baseline.json          ★ 固定 runner 上的基线
  golden/<case_id>.npz         ★ 外部参考输出
  adapters/                    ★ 与本仓库相关的胶水，如调用 C 版 S4 生成 golden
  inventory.json               ☆ 符号清单
  observed_edges.json          ☆ 运行时观察到的调用边，只用于发现
  dependency_graph.json        ☆ 目标已唯一解析的依赖边，供 SCC、队列和 stale 使用
  baseline/issues.json         ★ 阶段 0 冻结的 issue ID 集合
  state/final-attestation.json ☆ 阶段 D 的长期证明摘要，随发布物保存
  tasks/<task_id>.json         ☆ 任务卡
  contracts/<symbol_id>.json     契约
  decisions/<symbol_id>.json     非脚本判断的枚举记录
  state/proofs/<task_id>.json  ☆ 小型证明清单
  amend/<id>/                  ★ 修订记录
  status.json                  ◇ 进度视图，由 status.py 从证明重建
tests/numeric/                 对 golden 的回归
tests/negative/                对 negative_cases 的回归
tests/structure/               调用门禁的 pytest
```

检查器自身的自测（故意违规的样本、必须失败与必须通过的断言）属于检查器包，不放在业务仓库里。CI 制品保存完整报告，仓库只入库 `state/proofs/`。

### 清单

> **[R02、R19 变更｜S0-2、S2-1｜采纳、修正后采纳]** 原方案：清单含「全部函数、类、模块级变量」，调用边只有 `callers` / `callees`。现方案：清单范围缩小，边带可信度。原因：局部变量、闭包、数组视图数量大且不稳定；调用边来源不同，可靠程度不同。

`inventory.json` 只登记：模块、类、函数、模块级状态、配置字段、规范数据载体、缓存。普通局部变量不作为长期状态项，在任务内由 AST 临时分析。

```json
{
  "id": "fmm.closed.epsilon",
  "kind": "function",
  "qualname": "fmm.closed.epsilon",
  "file": "src/fmm/closed.py",
  "module": "fmm",
  "signature": "(g, pattern, materials) -> Epsilon",
  "ast_hash": "sha256:...",
  "edges_out": [
    {"to": "pattern.fourier", "confidence": ["static", "observed", "declared"]}
  ],
  "edges_in": [
    {"from": "orch.compute_layer_modes", "confidence": ["static", "observed"]}
  ],
  "in_api_snapshot": false
}
```

边的可信度：

| 级别 | 来源 | 用途 |
| --- | --- | --- |
| `static` | `ast` 解析的直接调用和 import | 硬事实的下限，不能证明「没有别的边」 |
| `declared` | 契约的 `callees` 字段 | 人为声明，必须被 static 或 observed 至少一种印证 |
| `observed` | 运行 golden、端到端和负向用例时用 `sys.setprofile` 或 `sys.monitoring` 记录 | 补 static 漏掉的多态、回调、别名 |
| `dynamic` | `getattr`、字符串入口、注册表、`importlib`，登记在 `dynamic_ref.toml` | 不能自动删除，必须人工决定 |

这四类边只用于发现，不直接排序。`resolve_deps` 另写 `dependency_graph.json`（R30）。契约的 `callees` 必须覆盖该符号在依赖图里的全部出边。

### 状态与证明

`status.json` 是派生视图，不是事实来源，不入库。状态取值：

| 状态 | 含义 |
| --- | --- |
| `untouched` | 还没有任务验收过 |
| `reviewed` | 存在证明清单，且当前重算的 `proof_hash` 与清单一致 |
| `deleted` | 已删除，inventory 里仍留墓碑直到阶段 D |
| `exempt` | 总纲点名不重构，必须有理由枚举 |
| `stale` | 已有证明，但重算的 `proof_hash` 不一致 |

> **[R03 变更｜S0-3｜采纳]** 原方案：`reviewed` 只绑定符号自身的 `ast_hash`。现方案：绑定 `proof_hash`。原因：函数体不变，直接依赖的签名、契约、规范量生产者、总纲、检查器版本、夹具、环境任一变化，验收都可能失效，只比较函数体会让它们错误地保持 `reviewed`。

```text
authority_digest = sha256(按路径排序的全部权威文件内容)
proof_hash(symbol) = sha256(
    symbol_ast
  + symbol_contract
  + authority_digest
  + fixture_digest
  + env.lock
  + gate_tool_sha256
  + sorted(direct_dependency_proof_hashes)
)
```

强连通分量里的符号共用一个 `group_proof_hash`。它的依赖只取离开该分量的边，因此组内互相引用不会把哈希算死循环。调用方的 `proof_hash` 引用的是这个组证明。

> **[R26 变更｜V1-S0-1｜采纳方案 A]** 原方案（v1）：`proof_hash` 只拼直接依赖的契约哈希，外加手工维护的 `constitution_version` 和该符号自己的规范量登记项。现方案：改为上面的递归式，并自动计算 `authority_digest`。`authority_digest` 覆盖 `constitution.md`、`modules.toml`、`options_schema.toml`、`canonical_vars.toml`、`chains.json`、`architecture_debt.toml`、`allowlists/*`、`tolerances.toml`、`invariants.toml`、`known_deviations.json`、`negative_cases.toml`、`api_snapshot.json`、`bench/baseline.json`、`baseline/issues.json`、`env.lock`。原因：被调用方只改函数体、不改契约文本时，v1 的调用方哈希不变，状态仍是 `reviewed`，队列不会重验它。手工版本号也会漏加。评审的方案 B（结构证明与行为证明分开）更精确，但 v1.1 先用方案 A：失效范围偏大，是保守且可复算的代价。

重算 `proof_hash` 不一致就把该符号标成 `stale` 并重新入队。调用方引用了依赖的 `proof_hash`，所以被调用方的实现、契约或权威文件一变，调用方会在同一次重算里连带失效，不另做一次人工传播。

> **[R04 变更｜S0-4｜修正后采纳]** 原方案：`mark.py` 是唯一写状态入口，事件日志用哈希链防篡改，报告引用 git 修订。现方案：事实来源改为 CI 在当前代码树上重跑全部结构门禁并重算 `proof_hash`；哈希链降为辅助。原因：Agent 能重写日志并重算哈希，本地状态文件没有信任价值。

> **[R27 变更｜V1-S0-2｜采纳]** 原方案（v1）：证明清单里有一个 `source_digest`，定义是「除 `refactor/state/` 之外全部受版本控制文件的摘要」，既要避开自引用，又被写成当前代码树的证明。现方案：废弃这一个字段，改成两个职责不同的摘要。`run_tree_digest` 仍是当时全仓（排除 `refactor/state/`）的摘要，只写进证明清单供追溯，**不**与当前树比较，也**不**决定 `reviewed` 还是 `stale`。当前是否有效只看 `proof_hash`；它只覆盖本符号、直接依赖证明、契约、夹具、权威文件、环境和工具，也就是评审所说的 `proof_scope_digest`。原因：任务 B 改了另一个模块后，全仓摘要必然从 T1 变成 T2。若要求旧证明的全仓摘要仍等于当前树，则每走一步全部旧证明失效，流程无法收敛；若不要求相等，它又不能证明当前状态。两个职责不能共用一个字段。

`state/proofs/<task_id>.json` 只保存：任务 id、符号列表、`proof_hash`、`run_tree_digest`、工具哈希、当时的每步报告引用。它记录「这个符号走过任务流程，以及当时跑在哪棵树上」。当前是否仍有效，由 CI 重算 `proof_hash` 决定。

## 4. 窗口与队列

### 4.1 上下游只含一跳

一步的窗口叫 W1，适用于 `implement` 类任务（4.3）。

- **当前函数**：可以改函数体、它的契约、它的数值夹具和对应单测。
- **下游，直接被调用方**：必须已经是 `reviewed`。本步只读，禁止改它们的文件。当前函数必须按它们已冻结的契约去调用。
- **上游，直接调用方**：本步还没轮到它们。`implement` 任务**不改签名**，因此不需要动任何调用点。
- **再远的一跳不打开。** 不读不改调用方的调用方，也不改被调用方的被调用方。
- **数据只追一跳。** 核对「调用方产出的、本函数读入的量」和「本函数产出的、被调用方读入的量」。不沿表达式再往外追。

> **[R05 变更｜S0-5｜采纳]** 原方案：上游直接调用方只允许改「调用当前函数的那一个表达式」，任务卡里只列一个上游。现方案：`implement` 任务冻结签名，不改调用点；改签名一律走 `migrate` 任务，一次列出全部直接调用者并原子更新。原因：一个函数常有多个调用者，只改一个会弄坏其余；逐个迁移又需要临时适配层，与「禁止新增包装」冲突。

组步是唯一的放宽。总纲把几个必须一起改的函数标成一个原子阶段（例如一块偏振基分解里共享中间矢量场的三步）时，这几个函数算同一个当前步。组的出口之外仍然只有一跳，而且组必须在 `chains.json` 里预先写好。Agent 不能在任务中途把窗口扩成组。

这个宽度是故意窄的。再宽，任务就会变成「顺手把上下游都整理一下」，局部修改和双向耦合会回到仓库里。

### 4.2 行走方向

业务链按用户操作记录，从入口到叶子，例如「算功率流」：

`api.get_power → orch.compute_layer_modes → fmm.closed.epsilon → pattern.fourier`

执行顺序是这条链的**依赖序**：被调用方先验收，调用方后验收。这样规定的原因：轮到某个函数时，它调用的函数契约已经冻住，它就不能再包一层适配器去凑旧接口。

> **[R06 变更｜S0-6｜采纳]** 原方案：队列脚本对调用图做拓扑排序；跨模块环阶段 A 就失败；模块内递归必须在 `chains.json` 声明，否则排序失败。现方案：先求强连通分量，再对压缩后的图排序。规模大于 1 的分量成为原子组，写入 `chains.json` 的 `groups`；跨模块的分量按 5A 进入 `architecture_debt.toml`。原因：相互递归、回调环、状态机互调都无法拓扑排序；而总纲禁止的边又必须靠阶段 B 才能消除，v0 的规则会两边互相等待。

> **[R30 变更｜V1-S0-5｜采纳]** 原方案（v1）：`scc_groups` 直接对 `static ∪ declared ∪ observed` 求强连通分量，dynamic 不进入这张图。现方案：原始调用边只用于发现。`resolve_deps` 生成 `dependency_graph.json`，SCC、队列和 stale 传播只用这张图。一条边要同时满足：目标是唯一的符号 ID；证据是已解析的 static、被 static 或 observed 印证的 declared，或总纲已确认目标的 dynamic。static 方法调用若指向不清，标 `ambiguous`，不进入 SCC，并阻断相关符号的队列，直到总纲消歧，或 observed / declared 能唯一确定目标。dynamic 一旦确认目标，也进入依赖图。declared 边允许用 static、observed 或已批准的 dynamic 印证，不要求一定有 static 或 observed，因为错误分支和延迟回调可能只有 dynamic 证据。原因：一条误解析的 static 边会把大量函数压成一个巨大分量；一条没观察到的回调环若不进图，排序仍然是错的。`trace_calls` 同时规定：新线程用 `threading.setprofile`；子进程各自写 trace 再合并；async 沿同一线程记录，但保留 task 与 scenario ID；C 扩展只记录 Python 边界，不声称看见内部调用。

一个函数出现在多条链上时，第一次出现的任务拥有修改权。后面的链只重新跑契约、数值夹具和端到端用例，任务卡写成 `verify_only`，diff 必须为空。

### 4.3 任务类型

> **[R05、R06 变更｜S0-5、S0-6｜采纳]** 原方案：只有 `edit` 与 `verify_only` 两种。现方案：如下五种。原因：签名变化、环、残留符号需要不同的窗口和证明。

| 类型 | 何时 | 可改 | 签名 |
| --- | --- | --- | --- |
| `implement` | 常规一步 | 当前符号、其契约与夹具 | 冻结 |
| `migrate` | 必须改签名 | 当前符号，加上全部直接调用者的调用点，加上契约与夹具 | 可变，但同一任务内所有调用点一起改 |
| `group` | 一个强连通分量或预先声明的原子组 | 组内全部符号 | 组内可变，对组外冻结 |
| `verify_only` | 已被别的任务验收，仅重跑 | 无 | 冻结 |
| `residual` | 阶段 C 的残留符号 | 见 5C | 冻结 |

`migrate` 的调用者来自 `dependency_graph.json` 里指向该符号的已解析入边，并附带测试、示例、文档里的字符串命中作为提示。规则：

- 直接调用者超过 `modules.toml` 里的 `max_callers_for_migrate`（默认 10）时，不允许硬迁移，必须先走修订，登记一个带到期任务的 `temp_shim` 进 `architecture_debt.toml`，之后再按 `verify_only` 之外的正常任务撤除。适配层因此有登记、有期限、只减不增。
- 主符号在 `api_snapshot.json` 里时，`migrate` 必须先有 `api_break_approved` 修订。

### 4.4 任务卡

`refactor-gate next` 写出下一张卡，Agent 不能自己选文件。

> **[R05、R18 变更｜S0-5、S1-8｜采纳、修正后采纳]** 原方案：任务卡含 `upstream_callsite_only` 一个字段。现方案：增加任务类型、全部调用者、新增抽象理由、`base_commit` 与总纲版本。原因：支持 `migrate`；过期卡不得使用。

```json
{
  "id": "solve_patterned.fmm_closed",
  "kind": "implement",
  "chain": "solve_patterned",
  "primary": ["fmm.closed.epsilon"],
  "downstream_frozen": ["pattern.fourier"],
  "callers_all": ["orch.compute_layer_modes"],
  "editable_files": [
    "src/fmm/closed.py",
    "refactor/contracts/fmm.closed.epsilon.json",
    "tests/numeric/test_fmm_closed.py",
    "tests/negative/test_fmm_closed.py"
  ],
  "callsite_files": [],
  "fixtures": ["golden:layer2d_circle", "neg:fmm_closed_missing_material"],
  "abstractions": [],
  "numeric_effect": "preserve",
  "intent": "structure",
  "base_commit": "abc123",
  "authority_digest": "sha256:..."
}
```

`check_scope.py` 比较 `git diff` 的路径与 `editable_files` 加 `callsite_files`，多出来的路径使任务失败。对 `callsite_files`，`check_callsite.py` 用 AST 要求调用方函数除该调用外的其余语句不变。对 `editable_files` 里的业务文件，还要求变化落在任务符号的 AST 节点及其必要 import 之内，而不只检查路径。

### 4.5 一步里面的顺序

1. `refactor-gate next` 取出任务卡。若上一张卡没有证明清单，或 `base_commit` 不再是当前分支的祖先，或权威文件在这之后变过，拒绝发卡。
2. 先写契约 JSON，再改函数。契约引用的下游符号必须已是 `reviewed`。
3. 改代码。纯透传包装直接删除。**新增**任何 class 或函数，必须在任务卡 `abstractions` 里登记理由，理由只能取枚举：`shared_invariant`（仓库内至少两处调用）、`public_boundary`、`backend_strategy`、`test_seam`；不接受「将来会用」。
4. 跑 `refactor-gate check --task <id>`。
5. 通过后 `refactor-gate mark` 写证明清单；CI 在当前树上重算并确认。
6. 门禁失败则留在同一步。需要动冻结的下游，或需要改调用方算法时，停止，走第 11 节的修订，不扩大 diff。

> **[R10 变更｜S1-1｜采纳]** 原方案：本步禁止新增 class；模块内函数个数不得净增；注释行不得净增（均为硬门禁）。现方案：纯透传仍硬失败；新增 class/def 必须登记理由并由脚本核对；不再限制注释数量。原因：计数指标会促使把逻辑塞进大函数、误杀值对象和策略对象，还可能用删一个无关函数换新增一个包装；过度封装的本质是「没有独立职责、没有第二个调用者的透传层」，理由枚举加调用者核对更直接。

> **[R18 变更｜S1-8｜修正后采纳，并行租约暂缓]** 原方案：只用 `task_started` 事件防止同一工作区同时开两张卡。现方案：明确 v1 是单 Agent 串行工作流；任务卡带 `base_commit` 与当时的权威文件摘要，过期即拒发；合并队列在最新目标分支上重跑 `gate --all`。原因：不同分支各持一张卡合并后，调用图和契约可能已变。并行需要服务端符号租约，v1 不做，见文末暂缓清单。v1.1 起，这个摘要就是 R26 的 `authority_digest`，不再用手写的 `constitution_version`。

「不用改」也要过门禁。结论枚举只能是 `already_conforms`。脚本仍会扫这个函数的 P2、P5、P6。扫干净才允许在没有 diff 的情况下标 `reviewed`。

## 5. 阶段

### 阶段 0：冻结行为

在任何结构调整之前做完。

1. 总纲写 `reference = external` 或 `reference = snapshot`。S4 移植默认 `external`。
2. `capture_golden` 适配器对固定算例表调用 C 版 S4（或当前 Python），写入 `refactor/golden/`。算例至少覆盖：均匀层、一维光栅、二维图案、实/复介电常数、含张量材料、两种以上傅里叶配方、复制层、近奇异或退化情形、改频率后缓存失效。
3. `baseline.json` 记录当时的反模式计数、`type: ignore` 数、`Any` 数。
4. `tolerances.toml` 对每个输出规定 `atol`、`rtol`、零值附近规则、NaN/Inf 规则和比较方式。比较方式包括：`elementwise`；`eigen_invariant`（特征值排序后比较，特征向量用相位归一化或退化子空间投影 `V Vᴴ` 比较，不逐元素比）；`scalar`。
5. `invariants.toml` 列物理不变量：无损情形能量守恒；复制层与原层等价；均匀层解析解；参考面平移的相位关系；互易性（适用时）。
6. `api_snapshot.json` 用 `inspect` 生成：导入路径、签名、默认值、返回注解，以及契约里声明的异常和返回 dtype、形状。
7. `bench/` 在固定 runner 上跑三档规模（小、中、大）的热点算例，预热后至少 7 次，记录中位数、MAD、峰值内存、线程数。
8. `env.lock` 记录 Python、numpy、scipy、BLAS/LAPACK、CPU 与线程配置、随机种子、locale。
9. 若用 external，同时写出 `known_deviations.json`，记下当前 Python 相对 C 的已知偏差，并为每个算例的每个输出定一个**上限**（当前偏差加噪声余量）。

> **[R07、R09、R16、R21 变更｜S0-7、S0-8、S1-6、S2-3｜采纳、修正后采纳]** 原方案：阶段 0 只有 golden 与 `baseline_vs_python.json`；`preserve` 任务要求偏差「不变大」，`bugfix` 任务才允许变小。现方案：增加容差规则、不变量、API 快照、基准、环境锁，偏差改为固定上限。原因：浮点结果会随 BLAS、线程数和循环顺序变化，「≤ 0 增长」既过严又不等于正确；修 bug 可能让某个样例略升、总体大降，逐样例不升的要求不合理；上限由修订降低而不能升高，仍保持棘轮。

`preserve` 任务的门槛：每个输出都在 `tolerances.toml` 的容差内，或在已知偏差的上限内；全部不变量成立。上限只能通过 `numeric_bugfix` 修订降低。

没有 `golden/`、`tolerances.toml`、`api_snapshot.json`、`env.lock` 时，`refactor-gate next` 不发任务卡。

### 阶段 A：顶向下发现、总纲、依赖方向、全量清单

脚本先出事实，Agent 再填总纲，校验通过才算本阶段结束。

> **[R15 变更｜S1-5｜采纳]** 原方案：阶段 A 只做总纲，随后自底向上处理。现方案：阶段 A 拆成 A1 顶向下发现、A2 事实生成、A3 总纲与债务、A4 分组，每步都有脚本判定的完成条件。原因：只从叶子函数开始，可能把旧 C 数据布局「正确地」冻结，上层之后只能继续适配。

**A1 顶向下发现。** 按用户业务场景写出每条链的端到端输入、输出、不变量，并把链和阶段 0 的 golden 算例建立对应。`check_chain_coverage.py` 要求：每条链至少被一个 golden 算例的运行覆盖（用 `observed_edges.json` 验证），且至少一个负向用例。

**A2 事实生成。**

- `inventory`：符号清单。
- `import_graph`：import 边、环、函数内 import、`importlib` 动态导入。
- `call_graph`：static 边。
- `trace_calls`：运行 golden、端到端和负向用例，写 `observed_edges.json`。跟踪范围见 R30。
- `resolve_deps`：把已唯一解析的边写成 `dependency_graph.json`。存在 `ambiguous` 边时，阶段 A 不能结束。
- `propose_modules`：按目录和调用紧密度给一个**建议**划分。建议不是总纲。

**A3 总纲与债务。** Agent 使用 skill `refactor-constitution`，把建议收成权威文件。

> **[R02、R06 变更｜S0-2、S0-6｜采纳]** 原方案：解析不了的动态调用写入 `dynamic_ref.toml` 草稿；`allowed_imports` 与真实 import 不一致时「差集被列成待消除」。现方案：动态入口按 `dynamic` 级别显式登记；差集必须落实到 `architecture_debt.toml`，每条债务指定消除任务和引入提交，规则是只减不增。原因：漏掉的动态边不会自动变成「解析失败」，草稿机制会漏；「待消除」没有形式化就会成为死锁或永久豁免。

`modules.toml` 的权威是边，不是层号。层号只用于让人读。每个模块必须写一句 `responsibility`（R20，见第 2 节）。每条实际 import 都必须出现在 `allowed_imports` 里，否则记入 `architecture_debt.toml`，不允许静默存在。

S4 的 Python 移植用下面这组模块作为起点。依赖只能沿箭头方向，同层的 fmm 与 rcwa 互不 import。

```text
api → orchestration → fmm → pattern → numeric
                    → rcwa → numeric
                    → gsel → numeric
orchestration → pattern
fmm → fft
rcwa → fft
sampling 与求解器之间没有边
```

这与 `Summary.md` 里 C 版的方向一致：前端只见编排；RCWA 只见数组，不见 `Simulation` 和图案；pattern 不见电磁量；FMM 的多种配方输出同一对 `Epsilon2`、`Epsilon_inv`。Python 里禁止把 `Simulation` 对象传进 fmm 或 rcwa。那是双向耦合最常见的入口。

`options_schema.toml` 列出可选开关和默认值，例如 Lanczos 平滑默认关闭。除此以外的键都是必填。

> **[R11 变更｜S1-2｜采纳]** 原方案：除 schema 外的键，`.get(..., default)` 非法，靠变量名判断。现方案：`modules.toml` 标出核心模块（fmm、rcwa、pattern、numeric）；这些模块的公开函数不接收 `dict`、`Mapping[str, Any]`、`**kwargs`，配置在 API 边界一次性解析成不可变的带类型对象，可选默认只在边界应用。名称到具体值类型的查找表（例如 `Mapping[str, Material]` 的材料表）合法。原因：按变量名识别 `.get` 误报漏报都多；把裸 dict 挡在核心之外，比逐个判断更确定，也不会误伤普通缓存字典。

`canonical_vars.toml` 至少登记：`omega`、`kx`、`ky`、`G`、`Epsilon2`、`Epsilon_inv`、`q`、`phi`、`kp`、层模式缓存、全堆叠振幅。每个量登记一个生产者函数和一个带类型载体（例如 `EpsilonMatrices` 数据类的字段）。

> **[R14 变更｜S1-4｜采纳]** 原方案：脚本禁止其他函数给该量的名字赋值。现方案：载体类型只能在生产者内构造，其他位置构造即失败；缓存用例统计生产者的调用次数。原因：换个变量名重算，或只重算公式的一部分，都绕得过按名字的赋值检查。

`negative_cases.toml` 在这里由 schema 和契约生成，并由 Agent 补充需要的错误类型：必填键缺失、形状错误、dtype 错误、材料引用不存在、层次结构非法，各自期望的异常类型。

> **[R08 变更｜S0-7｜修正后采纳]** 原方案：无负向用例。评审建议：在阶段 0 冻结当前负向行为。现方案：负向用例由 schema 和契约**规定**，在阶段 A 写成 `negative_cases.toml`，不从现有代码的行为反推。原因：现有代码在缺输入时静默继续，正是要消除的 P2；冻结它就把缺陷固化了。

**A4 分组与链。** `scc_groups` 对 `dependency_graph.json` 求强连通分量；`build_chains` 从入口和这张依赖图生成 `chains.json` 初稿。Agent 只做两件事：给链命名，确认或拆分原子组。脚本拒绝未覆盖的可达入口。

S4 移植的链按这个依赖序执行：

1. `numeric` 的分解与特征求解包装
2. `pattern` 的包含树、解析傅里叶、栅格化、流场
3. `gsel`
4. 各 FMM 配方，封闭形式先于 FFT、Kottke、偏振基
5. RCWA 层模式
6. S 矩阵、激励、振幅
7. 功率流与场
8. 编排层的缓存与失效
9. api 的参数编解码

阶段 A 结束条件，全部由 `check_constitution.py` 判定：

- 每条实际 import 要么在 `allowed_imports` 里，要么在 `architecture_debt.toml` 里。`allowed_imports` 本身无环。
- 每个文件恰好属于一个模块，每个模块有 `responsibility`。
- 每个可达入口都在某条链上，每条链被 golden 与负向用例覆盖。
- 每个规范量恰好一个生产者和一个载体，且都在 inventory 里。
- `public_api` 里的符号都存在，并且都在 `api_snapshot.json` 里。
- 动态引用被显式登记为 `dynamic`，或改成静态调用，不允许留着空草稿。
- SCC 已全部分组或登记债务。

### 阶段 B：沿链一步一任务

队列由 `refactor-gate queue` 按 4.2 生成。每个任务走 4.5。

本步要核对的 I/O，写在契约里，不写在聊天记录里。

> **[R12、R13 变更｜S1-2、S1-3｜采纳]** 原方案：契约里的 `shape` 是字符串，靠「函数前部是否引用 shape」检查；`raises` 必须非空。现方案：形状用受限表达式并引用 `dims`，由测试期运行时代理真实求值；`raises` 只对有前置条件的边界必填并必须对应负向测试；返回值区分 `optional_value` 与 `failure_sentinel`。原因：`shape` 引用一次就能通过，没有验证；纯数值内核在已验证输入上可能没有业务异常。

```json
{
  "symbol": "fmm.closed.epsilon",
  "schema_version": 2,
  "dims": {"n": "len(G)"},
  "inputs": [
    {"name": "G", "canonical_id": "G", "dtype": "int64", "shape": ["n", "2"],
     "required": true, "source": "gsel.select", "index_base": 0}
  ],
  "outputs": [
    {"name": "Epsilon2", "canonical_id": "Epsilon2", "dtype": "complex128",
     "shape": ["2*n", "2*n"], "nullable": "never"}
  ],
  "preconditions": ["len(materials) > 0"],
  "callees": ["pattern.fourier"],
  "mutates": [],
  "invalidates": [],
  "raises": [
    {"type": "KeyError", "when": "material tag not in materials",
     "test": "tests/negative/test_fmm_closed.py::test_missing_material"}
  ],
  "numeric_effect": "preserve"
}
```

`nullable` 取 `never`、`optional_value`、`failure_sentinel`；`failure_sentinel` 一律失败。

`check_contract.py`（静态）检查：

- 契约符合 schema，`dims`、`shape`、`preconditions` 只使用受限表达式。
- 必填输入在签名里没有默认值，函数体对必填输入不使用 `.get` / `getattr` 默认值 / `or` 默认值。
- `callees` 覆盖该符号在 `dependency_graph.json` 里的全部出边，且对端契约已是 `reviewed`。
- 跨函数传递的数据按 `canonical_id`、dtype、shape、unit，以及生产者与消费者符号对接。对不上就失败，而不是在本函数里再算一遍下游要的量。Python 参数名只供人读，改名本身不失败。

> **[R29 变更｜V1-S0-4｜采纳]** 原方案（v1）在这里要求「输出名字与下游契约的输入名字能对上」。现方案：按 `canonical_id` 对接。原因：v1 已经用 `canonical_id` 表示跨函数的数据身份，再要求局部参数名一致，会把无害的改名判成失败，也和「名字不是身份」矛盾。同一次修订还改了下面两处：单方法类的判定，以及 `new_optional` 的语义。
- `canonical_id` 的载体只在登记的生产者里构造。
- 核心模块的公开函数没有裸 `dict` / `Mapping[str, Any]` / `**kwargs` 形参。
- `raises` 里每一项的 `test` 引用真实存在的测试。

**运行时契约代理**（测试期）由检查器提供的 pytest 插件通过导入钩子注入，不需要在生产代码里加装饰器：入口和出口按 `dims` 求值形状与 dtype；对 `mutates` 之外的输入数组在调用前后比较哈希，发生变化即失败。

本步同时跑第 9 节的局部反模式脚本，范围是任务卡里的文件，外加集合棘轮：当前 issue ID 集合必须是 `baseline/issues.json` 的子集（R28）。个数可以写进报告，不能单独决定通过或失败。

本步还要通过：

> **[R09、R17 变更｜S0-8、S1-7｜采纳]** 原方案：只有数值夹具与反模式扫描。现方案：增加三项。原因：夹具通过不代表改动的分支被执行；AST 扫描不能证明低效写法已消除。

- **被改行的分支覆盖**：任务符号中新增或修改的行，分支覆盖率 100%；未覆盖的行列入报告并失败。不设全仓统一覆盖率数字。
- **边观察**：契约里声明的每条 `callees` 至少在夹具运行中被观察到一次。
- **性能与内存**：任务声明的基准算例，中位数不得慢于基线超过 `max(3×MAD/中位数, 10%)`，峰值内存不得超过基线的 1.25 倍。`intent = performance` 的任务还必须达到任务卡里声明的改进目标。硬失败只发生在固定 runner；其他机器只出报告。

每步结束，`refactor-gate mark` 写证明清单。inventory 本身重生成，不手改。

### 阶段 C：按模块清残留

进入条件：所有链上符号都是 `reviewed`、`deleted` 或 `exempt`。`residual` 列出仍然 `untouched` 的符号，并先做机器分类：

> **[R02、R23 变更｜S0-2、S2-5｜采纳]** 原方案：`dead_candidate` 是「静态不可达且不在动态引用名单」；`test_only` 直接删除。现方案：`dead_candidate` 要求四类证据同时成立；`test_only` 先核对 API 快照、文档和发布历史。原因：`ast` 会漏边，运行时也可能没覆盖到；测试是唯一使用者不等于无价值，可能是兼容或插件入口。

| 分类 | 脚本规则 | 随后的任务 |
| --- | --- | --- |
| `dead_candidate` | 静态不可达，且 `observed_edges.json` 里没有，且不在 `api_snapshot.json`，且不在 `dynamic_ref.toml`，且 `refs` 对名字、字符串、装饰器、注册表的检索为空 | 删除任务，删除后跑全量测试；任一证据不成立则改为人工分类 |
| `dynamic` | 不可达但在动态引用名单里 | 改成静态调用，或保留并写 `exempt` 理由 `dynamic_entry` |
| `test_only` | 只有测试调用 | 先核对 API 快照、文档、发布历史；确认无外部承诺再删除生产代码与只为它存在的测试，否则补进某条链或标 `frozen_public_api` |
| `side_path` | 可达但不在任何链上 | 补链（修订）或删除。不允许一直留在链外 |

每个残留符号仍是一张任务卡，窗口仍是 W1：当前符号、直接调用方、直接被调用方只读。以该函数为中心看它落在哪条调用链上，结论只能是三选一，写入 `decisions/`：

- `move_into_chain`：它属于某条已有链的哪一条边，修订 `chains.json` 后按阶段 B 的规则改。
- `delete`：死代码，本步只删定义和因此无人用的 import。
- `exempt`：理由枚举 `third_party_shim`、`dynamic_entry`、`frozen_public_api` 之一，并引用总纲段落。

`check_dupes.py` 给出的重复簇在这一阶段逐个处理。

> **[R14 变更｜S1-4｜采纳]** 原方案：每簇只留契约里登记的一个所有者，其余强制合并。现方案：每簇结论记为 `merge` 或 `independent`，`independent` 要写证据（不同公式来源或不同物理含义），`merge` 则走 `implement` 或 `migrate` 任务。原因：AST 归一化相同可能是两个合理的独立简单公式，AST 不同也可能是同一公式。

阶段 C 结束条件：不存在 `untouched`，不存在未分类的残留，重复簇全部有结论。

### 阶段 D：全局验收

`gate --all` 对全仓库重跑第 9 节的全部脚本，并重生成 inventory 与 `observed_edges.json`。通过条件：

> **[R07、R09、R16 变更｜S0-7、S0-8、S1-6｜采纳、修正后采纳]** 原方案：阶段 D 的数值条件是「golden 在容差内、`preserve` 任务偏差没有变大」，无 API、性能、债务条件。现方案：增加下面加粗的几项，并要求在最新目标分支和固定 runner 上重跑。原因：见变更总表。

- 状态只有 `reviewed`、`deleted`、`exempt`，没有 `untouched` 和 `stale`，且每个 `reviewed` 的 `proof_hash` 重算一致。
- import 与 `allowed_imports` 一致，无环，**`architecture_debt.toml` 为空**。
- 反模式计数满足终局：P2、P5 未登记项、P6、P8、P10、P17、P18 为 0；P9、P19 不高于基线且 P9 的剩余项都在延迟 import 名单里。
- 每条链的每条边都有契约，契约引用的符号存在。
- 规范量生产者与载体构造点唯一。
- 全部 golden 算例在容差内或已知偏差上限内；全部物理不变量成立；**全部负向用例通过**。
- **`api_snapshot.json` 逐项一致，或每处差异都有 `api_break_approved` 修订。**
- **固定 runner 上的基准无回退，峰值内存无回退。**
- `audit.md` 的每一节第一行是 `attestation: <长期地址> sha256: <哈希>`。`check_audit_report` 重新计算 `final-attestation.json` 的哈希。对不上则阶段 D 失败。Agent 不能用「已全局看过」代替这一行。

> **[R31 变更｜V1-S0-6｜采纳]** 原方案（v1）：这一行引用每步或终局的 CI 制品。现方案：每步详细报告仍放短期 CI 制品，允许过期。阶段 D 另写一份 `final-attestation.json`，内容是工具哈希、环境哈希、`authority_digest`、代码提交、`run_tree_digest`、各门禁摘要，以及详细报告的 SHA256。团队档由 CI 身份签名，存到 release 或不可变对象存储。单人本地档把这份摘要和报告压缩包的 SHA256 放进 release，仓库里保留 `refactor/state/final-attestation.json`，不保留本机路径。`audit.md` 只引用这个长期地址。原因：普通 CI 制品会过期，过期后审计里的哈希无法再核对，终局「全部通过」就只剩一串无法验证的摘要。

阶段 D 的 Agent 技能 `refactor-global-audit` 只做三件脚本做不到的事，而且每件都要落成枚举记录：抽查豁免名单里每一项理由是否仍匹配代码；确认没有两条链用不同公式算同一个 `canonical_id`；确认缓存失效列表覆盖「改几何、改材料、改频率、改 G」。做完必须再跑一次 `gate --all`。两次报告哈希都写进 `audit.md`。

## 6. 非脚本判断怎么收口

不能做成 AST 规则的判断，只允许出现在阶段 A 和修订里，运行中不再临场发明。

| 判断 | 谁做 | 记录 | 脚本随后强制什么 |
| --- | --- | --- | --- |
| 模块边界划在哪 | 阶段 A | `modules.toml` | 非法 import 失败 |
| 键是必填还是可选 | 阶段 A | `options_schema.toml` | 必填键上的默认值失败；核心模块不接裸 dict |
| 规范量谁生产 | 阶段 A | `canonical_vars.toml` | 载体在生产者外构造失败 |
| 双循环是几何遍历还是低效核 | 修订或阶段 A | `allowlists/loops.toml` 的理由枚举 | 数值模块使用 `geometry_traversal` 失败 |
| 透传要不要留 | 阶段 A 的 `public_api` | 不在名单里的透传失败 | 阶段 B 不能新增透传 |
| 新增抽象是否合理 | 任务卡 | `abstractions` 理由枚举 | 理由与调用者数量不符失败 |
| 动态引用是不是入口 | 阶段 C | `decisions/*.json` | 理由不在枚举里失败；引用的行号对不上失败 |
| 重复簇合并还是独立 | 阶段 C | `decisions/*.json` | `independent` 缺证据失败 |
| 数值变化是修 bug 还是改坏了 | 修订 | 任务卡 `numeric_effect` | 未标记 `bugfix` 的数值漂移失败 |
| 现存非法依赖何时消除 | 阶段 A | `architecture_debt.toml` | 旧 debt ID 只能保留或删除；不能把一条边换成另一条边来保持个数不变 |

`decisions/*.json` 的 `choice` 只能是脚本内置枚举。记录里必须有 `evidence`：文件路径和行号。`check_decision.py` 确认这些行仍然存在，并且与 choice 对应的结构还在（例如 choice 是 `delete` 时符号已不在 AST 里）。它不评价物理论证。

## 7. Skills

Skill 只规定读哪些工件、允许改什么、必须跑哪条命令、怎样才算完。策略的正文在本文和检查器里，skill 不复制一套可以漂掉的口吻要求。

四个 skill，都放在目标仓库 `.cursor/skills/`。

### `refactor-constitution`

- 何时：阶段 0 的基线已经生成，阶段 A 尚未通过 `check_constitution`。
- 先读：`propose_modules` 的报告、`Summary.md` 一类的参考结构（S4 移植必读本仓库那份摘要）、import 环与 SCC 报告、`observed_edges.json`。
- 允许写：`constitution.md`、`modules.toml`、`options_schema.toml`、`canonical_vars.toml`、`negative_cases.toml`、`architecture_debt.toml`，以及对 `chains.json` 的命名和分组。这些改动必须在带修订编号的提交里完成，不得同时改 `src/`。
- 禁止：改业务代码，把建议划分原样粘贴成总纲而不填 `allowed_imports` 和 `responsibility`，把违规依赖直接写进 `allowed_imports` 以求通过。
- 完成：`check_constitution` 退出码 0。

> **[R25 变更｜S0-1 延伸｜本次新增]** 原方案：允许写的文件里没有对提交范围的限制。现方案：权威文件的改动必须单独成提交并带修订编号。原因：本地档下要把「独立审批」变成可脚本判定的条件。

### `refactor-step`

- 何时：`/refactor-next` 发出一张 `implement`、`migrate`、`group` 或 `verify_only` 任务卡。
- 先读：任务卡、当前符号源码、下游契约、规范量表里与本符号有关的行。不读窗口外的实现文件。
- 允许写：任务卡的 `editable_files` 与 `callsite_files`。`implement` 任务不改签名。
- 禁止：新增未登记理由的 class 或函数；新增 `.get` 默认值；放宽容差、基线或 allowlist；修改下游函数体；重构调用方算法；新增只复述代码的叙述性注释。数学来源、单位和数值稳定性的说明可以写。
- 完成：`refactor-gate check` 退出码 0，然后只通过 `refactor-gate mark` 写证明清单。

### `refactor-residual`

- 何时：`residual` 报告链上符号已清空，且仍有 `untouched`。
- 先读：该符号的分类、引用报告、调用方列表、API 快照。
- 允许的结论：`move_into_chain`、`delete`、`exempt`。没有第四种。
- 完成：该符号不再是 `untouched`，`refactor-gate check` 对这张残留卡退出码 0。

### `refactor-global-audit`

- 何时：阶段 C 的结束条件已经满足。
- 先跑：`gate --all`。失败则回到对应阶段的任务，不在审计 skill 里直接改业务代码。
- 允许写：`audit.md`、对豁免项的 `decisions/` 记录。发现缺漏时只开修订或把符号打回 `stale`，不趁审计改无关文件。
- 完成：`check_audit_report` 退出码 0。

## 8. Commands

Command 是短入口，本身不做判断，只调用检查器并指出必须使用的 skill。建议放在目标仓库 `.cursor/commands/`。

| Command | 作用 | 拒绝条件 |
| --- | --- | --- |
| `/refactor-init` | 跑阶段 0 的捕获、基线、环境锁、API 快照、基准，再跑清单、观察图和总纲检查 | 覆盖已有基线时必须显式 `--force` 并走修订；仓库不干净 |
| `/refactor-next` | 打印下一张任务卡路径，并要求按卡上的 skill 执行 | 阶段 A 未通过；上一任务没有证明清单；`base_commit` 过期；工作区有任务卡之外的未提交改动 |
| `/refactor-step` | 对当前卡跑 `refactor-gate check`；通过则 `mark` | 没有当前卡；范围检查失败 |
| `/refactor-residual` | 生成残留分类并取出下一张残留卡 | 链上仍有未验收符号 |
| `/refactor-audit` | 跑 `gate --all` 和审计报告校验 | 阶段 C 未结束 |
| `/refactor-amend` | 第 11 节的唯一修订入口 | 工作区有未标记的业务 diff |
| `/refactor-status` | 只打印计数：各状态符号数、当前链、棘轮指标、债务数量、下一批任务 id | 无 |

不设「把这个函数也一起改了」的 command。扩大范围只有 `/refactor-amend`。

## 9. 硬编码脚本

> **[R01 变更｜S0-1｜采纳]** 原方案：脚本放在仓库 `scripts/refactor/`，与业务代码同权限。现方案：全部作为 `refactor-gate` 检查器的子命令，安装位置在工作区之外，版本由 `tool.lock` 固定。下表的「脚本」即子命令。原因：见 1.1。

检查器用 Python 标准库的 `ast` 做结构检查，不依赖仓库能跑通测试；数值、覆盖、基准部分依赖 numpy、coverage 与仓库自己的测试。每个子命令把 JSON 报告写到 CI 制品目录（本地写到不入库的目录），并向 stdout 打一行 `sha256`。退出码 0 为通过，1 为检查失败，2 为用法或工件缺失。

标 `▲` 的行相对 v0 有改动，标 `新增` 的行是 v1 新增。

| 脚本 | 检查 | 用在 |
| --- | --- | --- |
| `inventory` ▲ | 生成符号清单（范围见第 3 节）；边带可信度 | 每次门禁之前 |
| `import_graph` ▲ | 环、未授权边、函数内 import、星号 import、`importlib` 动态导入 | 阶段 A、每步、阶段 D |
| `call_graph` ▲ | static 边；动态调用登记名单 | 阶段 A、阶段 C |
| `trace_calls` 新增 | 运行 golden、端到端、负向用例，生成 `observed_edges.json` | 阶段 A、阶段 C、阶段 D |
| `propose_modules` | 只产出建议，退出码始终 0 | 阶段 A 之前 |
| `module_metrics` 新增 | 模块内调用占比、跨模块边数、public API 面积、扇入；只出报告 | 阶段 A、阶段 D |
| `resolve_deps` 新增 | 由发现边生成 `dependency_graph.json`；`ambiguous` 边阻断相关队列 | 阶段 A、每步、阶段 D |
| `scc_groups` 新增 | 只对 `dependency_graph.json` 求强连通分量 | 阶段 A、队列 |
| `build_chains` | 入口覆盖、组不与依赖序矛盾 | 阶段 A |
| `check_chain_coverage` 新增 | 每条链被 golden 与负向用例覆盖 | 阶段 A、阶段 D |
| `queue` ▲ | 对压缩后的图做拓扑序，生成任务卡 | `/refactor-next` |
| `check_constitution` ▲ | 阶段 A 的结束条件 | 阶段 A |
| `check_authority_commits` 新增 | 遍历 `base..HEAD`：改动权威文件的提交不得同时改业务代码，且必须带 `Refactor-Amend: <id>` 并有对应记录；`tool.lock` 变更同样受限 | 每步、阶段 D、CI |
| `check_debt` 新增 | 非法依赖 ⊆ `architecture_debt.toml`。比较的是 debt ID 集合：旧 ID 只能保留或删除，不能替换端点；允许改的只有撤除任务和说明。严重度不能降低，除非修订给出证据。新增只允许出现在阶段 A 的修订里 | 每步、阶段 D |
| `check_scope` ▲ | diff 路径 ⊆ 任务卡；业务文件的变化落在任务符号节点内 | 每步 |
| `check_callsite` ▲ | 调用点文件只有调用表达式变化；`migrate` 时核对调用者列表完整 | 每步 |
| `check_defensive` | P2、P6、P7、P8：`.get` 默认、`getattr` 默认、`or` 默认、裸 except、except 后 return/pass、链上签名默认值、内部 `**kwargs` | 每步和终局 |
| `check_core_params` 新增 | 核心模块公开函数没有裸 `dict` / `Mapping[str, Any]` / `**kwargs` 形参 | 每步和终局 |
| `check_wrappers` | 函数体只有一条、不做转换的转发调用：不在 `public_api` 即失败。类除 `__init__` 外只有一个方法时只出候选；仅当这个方法也是纯委托、类没有自己的不变量或资源生命周期，且任务卡没有登记 `backend_strategy` 或 `test_seam` 时才失败 | 每步和终局 |
| `check_abstraction` ▲ | 本步 diff 新增的每个 class/def 都在任务卡 `abstractions` 里有理由；`shared_invariant` 要求仓库内至少两处调用；不统计数量，不统计注释 | 每步 |
| `check_loops` ▲ | 候选发现：嵌套 `range` 循环且循环体是下标算术、`np.vectorize`、逐行 `apply`、嵌套推导式、`append` 热循环。豁免理由枚举：`geometry_traversal`、`sparse_assembly`、`mode_pairing`。`fmm` 与 `rcwa` 不得使用 `geometry_traversal` | 每步和终局 |
| `check_contract` ▲ | 第 5 节阶段 B 的静态契约规则 | 每步和终局 |
| `contract_runtime` 新增 | pytest 插件：入口与出口按 `dims` 求值形状与 dtype；`mutates` 之外的输入数组哈希不变 | 每步 |
| `check_negative` 新增 | `negative_cases.toml` 的每个用例抛出规定的异常类型；契约 `raises` 的 `test` 存在且通过 | 每步和终局 |
| `check_invariants` 新增 | 物理不变量成立 | 每步相关夹具，终局全量 |
| `check_dupes` ▲ | 归一化 AST 哈希聚类，只产出候选 | 阶段 C、阶段 D |
| `check_globals` | 可变默认参数、模块级可变容器、`print`、`breakpoint` | 每步和终局 |
| `check_issue_set` 新增 | 当前 issue ID 集合必须是 `baseline/issues.json` 的子集。覆盖 `Any`、`type: ignore`、防御性默认、未登记循环和架构债务 | 每步、阶段 D |
| `check_typing` | 产出 `Any` 与 `type: ignore` 的 issue ID；个数只写进报告 | 每步、阶段 D |
| `check_diff_coverage` 新增 | 任务符号中新增或修改的行分支覆盖 100%；声明的调用边被观察到 | 每步 |
| `check_api_snapshot` 新增 | 与 `api_snapshot.json` 逐项比较 | 每步涉及 `public_api` 时、阶段 D |
| `bench_check` 新增 | 固定 runner 上的中位数、MAD、峰值内存；非固定 runner 只出报告 | 每步相关基准、阶段 D |
| `check_proof` 新增 | 重算每个符号的 `proof_hash`，把不一致者标为 `stale` | 每次门禁 |
| `refs` ▲ | 删除前的名字引用，含字符串、装饰器、注册表 | 阶段 C 删除 |
| `residual` ▲ | 残留分类（四类证据） | 阶段 C |
| `check_decision` | 判断记录的枚举、行号、与 choice 一致的结构 | 修订、阶段 C、阶段 D |
| `numeric_check` ▲ | 按 `tolerances.toml` 比较；已知偏差不超过上限；本征分解按不变量比较 | 每步相关夹具，终局全量 |
| `capture_golden` | 生成 golden（通过 `adapters/` 调用参考实现） | 阶段 0 |
| `check` ▲ | 串联本步需要的上述检查 | `/refactor-step` |
| `gate` | `--step` 或 `--all` | 每步与阶段 D |
| `mark` ▲ | 写小型证明清单（含 `proof_hash`、`run_tree_digest`）；不是事实来源 | 每步结束 |
| `attest` 新增 | 写 `final-attestation.json`，并按 5D 存到长期位置 | 阶段 D |
| `check_audit_report` | 核对审计报告引用的长期证明哈希 | 阶段 D |
| `status` | 从证明清单与重算结果重建进度视图 | `/refactor-status` |

`gate --step` 的最小集合：`inventory`、`resolve_deps`、`check_proof`、`check_authority_commits`、`check_debt`、`check_issue_set`、`check_scope`、`check_callsite`、`check_defensive`、`check_core_params`、`check_wrappers`、`check_abstraction`、`check_loops`、`check_contract`、`check_negative`、`check_typing`、`check_diff_coverage`、`import_graph`、`numeric_check`、`check_invariants`（后两者只跑这张卡声明的夹具），以及任务声明的 `bench_check`。

`gate --all` 额外加入：`trace_calls`、`check_chain_coverage`、`check_dupes`、`check_globals`、`check_api_snapshot`、`residual` 必须为空、`numeric_check` 与 `check_invariants` 全量、`bench_check` 全量、`check_constitution`、`attest`。

第三方代码目录在总纲 `ignore_paths` 里列出，所有 AST 检查跳过。不设默认忽略 `tests/`：测试里的 `.get` 默认值不计入 P2，但测试里复制一份生产算法会计入 `check_dupes`。

## 10. 日志与指标

### 10.1 事件日志

> **[R04、R22 变更｜S0-4、S2-4｜修正后采纳]** 原方案：`events.jsonl` 入库，哈希链断裂时 `gate.py` 失败，作为不可篡改的依据。现方案：事件日志降为辅助，只用于发现误改与恢复现场；完整报告放 CI 制品，仓库里只留 `state/proofs/`。原因：Agent 可以重写日志并重算哈希；每步提交大量生成文件会淹没业务 diff。

`refactor/log/events.jsonl` 只追加，不入库时放在本地不入库目录。`mark` 和修订命令写入。手改导致哈希链断裂时，本地 `gate` 给出警告；团队档里判定以 CI 的重算为准。每行：

```json
{
  "ts": "2026-09-30T00:00:00Z",
  "event": "task_marked",
  "task_id": "solve_patterned.fmm_closed",
  "symbol_ids": ["fmm.closed.epsilon"],
  "run_tree_digest": "...",
  "proof_hash": "...",
  "report_ref": "ci-artifact://...",
  "prev_sha256": "..."
}
```

事件名只使用：`baseline_captured`、`phase_a_accepted`、`task_started`、`gate_failed`、`task_marked`、`amend_opened`、`amend_accepted`、`audit_accepted`。

`task_started` 到下一条 `task_marked` 或 `gate_failed` 之间不允许第二张 `task_started`。这只防止同一工作区里同时开两张卡；跨分支的过期由任务卡的 `base_commit` 处理（4.5）。

### 10.2 每步留下的数

`status` 和 CI 读取同一些字段。阈值如下。

> **[R07、R09、R10、R17 变更｜S0-7、S0-8、S1-1、S1-7｜采纳、修正后采纳]** 原方案：含 `def_net_increase`、`class_increase`、逐样例 `deviation_growth`。现方案：删除这三项，新增下面标 `新增` 的指标。原因：计数指标诱导错误设计；逐样例偏差不升对浮点不合理；新指标对应新增的证明与门禁。

| 指标 | 每步 | 阶段 D |
| --- | --- | --- |
| `scope_violation` | 0 | 0 |
| `authority_commit_violation` 新增 | 0 | 0 |
| `unauthorized_import` | 本步不得新增 | 0 |
| `import_cycle` | 0 | 0 |
| `architecture_debt_ids` | 旧 ID 只能保留或删除 | 空集 |
| `defensive_required` | 本步文件为 0；全库 ID 集合 ⊆ 基线 | 空集 |
| `core_raw_dict_param` 新增 | 本步文件为 0 | 0 |
| `silent_except` | 0 | 0 |
| `passthrough_illegal` | 本步为 0 | 0 |
| `abstraction_unjustified` 新增 | 0 | 0 |
| `nested_loop_unlisted` | 本步为 0 | 0 |
| `canonical_reassign` | 0 | 0 |
| `contract_gap` | 本步边为 0 | 0 |
| `contract_runtime_fail` 新增 | 0 | 0 |
| `negative_case_fail` 新增 | 0 | 0 |
| `invariant_fail` 新增 | 0 | 0 |
| `diff_branch_uncovered` 新增 | 0 | — |
| `declared_edge_unobserved` 新增 | 0 | 0 |
| `type_ignore_ids` | ⊆ 基线集合；个数只展示 | ⊆ 基线，终局可以为空 |
| `any_ids` | ⊆ 基线集合；个数只展示 | ⊆ 基线，终局可以为空 |
| `numeric_abs_max` | ≤ 该夹具容差或已知偏差上限 | 全部夹具 |
| `api_break` 新增 | 0，或有 `api_break_approved` | 0，或每处有批准 |
| `bench_regress` 新增 | 0（固定 runner） | 0 |
| `peak_mem_regress` 新增 | 0（固定 runner） | 0 |
| `proof_stale` 新增 | 队列中随任务下降 | 0 |
| `untouched_on_chain` | 随队列下降 | 0 |
| `untouched_off_chain` | 阶段 B 可不处理 | 0 |

检查器自测（故意违规样本必须失败、修正后必须通过）随检查器包发布，是检查器 CI 的一部分。业务仓库的测试分三层，都由 CI 调用，不靠 Agent 记得跑。

1. **`tests/structure/test_gate.py`**：在完整仓库上跑 `gate --step` 或 `--all`，把退出码变成 pytest 结果。CI 每个 PR 跑 `--all` 中除全量数值与基准以外的部分；全量数值在合并队列跑；基准在固定 runner 上跑。
2. **`tests/numeric/`**：每个链步至少一个夹具，直接对比 `refactor/golden/`，并检查不变量。编排层的缓存用例要断言：改区域、改材料、改频率、改 G 之后，下一次求解的结果与冷启动一致，且规范量的生产者被重新调用，用来守住 P16 与 P4。
3. **`tests/negative/`**：`negative_cases.toml` 的每个用例，以及契约 `raises` 里引用的测试。

不把「代码行数下降」或「圈复杂度」列成门槛。那些数变小并不表示 P1 到 P5 被除掉，还容易诱使 Agent 为了指标拆函数。

## 11. 修订

总纲、依赖边、可选默认值、循环豁免、容差与基线、检查器版本、数值从 `preserve` 改成 `bugfix`、把一步扩成组，都算修订。业务重构任务里面不做这些决定。

> **[R01、R06、R07、R16、R25 变更]** 原方案：理由枚举为 `break_cycle`、`wrong_owner`、`atomic_group`、`loop_exempt`、`numeric_bugfix`、`new_optional`。现方案：增加下表加粗的理由，并要求修订提交与业务提交分离。原因：检查器、容差、API、债务、临时适配层的变更需要有入口，且不能夹带业务改动。

`/refactor-amend` 的步骤：

1. 工作区除了修订说明没有未标记的 `src/` diff，否则拒绝。
2. 复制一份当前权威文件到 `refactor/amend/<id>/before/`。
3. Agent 只改权威文件，并写 `reason`，理由枚举：`break_cycle`、`wrong_owner`、`atomic_group`、`loop_exempt`、`numeric_bugfix`、`new_optional`、**`scc_group`**、**`debt_temp_shim`**、**`tolerance_change`**、**`api_break_approved`**、**`tool_upgrade`**。
4. 提交信息必须带 `Refactor-Amend: <id>`，提交不得同时包含 `src/` 改动；`check_authority_commits` 在每次门禁里核对。
5. `check_constitution` 与受影响链的 `numeric_check` 必须通过。`numeric_bugfix` 只能降低已知偏差的上限，不能升高。`tolerance_change` 不得松于基线，除非同时有 `tool_upgrade` 或参考实现变更的说明。`new_optional` 要求默认值写在 schema 里，并且只在公开 API 边界应用：缺键时套用这个默认值，再传给核心链；核心链看到的是已经存在的字段，不再处理缺键。若缺键必须报错，这个键仍是必填，不能用 `new_optional`。默认值不能加在内部链上。`debt_temp_shim` 必须指定撤除任务。

> **[R29 续｜V1-S0-4｜采纳]** 原方案（v1）写的是「`new_optional` 要求缺键时仍在公开 API 边界抛错」。现方案：可选键缺省时套用已登记默认值；必须抛错的键保持必填。原因：缺键仍抛错就是必填，不是可选，两条规则不能同时成立。

> **[R29 续｜V1-S0-4｜采纳]** `check_wrappers` 的现方案见第 9 节。原方案（v1）把「类除 `__init__` 外只有一个方法，且不在 `public_api`」直接判失败。现方案：这种情况只出候选；只有该方法也是纯委托、类没有自己的不变量或资源生命周期，并且没有登记 `backend_strategy` 或 `test_seam` 时才失败。原因：FFT 后端、线性求解后端通常只有一个公开方法，也不是顶层 `public_api`，v1 会把第 4.5 节允许的 `backend_strategy` 直接判死。
6. 通过后重算 `proof_hash`，凡是依赖该决定的符号都变为 `stale`，队列在这些符号上重做。不允许「修订完就当全库已验收」。

团队档里，修订还需要 `CODEOWNERS` 审批。

## 12. 一次任务长什么样

以「图案层的封闭形式介电矩阵」为例。队列已经验收完 `pattern.fourier` 和 `gsel.select`，还没轮到 `orch.compute_layer_modes`。

任务卡的当前符号是 `fmm.closed.epsilon`，类型 `implement`，签名冻结。下游 `pattern.fourier` 只读。调用者 `orch.compute_layer_modes` 不需要改动。

本步要做的事只包括：

- 契约写明输入 `G`、图案、材料表，输出 `Epsilon2` 与 `Epsilon_inv`，`canonical_id` 使用总纲里的载体；`dims` 里 `n = len(G)`；`raises` 登记 `KeyError` 并指向负向测试。
- 删掉缺材料时回落到真空介电常数的 `.get`。缺材料抛 `KeyError`。
- 若函数只是把 `Simulation` 拆开再转给另一个同名内部函数，把内部函数抬成这一层的实现，删掉外壳。此时签名会变，因此这张卡应当是 `migrate`：任务卡列出全部调用者，一起改。若这层是 fmm 模块的 `public_api`，可以保留一个与 C 侧 `FMMGetEpsilon_ClosedForm` 同签名的入口，但入口必须真的列在 `public_api` 里。
- 双层 `range(n)` 填充矩阵的部分改成数组运算。几何上逐形状求交的循环若在 `pattern` 里，不在这一步；这一步在 `fmm`，不能用 `geometry_traversal` 豁免。
- 不在这里重算 `G` 或包含树。生产者不是这个函数。
- 需要新增辅助函数时，在任务卡 `abstractions` 里写理由，并保证仓库内至少两处调用（`shared_invariant`）。

门禁通过后，这个符号获得证明清单。编排函数里其余的逻辑留到它自己的那一步：那一步会检查它没有第二次组装 `Epsilon2`，并检查改频率时 `invalidates` 含模式缓存。

## 13. 在目标仓库里的落地顺序

先验证方案本身，再落地检查，最后才允许 Agent 改业务代码。

> **[R24 变更｜评审 §9｜采纳]** 原方案：先写一批检查脚本，然后开工。现方案：第 0 步先做三个原型。原因：先验证信任边界、调用图漏边率、任务原子性这三个最关键的假设。任一原型失败都说明设计需要继续调整，不能带着问题铺开二十多个脚本。

0. **三个原型**，任一失败则回到设计。
   1. **信任边界原型**：从固定提交安装最小检查器，证明业务 PR 修改仓库内同名脚本、基线、容差都不能改变 CI 结果；`check_authority_commits` 能识别把权威文件与业务代码混在一个提交里。
   2. **调用图原型**：选一条含方法调用、回调和一个动态入口的真实业务链，对比 static、observed 与人工整理的调用图，量化漏边率，并确认 `declared` 与 `observed` 能补上 static 的缺口。
   3. **任务原子性原型**：选一个有三个调用者的函数，分别演练保持签名的 `implement` 与改签名的 `migrate`，确认 W1、`proof_hash`、`stale` 传播能闭环，且不需要在任务里新增适配层。
1. 检查器的核心子命令与它自己的自测：`inventory`、`import_graph`、`call_graph`、`check_defensive`、`check_loops`、`check_wrappers`、`check_authority_commits`、`check_proof`。
2. 阶段 0：`capture_golden`、容差、不变量、API 快照、基准、环境锁，以及一批真实算例。
3. 阶段 A：总纲文件、负向用例、债务表、`check_constitution`、`trace_calls`、`scc_groups`。
4. 任务卡、`check_scope`、`check_callsite`、`mark`、日志。
5. 四个 skill 和七个 command，内容指向上述检查器，不另写一套规则。
6. 然后才允许 `/refactor-next` 发出第一张业务任务卡。

缺第 1 到第 4 步时，不开始清代码。否则又会变成一边改一边靠对话记得「刚才检查过」。

## 附：暂缓事项与触发条件

这些事项评审提出过，问题成立，但 v1 不实现。

| 事项 | 评审项 | 为什么暂缓 | 何时启用 |
| --- | --- | --- | --- |
| 变异测试：删掉输入校验后测试是否会失败 | S1-7 | 成本高，且依赖负向用例先成型；v1 用「契约 `raises` 必须有真实测试」加「被改行分支覆盖 100%」先兜底 | 阶段 D 发现有校验代码删除后测试仍全绿时，先对边界函数的必填输入校验抽样启用 |
| 多 Agent 并行与符号租约 | S1-8 | 需要任务服务端；v1 明确串行，靠 `base_commit` 拒绝过期卡 | 单 Agent 串行的吞吐成为瓶颈，且已能保证同一 SCC、同一规范量、同一模块总纲的任务互斥 |
| 把检查器发布成对外的独立包并走完整发布流程 | S0-1 | 单人本地档用只读安装加提交历史检查即可满足 v1 的目标；发布流程是团队档的成本 | 换到多人协作或需要外部审计时 |
