# 《确定性重构工作流》审查意见

审查对象：`RefactorWorkflow.md`

## 1. 结论

方案已经具备一套完整工作流的骨架：先冻结行为，再确定架构总纲，随后按调用链处理，最后清残留并做全局验收。任务卡、单步窗口、状态哈希、数值 golden、棘轮指标和修订流程都比单纯增加 skill 或 hook 更可靠。

但按当前设计直接实施，还不能达到“高度确定性”。主要原因不是规则数量不够，而是下面四个基础假设不成立：

1. 检查脚本、基线、allowlist、状态和业务代码都由同一个 Agent 修改，控制面并不可信。
2. Python 的调用关系、动态入口和等价计算不能仅凭标准库 `ast` 完整识别。
3. 一函数一任务与只开放一跳，在签名变化、多调用者、模块调用环上会出现流程死锁。
4. 方案强调结构门禁，但缺少可重复的性能门禁、错误行为基线和公开 API 兼容性门禁。

因此建议把现方案定为 **v0 设计草案**，先修复本审查中的 S0、S1 项，再开始实现检查脚本。否则门禁可能表现得很严格，却仍可被绕过、误判或把错误架构冻结下来。

## 2. 值得保留的设计

以下内容方向正确，不建议推倒重来：

- 把可脚本化事实和 Agent 判断分开。
- 用外部 S4 参考实现生成数值基准，而不是只冻结 Python 当前输出。
- 用模块依赖白名单代替模糊的“低耦合”要求。
- 每个规范物理量指定唯一生产者。
- 以任务卡限定文件范围，并让状态绑定代码哈希。
- 用 `stale` 使已验收代码在变化后重新进入队列。
- 把不能自动判断的事项限制成枚举决策，并要求证据位置。
- 阶段 C 专门处理链外符号和死代码，避免只重构主路径。
- 对 `Any`、`type: ignore`、延迟 import 等存量问题使用棘轮，而不是要求一次归零。
- 把修订从普通重构任务中分离。

这些机制构成了方案的主干。问题主要集中在“谁能修改门禁”“脚本能证明什么”“任务原子性如何定义”三个方面。

## 3. 阻塞问题

### S0-1：控制面和被检查代码没有隔离

当前方案把 `scripts/refactor/`、`baseline.json`、allowlist、golden、任务状态和业务代码放在同一仓库，也允许 Agent 逐步建设这些脚本。即使有 `tests/refactor_self/`，同一个 Agent 仍可以同时放宽检查器和修改自测样本，使 CI 全绿。

哈希链不能解决这个问题。Agent 可以重写事件日志、重算哈希，或者直接修改负责校验哈希的脚本。`mark.py` 是“唯一写状态入口”只是一条约定，不是权限边界。

必须改成：

- 检查器打包为独立、版本化的工具，例如 `refactor-gate==1.4.2`，CI 按版本和制品 SHA256 安装。
- 业务重构 PR 不允许修改检查器包、CI workflow、基线生成器和已有 golden。
- 用分支保护或 `CODEOWNERS` 要求检查器、`modules.toml`、schema、allowlist、golden 的修改单独审批。
- CI 使用受保护分支上的检查器运行，而不是使用 PR 分支里同路径的脚本。
- allowlist 每项带稳定 ID、责任人、到期条件和引入提交；普通步骤只能消费，不能新增。

如果无法建立仓库权限边界，至少要让 CI 从固定 commit 下载检查器，并校验配置文件是否经过单独的 constitution/amend 提交。否则“硬编码门禁”仍然只是 Agent 可编辑的提示词。

### S0-2：`ast` 调用图不能作为完整事实来源

标准库 `ast` 可以可靠抽取语法层 import，却无法可靠解析以下 Python 调用：

- 别名、重新导出和条件 import；
- 实例方法的真实接收类型；
- 装饰器替换后的函数；
- 回调、函数参数、高阶函数和注册表；
- `Protocol`、抽象基类和多态分派；
- `getattr`、插件发现、字符串入口；
- monkey patch、依赖注入容器；
- C 扩展与 numpy 内部回调。

因此 `call_graph.py` 生成的结果不能直接用于证明“入口覆盖完整”“该符号是死代码”或“任务队列是完整拓扑序”。把解析失败写进 `dynamic_ref.toml` 也不够，因为漏掉的动态边不会自动变成解析失败。

建议使用三类证据并标注可信度：

1. `import graph`：静态硬门禁，可证明模块依赖方向。
2. `declared call graph`：函数契约显式声明调用边，静态扫描只核对声明中存在的边。
3. `observed call graph`：运行 golden、集成测试和公开入口时，用 profiler/trace 记录实际调用边。

死代码只有在“静态不可达、运行未覆盖、无公开 API/插件声明、字符串与注册表检索为空”同时成立时才可自动进入删除候选；仍需删除后跑全量测试。调用图应包含 `confidence = declared|static|observed|dynamic`，不能把所有边当成同一可信度。

### S0-3：任务状态只绑定 `ast_hash`，没有绑定依赖和契约

当前符号的函数体不变，并不代表它仍然通过验收。下面任一变化都可能让它失效：

- 被调用函数签名或异常行为改变；
- 输入/输出契约改变；
- `canonical_vars.toml` 的生产者改变；
- 模块依赖总纲改变；
- numpy、scipy、BLAS 版本改变；
- golden 或容差改变。

只比较函数 `ast_hash` 会把这些符号错误地保持为 `reviewed`。

状态证明至少要绑定：

```text
proof_hash = hash(
  primary_symbol_ast
  + direct_dependency_contract_hashes
  + task_contract_hash
  + constitution_version
  + gate_tool_sha256
  + test_fixture_hashes
  + environment_lock_hash
)
```

任一组成项变化，都要沿反向依赖图把直接消费者标成 `stale`。如果修改公开契约，还要继续传播到所有消费者，不能只看“AST 是否与契约不符”。

### S0-4：单步提交和证明哈希存在循环定义

方案写到 `mark.py` 记录 git 修订，但没有规定顺序：

1. 门禁报告在提交前生成时，报告引用的是旧 HEAD。
2. 把报告和状态提交后，HEAD 又变化。
3. 若重新生成报告，提交内容再次变化。

这会导致“报告证明哪个代码树”不明确，也使审计难以复现。

建议以 Git tree hash 而非 commit hash作为证明对象：

1. 工作区修改完成。
2. 暂存任务允许的文件。
3. 用 `git write-tree` 得到 `source_tree`。
4. 门禁对该 tree 的临时 checkout 运行，报告记录 `source_tree`。
5. bot 写入不可编辑的 CI artifact/attestation；状态文件只保存 attestation ID。
6. 最终提交允许多出状态元数据，但业务文件 tree 必须与 `source_tree` 一致。

如果不引入 bot，最简单的替代是：状态完全由 CI 根据当前 commit 重算，不把 `status.json` 视为事实来源。

### S0-5：W1 窗口无法安全处理多调用者和签名迁移

任务卡示例只列出一个上游调用方。但一个函数可能有多个直接调用者，分布在多条链、测试、脚本入口和外部 API 中。只修改一个调用点会破坏其他调用者；逐个迁移又需要临时兼容层，而方案同时禁止新增适配包装。

需要把任务分成两种：

- **实现任务**：保持公开签名不变，只清理函数内部。
- **契约迁移任务**：把一个符号、所有仓库内直接调用点、公开 API 兼容测试作为一个原子 cutover。

契约迁移卡必须列出所有可信来源发现的直接调用者，而不是“某条链上的上游”。若调用者数量超过阈值，则先做专门的 API 迁移计划，不能硬套 W1。

外部调用者不在仓库中时，必须有 `public_api_snapshot.json`，对 import 路径、签名、默认值、异常、返回 dtype/shape 做兼容性检查。需要破坏兼容性时走版本迁移，不应被普通重构任务顺便修改。

### S0-6：拓扑排序没有正确处理强连通分量

“模块内部递归声明后继续拓扑排序”不足以处理：

- 相互递归；
- 回调形成的环；
- 状态机节点互调；
- cache/provider 与计算函数的内部环。

任何有向图都应先压缩强连通分量（SCC），再对凝聚图拓扑排序。SCC 中多个函数必须作为一个原子组，或先做专门的破环任务。这个分组应由脚本根据图生成候选，constitution 审批，不要求 Agent 事先手工猜全。

阶段 A 对现有非法模块边也可能形成死锁：总纲要求边不存在，但修边所需任务又因阶段 A 未通过无法开始。应增加 `architecture_debt.toml`：

- 记录当前非法边；
- 每条边指定消除任务；
- 阶段 B 只允许数量下降；
- 新非法边始终失败；
- 阶段 D 必须归零。

### S0-7：数值正确性门禁不充分且部分阈值不可执行

`deviation_growth <= 0` 对浮点计算过严，也不等于正确。更换循环顺序、BLAS、线程数或平台就可能出现最后几位变化。修复某个算法也可能让一个样例误差略升、总体误差显著下降，不能要求逐样例 Pareto 改善。

应改为：

- 固定 Python、numpy/scipy、BLAS、线程数、随机种子和 CPU 架构范围。
- 每个输出定义 `atol`、`rtol`、零值附近规则、NaN/Inf 规则；必要时使用 ULP。
- 矩阵本征向量按相位、符号、排序和简并子空间不变量比较，不能直接逐元素比较。
- S4 类问题增加守恒和互易等不变量：能量守恒、层复制等价、参考面平移关系、均匀层解析结果。
- golden 清单记录输入覆盖维度，至少含均匀/图案、1D/2D、实/复介电常数、各 FMM 分支、退化/近奇异情况、缓存失效。
- baseline 不只保存输出，还保存参考实现版本、构建参数和环境锁。

此外，P2 的目标是“缺少输入必须报错”，但阶段 0 只强调成功算例。必须冻结负向行为：缺键、错误 shape、错误 dtype、无效材料引用、非法层次结构分别抛什么异常。否则重构后虽然数值相同，静默默认仍可能存在。

### S0-8：没有性能验收，无法证明 P5 已解决

AST 检测双层 `range` 只能发现一种写法：

- `np.vectorize` 仍是 Python 循环，可能漏报；
- `DataFrame.apply`、列表推导、生成器嵌套也可能很慢；
- 向量化可能制造巨型临时数组，时间变快但内存爆炸；
- 一个有界小循环可能比复杂广播更清楚、更快，却被误判。

既然目标明确包含低效 Python 写法，必须增加 benchmark：

- 为 RCWA/FMM 数值热点建立固定规模的小、中、大三组基准。
- 预热后多次运行，记录中位数和 MAD，而不是单次时间。
- 固定线程数，分开记录 wall time、峰值 RSS 和临时数组规模。
- 普通任务不得比基线慢超过预设噪声带；标记为 performance 的任务必须达到明确改进。
- 性能门禁运行在固定 runner；普通开发机只出报告，不做硬失败。

`check_loops.py` 可以保留为候选发现器，但不应独自证明性能问题已经消除。

## 4. 高优先级问题

### S1-1：用 class/def 数量和注释净增作为硬门禁会诱导错误设计

“class 不增、def 不净增、注释不净增”容易被游戏化：

- Agent 会把逻辑塞进大函数以避免增加 helper。
- 一个合理的值对象或策略对象会被误杀。
- 删除一个无关函数即可换取新增一个包装函数。
- 为复杂数学公式增加必要说明也会失败。

过度封装的本质不是定义数量，而是没有独立职责、没有第二个调用者、没有替换边界价值的透传层。建议：

- `check_wrappers.py` 对确定的纯透传保持硬失败。
- class/def 净增只作为审查信号，不直接失败。
- 新增抽象时必须在任务卡写枚举理由：`shared_invariant`、`public_boundary`、`backend_strategy`、`test_seam`。
- 禁止理由 `future_use`。
- 注释只禁用重复代码的叙述性注释；数学来源、单位、数值稳定性和不变量说明允许增加。

### S1-2：`.get` 检查应推动“边界解析后不再传 dict”

按变量名判断 `.get` 是否对应必填配置容易误判。更确定的设计是：

1. 公开 API 边界一次性解析 mapping。
2. 必填键使用 `config["key"]` 或显式 schema 验证。
3. 可选默认只在这个边界应用。
4. 核心调用链接收带类型的不可变配置对象或显式参数，不再接收原始 dict。

门禁可以硬编码为：core/fmm/rcwa/pattern 模块的公开函数参数不允许 `Mapping[str, Any]` 或裸 `dict`；这些模块中不允许读取配置字符串键。这样比逐个识别 `.get` 更可靠，也不会误伤普通缓存字典的 `.get`。

不建议要求每个函数的 `raises` 必须非空。纯数值函数在已验证输入上可能没有预期业务异常。正确门禁是：有前置条件的边界必须声明并测试异常；任何函数都不得吞异常或把失败静默转成默认值。

同理，`None` 有时是合法领域值，不能全局禁止。契约应区分 `optional_value` 和 `failure_sentinel`。

### S1-3：契约目前是文本，容易形式通过

`shape = "(2n, 2n)"` 没有定义 `n` 从哪里来。检查“函数前部引用 shape”可以用无效断言轻易通过，例如只访问一次 `.shape`。

建议把契约做成可执行 schema：

- 每个维度引用输入字段，例如 `n = len(G)`。
- 前置条件写成受限表达式，例如 `Epsilon2.shape == [2*n, 2*n]`。
- 测试包装器在函数入口和出口实际执行表达式。
- `dtype` 明确是否允许安全转换。
- `mutates` 用运行时只读副本或 array hash 验证，而不是只扫赋值左侧。
- 数据语义使用稳定的 `canonical_id`，不能依赖 Python 局部变量名。

生产环境可关闭昂贵检查，但 CI 夹具必须执行。

### S1-4：重复计算与重复实现不能只靠 AST 和变量名证明

相同 AST 可能是两个合理的简单公式；同一公式也能写成完全不同的 AST。`check_dupes.py` 适合生成候选，不适合把每个哈希簇强制合成一个函数。

规范量的唯一生产者也不能只查变量赋值名。代码可以换个变量名重算，或者只重算公式的一部分。

建议：

- `check_dupes.py` 只产生审查列表，结论是 `merge` 或 `independent`，两者都要记录。
- 规范量通过带类型的数据对象或返回字段传播，例如 `LayerModes.q`、`EpsilonMatrices.inverse`，检查构造点唯一。
- 性能 profiler 记录热点函数调用次数，发现同一请求中的重复计算。
- 缓存测试统计生产者调用次数，而不是只核对 `invalidates` 文本列表。

### S1-5：底向上的实现顺序可能过早冻结错误契约

从 numeric、pattern 开始逐层向上实现，有利于依赖稳定，但业务目标在 api/orchestration。若没有先做完整的顶向下契约发现，底层可能按照旧 C 数据布局被“正确地”冻结，随后上层只能继续适配它。

建议把阶段 A 拆成：

1. **顶向下发现**：按用户业务场景定义端到端输入、输出、不变量和模块职责，不改实现。
2. **底向上实施**：按依赖序改代码。

每条业务链先有一条端到端 characterization test，再允许处理叶子函数。这样既保留依赖序，又避免低层局部最优。

### S1-6：公开 API 与序列化兼容性没有独立门禁

`public_api` 当前只检查符号存在，不能防止：

- import 路径改变；
- 参数位置、名称或默认值改变；
- 异常类型改变；
- 返回对象类型、dtype、shape 改变；
- pickle/JSON 配置不兼容。

应在阶段 0 生成 API snapshot，并在阶段 D 比较。允许变化时必须有迁移说明、弃用周期和兼容测试。

### S1-7：测试覆盖指标缺少“边是否真正执行”

数值夹具通过不代表任务声明的调用链真的被覆盖。建议每个任务报告：

- primary symbol 的 branch coverage；
- 任务契约中每个异常路径的负向测试；
- 声明调用边是否至少观察到一次；
- 对输入验证代码使用 mutation test，确认删掉检查后测试会失败。

不建议设一个全仓统一的行覆盖率数字。门禁应针对本任务新增/修改分支，以及关键错误处理。

### S1-8：并发、多 Agent 和合并后的失效没有定义

事件日志只防止同一工作区同时开两张任务卡，不能防止两个 Agent 在不同分支各自拿到过期卡片。合并后调用图、契约和 proof hash 都可能变化。

如果允许并行：

- 任务服务端对 symbol ID 加租约。
- 任务卡记录 base commit 和 constitution version。
- 提交前检查 base 是否仍包含在目标分支。
- 合并队列在最新目标分支上重跑 `gate --all`。
- 修改同一模块总纲、同一规范量或相邻 SCC 的任务不可并行。

如果不打算支持并行，应明确工作流是单 Agent 串行，不要让任务卡机制暗示可安全并发。

## 5. 中优先级问题

### S2-1：“全部变量清单”范围应缩小

Python 局部临时变量、闭包单元、动态属性、数组视图数量巨大且不稳定。把它们全部写进 inventory 会制造噪声。

建议清单只覆盖：

- 模块、类、函数；
- 公开常量和模块级可变状态；
- dataclass/TypedDict/配置字段；
- 契约中的 canonical data；
- 缓存和跨函数传播的状态。

普通局部变量通过 AST 临时分析，不作为长期状态项。

### S2-2：模块“高内聚”不能从 import 白名单直接推出

无环和单向依赖能证明部分低耦合，但不能证明模块内高内聚。`utils` 扇入阈值也可能误伤真正稳定的底层数学模块。

建议将以下指标作为审查证据而非硬阈值：

- 模块内调用边占比；
- 模块跨边数量；
- public API 面积；
- 变更共同发生率；
- 模块职责描述中的术语是否集中。

最终模块边界仍由 constitution 决定，脚本只检查决定是否被遵守。

### S2-3：环境可复现性应进入正式工件

应增加：

- `environment.lock` 或 lockfile hash；
- Python、numpy、scipy、BLAS/LAPACK、CPU/thread 配置；
- locale、时区、随机种子；
- golden 生成器的编译版本。

这些信息需要进入 proof hash 和数值报告。

### S2-4：报告和生成物可能让 PR 难以审查

每步提交大量 `reports/*.json`、inventory 和状态变更，会淹没业务 diff。建议：

- 详细报告作为 CI artifact，不提交仓库。
- 仓库只提交小型 manifest：任务 ID、tree hash、工具 hash、报告 artifact ID。
- inventory 在 CI 重生成并比较；只有架构权威文件和人工决策需要版本控制。

### S2-5：删除测试的规则需要更保守

“只有测试调用就删除生产代码与只为它存在的测试”可能删掉为兼容性、调试或未来插件保留的公开接口。应先核对 API snapshot、文档和发布历史。测试本身是唯一使用者并不能证明代码无价值。

## 6. 建议的修订版结构

建议把方案收敛成五个平面。

### 6.1 受保护控制面

- 固定版本的 gate 工具。
- CI、constitution、schema、allowlist、golden 的独立审批。
- 普通 Agent 不能修改。

### 6.2 行为基线面

- 外部 C 参考输出。
- 负向输入行为。
- 数值不变量。
- 公开 API snapshot。
- 固定 runner 上的性能和内存基线。

### 6.3 架构事实面

- import 图是硬事实。
- 调用图分 declared/static/observed/dynamic 四种可信度。
- 图先压缩 SCC，再生成任务。
- 现存非法边进入只减不增的架构债务表。

### 6.4 任务执行面

- 实现任务保持签名。
- 契约迁移任务原子更新所有调用者。
- 当前函数文件内只允许当前 AST 节点、必要 import 和指定调用点变化，不能只检查文件路径。
- proof hash 绑定实现、契约、依赖、工具和环境。

### 6.5 验收面

- 结构门禁。
- 成功与失败行为测试。
- 数值 golden 与物理不变量。
- API 兼容性。
- 性能与峰值内存。
- 最新目标分支上的全量重跑。

## 7. 建议调整后的阶段退出条件

### 阶段 0

- 固定工具版本和环境。
- 生成成功用例、失败用例、API snapshot、数值不变量和性能基线。
- 检查器自测由受保护工具包提供。

### 阶段 A

- 模块依赖总纲通过。
- 动态入口显式登记。
- 调用边带可信度。
- SCC 已分组。
- 每条业务链先有端到端测试。
- 非法现存依赖全部进入有负责人和消除任务的债务表。

### 阶段 B

- 普通实现任务不改签名。
- 签名变化只能走契约迁移任务。
- 每步 proof hash 可复算。
- 相关数值、异常、调用边和性能夹具通过。

### 阶段 C

- 死代码需多证据确认。
- 重复 AST 只作为候选，不自动合并。
- 链外可达代码必须补业务链或明确声明为运维/插件入口。

### 阶段 D

- 在最新目标分支和固定 runner 上重跑。
- 架构债务归零。
- 无 stale proof。
- API、数值、错误行为、性能和内存全部通过。
- 详细报告来自受保护 CI artifact。

## 8. 对原方案逐项判定

| 原设计 | 判定 | 调整 |
| --- | --- | --- |
| golden 优先于结构调整 | 保留 | 增加负向行为、API、不变量、性能 |
| import 白名单 | 保留为硬门禁 | 现存违规进入债务表 |
| 调用图生成任务 | 有条件保留 | 多证据、可信度、SCC |
| W1 一跳窗口 | 保留为默认 | 签名迁移改成全调用点原子任务 |
| `ast_hash` 状态 | 不足 | 改 proof hash |
| 事件哈希链 | 不能建立信任 | 由受保护 CI attestation 替代 |
| `.get` AST 扫描 | 保留为辅助 | 核心禁止裸 dict，边界 schema 验证 |
| class/def 不净增 | 降级为信号 | 纯透传仍硬失败 |
| 注释不净增 | 删除 | 只禁止低价值叙述注释 |
| 嵌套循环扫描 | 保留为候选发现 | 加固定 runner benchmark |
| 重复 AST 聚类 | 降级为候选 | 人工枚举 merge/independent |
| 每函数 `raises` 非空 | 删除 | 只要求有前置条件的边界声明异常 |
| 变量唯一生产者 | 保留 | 用 canonical data 构造点和运行计数验证 |
| 每步报告进仓库 | 调整 | 详细报告放 CI artifact |
| `deviation_growth <= 0` | 删除 | 使用固定容差和物理不变量 |

## 9. 建议实施顺序

在修改 `RefactorWorkflow.md` 前，先验证三个原型。原型失败意味着总体设计需要继续调整，不应开始全仓重构。

1. **可信门禁原型**：从固定 commit 安装一个最小 gate 工具，证明业务 PR 修改本地同名脚本不能影响 CI。
2. **调用图原型**：选一条含方法调用、回调和一个动态入口的真实业务链，对比静态图、运行图和人工图，量化漏边。
3. **任务原子性原型**：选择一个有三个调用者的函数，分别演练保持签名的实现任务和改变签名的迁移任务，确认 W1 与 proof hash 能闭环。

三个原型通过后，再实现 golden、constitution 和完整门禁。这个顺序比一次写完二十多个检查脚本风险更低，因为它先验证方案最关键的信任边界和任务模型。

## 10. 最终意见

方案应继续推进，但不建议按当前文本直接落地。它已经解决了“不要只依赖 skill 和 hook”的方向问题，却尚未解决“谁约束检查器本身”和“Python 动态行为怎样形成可证明覆盖”这两个根问题。

完成以下最低修改后，可以进入实现：

1. 检查器与业务仓库形成受保护信任边界。
2. `ast_hash` 升级为包含契约、依赖、工具和环境的 proof hash。
3. 调用图改成多证据和可信度模型，并用 SCC 生成任务。
4. 区分保持签名的实现任务与全调用点契约迁移任务。
5. 数值门禁增加负向行为、物理不变量、API 兼容性和固定环境。
6. 增加可重复的性能与峰值内存基准。
7. 把 class/def 数量、注释数量、重复 AST 从硬门禁降为审查信号。

做到这些后，工作流才从“规则非常详尽”变成“关键结论可复算、控制面不可由被检查者篡改、失败模式有明确退出路径”的确定性机制。

---

# 第二轮审查：v1 修订版

审查对象：`RefactorWorkflow.md` 的 v1（提交 `089e164`）

说明：本轮内容只追加在第一轮意见之后，不修改第一轮审查原文。编号使用 `V1-S0-*`、`V1-S1-*`、`V1-S2-*`，避免与第一轮编号混淆。

## 11. 第二轮结论

v1 对第一轮的 21 项意见逐项给出处置，并实质性修复了多数核心问题：

- 控制面与业务面已分离。
- `ast_hash` 已升级为 `proof_hash`。
- 调用边已有可信度模型。
- 队列已考虑强连通分量。
- 普通实现和签名迁移已拆成不同任务。
- 数值、API、负向行为、物理不变量、性能和内存都进入门禁。
- 原先容易诱导错误设计的 class/def/注释计数门槛已经取消。

因此，v1 已从“概念方案”进展到“可以做原型验证的规格”。但仍不宜直接铺开完整重构。当前有六项规格级阻塞问题：依赖证明仍不完整、全仓 `source_digest` 的生命周期不清、棘轮仍按数量而非问题身份、三处硬规则相互矛盾、调用图证据与 SCC 输入没有区分“确定边”和“猜测边”、最终审计制品缺少长期保存规则。

建议先修完本轮 S0 项，再执行文档中的三个原型。S1 项可以在原型阶段一并验证；S2 项可在完整门禁实现前补齐。

## 12. 上一轮意见的落实评估

| 第一轮意见 | v1 落实情况 | 第二轮判定 |
| --- | --- | --- |
| S0-1 控制面隔离 | 增加团队 CI 档与单人本地档，检查器移出工作区 | 基本解决；本地档明确不防主动绕过，边界诚实 |
| S0-2 动态调用图 | 增加 static/declared/observed/dynamic 四级证据 | 方向正确；SCC 和任务排序如何消费这些证据仍需修订 |
| S0-3 状态只绑 AST | 改为 `proof_hash` | 部分解决；仍未绑定直接依赖的实现证明和全部权威文件 |
| S0-4 提交与证明循环 | 增加排除证明目录的 `source_digest` | 避开自引用；但全仓摘要会随任意后续任务变化，语义尚不清 |
| S0-5 多调用者迁移 | 增加 `implement` 与 `migrate` | 解决 |
| S0-6 SCC 和现存非法边 | 增加 SCC、`architecture_debt.toml` | 基本解决；SCC 使用的边集合仍可能过度近似 |
| S0-7 数值门禁 | 增加容差、不变量、等价比较、环境锁、负向用例 | 解决；本征问题比较规则仍需通过真实算例原型验证 |
| S0-8 性能门禁 | 增加固定 runner、中位数、MAD、峰值内存 | 方向正确；测量工具和重试规则未定义 |
| S1-1 抽象计数 | 改为透传硬失败、抽象理由枚举 | 大体解决；单方法类规则仍与策略对象矛盾 |
| S1-2 裸 dict 与异常 | 核心边界改为带类型配置；`None` 分类 | 基本解决；`new_optional` 的修订语义仍自相矛盾 |
| S1-3 可执行契约 | 增加测试期运行时代理 | 方向正确；代理覆盖的对象类型和受限表达式执行器未定义 |
| S1-4 重复与唯一生产者 | 重复 AST 降级为候选；增加载体构造点与调用次数 | 部分解决；“载体只能由生产者构造”过于严格 |
| S1-5 顶向下发现 | 阶段 A 增加端到端链发现 | 解决 |
| S1-6 API 兼容 | 增加 API 快照 | 基本解决；导入副作用和参数相关 shape 需补规格 |
| S1-7 覆盖 | 增加 diff 分支覆盖和边观察 | 方向正确；100% 分支覆盖需豁免模型 |
| S1-8 并行 | v1 明确串行，任务卡带 base commit | 在 v1 范围内解决 |
| S2-1 清单范围 | 缩小为跨函数状态 | 解决 |
| S2-2 高内聚 | 增加 responsibility，指标降级为报告 | 解决 |
| S2-3 环境复现 | 增加 `env.lock` | 解决 |
| S2-4 报告体积 | 完整报告放 CI artifact | 解决短期体积问题；长期审计保存仍缺规则 |
| S2-5 删除测试 | 增加 API、文档和发布历史核对 | 解决 |

## 13. 第二轮阻塞问题

### V1-S0-1：`proof_hash` 仍没有绑定直接依赖的实现证明

v1 的 `proof_hash` 包含“全部直接依赖的契约哈希”，但不包含依赖函数的实现哈希或 `proof_hash`。

例如：

1. `pattern.fourier` 已验收。
2. `fmm.closed.epsilon` 依赖它并完成验收。
3. 后续任务修改 `pattern.fourier` 的实现，但保持契约文本不变。
4. `fmm.closed.epsilon` 自身 AST 和依赖契约都没变，所以其 `proof_hash` 仍一致，状态继续是 `reviewed`。

这与 v1 所说“任一组成项变化时沿反向依赖图传播 stale”并不冲突，因为被调用方实现根本不在调用方的组成项里。最终 `gate --all` 可能通过端到端测试发现问题，但中间状态已经把一个受影响调用者错误地标为有效，任务队列也不会重验它。

必须改为以下二选一：

**方案 A：递归依赖证明。**

```text
proof_hash(symbol) = hash(
  symbol_ast
  + symbol_contract
  + authority_digest
  + fixture_digest
  + environment_digest
  + sorted(direct_dependency_proof_hashes)
)
```

被调用方的实现一变，其 proof 变化，直接调用者自动 stale，再向上传播。依赖图已压缩 SCC，因此组内函数共用一个 group proof，不会递归死循环。

**方案 B：结构证明和行为证明分开。**

- `structure_proof` 只绑定当前符号及契约。
- `behavior_proof` 绑定任务夹具运行时实际经过的全部符号实现哈希。
- 状态只有两者都有效时才是 `reviewed`。

方案 B 精确度更高，但实现复杂。v1 追求确定性，建议先用方案 A；失效范围大是可接受的保守代价。

此外，当前 proof 只写了 `constitution_version` 和“该符号在 canonical_vars 中的登记项”，没有自动绑定以下权威文件：

- `modules.toml`
- `options_schema.toml`
- `chains.json`
- `architecture_debt.toml`
- `allowlists/*`
- `tolerances.toml`
- `invariants.toml`
- `known_deviations.json`
- `negative_cases.toml`
- `api_snapshot.json`
- `bench/baseline.json`

手工维护 `constitution_version` 会漏增。应计算 `authority_digest = hash(全部适用权威文件内容)`，直接进入 proof，不依赖人工改版本号。

### V1-S0-2：全仓 `source_digest` 会在每个后续任务后失配

v1 将 `source_digest` 定义为“除 `refactor/state/` 之外全部受版本控制文件的摘要”。它避开了证明清单自引用，但带来另一个问题：

- 任务 A 完成时，proof A 保存全仓摘要 T1。
- 任务 B 修改另一模块后，全仓摘要变成 T2。
- proof A 的 `source_digest` 必然不等于当前树。

文档一处说状态由 `proof_hash` 重算决定，另一处又说 proof 清单中的 `source_digest` 用于证明代码树，审计需要报告哈希。没有明确规定旧 proof 的 `source_digest` 是否必须匹配当前树。

若必须匹配，则每做一步都会让全部旧 proof 失效，流程无法收敛。若不匹配也没关系，则这个字段只是“当时运行在哪棵树上”的历史信息，不能证明当前状态。

建议拆成两个字段并明确用途：

- `run_tree_digest`：门禁运行时的全仓摘要，只用于追溯，不参与当前状态判定。
- `proof_scope_digest`：当前任务可编辑符号、直接依赖 proof、契约、夹具和权威文件的摘要，参与当前状态判定。

CI artifact 记录 `run_tree_digest`；`status` 只重算 `proof_scope_digest`。不要让同一个 `source_digest` 同时承担历史追溯和当前有效性两种相反职责。

### V1-S0-3：多个“棘轮”仍按数量判断，可以被等量替换绕过

v1 仍使用：

- `type_ignore_count ≤ 基线`
- `any_count ≤ 基线`
- `architecture_debt` 数量不得高于上一个权威提交
- 反模式总计不高于 baseline

数量棘轮允许：

- 删除 A 文件的一个 `type: ignore`，在 B 文件新增一个，数量不变。
- 删除一个已知依赖债务，新增另一条更严重的逆向依赖，数量不变。
- 删除一个 `.get(default)`，在当前任务外新增另一个，数量不变。

因此“只允许下降”必须按**问题身份集合**而不是计数判定：

```text
current_issue_ids ⊆ baseline_issue_ids - resolved_issue_ids
```

每个 issue ID 使用稳定字段生成，例如：

```text
hash(rule_id + module + qualname + normalized_ast_path)
```

行号不能单独作为 ID，否则格式化会把所有问题变成“删除旧问题、新增新问题”。允许同一问题在符号内部小范围移动时，采用 qualname 加归一化 AST 路径；跨符号移动则视为新问题并失败。

`architecture_debt.toml` 还应要求：

- 旧 debt ID 只能保留或删除，不能替换。
- 允许更新的字段只有撤除任务和说明，不能换边的端点。
- 严重度不能降低，除非走修订并给证据。

指标表可以继续显示数量，但门禁必须比较集合。

### V1-S0-4：三处硬规则存在内部矛盾

#### 1. 单方法类与合法策略对象冲突

v1 表示“纯透传仍硬失败”，并允许新增抽象理由 `backend_strategy`。但 `check_wrappers` 仍定义为：

> 函数体只有一条 return 调用，**或类除 `__init__` 外只有一个方法**；不在 `public_api` 即失败。

一个合法的 FFT backend、线性求解 backend、策略对象往往恰好只有一个公开方法，也通常不是顶层 `public_api`。它会被无条件判失败，与 `backend_strategy` 理由冲突。

应改成：

- 函数只有一条无转换的转发调用：硬失败，除非 public boundary。
- 类只有一个方法：只报候选。
- 仅当这个方法也是纯委托、类没有自有不变量/资源生命周期、且未登记 `backend_strategy` / `test_seam` 时才失败。

#### 2. `new_optional` 与“缺键仍抛错”冲突

第 11 节仍写：

> `new_optional` 要求默认值写在 schema 里，且调用点在缺键时仍会在公开 API 边界抛错。

如果缺键必须抛错，它就是 required，不是 optional。optional 的正确行为应该是：

- API 边界缺键时应用 schema 中唯一登记的默认值。
- 转换成带类型配置后，核心链只看到一个已经存在的字段，不再处理“缺键”。
- 若公开 API 为保持兼容必须对缺键报错，则不能使用 `new_optional`，而应保持 required。

#### 3. 输出按名字匹配与 `canonical_id` 冲突

阶段 B 仍要求“输出名字与下游契约的输入名字能对上”。但 v1 已引入 `canonical_id`，局部参数改名不应破坏契约。

跨函数数据边应按下面三项匹配：

- `canonical_id`
- dtype / shape / unit
- producer / consumer symbol

局部 Python 名称只用于可读性，不应成为硬门禁。

这些不是文字小问题。若不先统一，检查器实现者会在两个相互冲突的规则中任选一个，确定性反而下降。

### V1-S0-5：SCC 使用的并集图混合了确定边和近似边

v1 用 `static ∪ declared ∪ observed` 做 SCC 和任务队列。这比单一 AST 好，但三种边不具备同样的含义：

- static 方法调用可能因类型解析不准而指向错误符号。
- observed 只证明“这次运行发生了”，不能证明其他合法运行没有边。
- declared 是设计声明，可能还没有被某个夹具触发。
- dynamic 虽然单独登记，却没有进入 SCC 并集；它仍可能形成真实调用环。

后果有两种：

1. 一个误解析的 static 边可能把大量函数压成一个巨大 SCC，失去小任务窗口。
2. 一个未观察到的 dynamic 回调环可能不在 SCC 里，队列仍按错误 DAG 排序。

需要增加一张专用于任务排序的 `dependency_graph.json`，不要直接使用原始调用边并集：

- 边必须有唯一、已解析的 symbol ID。
- static 方法边无法唯一解析时标 `ambiguous`，不得直接进入 SCC；它阻断相关队列，直到 constitution 决策或用 observed/declared 消歧。
- dynamic 调用一旦确认目标，也要进入 dependency graph。
- declared 边允许用 `static`、`observed` 或已批准的 `dynamic` 证据印证。不能强制只有 static/observed，因为错误分支、插件入口和延迟回调可能只能用 dynamic 证据。
- 原始 call graph 用于发现；dependency graph 才用于 SCC、顺序和 stale 传播。

运行时跟踪还要写清：

- 新线程使用 `threading.setprofile` 或等价机制。
- 子进程单独写 trace 后合并。
- async task 沿同一线程记录，但要保留 task/scenario ID。
- C 扩展只记录 Python 边界，不声称看见内部调用。

否则 observed 的含义会随测试运行方式变化。

### V1-S0-6：CI artifact 会过期，最终审计证据不持久

v1 把完整报告放 CI artifact，仓库只保存 artifact 引用和哈希。这解决了 PR 噪声，但多数 CI artifact 有保存期限。几个月后：

- `audit.md` 的 URI 可能失效。
- 无法下载原报告并核对 SHA256。
- 最终“所有门禁通过”的证据只剩一串不可验证的哈希。

建议分层保存：

- 每步详细报告：短期 CI artifact，可以过期。
- 阶段 D 最终报告：压缩为稳定的 `final-attestation.json`，包括工具哈希、环境哈希、authority digest、代码 commit、所有门禁摘要、详细制品 SHA256。
- `final-attestation.json` 存在仓库 release、不可变制品库或长期对象存储；团队档使用 CI 身份签名。
- 仓库里的 `audit.md` 引用长期地址，不引用普通流水线临时 artifact。

单人本地档至少把最终 attestation 和报告压缩包的 SHA256 放进 release，不能只留本机路径。

## 14. 第二轮高优先级问题

### V1-S1-1：规范量载体“只能在生产者内构造”过于严格

测试夹具、反序列化、clone、缓存恢复、类型转换都可能合理构造 `EpsilonMatrices` 或 `LayerModes`。把类构造点限制为唯一生产者会诱导所有地方绕到一个全局工厂，增加耦合。

应限制的是**生产语义**而不是 Python 构造语法：

- 生产调用链里，某个 `canonical_id` 只有一个 owner/producer。
- 测试工厂、反序列化器和 clone 显式标角色，不算重新计算。
- 从已有 canonical data 复制或恢复要写 `derivation = copy|deserialize|cache_restore`。
- 新计算写 `derivation = compute`，只有 owner 可以使用。

这可以通过载体元数据或构造工厂的枚举入口实现，但不应全局禁止类构造。

### V1-S1-2：运行时契约代理只覆盖 numpy 数组变异

当前设计用数组哈希检查 `mutates`，但函数可能修改：

- list / dict；
- dataclass 字段；
- numpy view 的 base；
- 稀疏矩阵内部数组；
- 自定义缓存对象；
- memmap 或设备数组。

建议契约明确每种输入的 `mutation_policy`：

| 策略 | 检查 |
| --- | --- |
| `immutable_scalar` | 不检查变异 |
| `numpy_readonly` | 测试时设置 `writeable=False`，优先于调用前后全量哈希 |
| `numpy_snapshot` | 对允许 view/底层库写入的情况比较内容摘要 |
| `object_snapshot` | 用稳定序列化或字段级 snapshot |
| `mutable_declared` | 只允许契约列出的字段变化 |

形状受限表达式不能直接用 Python `eval`。检查器需要一个明确语法（整数、名称、`+ - * //`、`len`）和解释器；出现属性调用、下标读取、函数调用就拒绝。

### V1-S1-3：API 快照生成需要隔离导入副作用

用 `inspect` 生成 API 快照会 import 未审查仓库。模块可能在导入时：

- 读环境变量；
- 初始化线程池；
- 加载动态库；
- 访问 GPU；
- 注册插件；
- 写缓存文件。

应在受控子进程中生成快照，固定环境变量、工作目录、网络策略和超时。导入失败本身是阶段 0 的阻塞错误，不能静默跳过。

返回 dtype/shape 往往依赖输入，不能作为一个函数级静态字段快照。应放进按 `case_id` 区分的 API characterization：

```text
api_case = import_path + signature + input_case_id
           -> return_type + dtype + shape + exception
```

静态 API snapshot 只保存导入路径和签名，运行行为由 case 记录。

### V1-S1-4：分支覆盖 100% 缺少受控豁免

“被改行分支覆盖率 100%”目标清晰，但有些分支只在：

- 特定操作系统；
- 可选 LAPACK/FFTW backend；
- 内存分配失败；
- 不可构造的第三方库错误；
- `TYPE_CHECKING`

下触发。没有豁免时，Agent 可能删除必要防御分支或伪造测试。

建议豁免枚举：

- `platform_only`
- `optional_backend`
- `uninjectable_external_failure`
- `type_checking_only`

每项必须引用分支 AST、对应运行矩阵或第三方错误说明。普通业务输入分支和 P2/P6 相关错误分支不得豁免。阶段 D 在 CI 矩阵合并 coverage 后再计算，避免 Linux job 单独误报 Windows 分支。

### V1-S1-5：性能门槛缺少测量器和重试协议

v1 有中位数、MAD、10% 和峰值内存阈值，但还需定义：

- 使用 `pyperf`、`pytest-benchmark` 还是自研 runner；
- CPU governor、亲和性、turbo、后台负载；
- BLAS 线程数；
- 峰值内存是 Python allocator、RSS 还是 native heap；
- 第一次失败是否允许自动复测；
- 两次结果冲突时如何裁决。

建议固定：

- wall time 用 `pyperf` 的校准和进程隔离。
- native 峰值 RSS 用操作系统指标；Python 分配用 `tracemalloc` 只作附加报告。
- BLAS/OpenMP 线程数写入 `env.lock`。
- 首次越界自动重跑一次完整基准；两次任一通过不能算通过，以两次合并样本重新统计。
- 固定 runner 升级硬件或镜像时必须走 `tool_upgrade`/基线修订，不能沿用旧基线。

### V1-S1-6：`check_authority_commits` 在 squash/merge 流程下语义不稳定

v1 遍历 `base..HEAD`，要求权威文件和业务文件不在同一提交。这在 PR squash merge 后会合成一个提交，目标分支上的历史不再满足规则。

应明确检查范围：

- PR 阶段检查 PR 内原始 commits；CI 平台提供 commit 列表。
- 合并后的目标分支不再按 commit 粒度重查，而是核对已批准的修订 attestation。
- 如果团队强制 squash，则权威文件修订必须单独 PR，不能只要求“同一 PR 的不同 commit”。
- 单人本地档若依赖 commit 粒度，就不得 squash 这些提交，或在 squash 前生成长期修订证明。

否则同一套规则在 PR 中通过、合并后又失败。

## 15. 第二轮中优先级问题

### V1-S2-1：`check_scope` 需要定义格式化和 import 重排

任务只允许当前 AST 节点和必要 import 变化。运行 formatter 或 import sorter 时可能改完整文件，造成无关 diff。应规定：

- 任务开始前先跑一次固定版本 formatter，保证基线已规范化。
- 任务中只允许同一固定版本。
- scope 检查用 AST 节点和语义 import 集合，不用行号。
- formatter 引起的纯空白变化可以忽略，但不能借此改变其他字符串、注释或语句顺序。

### V1-S2-2：动态证据和 allowlist 也应进入 proof

`dynamic_ref.toml` 已在 authority 文件中，但 `observed_edges.json` 是生成物。若测试场景变化导致 observed 边变化，相关任务顺序和覆盖结论也会变化。

proof 应绑定当前符号的：

- resolved dependency edges；
- 每条边的证据类型；
- 覆盖这条边的 scenario IDs。

不必绑定整个 `observed_edges.json`，否则任何无关边变化都会让全库 stale。

### V1-S2-3：阶段 D 要区分“零债务”和“合法例外”

`architecture_debt.toml` 阶段 D 必须为空是合理的；合法的 dynamic entry、延迟 import、循环豁免不属于债务，应只存在 allowlist。文档已经隐含区分，但建议明确：

- debt 是目标架构违反项，终局必须为零。
- allowlist 是目标架构允许、但规则扫描会误报的例外，可以非零。
- allowlist 每项也要有 evidence 和复审条件，但不要求阶段 D 清空。

否则实施者可能为了“债务清零”把真实违规移进 allowlist。

### V1-S2-4：`api_break_approved` 需要迁移期而不只是批准

批准 API 破坏只能说明变化有意，不能保护外部调用者。建议修订记录还必须写：

- 旧 API；
- 新 API；
- deprecation 期或明确的 major version；
- 迁移示例；
- 兼容 shim 的撤除任务（如使用）；
- 受影响的文档和示例。

## 16. 第二轮建议修改清单

### 在三个原型之前必须修改

1. `proof_hash` 加入直接依赖 proof 和自动计算的 `authority_digest`。
2. 把 `source_digest` 拆成历史 `run_tree_digest` 与当前 `proof_scope_digest`。
3. 所有棘轮改成 issue ID 集合包含关系，而不是数量关系。
4. 修正单方法类、`new_optional`、输出名称匹配这三处规则冲突。
5. 从原始调用图中分离 `dependency_graph.json`，只有已解析目标的边进入 SCC；dynamic 确认边也进入。
6. 定义最终 attestation 的长期保存位置和签名/哈希规则。

### 在完整门禁实现之前修改

7. 把 canonical data 的“唯一构造者”改成“唯一 compute owner”，给 copy/deserialize/cache restore 明确角色。
8. 为运行时契约定义 mutation policy 和安全的受限表达式解释器。
9. API 快照在隔离子进程生成，返回行为改成 scenario 级记录。
10. 给 diff coverage 增加严格枚举的豁免和 CI 矩阵合并规则。
11. 固定性能测量器、峰值内存指标和失败复测协议。
12. 明确 squash/merge 下权威提交检查的行为。

## 17. 第二轮最终意见

v1 对第一轮意见的响应是有效的，不是只加文字：任务模型、信任模型、证据模型和验收范围都有实质变化。尤其是 `implement` / `migrate` 分离、SCC、负向用例不冻结错误行为、API/性能门禁，都是正确修正。

但 v1 目前仍有“规则写得更强，规则之间却未完全闭合”的问题。最关键的是：

- 调用方 proof 不随被调用方实现变化；
- 全仓摘要既像历史记录又像当前有效性证明；
- 数量棘轮仍可被等量替换绕过；
- wrapper、optional、数据匹配各有一处语义冲突；
- SCC 的输入图没有把确定边与近似边分开。

修复第 16 节前六项后，可以开始三个原型。三个原型通过后，可以实现完整检查器；在此之前仍不建议让 Agent 开始全仓业务重构。
