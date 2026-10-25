# 验证、结构化输出、复核与报告

### 阶段 3：独立验证每个候选

在干净的覆盖度批判通过或明确记录提前停止后，按稳定指纹和根因合并阶段 2 候选与沿用的同源先前确认。将每个唯一的拟议 `confirmed` 和 `needs_validation` 候选交给一个未搜寻过它的全新 `general` 验证器。沿用的先前确认遵循相同的当前验证路径，即使搜寻器排除了该未变更的根因。验证器可以读取搜寻器或先前产物，但必须重新读取每个引用的当前源位置，并独立运行其能安全复现的任何决定性检查。

为每个验证器分配规范的小写唯一 ID 和 `<output-dir>/agents/<verifier-id>/scratch/` 以及父级拥有的 `artifacts/`。验证器只写入 `scratch/`，绝不写入保留产物。它仅接收候选、其关联的覆盖单元检查和产物路径、解释该路径所需的架构事实、确切的相关伴随验证块、下方的提升程序块、源/本地执行边界、逐字复制的 `report-schema.json` 中 `confirmed`、`needs_validation` 和 `rejected` 分支，以及具有相同指纹的先前记录。它不得接收另一个验证器的结论。

#### 候选验证器提示

```text
You did not write this candidate. Try to refute it from repository source and bounded
local evidence. Do not contact deployed endpoints or external/shared services. Run
target-controlled code only inside the approved OS-enforced sandbox: no external
network, empty allowlisted environment, read-only target and tools, scratch-only
writes, and explicit low resource and wall-clock limits. If any control is unavailable,
do not execute; retain the exact missing capability as a needs_validation blocker.
Treat every scratch entry as target-controlled after execution. After the sandbox and
all its processes terminate, only trusted parent-side code may promote a predeclared
scratch-relative file, following the promotion procedure block included verbatim in
this prompt. You and target code never write retained artifacts. If promotion is
unavailable or fails, do not use that file as evidence.

1. Verify every trace and evidence file, positive line number, scope, and description.
   Confirm the first entry is a real lower-trust entrypoint and the last is the
   claimed sink or boundary effect.
2. Reconstruct the strongest source-visible validation, identity, authorization,
   normalization, lifecycle, framework, and containment controls on the path.
   Where the architecture summary names a comparable baseline, note whether it
   shares the pattern — as calibration, never as grounds to dismiss.
3. For a proposed confirmed candidate, independently reproduce the minimum observed
   result when possible. Verify inputs, interface shape, conditions, and affected
   dummy principal/resource. Do not infer a stronger result or continue after it.
4. Verify that likelihood, impact, confidence, and the proposed source fix match only
   what the evidence establishes.
5. For a proposed needs_validation candidate, decide whether the blocker is genuinely
   outside source/local observation. If source refutes the trace, reject it. If the
   missing fact remains decisive, keep needs_validation and make the local and
   owner-observed plans exact and non-destructive.
6. Preserve the fingerprint for the same source-derived root cause across every state.

Return exactly one JSON object and no surrounding prose:
{"decision": "confirmed|needs_validation|rejected", "record": { ... }}
where record exactly matches the decision's verdict branch of the schema included
in this prompt. A corrected record replaces the hunter's wording.
```

将此提升程序逐字复制到每个候选验证器提示中：

```text
Artifact promotion procedure (trusted parent-side code only):
Reference only for you: the parent performs these steps; you never perform them.

Before execution, the parent opens and retains trusted, non-inheritable directory
descriptors for the agent's scratch/ and artifacts/ roots, and records an allowlist
of expected scratch-relative artifact files plus explicit per-file and cumulative
byte limits. Never pass those descriptors to the agent or sandbox. After the sandbox
and all its processes terminate, trusted parent-side code promotes each allowlisted
file separately:

1. Validate the declared relative path: reject absolute, empty, `.`, `..`, or
   symlinked components.
2. Walk each parent component from the retained scratch-root descriptor with
   no-follow directory-relative operations; never reopen by path.
3. Open the leaf no-follow and nonblocking.
4. Verify with `fstat` that it is a regular file with link count exactly one and
   within the recorded per-file and cumulative byte limits.
5. Enforce those limits again while reading from that descriptor.
6. Copy exactly the verified size, repeat `fstat`, and reject a changed identity,
   type, link count, or size.
7. For the destination, walk every parent component from the retained
   artifacts-root descriptor with no-follow directory-relative operations; require
   each existing component to be a real directory, and create any missing directory
   exclusively before reopening and verifying it no-follow.
8. Create the leaf exclusively without following links, verify that the opened
   destination is a regular file with link count exactly one, and copy from the
   verified source descriptor without reopening either path.
9. Use equivalent race-safe APIs on non-POSIX systems.
10. Never recursively copy or glob scratch, extract an archive into artifacts, or
    open or promote a symlink, FIFO, socket, device, directory, hard-linked file,
    changing file, or file that exceeds its bound.
11. If any check is unavailable, cannot be enforced, or fails, discard the scratch
    entry; if it is decisive evidence, retain `needs_validation` with the exact
    promotion blocker.
```

验证器只有在独立建立完整路径和有界观察结果后，才能将 `needs_validation` 提升为 `confirmed`。当特定部署或运行时事实仍未知时，将拟议确认降级为 `needs_validation`。当源、本地行为、可见控制、缺失有意义的影响或不可能的前提反驳该声明时，使用 `rejected`。`needs_validation` 绝不是推测性想法的存放处。

父级检查每个验证器返回相同的指纹，除非它识别出真正不同的根因。合并更正，在每个关联覆盖单元中记录决定，并确保每个指纹有一条最终记录。丢弃格式错误或包裹在散文中的验证器结果，不进行修复；在预算允许时用全新验证器重新运行该候选，否则它根据不完整运行规则保留为未验证的账本候选。

当验证器证据更新账本检查时，将该检查的 `agent_id` 设为验证器的规范 ID，并列出其非空的仓库相对 `reviewed_paths`。保持单元级 `reviewed_paths` 等于各检查的并集。仅源审查使用 `method: "source"` 和 `artifact: null`。仅当文件由可信父级代码成功提升到 `agents/<check.agent_id>/artifacts/` 下时，才使用 `method: "local"`。单元保留其原始分配所有者，因此独立拥有的搜寻器和验证器检查可以共存。对于沿用的先前记录的种子 `planned` 单元，没有先前所有者：重新检查它的验证器成为该单元的分配所有者，其重新检查是该单元的第一个检查，将单元移至 `candidate` 并携带指纹。

如果严格的总代理预算无法覆盖每个候选，将运行状态设为不完整，并遵循 `SKILL.md` 中的确定性预算规则。未验证的候选仅保留在账本中。它不会以任何裁决进入 `findings.json`。

### 阶段 4：写入并验证 `findings.json`

父级将所有独立决定的记录写入 `<output-dir>/findings.json`，按指纹排序。包括：

- `confirmed`：有源依据的漏洞，具有完整的本地执行证据、条件、具体修复、可能性/影响/总体严重性和置信度。
- `needs_validation`：有源依据的候选，具有确切未解决的阻塞因素和至少一个适用的本地或所有者观察的部署计划。
- `rejected`：在验证期间被反驳的有源依据候选，保留以便未来运行不会在没有变更证据的情况下重复无支持的声明。

写入前立即读取 `report-schema.json`。它使用 `additionalProperties: false`；不要将搜寻器包装字段带入记录。保持这些裁决契约不同：

- `confirmed` 记录使用 `root_cause`、`intended_behavior`、`conditions`、`execution`、`remediation`、`severity` 和 `confidence`。它不得使用 `claimed_root_cause`、`blockers`、`validation_plan` 或 `reason`。`execution` 是目标中立的，使用目标的本机接口：API/HTTP 输入、CLI 调用、库调用、消息、文件夹具、浏览器操作、渲染策略或本地测试装置（视情况而定）。`observed_result` 非空且为事实。
- `needs_validation` 记录使用 `claimed_root_cause`、`trace`、`evidence`、`blockers`，以及至少一个非空的 `validation_plan.local` 或 `validation_plan.deployment` 字段。仅当两个上下文都能解决不同事实时才包含两者。它不得使用严重性、执行、修复、原因或确认的根因。
- `rejected` 记录使用 `claimed_root_cause`、`trace`、`evidence` 和 `reason`。它不得使用严重性、执行、修复、阻塞因素、验证计划或确认的根因。

每条记录都有稳定的指纹、标题、描述和仓库相对源路径。多步跟踪以 `entrypoint` 开始，以 `sink` 结束，仅在它们之间使用 `propagation`。单条目跟踪使用 `entrypoint` 或 `sink`。总体严重性不能超过已证明的影响。

运行：

```sh
node <skill-dir>/validate-findings.cjs <output-dir>/findings.json
node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json
```

在继续之前修复每个结构和语义错误。findings 验证器拒绝超过 5 MiB、1,000 个顶层 findings 或 64 层嵌套的输入，并将报告的错误输出限制为 100 条消息。验证器成功仅证明格式和账本一致性。

### 阶段 5：以全新视角验证最终记录

为每个最终 `confirmed` 和 `needs_validation` 记录并行启动一个全新的 `research` 验证器。此验证器检查结构化记录，而非搜寻器报告，并保持在源/本地边界内。

在 `quick` 运行中，阶段 3 和阶段 5 合并：阶段 3 验证器也执行这些记录检查并返回最终 schema 形状的记录，因此每个候选获得一个全新的独立审查者而非两个。其他所有配置文件保持两遍分离。在任何配置文件中都不得跳过对 `confirmed` 记录的独立审查。

对于 `confirmed`，要求它检查：

1. 每个仓库相对跟踪/证据路径、行、范围和描述的操作。
2. 真实入口接口和确切的本地输入形状。
3. 每个条件、解析器/策略步骤、源可见的阻止层和观察到的本地结果。
4. 受影响的主体/资源和已证明的影响。
5. 严重性分离：现实可能性、已证明影响、总体不大于影响。
6. 修复策略和任何 `code_changes`，包括修复是否强制执行不变量而不仅仅是转移信任。

对于 `needs_validation`，要求它检查：

1. 源路径真实且仅支持所声明的 `claimed_root_cause`。
2. 每个列出的阻塞因素都是决定性的，且不能已在本地回答。
3. 候选命名了一个边界和一个可能的具体结果，而非泛泛的担忧。
4. 至少一个验证计划字段存在且确切。`local` 使用有界夹具；`deployment` 要求所有者观察配置、身份、路由、策略或运行时事实。不要为不适用的上下文发明计划，且绝不向部署发送审计流量。
5. 指纹与相同根因的先前/当前记录匹配。

每个验证器返回恰好一个 JSON 对象：`{"decision":"verified","fingerprint":"..."}` 或 `{"decision":"replace","reason":"...","record":{...}}`，周围无散文。替换记录必须匹配其 `confirmed`、`needs_validation` 或 `rejected` schema 分支。以与阶段 3 相同的方式处理格式错误或包裹在散文中的阶段 5 结果：丢弃它而不修复，并在预算允许时用全新验证器重新运行。

当阶段 5 替换将记录提升为更强的裁决（包括任何提升为 `confirmed`），或实质性改变根因、跟踪、执行输入或观察结果、已证明影响或严重性时，不要将其作为最终应用。将该完整替换交给一个新的独立验证器，该验证器未搜寻、未执行阶段 3 验证、也未提出阶段 5 替换。新验证器重新检查当前源并在执行边界下独立复现任何决定性本地结果，然后返回 `verified` 或另一个替换。仅在此全新验证后应用实质性替换。如果产生另一个实质性替换，用全新验证器重复。如果预算或独立性不可用，从 `findings.json` 中移除有争议的记录，将其账本单元保留为未解决的候选，并设置 `run_status: "incomplete"` 和确切的 `incomplete_reason`。非实质性措辞或仓库行更正可在不改变含义或证据时直接应用。

每次应用替换后，重新运行两个验证器并更新关联的账本决定。如果最终验证器识别出单独的根因，分配新指纹并在纳入前通过独立候选验证发送。仅当每个账本候选都有独立的最终处置且每条保留记录通过阶段 5 时，才设置 `run_status: "complete"`。

不要仅验证 `confirmed` 记录。误导性的 `needs_validation` 交接浪费所有者时间，并可能保留错误前提。

### 阶段 6：从最终记录生成目标中立的报告

仅在 `findings.json` 中保留的每条记录通过阶段 5 后，从最终记录、账本和保留在账本簿记中的搜寻器 `hardening` 注释派生散文。不完整的运行可以报告独立验证的记录，但必须识别每个未解决的账本候选，且不得将其呈现为 finding。散文文件绝不改变裁决、严重性、阻塞因素或已证明影响。

#### `REPORT.md`

写入：

1. 运行配置文件、范围、预算（如设置）及已用与计划代理数、源引用、沙箱化仅源和本地执行声明、先前运行使用，以及明确推迟和范围外覆盖。命名沿用的同源确认和变更源重新验证。`quick`、限定范围、预算受限或不完整的运行明确声明其为部分通过。如果候选验证耗尽了严格预算，声明运行不完整并列出每个未验证指纹和关联单元；不要将这些候选描述为 findings。如果预算阻止了强制批判，声明哪个批判未运行，且不做干净覆盖声明。
2. 一段简短的安全态势摘要。
3. 确认 findings 表：严重性、标题、受影响边界和一行观察结果。
4. 每个确认 finding：仓库源位置、较低信任主体、目标本机有界复现、条件、实际结果、影响、优先级理由和最小源修复。
5. 单独的 `NEEDS VALIDATION` 表。给出每个线索的标题、仓库跟踪、确切阻塞因素、有界本地下一步和安全的所有者观察部署检查。不要分配严重性或称其为确认漏洞。
6. 单独的加固注释和正面源模式。
7. 来自账本的覆盖摘要：已覆盖、候选、阻塞和推迟计数，以及重要排除项和最终批判结果。

不要将拒绝的记录描述为 findings。仅当它们解释先前分歧或覆盖决定时才提及其指纹。

#### `FINDINGS-DETAIL.md`

对于每个确认的 `medium`、`high` 或 `critical` 记录，复制完整源路径和目标中立本地复现：

- 有序的仓库相对跟踪和证据；
- 虚拟攻击者/主体和受影响的虚拟资源；
- 本机输入、调用或夹具及确切有界指令；
- 观察到的输出及其证明的安全不变量；
- 条件和遏制；
- 源级修复和回归用例。

#### `NEEDS-VALIDATION.md`

对于每条未解决的记录，复制源跟踪、已验证证据、确切阻塞因素、受影响边界，以及每个适用的有界本地或所有者观察的解决计划。将这些保留为优先线索，不分配严重性。不要将其转化为实时测试指导或假设缺失的部署事实。

HTTP 是一种可能的原生接口，而非默认。库 finding 可能使用函数调用，解析器使用夹具，CLI 使用命令，桌面应用使用 IPC 或文件操作，基础设施使用本地渲染的策略。不要要求目标不具备的端点、外部账户或实时环境。

保持报告与证据成比例。干净的运行可能有零条确认记录。声明该结果和剩余覆盖/验证限制，而不发明 LOW findings。