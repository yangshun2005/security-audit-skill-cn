# 侦察

### 阶段 1：梳理源码并规划覆盖范围

父级初始化 `run-metadata.json`，执行 `SKILL.md` 中严格的侦察前预算门禁，然后创建 agent 临时根目录和共享账本，再开始搜寻。若门禁未通过，在元数据中记录未完成状态，且不启动任何侦察 agent。侦察只读取目标及本地可用的构建/配置状态，不接触已部署端点、外部身份提供方、镜像仓库、消息代理、云 API 或其他共享服务。

并行启动多个 `research` agent。它们向父级返回结构化事实，不写文件。

**Agent 1a：产品、技术栈与本地运行**

```text
读取 <target> 处的目标。禁止使用网络访问。返回：
1. 产品类型、用户、运维人员，以及常规的信任敏感操作。
2. 语言、框架、构建系统、运行时，以及本地可见的部署模型。
3. 仓库相对路径的入口点与子系统边界。
4. 可用本地依赖离线执行的精确构建与测试命令、其预期写入位置，以及其处理的目标可控输入。侦察期间不要运行它们。
5. 从本地文档和依赖中可见的同类软件或协议。若没有基于源码的有用对比，请说明。
6. 限制有界执行的缺失本地工具链或运行时事实。
仅返回带仓库相对 file:line 引用的源码事实。
```

**Agent 1b：主体、权限与控制**

```text
读取所有确立身份、授权、隔离与特权的源码。梳理：
1. 每个低信任主体及其设计上拥有的操作。
2. 每个入口面上的认证或对等身份。
3. 按资源的授权与租户/所有者范围。
4. 进程、浏览器、工作负载、CI、插件、模型/工具、设备或本地 IPC 权限。
5. 权限变更、确认、撤销、恢复与回退路径。
6. 哪些控制源码可见，哪些依赖未观测到的部署事实。
返回信任边界与控制位置，附仓库相对 file:line 引用。不要推断实际可达性。
```

**Agent 1c：入口面、副本与汇聚点**

```text
清点每个源码可见的外部或低信任输入进入点：
- HTTP/浏览器、RPC/消息/协议、文件/归档/文档、CLI/环境变量/配置、插件/依赖/CI、云事件/IAM 选择器、模型上下文/工具参数、移动端/深链/webview，以及本地 IPC。
对每个入口面，追踪主要变换、存储或派生的副本，以及安全相关的汇聚点。记录源码可见的限制与通往同一效果的并行路径。
返回仓库相对路径与行号。要完整，但不要执行或发送输入。
```

**Agent 1d：本地执行与部署可见性**

```text
读取测试、构建定义、清单、打包与维护中的环境覆盖层。返回：
1. 可在所需 OS 强制沙箱内用虚拟数据验证信任边界的小型离线测试或现有夹具。
2. 可使用隔离回环网络命名空间且无外部或共享依赖的进程。
3. 会拉取依赖、发布构建产物、接触付费/提供方 API 或影响共享状态的命令；标记为本次运行禁止。
4. 源码无法确立、因而若具决定性则需 needs_validation 的已部署控制与挂载。
5. 仅在仓库确定性选择时，每种部署模式的最终生效源码路径。
6. 本地平台能否强制空允许列表环境、无外部网络、只读目标/工具链挂载、仅临时目录写入，以及显式 CPU、内存、进程、文件大小、磁盘与墙钟限制。缺失控制将阻止目标可控执行。
7. 受信父级代码能否通过路径受限的无跟随描述符遍历、非阻塞常规文件检查、对每个目标父目录的无跟随遍历、独占常规文件目标创建，以及显式单文件与累计大小上限，来提升预声明的临时文件。缺失提升控制将阻止将临时文件用作证据。
```

针对这四个 agent 未覆盖的、实质不同的部署模式或子系统，增加专项侦察 agent。不要静默省略：若 `SKILL.md` 中的预算门禁阻止了专项 agent，则不为它启动任何东西，将该未覆盖区域作为 `deferred` 账本单元播种，原因记为 `budget_cannot_reserve_critics_and_validation`，并在报告中披露该缺口。

## 先前运行输入

在选择工作之前，父级读取同一仓库所有可用的先前 `coverage-ledger.json` 与 `findings.json`：

- 将每条先前记录和单元的源码位置、控制、条件与源码派生身份与当前源码比对。
- 仅当其相关源码、条件与合格证据仍然适用时，将未变的先前 `confirmed` 记录以相同指纹带入当前候选集。用 `prior_status: "prior_confirmed_same_source"` 将其链接到当前 `planned` 单元，并仅将该根因放入 hunter 排除列表。重新验证该带入记录的阶段 3 验证者成为该单元的分配所有者；其源码复查是该单元的首项检查，并将单元以带入的指纹移至 `candidate`。
- 当任何相关源码或条件发生变化时，构建当前计划的 `prior_confirmed_changed_source` 重新验证单元。不要将该根因从搜寻中排除，也不要假定先前结论仍然适用。
- 为每个先前 `needs_validation`、`deferred`、`blocked`、`out_of_scope` 及变更源码单元构建当前工作单元。这些状态是优先输入，绝不是去重或抑制键。
- 仅当当前源码支持其追踪时，才以相同指纹带入仍受阻的先前 `needs_validation` 记录。用 `prior_status: "prior_needs_validation"` 将其链接到当前 `planned` 单元；该记录保留未解决的阻塞项。重新检查该带入记录的阶段 3 验证者成为该单元的分配所有者；其复查是该单元的首项检查，并将单元以带入的指纹移至 `candidate`。将该记录纳入最终验证。
- 除非当前证据改变了失败的追踪或缺失条件，否则将先前被拒记录视为过期声明。未变的拒绝仅抑制该确切声明，不抑制对该覆盖单元的审查。
- 记录缺失或不兼容的账本，而不是将其视为空覆盖。

在 `run-metadata.json` 中说明所用路径与源码引用。仅在 `architecture.md` 中总结覆盖后果。

## 架构摘要与配套文件选择

父级合成 `<output-dir>/architecture.md`，硬性上限约 1,000 词。包括：

1. 产品、主体、常规权限与受保护资源。
2. 来自 Agent 1a 的同类软件基线（当有源码依据时）：同类软件接受哪些安全权衡。用它校准投入与严重性，绝不用来否定已证实的发现；若同类软件共享在实践中重要的缺陷模式，则加强该发现。无有意义的同类软件时省略此行。
3. 技术栈、源码可见的部署路径，以及离线构建/测试限制。
4. 入口面及重要的源码到汇聚点或生命周期路径。
5. 信任边界及每个边界上最强的源码可见控制。
6. 仓库相对起始路径。
7. 先前覆盖缺口、变更源码与受阻重新验证目标，以及同源已确认排除项。
8. 从 [ATTACK-CLASSES.md](ATTACK-CLASSES.md) 派生的简短配套文件选择摘要：选中的文件及需要它们的源码可见边界。

将分配级别的普通块、选中的配套块及带原因的排除块保留在每个账本单元中，而非 `architecture.md`。这使架构词数上限在大规模运行中仍然有效，并使确切的 hunter 提示映射可机器校验。

不要仅因语言或依赖名称出现就选择配套文件。选择它是因为侦察发现了其 `When to use this file` 节所描述的信任敏感边界。不要仅因另一个 agent 将审查相关类别就排除可见边界。

## 确定性覆盖账本

父级将 `<output-dir>/coverage-ledger.json` 写为顶层 JSON 数组。按运行配置设定的粒度（`quick` 使用一个全在范围内的子系统身份；`deep` 增加生命周期模式），为入口面、信任边界、子系统及适用的普通或配套攻击类的每种实质组合派生一个单元。对于限定范围的运行，播种范围内的入口面用于分配，并将发现的排除入口面保留为 `out_of_scope` 单元，以便后续完整运行将其转为当前工作。

每个维度在 `canonical_refs` 中有一个人类标签和一个稳定的源码派生值。同一源码对象在多次运行中使用相同的规范引用，即使其显示标签变化。合适的引用包括：仓库相对入口路径加导出范围、源码中定义的路由或消息身份、定义边界的源码控制、仓库包路径，以及确切的攻击类块引用。块引用是 `FILE.md#` 加上该文件中以粗体或标题形式书写的确切类名——一个与文件文本匹配的稳定标识符，而非渲染后的 HTML 锚点。对于配套节块，使用任何括号限定词之前的标题文本（例如 `Core discipline`）。不要通过小写化或 slug 化显示标签来派生引用。

无损派生 `coverage_id`：

1. 要求每个引用为 Unicode NFC、具有有效标量值、可见内容，无控制、格式、行/段分隔符或默认可忽略码点，且无前后空白。
2. 用 RFC 3986 百分号编码其 UTF-8 字节：仅保留 `A-Z a-z 0-9 - . _ ~` 不转义，其他每个字节使用大写 `%HH`。
3. 用 `::` 连接编码后的 `surface`、`boundary`、`subsystem` 与 `attack_class` 引用；存在时追加编码后的 `lifecycle`。

对 quick 配置的粗化子系统维度使用固定规范值 `profile/quick/all-in-scope-subsystems`。不要在引用或 ID 中包含波次号、agent、结论、严重性或行号。每次分配前按 `coverage_id` 字典序排序单元。任何重复 ID 均失败。若重复 ID 具有不同语义字段，视为规范身份冲突；绝不合并或静默覆盖。验证器还拒绝由不同规范引用表示的同一语义元组。

每个单元记录：

```json
{
  "coverage_id": "...",
  "canonical_refs": {
    "surface": "src/router.ts#POST /users/:id",
    "boundary": "src/authz.ts#requireOwner",
    "subsystem": "packages/api",
    "attack_class": "ATTACK-CLASSES.md#Access control"
  },
  "surface": "...",
  "boundary": "...",
  "subsystem": "...",
  "attack_class": "...",
  "starting_paths": ["repo/relative/path"],
  "ordinary_attack_class_block": "ATTACK-CLASSES.md#Access control",
  "selected_companion_blocks": ["FILE.md#section"],
  "excluded_blocks": [{"block": "FILE.md#section", "reason": "..."}],
  "prior_status": "new|prior_confirmed_same_source|prior_confirmed_changed_source|prior_needs_validation|prior_deferred|prior_blocked|prior_out_of_scope|prior_covered_same_source|prior_covered_changed_source|prior_rejected_claim_changed|none",
  "attempts": [],
  "wave": 1,
  "status": "planned",
  "agent_id": null,
  "reviewed_paths": [],
  "local_checks": [],
  "result_fingerprints": [],
  "unresolved": []
}
```

当 `lifecycle` 具有实质意义时，同时添加 `canonical_refs.lifecycle` 与人类可读的 `lifecycle` 字段。仅当无普通块适用时 `ordinary_attack_class_block` 为 null。选中的配套列表包括每个适用类及其配套 `Core discipline`、`Universal moves` 与 `Validation rules`；`excluded_blocks` 记录每个已考虑但未选中的块及排除它的源码事实。

父级可添加簿记字段，但保持上述语义字段稳定。在 `prior_status` 中，当存在兼容的先前账本时，`new` 标记本次运行首次见到的入口面；当无兼容先前账本可用时，`none` 标记播种的单元。先前的 `deferred`、`blocked`、`out_of_scope` 及变更源码单元在现已纳入范围时初始化为当前 `planned` 工作。先前同源已覆盖单元在当前账本中保持可见；优先分配变更源码、重要生命周期路径与确切冲突，然后用覆盖评审决定是否需要再一轮。

`attempts` 是仅追加的归档，用于被覆盖评审重新打开的、带证据的分配。重新分配前，追加先前单元的确切 `wave`、`status`、`agent_id`、`reviewed_paths`、`local_checks`、`result_fingerprints` 与 `unresolved`，以及评审的源码支持 `reassignment_reason`。仅 `blocked`、`covered` 与 `candidate` 状态可归档。归档尝试保留与活动单元相同的状态与证据不变量，使用严格递增且低于当前波次的波次，并保留其产出所有者与构建产物。下一次分配递增 `wave`，使用全新所有者，并以空的活动证据开始。若配置或预算阻止另一次分配，递增 `wave` 并使用活动 `deferred` 状态，所有者 null、证据为空，并记录停止原因。绝不将归档所有者的检查或构建产物复制到活动状态。后续活动终态仅包含新尝试的证据；归档保持不变。

严格强制执行此状态表：

| 状态 | 单元 `agent_id` | `reviewed_paths` / `local_checks` | `result_fingerprints` | `unresolved` |
|---|---|---|---|---|
| `planned` | null | 空 | 空 | 空 |
| `not_applicable`、`out_of_scope`、`deferred` | null | 空 | 空 | 非空原因 |
| `in_progress` | 规范所有者 | 空 | 空 | 空 |
| `blocked` | 规范所有者 | 两者非空，为所有者部分证据 | 空 | 非空阻塞项 |
| `covered` | 规范所有者 | 两者非空 | 空 | 空 |
| `candidate` | 规范所有者 | 两者非空 | 非空 | 可选 |

规范 agent ID 匹配 `^[a-z0-9][a-z0-9_-]{0,63}$`，且不是 Windows 设备名。小写是强制的，因此一个账本不能包含大小写折叠别名。单元 `agent_id` 记录分配所有者。每项检查记录自己的 `agent_id` 与非空 `reviewed_paths`；单元级 `reviewed_paths` 恰好是它们的并集。仅源码检查使用 `artifact: null`。本地检查要求常规文件仅由受信父级代码在恰好 `agents/<check.agent_id>/artifacts/` 下提升。这使 hunter 与验证者检查可共存于一个单元。临时路径、输出根文件、符号链接、特殊文件及其他检查所有者的构建产物不是证据。

账本是覆盖声明。架构摘要、agent 数量或泛泛的"已审查认证"句子不是覆盖证据。阶段 2 仅根据 hunter 结构化结果中的路径与检查关闭单元。

在播种后、每次父级更新后及阶段 6 前运行 `node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json`。验证器拒绝超过 5 MiB、64 层嵌套、10,000 个单元、嵌套集合中 1,000 个条目或 500,000 个遍历值的输入，并将报告的验证错误上限设为 100。实践中 5 MiB 字节限制约容纳 2,000-5,000 个真实单元，因此它先于 10,000 单元上限生效。在分配工作或做出覆盖声明前修复每个错误。