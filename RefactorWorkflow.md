# 确定性重构工作流

这份文档设计一套用于「未经严格审查的 Python 移植仓库」的重构机制。典型对象是按 S4 C/C++ 源码移植的 Python 版，再加上后来按零散需求打上的局部补丁。目标是在数值行为保持不变的前提下，清掉 Agent 式过度封装、缺输入却静默继续、双向依赖、重复计算，以及照搬 C 的低效双循环等问题。

机制放在**被重构的那个 Python 仓库**里。本文档只规定要落地的 skill、command、脚本、日志和单测，不在本 C 仓库里实现它们。S4 自身的模块边界见 `Summary.md`，下文用它作为 Python 移植的目标结构样例。

确定性不来自更长的提示词。Agent 不挑选下一处要改的文件，不手写「已完成」，也不靠自觉遵守风格。下一处改什么由队列脚本给出；能不能结束由检查脚本的退出码决定；状态只由 `mark.py` 写入。

## 1. 设计原则

1. **事实与判断分开。** 符号清单、调用图、import 图、AST 反模式由脚本生成。Agent 只在脚本留出的枚举项里做判断，并把判断写成可校验的记录。
2. **判断尽量前移到一次总纲。** 某个量是必填还是可选、某条依赖允不允许、某个函数是不是模块的公开入口，都在阶段 A 写进 schema。阶段 B 只对照 schema 改代码，不再临时决定。
3. **棘轮。** 防御性默认、`type: ignore`、`Any`、透传包装、未登记双循环的数量只允许下降。相对基线增加即失败。
4. **功能锁在外部参考上。** S4 移植的数值基准是 C 版 S4 在固定算例上的输出，不是脏 Python 自己的输出。重构不得让偏差变大。没有 C 参考时才退化为冻结当前 Python 输出，并在总纲里写明。
5. **一次只走一步。** 一步是调用图上的一个函数，或总纲里预先分组的一个原子阶段。窗口之外的文件出现在 diff 里，任务失败。
6. **失败即停。** 门禁非零就留在当前任务里修。需要扩大窗口时停止改代码，走修订流程，不在任务里偷偷多改。

## 2. 要消除的问题

前五项对应本次要求。后面是同一类仓库里会反复出现、并且能用同一套门禁挡住的问题。

| 编号 | 问题 | 判定 | 何时钉死 |
| --- | --- | --- | --- |
| P1 | 过度封装：函数只转发给另一个函数，单方法类，为了一处调用新加一层 | 脚本识别透传；合法透传只能是 `modules.toml` 里该模块的 `public_api` | 阶段 A 登记公开入口；阶段 B 禁止净增 class/def |
| P2 | `dict.get(k, default)`、`getattr(obj, k, default)`、`pop(k, default)`、`x or default` 让必填输入缺失时继续跑 | 脚本扫 AST；必填键必须用会抛 `KeyError`/`ValueError` 的访问 | 阶段 A 的 options schema 列出仅有的合法默认值 |
| P3 | 局部修改造成的跨模块调用和双向 import | import 图相对 `allowed_imports` 的差集；调用图上的跨模块边同样检查 | 阶段 A 的依赖总纲 |
| P4 | 同一物理量沿调用链算了多次 | 总纲指定每个规范量的唯一生产者；脚本禁止其他函数给该名字赋值；契约上的 `formula_id` 冲突也失败 | 阶段 A 登记生产者；阶段 B 每步核对 |
| P5 | 直接双层 `for i in range` / `for j in range` 做逐点算术，把 C 循环原样搬进 Python | AST 识别；几何遍历、稀疏装配可以豁免，数值内核不能借用几何豁免 | 豁免名单，且理由只能取枚举 |
| P6 | 吞掉异常后返回默认值或 `None`：裸 `except`、`except Exception: return/pass` | 脚本 | 全程禁止，无默认豁免 |
| P7 | 用 `None` 表示失败，调用方再判空 | 链上函数的返回注解和契约不得把失败写成 `None`；失败必须抛异常 | 契约 schema |
| P8 | `**kwargs` 把签名藏起来，上下游对不齐 | 内部函数禁止 `**kwargs` / `*args`；公开 API 若保留，必须在契约里逐键展开 | 阶段 B 的签名检查 |
| P9 | 函数内部 import，用来掩盖环 | 脚本统计；棘轮下降，终局除总纲点名的延迟 import 外必须为 0 | 阶段 A 点名，之后只减不增 |
| P10 | 可变默认参数、星号 import、模块级可变全局量 | 脚本，终局为 0 | 全程 |
| P11 | 热循环里 `list.append` 再转数组；对标量反复调 BLAS | 并入 P5 的循环检查 | 阶段 B |
| P12 | 两段 AST 归一化后相同的实现并存 | 脚本聚类，每簇只留契约里登记的一个所有者 | 阶段 C 清残留时必须处理完 |
| P13 | 0-based / 1-based 混用、复数用实部虚部交错数组和 `complex` dtype 混用 | 契约写 `index_base` 和 `dtype`；脚本核对链上边界的契约字段是否填了 | 不能证明哪一次 `+1` 是错的，所以做成必填字段，而不是猜 |
| P14 | 广播碰巧成功，形状错了也不报 | 数值内核入口要有形状断言；脚本检查契约中的 `shape` 是否在函数体前部被读取 | 启发式，未引用 `shape` 即失败 |
| P15 | 就地改写调用方还要复用的数组 | 契约 `mutates` 列出被改的参数；脚本看参数名是否出现在赋值左侧。未声明的就地写失败 | 每步契约 |
| P16 | 几何或频率变了，模式缓存仍在 | 编排层函数的契约 `invalidates` 必须覆盖总纲里的缓存名；缺一即失败 | 阶段 B 编排步 |
| P17 | `utils` / `common` / `helpers` 成为谁都 import 的枢纽 | 脚本：扇入超过总纲阈值且不在 `public_api` 模块列表里即失败 | 阶段 A 禁止预留这种模块 |
| P18 | 调试 `print`、`breakpoint`、无单号的 TODO | 脚本，终局为 0 | 全程 |
| P19 | `Any` 与 `# type: ignore` 扩散 | 计数棘轮 | 全程 |
| P20 | 注释和 docstring 代替结构，重构后与代码漂移 | 阶段 B 禁止新增叙述性注释和扩写 docstring。不设自然语言质量分 | 步骤 skill 的硬禁止，由 diff 检查「注释行净增」辅助 |
| P21 | 死代码、只有测试引用的代码、字符串动态调用 | 从入口做可达性；动态访问单独列出，不能直接删 | 阶段 C |
| P22 | 为通过测试而放宽数值容差 | 容差只在基线文件里；变松必须走修订，且修订脚本拒绝松于基线 | 门禁 |

P13、P14、P16 脚本不能独自判断物理对错。它们被收成必填字段和引用检查之后，Agent 的自由只剩「填哪一个枚举值」。填完仍由脚本核对字段在不在、引用的行还在不在。

## 3. 目标仓库里的工件

路径都相对于 Python 仓库根目录。生成物可以手改的只有总纲和 schema；清单和状态不行。

```
refactor/
  constitution.md              人读的总纲，章节标题固定，供脚本核对
  modules.toml                 模块、层号、允许的 import 边、public_api
  options_schema.toml          仅有的合法默认值
  canonical_vars.toml          规范物理量 → 唯一生产者
  chains.json                  业务流程 DAG，由脚本生成后只经修订命令改
  inventory.json               全部函数、类、模块级变量；脚本生成
  status.json                  每个符号的状态；只由 mark.py 写
  contracts/<symbol_id>.json   该步的 I/O 契约
  decisions/<symbol_id>.json   非脚本判断的枚举记录
  allowlists/
    loops.toml
    delayed_import.toml
    dynamic_ref.toml
  baseline.json                阶段 0 冻结的计数与容差
  golden/<case_id>.npz         外部参考输出
  tasks/<task_id>.json         脚本生成的任务卡
  reports/<run_id>.json        每次门禁的完整结果
  log/events.jsonl             只追加的事件日志
scripts/refactor/              第 9 节的检查脚本
tests/numeric/                 对 golden 的回归
tests/structure/               调用门禁脚本的 pytest
tests/refactor_self/           用玩具仓库验证门禁本身会失败、会通过
.cursor/skills/                第 7 节
.cursor/commands/              第 8 节
```

`inventory.json` 里每个符号至少有：

```json
{
  "id": "fmm.closed.epsilon",
  "kind": "function",
  "qualname": "fmm.closed.epsilon",
  "file": "src/fmm/closed.py",
  "module": "fmm",
  "signature": "(g, pattern, materials) -> Epsilon",
  "callers": ["orch.compute_layer_modes"],
  "callees": ["pattern.fourier"],
  "ast_hash": "sha256:...",
  "reachable": true
}
```

`status.json` 不进 inventory，避免 Agent 改清单时顺手改状态。状态取值：

| 状态 | 含义 |
| --- | --- |
| `untouched` | 还没有任务验收过 |
| `reviewed` | 当前 `ast_hash` 已通过该任务门禁 |
| `deleted` | 已删除，inventory 里仍留墓碑直到阶段 D |
| `exempt` | 总纲点名不重构，必须有理由枚举 |
| `stale` | 源码哈希与验收时不一致，由下次生成清单时自动打上 |

`reviewed` 绑定哈希。有人绕过流程改了函数体，下一次 `inventory.py` 会把状态打回 `stale`，队列重新插入该符号。Agent 不能通过编辑 JSON 把 `stale` 改回 `reviewed`：`mark.py` 只接受「刚跑过的报告文件」，并核对报告里的哈希。

## 4. 窗口与队列

### 4.1 上下游只含一跳

一步的窗口叫 W1。

- **当前函数**：可以改函数体、它的契约、它的数值夹具和对应单测。
- **下游，直接被调用方**：必须已经是 `reviewed`。本步只读，禁止改它们的文件。当前函数必须按它们已冻结的契约去调用。
- **上游，直接调用方**：本步还没轮到它们。只允许改「调用当前函数的那一个表达式」，使参数与新签名一致。禁止改调用方的算法、禁止改调用方调用别人的方式。
- **再远的一跳不打开。** 不读不改调用方的调用方，也不改被调用方的被调用方。
- **数据只追一跳。** 核对「调用方产出的、本函数读入的量」和「本函数产出的、被调用方读入的量」。不沿表达式再往外追。

组步是唯一的放宽。总纲把几个必须一起改的函数标成一个原子阶段（例如一块偏振基分解里共享中间矢量场的三步）时，这几个函数算同一个当前步。组的出口之外仍然只有一跳，而且组必须在 `chains.json` 里预先写好。Agent 不能在任务中途把窗口扩成组。

这个宽度是故意窄的。再宽，任务就会变成「顺手把上下游都整理一下」，局部修改和双向耦合会回到仓库里。再窄，只改当前函数、不看调用点，签名一变编译或运行就会在别的任务里才爆。

### 4.2 行走方向

业务链按用户操作记录，从入口到叶子，例如「算功率流」：

`api.get_power → orch.compute_layer_modes → fmm.closed.epsilon → pattern.fourier`

执行顺序是这条链的**依赖序**：被调用方先验收，调用方后验收。队列脚本对调用图做拓扑排序。

这样规定的原因：轮到某个函数时，它调用的函数契约已经冻住，它就不能再包一层适配器去凑旧接口。调用方尚未重构，所以本步只改调用点，不把调用方的清理提前做完。

跨模块的调用环在阶段 A 就失败，不进入阶段 B。模块内部的递归必须在 `chains.json` 里声明，否则拓扑排序失败。

一个函数出现在多条链上时，第一次出现的任务拥有修改权。后面的链只重新跑契约和数值夹具，任务卡写成 `verify_only`，diff 必须为空。

### 4.3 任务卡

`scripts/refactor/next_task.py` 写出下一张卡，Agent 不能自己选文件。卡的内容：

```json
{
  "id": "solve_patterned.fmm_closed",
  "chain": "solve_patterned",
  "primary": ["fmm.closed.epsilon"],
  "downstream_frozen": ["pattern.fourier"],
  "upstream_callsite_only": ["orch.compute_layer_modes"],
  "editable_files": [
    "src/fmm/closed.py",
    "refactor/contracts/fmm.closed.epsilon.json",
    "tests/numeric/test_fmm_closed.py"
  ],
  "callsite_files": ["src/orch/layers.py"],
  "checks": ["step"],
  "mode": "edit"
}
```

`check_scope.py` 比较 `git diff` 的路径与这两份文件列表。多出来的路径使任务失败。`callsite_files` 上的 diff 再交给 `check_callsite.py`：只允许调用点所在行变化，用 AST 比较调用方函数除该调用外的其余语句必须相同。

### 4.4 一步里面的顺序

1. `next_task.py` 取出任务卡。若上一张卡的状态不是 `reviewed` 或 `deleted`，拒绝发下一张。
2. 先写契约 JSON，再改函数。契约引用的下游符号必须已是 `reviewed`。
3. 改代码。本步禁止新增 class；模块内函数个数不得净增。要删掉的透传包装算减少，允许。
4. 跑 `check_step.py --task <id>`。
5. 通过后 `mark.py` 写状态并追加日志。
6. 门禁失败则留在同一步。需要动冻结的下游，或需要改调用方算法时，停止，走第 11 节的修订，不扩大 diff。

「不用改」也要过门禁。结论枚举只能是 `already_conforms`。脚本仍会扫这个函数的 P2、P5、P6。扫干净才允许在没有 diff 的情况下标 `reviewed`。

## 5. 阶段

### 阶段 0：冻结行为

在任何结构调整之前做完。

1. 总纲写 `reference = external` 或 `reference = snapshot`。S4 移植默认 `external`。
2. `capture_golden.py` 对固定算例表调用 C 版 S4（或当前 Python），写入 `refactor/golden/`。算例至少覆盖：均匀层、一维光栅、二维图案、两种以上傅里叶配方、复制层、改频率后缓存失效。
3. `baseline.json` 记录当时的反模式计数、`type: ignore` 数、`Any` 数、以及各算例容差。
4. 若用 external，同时写出 `baseline_vs_python.json`，记下当前 Python 相对 C 的已知偏差。之后偏差允许变小，不允许变大。变小只有在任务卡 `numeric_effect = bugfix` 且修订记录点名算例时成立；默认任务的 `numeric_effect = preserve`，偏差必须保持在容差内。

没有 `golden/` 和 `baseline.json` 时，`next_task.py` 不发任务卡。

### 阶段 A：总纲、依赖方向、全量清单

对应设想的 a。脚本先出事实，Agent 再填总纲，校验脚本通过才算本阶段结束。

脚本产出：

- `inventory.py`：全部函数、类、模块级变量。
- `import_graph.py`：import 边、环、函数内 import。
- `call_graph.py`：静态调用边；解析不了的 `getattr`、字典派发写入 `dynamic_ref.toml` 草稿，不得假装成已解析。
- `propose_modules.py`：按目录和调用紧密度给一个**建议**划分。建议不是总纲。

Agent 使用 skill `refactor-constitution`，把建议收成下面三份权威文件。

`modules.toml` 的权威是边，不是层号。层号只用于让人读。每条实际 import 都必须出现在 `allowed_imports` 里，否则阶段 A 的 `check_constitution.py` 失败。

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

`options_schema.toml` 列出可选开关和默认值，例如 Lanczos 平滑默认关闭。除此以外的键都是必填，`.get(..., default)` 非法。

`canonical_vars.toml` 至少登记：`omega`、`kx`、`ky`、`G`、`Epsilon2`、`Epsilon_inv`、`q`、`phi`、`kp`、层模式缓存、全堆叠振幅。每个量一个生产者函数。

`chains.json` 由 `build_chains.py` 从入口和调用图生成，Agent 只做两件事：给链命名、把必须原子化的连续函数收成组。脚本拒绝未覆盖的可达入口。

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

- `allowed_imports` 无环，且与真实 import 图一致，或差集被列成「待消除」并因此阻断阶段 B 中对应模块之外的任务。
- 每个文件恰好属于一个模块。
- 每个可达入口都在某条链上。
- 每个规范量恰好一个生产者，且生产者在 inventory 里。
- `public_api` 里的符号都存在。
- 动态调用草稿被显式接受或改成静态调用，不允许留着空草稿进入阶段 B。

### 阶段 B：沿链一步一任务

对应设想的 b。队列由 `build_queue.py` 按第 4.2 节生成。每个任务走第 4.4 节。

本步要核对的 I/O，写在契约里，不写在聊天记录里：

```json
{
  "symbol": "fmm.closed.epsilon",
  "inputs": [
    {"name": "G", "dtype": "int64", "shape": "(n, 2)", "unit": "1", "required": true, "source": "gsel.select"}
  ],
  "outputs": [
    {"name": "Epsilon2", "dtype": "complex128", "shape": "(2n, 2n)", "unit": "1", "formula_id": "epsilon2", "consumers": ["rcwa.solve_layer"]}
  ],
  "mutates": [],
  "invalidates": [],
  "raises": ["KeyError", "ValueError"],
  "decision": "vectorized"
}
```

`check_contract.py` 在本步检查：

- 必填输入在签名里没有默认值。
- 函数体对必填输入不使用 `.get` / `getattr` 默认值 / `or` 默认值。
- 输出名字与下游契约的输入名字能对上；对不上就失败，而不是在本函数里再算一遍下游要的量。
- `formula_id` 与 `canonical_vars.toml` 的生产者一致。若本函数不是生产者，却给规范量赋值，失败。
- `shape` 字段在函数前部被断言引用。
- `raises` 非空。链上函数不允许把错误变成返回值。

本步同时跑第 9 节的局部反模式脚本，范围是任务卡里的文件，外加棘轮：全仓库计数不得高于 `baseline.json`。

每步结束，`mark.py` 把当前符号标成 `reviewed`，记下 `ast_hash`、报告哈希、git 修订。inventory 本身重生成，不手改。

### 阶段 C：按模块清残留

对应设想的 c。进入条件：所有链上符号都是 `reviewed`、`deleted` 或 `exempt`。`residual.py` 列出仍然 `untouched` 的符号，并先做机器分类：

| 分类 | 脚本规则 | 随后的任务 |
| --- | --- | --- |
| `dead_candidate` | 从全部入口沿静态调用不可达，且不在 `dynamic_ref.toml` | 删除任务。删除前 `refs.py` 再搜字符串和装饰器；搜到则改成人工枚举，不能删 |
| `dynamic` | 不可达但在动态引用名单里 | 改成静态调用，或保留并写 `exempt` 理由 `dynamic_entry` |
| `test_only` | 只有测试调用 | 删除生产代码与只为它存在的测试，或把它补进某条链 |
| `side_path` | 可达但不在任何链上 | 补链（修订）或删除。不允许一直留在链外 |

每个残留符号仍是一张任务卡，窗口仍是 W1：当前符号、直接调用方的调用点、直接被调用方只读。以该函数为中心看它落在哪条调用链上，结论只能是三选一，写入 `decisions/`：

- `move_into_chain`：它属于某条已有链的哪一条边，修订 `chains.json` 后按阶段 B 的规则改。
- `delete`：死代码，本步只删定义和因此无人用的 import。
- `exempt`：理由枚举 `third_party_shim`、`dynamic_entry`、`frozen_public_api` 之一，并引用总纲段落。

阶段 C 结束条件：不存在 `untouched`，不存在未分类的残留，重复实现聚类里没有未登记的第二份。

### 阶段 D：全局验收

对应设想的 d。`gate.py --all` 对全仓库重跑第 9 节的全部脚本，并重生成 inventory。通过条件：

- 状态只有 `reviewed`、`deleted`、`exempt`，没有 `untouched` 和 `stale`。
- import 与 `allowed_imports` 一致，无环。
- 反模式计数满足终局：P2、P5 未登记项、P6、P8、P10、P17、P18 为 0；P9、P19 不高于基线且 P9 的剩余项都在延迟 import 名单里。
- 每条链的每条边都有契约，契约引用的符号存在。
- 规范量生产者唯一，且与真实赋值位置一致。
- 全部 golden 算例在容差内；`preserve` 任务的偏差相对阶段 0 没有变大。
- `audit.md` 的每一节第一行是 `report: <路径> sha256: <哈希>`。`check_audit_report.py` 重新计算报告哈希。对不上则阶段 D 失败。Agent 不能用「已全局看过」代替这一行。

阶段 D 的 Agent 技能 `refactor-global-audit` 只做三件脚本做不到的事，而且每件都要落成枚举记录：抽查豁免名单里每一项理由是否仍匹配代码；确认没有两条链用不同公式算同一个 `formula_id`；确认缓存失效列表覆盖「改几何、改材料、改频率、改 G」。做完必须再跑一次 `gate.py --all`。两次报告哈希都写进 `audit.md`。

## 6. 非脚本判断怎么收口

不能做成 AST 规则的判断，只允许出现在阶段 A 和修订里，运行中不再临场发明。

| 判断 | 谁做 | 记录 | 脚本随后强制什么 |
| --- | --- | --- | --- |
| 模块边界划在哪 | 阶段 A | `modules.toml` | 非法 import 失败 |
| 键是必填还是可选 | 阶段 A | `options_schema.toml` | 必填键上的默认值失败 |
| 规范量谁生产 | 阶段 A | `canonical_vars.toml` | 第二处赋值失败 |
| 双循环是几何遍历还是低效核 | 修订或阶段 A | `allowlists/loops.toml` 的理由枚举 | 数值模块使用 `geometry_traversal` 失败 |
| 透传要不要留 | 阶段 A 的 `public_api` | 不在名单里的透传失败 | 阶段 B 不能新增透传 |
| 动态引用是不是入口 | 阶段 C | `decisions/*.json` | 理由不在枚举里失败；引用的行号对不上失败 |
| 数值变化是修 bug 还是改坏了 | 修订 | 任务卡 `numeric_effect` | 未标记 `bugfix` 的数值漂移失败 |

`decisions/*.json` 的 `choice` 只能是脚本内置枚举。记录里必须有 `evidence`：文件路径和行号。`check_decision.py` 确认这些行仍然存在，并且与 choice 对应的结构还在（例如 choice 是 `delete` 时符号已不在 AST 里）。它不评价物理论证。

## 7. Skills

Skill 只规定读哪些工件、允许改什么、必须跑哪条命令、怎样才算完。策略的正文在本文和脚本里，skill 不复制一套可以漂掉的口吻要求。

四个 skill，都放在目标仓库 `.cursor/skills/`。

### `refactor-constitution`

- 何时：阶段 0 的 golden 已经生成，阶段 A 尚未通过 `check_constitution.py`。
- 先读：`propose_modules.py` 的报告、`Summary.md` 一类的参考结构（S4 移植必读本仓库那份摘要）、现有 import 环报告。
- 允许写：`constitution.md`、`modules.toml`、`options_schema.toml`、`canonical_vars.toml`，以及对 `chains.json` 的命名和分组。
- 禁止：改 `src/` 业务代码，改 `status.json`，把建议划分原样粘贴成总纲而不填 `allowed_imports`。
- 完成：`check_constitution.py` 退出码 0，并追加一条 `phase_a_accepted` 日志。

### `refactor-step`

- 何时：`/refactor-next` 发出一张 `mode = edit` 或 `verify_only` 的任务卡。
- 先读：任务卡、当前符号源码、下游契约、规范量表里与本符号有关的行。不读窗口外的实现文件。
- 允许写：任务卡的 `editable_files` 与调用点文件。
- 禁止：新增 class；函数个数净增；新增 `.get` 默认值；放宽容差；修改下游函数体；重构调用方算法；加叙述性注释来解释改动。
- 完成：`check_step.py` 退出码 0，然后只通过 `mark.py` 更新状态。

### `refactor-residual`

- 何时：`residual.py` 报告链上符号已清空，且仍有 `untouched`。
- 先读：该符号的分类、引用报告、调用方列表。
- 允许的结论：`move_into_chain`、`delete`、`exempt`。没有第四种。
- 完成：该符号不再是 `untouched`，`check_step.py` 对这张残留卡退出码 0。

### `refactor-global-audit`

- 何时：阶段 C 的结束条件已经满足。
- 先跑：`gate.py --all`。失败则回到对应阶段的任务，不在审计 skill 里直接改业务代码。
- 允许写：`audit.md`、对豁免项的 `decisions/` 记录。发现缺漏时只开修订或把符号打回 `stale`，不趁审计改无关文件。
- 完成：`check_audit_report.py` 退出码 0。

## 8. Commands

Command 是短入口，本身不做判断，只调用脚本并指出必须使用的 skill。建议放在目标仓库 `.cursor/commands/`。

| Command | 作用 | 拒绝条件 |
| --- | --- | --- |
| `/refactor-init` | 跑阶段 0 的捕获与基线，再跑清单和总纲检查 | 覆盖已有 golden 时必须显式 `--force`，且仓库干净 |
| `/refactor-next` | 打印下一张任务卡路径，并要求按卡上的 skill 执行 | 阶段 A 未通过；上一任务未标记；工作区有任务卡之外的未提交改动 |
| `/refactor-step` | 对当前卡跑 `check_step.py`；通过则 `mark.py` | 没有当前卡；范围检查失败 |
| `/refactor-residual` | 生成残留分类并取出下一张残留卡 | 链上仍有未验收符号 |
| `/refactor-audit` | 跑 `gate.py --all` 和审计报告校验 | 阶段 C 未结束 |
| `/refactor-amend` | 第 11 节的唯一修订入口 | 工作区有未标记的业务 diff |
| `/refactor-status` | 只打印计数：各状态符号数、当前链、棘轮指标、下一批任务 id | 无 |

不设「把这个函数也一起改了」的 command。扩大范围只有 `/refactor-amend`。

## 9. 硬编码脚本

脚本用 Python 标准库的 `ast`，不依赖仓库能跑通测试才开始做结构检查。数值检查单独依赖 numpy。每个脚本把 JSON 报告写到 `refactor/reports/`，并向 stdout 打一行 `sha256`。退出码 0 为通过，1 为检查失败，2 为用法或工件缺失。

| 脚本 | 检查 | 用在 |
| --- | --- | --- |
| `inventory.py` | 生成符号清单；哈希变化时把状态打成 `stale` | 每次门禁之前 |
| `import_graph.py` | 环、未授权边、函数内 import、星号 import | 阶段 A、每步、阶段 D |
| `call_graph.py` | 静态调用；动态调用与解析失败名单 | 阶段 A、阶段 C |
| `propose_modules.py` | 只产出建议，退出码始终 0 | 阶段 A 之前 |
| `build_chains.py` | 入口覆盖、组不与依赖序矛盾 | 阶段 A |
| `build_queue.py` | 拓扑序任务队列 | `/refactor-next` |
| `check_constitution.py` | 第 5 节阶段 A 的结束条件 | 阶段 A |
| `check_scope.py` | diff 路径 ⊆ 任务卡 | 每步 |
| `check_callsite.py` | 上游文件只有指定调用表达式变化 | 每步 |
| `check_defensive.py` | P2、P6、P7、P8：`.get` 默认、`getattr` 默认、`or` 默认、裸 except、except 后 return/pass、链上签名默认值、内部 `**kwargs` | 每步和终局 |
| `check_wrappers.py` | 函数体只有一条 return 调用，或类除 `__init__` 外只有一个方法；不在 `public_api` 即失败 | 每步和终局 |
| `check_loops.py` | 嵌套 `range` 循环且循环体是下标算术；`append` 热循环。豁免理由枚举：`geometry_traversal`、`sparse_assembly`、`mode_pairing`。`fmm` 与 `rcwa` 不得使用 `geometry_traversal` | 每步和终局 |
| `check_contract.py` | 第 5 节阶段 B 的契约规则；规范量重复赋值；`formula_id` 冲突 | 每步和终局 |
| `check_dupes.py` | 归一化 AST 哈希聚类 | 阶段 C、阶段 D |
| `check_globals.py` | 可变默认参数、模块级可变容器、`print`、`breakpoint` | 每步和终局 |
| `check_typing.py` | `Any` 与 `type: ignore` 计数相对基线 | 每步棘轮 |
| `check_abstraction.py` | 本步 diff 的 class 数不增、def 数不净增、注释行不净增 | 每步 |
| `refs.py` | 删除前的名字引用，含字符串 | 阶段 C 删除 |
| `residual.py` | 残留分类 | 阶段 C |
| `check_decision.py` | 判断记录的枚举、行号、与 choice 一致的结构 | 修订、阶段 C、阶段 D |
| `numeric_check.py` | golden 的最大绝对误差与最大相对误差；与 `baseline_vs_python.json` 比较偏差是否变大 | 每步相关夹具，终局全量 |
| `capture_golden.py` | 生成 golden | 阶段 0 |
| `check_step.py` | 串联本步需要的上述检查 | `/refactor-step` |
| `gate.py` | `--step` 或 `--all` | 每步与阶段 D |
| `mark.py` | 验证最新报告哈希与 git HEAD，写 `status.json`，追加日志 | 唯一写状态的入口 |
| `check_audit_report.py` | 审计报告引用的哈希 | 阶段 D |
| `status.py` | 人读的进度 | `/refactor-status` |

`gate.py --step` 的最小集合：`inventory.py`、`check_scope.py`、`check_callsite.py`、`check_defensive.py`、`check_wrappers.py`、`check_loops.py`、`check_contract.py`、`check_abstraction.py`、`check_typing.py`、`import_graph.py`、`numeric_check.py`（只跑这张卡声明的夹具）。

`gate.py --all` 额外加入：`check_dupes.py`、`check_globals.py`、`residual.py` 必须为空、`numeric_check.py` 全量、`check_constitution.py`。

第三方代码目录在总纲 `ignore_paths` 里列出，所有 AST 脚本跳过。不设默认忽略 `tests/`：测试里的 `.get` 默认值不计入 P2，但测试里复制一份生产算法会计入 `check_dupes.py`。

## 10. 日志与指标

### 10.1 事件日志

`refactor/log/events.jsonl` 只追加。`mark.py` 和修订脚本写入。手改导致哈希链断裂时，`gate.py` 失败。每行：

```json
{
  "ts": "2026-09-30T00:00:00Z",
  "event": "task_marked",
  "task_id": "solve_patterned.fmm_closed",
  "symbol_ids": ["fmm.closed.epsilon"],
  "git": "abc123",
  "report_sha256": "...",
  "prev_sha256": "..."
}
```

事件名只使用：`golden_captured`、`phase_a_accepted`、`task_started`、`gate_failed`、`task_marked`、`amend_opened`、`amend_accepted`、`audit_accepted`。

`task_started` 到下一条 `task_marked` 或 `gate_failed` 之间不允许第二张 `task_started`。这挡住并行改同一棵工作区。

### 10.2 每步留下的数

`status.py` 和 CI 读取同一些字段。阈值如下。

| 指标 | 每步 | 阶段 D |
| --- | --- | --- |
| `scope_violation` | 0 | 0 |
| `unauthorized_import` | 本步不得新增 | 0 |
| `import_cycle` | 0 | 0 |
| `defensive_required` | 本步文件为 0，全库不高于基线 | 0 |
| `silent_except` | 0 | 0 |
| `passthrough_illegal` | 本步为 0 | 0 |
| `nested_loop_unlisted` | 本步为 0 | 0 |
| `canonical_reassign` | 0 | 0 |
| `contract_gap` | 本步边为 0 | 0 |
| `def_net_increase` | ≤ 0 | — |
| `class_increase` | 0 | — |
| `type_ignore_count` | ≤ 基线 | ≤ 基线 |
| `any_count` | ≤ 基线 | ≤ 基线 |
| `numeric_abs_max` | ≤ 该夹具容差 | 全部夹具 |
| `deviation_growth` | `preserve` 时 ≤ 0 | ≤ 0 |
| `untouched_on_chain` | 随队列下降 | 0 |
| `untouched_off_chain` | 阶段 B 可不处理 | 0 |

单测分成三层，都由 CI 调用，不靠 Agent 记得跑。

1. **`tests/refactor_self/`**：带小段故意违规的 Python 样本。断言 `check_defensive.py`、`check_loops.py`、`check_wrappers.py`、`import_graph.py` 的退出码为 1；修正样本后退出码为 0。这层保护检查脚本自身不被改松。
2. **`tests/structure/test_gate.py`**：在完整仓库上跑 `gate.py --step` 或 `--all`，把退出码变成 pytest 结果。CI 每个 PR 跑 `--all` 中除全量数值以外的结构部分；全量数值在合并队列跑。
3. **`tests/numeric/`**：每个链步至少一个夹具，直接对比 `refactor/golden/`。编排层的缓存用例要断言：改区域或改频率之后，下一次求解的结果与冷启动一致，用来守住 P16。

不把「代码行数下降」或「圈复杂度」列成门槛。那些数变小并不表示 P1 到 P5 被除掉，还容易诱使 Agent 为了指标拆函数。

## 11. 修订

总纲、依赖边、可选默认值、循环豁免、数值从 `preserve` 改成 `bugfix`、把一步扩成组，都算修订。业务重构任务里面不做这些决定。

`/refactor-amend` 的步骤：

1. 工作区除了修订说明没有未标记的 `src/` diff，否则拒绝。
2. 复制一份当前权威文件到 `refactor/amend/<id>/before/`。
3. Agent 只改权威文件，并写 `reason`，理由枚举：`break_cycle`、`wrong_owner`、`atomic_group`、`loop_exempt`、`numeric_bugfix`、`new_optional`。
4. `check_constitution.py` 与受影响链的 `numeric_check.py` 必须通过。`numeric_bugfix` 还要求偏差变小而不是仅仅换了一组数。`new_optional` 要求默认值写在 schema 里，且调用点在缺键时仍会在公开 API 边界抛错；默认值不能加在内部链上。
5. 通过后把依赖该决定、且 AST 已与契约不符的符号打成 `stale`，队列在这些符号上重做。不允许「修订完就当全库已验收」。

## 12. 一次任务长什么样

以「图案层的封闭形式介电矩阵」为例。队列已经验收完 `pattern.fourier` 和 `gsel.select`，还没轮到 `orch.compute_layer_modes`。

任务卡的当前符号是 `fmm.closed.epsilon`。下游 `pattern.fourier` 只读。上游 `orch.compute_layer_modes` 只允许改调用表达式。

本步要做的事只包括：

- 契约写明输入 `G`、图案、材料表，输出 `Epsilon2` 与 `Epsilon_inv`，`formula_id` 使用总纲里的生产者。
- 删掉缺材料时回落到真空介电常数的 `.get`。缺材料抛 `KeyError`。
- 若函数只是把 `Simulation` 拆开再转给另一个同名内部函数，把内部函数抬成这一层的实现，删掉外壳。若这层是 fmm 模块的 `public_api`，可以保留一个与 C 侧 `FMMGetEpsilon_ClosedForm` 同签名的入口，但入口必须真的列在 `public_api` 里。
- 双层 `range(n)` 填充矩阵的部分改成数组运算。几何上逐形状求交的循环若在 `pattern` 里，不在这一步；这一步在 `fmm`，不能用 `geometry_traversal` 豁免。
- 不在这里重算 `G` 或包含树。生产者不是这个函数。

门禁通过后，只有这个符号变成 `reviewed`。编排函数里其余的逻辑留到它自己的那一步：那一步会检查它没有第二次组装 `Epsilon2`，并检查改频率时 `invalidates` 含模式缓存。

## 13. 在目标仓库里的落地顺序

先落地检查，再允许 Agent 改业务代码。顺序固定：

1. `inventory.py`、`import_graph.py`、`call_graph.py`、`check_defensive.py`、`check_loops.py`、`check_wrappers.py`，以及 `tests/refactor_self/`。
2. 阶段 0 的 `capture_golden.py` 与一批真实算例。
3. 阶段 A 的总纲文件和 `check_constitution.py`。
4. 任务卡、`check_scope.py`、`mark.py`、日志。
5. 四个 skill 和七个 command，内容指向上述脚本，不另写一套规则。
6. 然后才允许 `/refactor-next` 发出第一张业务任务卡。

缺第 1 到第 4 步时，不开始清代码。否则又会变成一边改一边靠对话记得「刚才检查过」。
