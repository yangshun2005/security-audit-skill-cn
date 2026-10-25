

> **[中文化] security-audit-skill**
>
> 此项目是 [cloudflare/security-audit-skill](https://github.com/cloudflare/security-audit-skill) 的中文翻译版本。
> - 原项目 Stars: 45592
> - 主语言: JavaScript
> - 许可证: MIT
> - 翻译日期: 2026-10-25
> - 原始 README: [README_en.md](README_en.md)
>
> 说明：下方 README 正文为**原项目文档的中文翻译**，力求忠于原文；本页顶部的「中文快速上手」为**面向中文用户的补充导读**（非原文内容），目的是让你不用读完整篇 README 就能先把项目跑起来。
>
> 如有翻译不准确之处，欢迎提 Issue 或 PR。

---


## 🚀 中文快速上手（补充导读，非原文）

### 这是什么 / 解决什么问题

`security-audit` 是一个给编码智能体（coding agent）用的「技能」（skill），装上之后可以让你的 agent 扮演安全审计员，对代码库做结构化的漏洞审计。它把审计拆成六个阶段：侦察、按覆盖清单狩猎、候选验证、结构化输出、独立记录复核、目标无关的报告生成。它解决的是「让 AI 做安全审计时流程散乱、结论不可信」的问题——通过隔离的 agent 分工、对抗式验证（查的人不是找的人）和可校验的 JSON 记录，把审计结果沉淀成可复核的产物。项目源自 Cloudflare 的漏洞发现框架，是其单仓库起点版本。

### 快速上手（最短路径）

1. 用 Skills CLI 安装该技能（照抄原文命令）：

   ```bash
   npx skills add https://github.com/cloudflare/security-audit-skill \
     --skill security-audit
   ```

   如需用户级安装，加 `--global`：

   ```bash
   npx skills add https://github.com/cloudflare/security-audit-skill \
     --skill security-audit \
     --global
   ```

2. 如需了解 agent 选择和非交互选项，运行 `npx skills --help`。

3. 在你的编码智能体中打开（或指向）要审计的代码库。

4. 直接向 agent 提出审计请求，例如：

   ```
   security audit this codebase
   ```

   或指定目录 / 输出位置：

   ```
   find security vulnerabilities in ./src
   ```

   ```
   do a security review, output to ~/audits/my-project
   ```

5. 当请求匹配触发词（security audit、find vulnerabilities、pen-test the code 等）时技能会自动激活。完整审计模式下若未指定输出目录，默认写到 `~/security-audit-skill/<repo-name>/run-<N>`。

### 环境要求

- **操作系统**：原文未说明具体支持哪些操作系统。
- **硬件**：原文未说明。
- **依赖**：
  - 一个支持工具调用（tool use）和并行子 agent 的编码智能体及模型；
  - Node.js（用于零依赖的 findings 与 coverage-ledger 校验脚本）。
- **沙箱要求**：需要一个由操作系统强制执行的沙箱，用于运行目标代码的构建、测试、进程、浏览器、模拟器、fuzzer 和 fixture。该沙箱必须禁用外部网络、使用净化后的白名单环境、强制资源限制，并且只允许写入指定的临时路径。若缺少这些控制，工作流会把相关线索保留为 `needs_validation`，而不执行目标代码。
- **版本要求**：原文未说明 Node.js 的具体版本。
- **国内网络提示**：安装通过 `npx skills add` 从 GitHub 拉取，国内网络可能需要配置 npm 镜像源或使用代理加速。

### 常见坑与注意事项

- **必须有 OS 级沙箱**：没有满足上述条件的沙箱时，涉及执行目标代码的环节不会真正运行，相关结论只能停留在 `needs_validation`，审计深度会受限。
- **依赖支持并行子 agent 的模型**：该技能依赖并行子 agent 做隔离分工，模型或 agent 不支持并行子 agent 时无法按设计运行（原文未说明降级方案）。
- **输出目录默认位置**：完整审计模式下不指定输出目录会写到 `~/security-audit-skill/<repo-name>/run-<N>`；只有在显式选择被版本控制忽略的目录时，工作流才会写入目标仓库内部。
- **单次运行覆盖不全**：原文明确说明，测试中单次运行大约只找到重复运行所能发现漏洞总量的一半，多次运行是叠加的，建议对同一仓库多次运行以提升覆盖率。

---



---

# security-audit

一个编码智能体技能，可将你的智能体变成安全审计员。它通过侦察、覆盖驱动的狩猎、候选验证、结构化输出、独立记录核验和目标中立报告来编排隔离的智能体。

这个技能催生了 Cloudflare 的漏洞发现框架，详见 [构建你自己的漏洞框架](https://blog.cloudflare.com/build-your-own-vulnerability-harness)。该框架后来发展成多阶段、全舰队范围的系统；本技能是它演化而来的单仓库起点。

## 功能

该技能分六个阶段运行结构化审计：

1. **侦察** -- 在 `architecture.md` 和 `coverage-ledger.json` 中梳理架构、信任边界、输入面、既有证据和确定性覆盖。
2. **覆盖驱动的狩猎** -- 从账本单元分配隔离的狩猎者，记录其检查，并用覆盖评审者找出缺口。
3. **候选验证** -- 将每个唯一候选交给全新的验证者，由其尝试证伪。
4. **结构化输出** -- 将 `confirmed`、`needs_validation` 和 `rejected` 记录写入 `findings.json`，并对照 `report-schema.json` 校验。
5. **独立记录核验** -- 全新智能体核验最终来源声明。实质性替换需再经一名独立验证者。
6. **目标中立报告** -- 从已核验的记录和覆盖账本生成 `REPORT.md`、`FINDINGS-DETAIL.md` 和 `NEEDS-VALIDATION.md`。

父级在创建账本后以及之后每次账本更新后运行 `validate-coverage-ledger.cjs`。它在阶段 4 运行 `validate-findings.cjs`，并在阶段 5 每次替换后再次运行。

三种判定各不相同：`confirmed` 有完整的来源追踪和有界的观测结果，`needs_validation` 有确切未解决的事实且无严重性，`rejected` 记录被证伪的候选。

对同一仓库的多次运行是累加的。该技能利用先前的账本和发现来定位缺口、重新验证变更的来源，并延续当前来源的证据，而不将过时或未解决的工作视为已覆盖。

## 文件

| 文件 | 用途 |
|------|---------|
| `SKILL.md` | 设置、核心原则、平台术语、工作流概览和审计反模式 |
| `RECONNAISSANCE.md` | 阶段 1 侦察提示和综合说明 |
| `HUNTING.md` | 阶段 2 编排、狩猎方法和验证规则 |
| `ATTACK-CLASSES.md` | 核心、通配和显而易见事项的攻击提示 |
| `MEMORY-SAFETY-AND-BINARY.md` | 面向原生目标的内存安全、二进制和内核狩猎类别 |
| `AI-AND-LLM.md` | 面向 LLM 支撑目标的提示注入、智能体/工具和输出处理狩猎类别 |
| `WEB-PROTOCOL-AND-AUTH.md` | 面向 HTTP 协议和认证目标的 HTTP 请求分帧、缓存和认证协议狩猎类别 |
| `CLIENT-SIDE.md` | 面向客户端/浏览器目标的 DOM 注入、消息信任、UI 伪装和原型污染狩猎类别 |
| `SUPPLY-CHAIN-AND-RELEASE.md` | 依赖、CI、发布、签名、更新、插件和扩展狩猎类别 |
| `CLOUD-AND-DEPLOYMENT.md` | IAM、基础设施即代码、容器、无服务器、入口和运行时配置狩猎类别 |
| `PROTOCOLS-RPC-AND-MESSAGING.md` | RPC、序列化、队列、代理、Webhook 和流式协议狩猎类别 |
| `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md` | 共享资源、配额、队列、工作进程和运营支出狩猎类别 |
| `DATA-ISOLATION-AND-LIFECYCLE.md` | 租户隔离、缓存、搜索、导出、备份、迁移、删除和恢复狩猎类别 |
| `DESKTOP-MOBILE-AND-LOCAL-IPC.md` | 原生应用、深链、webview、导出组件、辅助进程、守护进程和本地 IPC 狩猎类别 |
| `VALIDATION-AND-REPORTING.md` | 阶段 3–6 候选验证、结构化输出、记录核验和报告 |
| `report-schema.json` | 三种 `findings.json` 判定的 JSON Schema |
| `validate-findings.cjs` | 阶段 4 和 5 中 `findings.json` 的零依赖校验器 |
| `validate-findings.test.cjs` | 发现校验器测试和生产者兼容的夹具检查 |
| `validate-coverage-ledger.cjs` | 阶段 1–5 中 `coverage-ledger.json` 的零依赖校验器 |
| `validate-coverage-ledger.test.cjs` | 覆盖账本校验器测试 |

## 安装

使用 [Skills CLI](https://skills.sh) 安装该技能：

```bash
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit
```

使用 `--global` 进行用户级安装：

```bash
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit \
  --global
```

运行 `npx skills --help` 查看智能体选择和非交互选项。

## 用法

在要审计的代码库中（或指向该代码库）启动编码智能体，然后让它执行安全审计：

```
security audit this codebase
```

```
find security vulnerabilities in ./src
```

```
do a security review, output to ~/audits/my-project
```

当请求匹配其触发条件（安全审计、查找漏洞、对代码进行渗透测试等）时，该技能会自动激活。直接的代码库审计或渗透测试请求使用完整审计模式。安全问题及聚焦的漏洞工作使用指导模式，除非你要求生成报告产物。在完整审计模式下，未指定输出目录时默认为 `~/security-audit-skill/<repo-name>/run-<N>`。仅当你显式选择版本控制忽略的目录时，工作流才会写入目标仓库内部。

## 要求

- 支持工具使用和并行子智能体的编码智能体
- 用于零依赖发现和覆盖账本校验器的 Node.js
- 用于目标控制的构建、测试、进程、浏览器、模拟器、模糊测试器和夹具的操作系统强制沙箱。它必须禁用外部网络、使用经清理的允许列表环境、强制执行资源限制，并仅允许写入分配的临时路径。缺少这些控制时，工作流会将线索保持为 `needs_validation`，而不执行目标代码。

## 设计原则

- **仅确认已确立的边界失效。** 将有来源依据的受阻线索保持为 `needs_validation`，并附其确切未解决的事实。
- **对抗性验证。** 检查发现的智能体绝不是发现它的智能体。
- **严重性需要影响。** 可能性 × 影响，而非偏离检查清单。
- **纵深防御缺口不是漏洞。** 如果 A 层阻止了攻击，B 层的缺失只是加固说明。
- **多次运行提升覆盖。** 在我们的测试运行中，单次运行发现的漏洞约为重复运行总计发现量的一半。

## 联系方式

问题、反馈或关于 AI 驱动安全工具的交流：security-ai-research@cloudflare.com

## 许可证

MIT -- 见 [LICENSE](LICENSE)。