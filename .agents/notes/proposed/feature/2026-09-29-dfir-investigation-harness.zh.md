# Agent Note: 不含中国托管服务与遥测的 DFIR 调查 harness

Status: proposed

[English](2026-09-29-dfir-investigation-harness.md) | 中文

## 问题

本 harness 以通用编程 agent 的形态发布，其中四项默认设置与事件响应和数字取证工作相冲突。

1. **工具面可以改变现场。** `dsh-base` 挂载了 `tool-bash`、`tool-pwsh`、`tool-fs` 的写入与编辑、终端、`ptc-runtime`、`plugin-manager`、`hmr`、`config-editor` 以及由模型编写的插件挂载。调查人员需要对证据的只读访问，而 agent 不得改动证据、harness 或自身配置。
2. **Session 日志不具备证据级别。** 日志只追加并以带校验和的帧存储，但没有摘要把一个事件与下一个事件相连，没有签名覆盖，也没有命令可以验证。附件按 sha256 内容寻址，但没有任何记录说明谁在何时从何处获取了某个条目。也没有任何东西对案件、发现、时间线条目或入侵指标建模。
3. **案件数据默认离开本机。** `session-telemetry-otel` 导出到 DeepSeek 所有的 OTLP 收集器，`session-log-deepseek` 把规范日志附加到官方请求上，`product-analytics` 上报桌面事件，桌面更新策略检查发送客户端元数据，并且每个 profile 都会创建匿名用户 id。事件数据是保密的，往往还处于法律保全之下，因此这些路径中的任何一条都构成泄露。
4. **默认模型与支持服务由中国托管。** 默认路由是位于 `api.deepseek.com` 的 `deepseek-official`，网页搜索使用同一个 API，账户调用发往 `platform.deepseek.com`，插件管理器回退到 `registry.npmmirror.com`，桌面版本通过腾讯 COS 分发，联系表单托管在飞书上。`llm-pi-ai` 目录还暴露了智谱、Moonshot、MiniMax、Qwen 等由中国托管的路由。处理受监管或敏感事件数据的组织需要排除这些依赖，并证明它们不存在。

## 方案

DFIR harness 是一个新的 profile 与 bundle，外加一小组调查 package。本 note 不改动通用 bundle。硬性要求（无遥测、无中国托管端点、不写入证据）通过移除插件和拒绝加载来强制执行，而不是通过可以被改回去的配置开关。

### Profile 与 bundle

- 新的 bundle package `packages/bundle/dfir` 在 `dsh-base` 之后叠加，并移除或替换下面列出的行。名为 `dfir` 的 profile 在 `dsh.profile.bundles` 中列出它。
- 该 profile 只通过 `dsh --profile dfir` 启动，符合[应用启动](../../../../docs/architecture.zh.md)的要求。headless 变体用于脚本化分诊，web 变体用于交互式调查。
- 组合检查 `verify-dfir-profile-graph` 会解析该 profile，当下文移除清单中的任意行，或移除中国托管依赖中拒绝清单上的任意 package 出现在图中时失败。同一项检查在 profile 加载时运行，因此用户补丁重新加入被移除的行时，会被拒绝并在错误中给出该行的 id。

### 调查领域 package

新的 package 组 `packages/dfir/` 承载该领域。每个 package 遵循[新增 package](../../../../docs/cookbook/adding-a-package.zh.md)与[新增工具](../../../../docs/cookbook/adding-a-tool.zh.md)两份 cookbook，包括每个工具结果的展示。

| Package | 职责 | 工具 |
|---|---|---|
| `dfir-case` | 案件记录：id、标题、范围、响应人员、开启与关闭时间。一棵 Session 树属于一个案件。 | `case_open`、`case_note` |
| `dfir-evidence` | 建立在 `attachment-local` 之上的证据登记。每个条目记录 sha256、字节长度、来源主机或路径、获取时间、获取用户和自由文本的处理说明。条目登记后不可变。 | `evidence_add`、`evidence_list`、`evidence_read` |
| `dfir-timeline` | 有序的时间线条目，含事件时间、来源证据 id 以及偏移或行引用，统一使用 UTC。 | `timeline_add`、`timeline_query` |
| `dfir-finding` | 带置信度的发现与假设。发现必须引用至少一个证据 id 和位置，否则工具将其标记为 `hypothesis`。入侵指标有类型（哈希、IP、域名、路径、注册表键、账户）。 | `finding_record`、`ioc_extract` |
| `dfir-report` | 把案件、发现和时间线渲染为 Markdown 与 JSON 报告，并通过 `tool-present` 声明。每条陈述都带证据引用。 | `report_generate` |

每次写入都经过 Session 事件，因此“模型可见即已记录”的规则成立。新的事件类型在读取时必需，`SessionEventMap` 与两个 SDK 的投影在同一变更中更新。

### 取证安全的工具策略

- 证据根目录在 harness 启动之前，在操作系统层面以只读方式挂载。`dfir` bundle 将其记录为前置条件，当 headless runner 能够写入已配置的证据根目录时，拒绝启动。
- bundle 固定使用 `read-only` 权限预设。`fs-sandbox` 以 `FS_SANDBOX_DENIED` 拒绝单一案件输出目录之外的所有变更。模型无法请求更宽的模式，因为 bundle 移除了 `sandbox-policy` 中的升级请求路径。
- `tool-bash` 与 `tool-pwsh` 被 `tool-dfir-exec` 取代，后者运行一份只读分析程序的允许清单（例如 `strings`、`file`、`xxd`、`sha256sum`、`grep`、`jq`、`sqlite3 -readonly`、`tshark -r`、`vol`、`plaso`）。允许清单由执行操作强制，而不是由工具 schema 强制，每次调用都记录其参数向量与退出状态。该允许清单是经过校验的 `Config` 字段。
- bundle 移除 `plugin-manager`、`tool-plugin-manager`、`hmr`、`config-editor`、`settings` 写入、`tool-cordis` 创建者模式、`skill-filesystem` 写入、`schedule`、`webhook`、`hooks`、`goal-round-driver`、`tool-goal`、`subagent-fork` 与 `tool-ralph`。子 agent 仍可用于并行的只读分析，并继承父级策略。
- `ptc-runtime` 与 `workflow-ptc` 仅在只读沙箱且无网络访问的条件下保持挂载。
- `credentials-local` 无法把机密与 agent 隔离。bundle 从不在其中存储案件凭据，远程获取使用运维人员提供的只读挂载。

### 监管链

- 每个 Session 事件增加一个可选的信封字段 `prev`，保存前一事件规范字节的 SHA-256。该字段是 `ignorable`，因此不认识它的构建仍可读取日志，只有结构性格式变更才会提升 `SESSION_FORMAT_VERSION`。`dfir` profile 在每个事件上写入它，验证器会拒绝任何 `prev` 缺失或错误的日志。
- 新命令 `dsh dfir verify <session>` 遍历该链，重新计算每个被引用附件的哈希，并报告第一处分歧。可选的 Ed25519 签名覆盖最终摘要，使用运维人员持有、从不交给 agent 进程的密钥，在案件关闭时封存日志。
- `dsh dfir export <session>` 生成案件包：Session 树、附件、证据登记、报告、摘要清单和签名。该路径无需 Web UI 即可工作，并复用 `session-log-export` 的归档写入器。
- `dfir` profile 的 spill 保留期不设上限，因此日志事件引用的原始工具输出始终可以取回。压缩、工具结果裁剪和图像卸载只改变模型请求，并有一个测试断言启用与否日志字节完全一致。
- `workspace-changes` 在证据根目录和案件输出目录上运行，证据根目录中出现非空变更集会使该轮次失败并被记录。

### 调查方法

- bundle 提供以下 skill：分诊顺序、时间线构建、Windows 事件与注册表工件、Linux 认证与审计日志、网络抓包审查、邮件头与钓鱼分析，以及云审计日志。
- agent 指令要求证据优先的推理：每条事实性陈述都引用证据 id 和位置，其余一律标为假设，绝不把推断表述为观察到的事实。
- 模型不会访问案件之外的内容。网页抓取与搜索默认不存在，因为抓取的内容会污染案件记录，且每次请求都是出站流量。运维人员可以在 profile 补丁中添加带主机允许清单的抓取 provider，图检查会在输出中报告这一选择加入。

### 移除中国托管依赖

`dfir` profile 不包含下列任何一项，`verify-dfir-profile-graph` 与新的仓库门禁 `verify-no-prohibited-endpoints` 会拒绝它们。

| 领域 | 当前依赖 | 处理 |
|---|---|---|
| 默认模型 | `llm-deepseek`、`llm-deepseek-api-key`、`llm-deepseek-account`、路由 `deepseek-official`、`DEEPSEEK_API_KEY`、`DEEPSEEK_BASE_URL`、`api.deepseek.com` | 从 `dfir` bundle 中移除。默认路由改为运维人员在 `llm-pi-ai` 上配置的路由，不带内置默认值，因此缺失时在加载阶段失败。 |
| Provider 目录 | `llm-pi-ai` 路由 `deepseek`、`zai*`、`moonshotai*`、`kimi-coding`、`minimax*`、`qwen-token-plan*`、`xiaomi*`、`ant-ling` | 在 `dfir` 组合中加入 provider 拒绝清单，在路由注册与加载时强制执行。自托管的 OpenAI 兼容端点仍然允许，包括气隙环境中的端点。 |
| 账户与平台 | `deepseek-account`、`deepseek-account-platform`、`platform.deepseek.com` | 移除。 |
| 网页搜索 | `web-search-deepseek` | 移除。见调查方法。 |
| 请求元数据 | `deepseek-llm-api-extensions`、`plugin-package-inventory-deepseek` | 移除。 |
| Package 注册表 | `plugin-manager` 中的 `NPMMIRROR_REGISTRY` 回退、`ui-plugin-manager` 中的注册表探测、`scripts/wine-windows-gates.sh` 中的镜像回退 | `dfir` profile 没有插件管理器。仓库门禁拒绝所有源码、配置和脚本路径中的 `npmmirror` 主机。 |
| 桌面分发 | 腾讯 COS（`cos-nodejs-sdk-v5`、`desktop-cos*`）、`download.deepseek.com`、飞书测试登录 | 不在 `dfir` 范围内。桌面应用不属于该 profile。 |
| 支持链接 | 飞书 `contactFormUrl` | 对 `dfir` 设为空。该值为空时，客户端隐藏联系入口。 |

拒绝清单是一份由源码持有的主机、环境变量名和 provider id 列表，图检查与仓库门禁共用。第三方依赖内部的端点（例如 `@earendil-works/pi-ai` 中的端点）由 provider 拒绝清单覆盖，因为它们的主机不在本仓库中。

### 遥测移除

`dfir` profile 除访问运维人员配置的模型端点外，不发起任何网络调用。

- 从 `dfir` 图中移除 `otel`、`session-telemetry-otel`、`session-log-deepseek`、`desktop-product-telemetry`、`product-analytics`、`product-telemetry-otel` 与 `anonymous-user-id`。反馈命令保留为本地 Session 事件，没有上传路径。
- 从 `dfir` profile 上下文中移除 `DSH_TELEMETRY_*` 与 `DSH_PRODUCT_ANALYTICS_*` 的处理，并且不再把 `DSH_TELEMETRY_DISABLED` 当作开关。移除使该设置失去意义，因此任何环境变量取值都无法启用收集器。
- 运行时不变量与崩溃报告保留为 `$DSH_HOME` 下的本地文件。
- `verify-dfir-profile-graph` 会拒绝任何其 package 声明了出站 OTLP、分析或更新检查端点的插件，方法是让每个有出站调用的插件在 package manifest 中声明明确的 `dsh.egress` 字段。没有该字段的插件被视为不发起出站调用，`dfir` 组合测试中的网络测试会强制验证这一声明。
- `sandbox` 网络策略拒绝除配置的模型主机之外的所有出站连接。组合测试使用记录型解析器和已录制的模型启动 headless runner，运行完整的调查快照，并断言对任何其他主机的连接尝试为零。

### 兼容性

- 公共 API 处于预稳定阶段，因此被移除的行和环境变量在 `dfir` 中无需兼容垫片。通用 bundle 保持当前行为，直到另一份 note 将其退役。
- 新的事件类型与 `prev` 字段会更新两个 SDK 的预期输出和 session 格式状态文档。`prev` 是 `ignorable`，无需提升版本。
- 新的升级指南条目记录 `dfir` profile、新命令以及新的 `dsh.egress` manifest 字段。
- 一份录制的无密钥 session 快照通过已发布的 `dfir` profile 覆盖端到端调查：登记证据、构建时间线、记录发现、生成报告、验证并导出。

## 备选方案

**仅配置的 profile。** 在补丁中禁用行不会从仓库中移除任何内容，之后的补丁可以重新启用它们。图检查与加载拒绝使这些约束对每一种组合都成立，因此仅补丁的 profile 被否决。

**fork 仓库并删除 package。** fork 移除了代码，但失去上游更新，并重复了 session、沙箱和附件方面的工作。bundle 方式保持单一代码库。物理删除中国托管与遥测 package 仍是后续工作，待通用 profile 不再需要它们之后进行。

**遥测关闭环境变量。** `DSH_TELEMETRY_DISABLED` 已经存在。开关只在被设置时才起保护作用，而且合规声明的审阅者看不到它。移除插件并断言出站清单为空，可以从解析后的图和网络测试中验证。

**用 Claude Code 钩子做策略。** 钩子桥接在工具调用构造之后才起作用，并且属于无人值守自动化，而 `dfir` profile 已将其移除。在执行操作中强制策略才是本地约定。

**加入带审批的写入权限。** 审批提示会让操作人员养成盲目点击的习惯。必须修改数据的分析应在案件输出目录中的副本上进行。

## 验收标准

- 解析后的 `dfir` 图不包含移除清单中的任何行或 package，也不含任何被拒绝的主机、环境变量或 provider id，且 `verify-dfir-profile-graph` 在补丁重新加入其中之一时失败。
- 录制的调查快照除已配置的模型端点外，不向任何主机建立网络连接。
- 在未配置模型路由的情况下加载 `dfir` profile 会失败，并给出指明缺失路由的信息。
- 通过执行器对证据根目录下的写入、重命名或删除，以 `FS_SANDBOX_DENIED` 失败，允许清单之外的 shell 命令不经运行即被拒绝。
- `dsh dfir verify` 接受未被改动的日志，并在改动一个事件、一个附件字节或事件顺序时报告第一个分歧事件。
- 在另一台机器上仅凭公钥即可验证 `dsh dfir export` 生成的案件包。
- 没有证据引用的发现被存为假设，报告也将其标为假设。
- 启用与关闭压缩、工具结果裁剪和图像卸载时，Session 日志字节完全一致。
- 两个 SDK 的预期输出、session 格式状态文档和升级指南均已更新，并且 `pnpm run test`、`pnpm run typecheck`、`pnpm run lint`、`pnpm run doc-sync` 与快照套件通过。

## 风险

- [SAFETY.md](../../../../SAFETY.zh.md) 声明本 harness 是实验性的且未经审计。哈希链只能事后显示篡改，并不能使 harness 成为可信的取证工具。本 note 不对法律上的可采性作任何主张。
- 案件数据仍会到达运维人员配置的任何模型端点。移除中国托管服务并不会使第三方模型 provider 变得安全。自托管的气隙模型是唯一没有数据离开现场的路线，代价是分析质量有所下降。
- `read-only` 沙箱限制文件效果，不限制读取或网络。这些保证由网络策略和操作系统挂载承担，缺失挂载属于部署故障，runner 必须在启动时检测到。
- `dsh.egress` 字段只是一项声明。发起未声明调用的插件能通过图检查，只有网络测试才能发现。该测试的覆盖度取决于录制的快照是否覆盖每一个已挂载的工具。
- 执行允许清单限制的是程序，而不是程序的行为。`sqlite3` 或 `tshark` 之类的程序在只读沙箱下仍可能读取证据根目录之外的内容，因此必须把允许清单作为案件环境的一部分进行审阅。
- provider id 拒绝清单会随着 `@earendil-works/pi-ai` 增加路由而过时。组合检查在加载时读取目录，因此未列出的新中国托管路由必须通过审阅目录更新来发现。
- 哈希链为每个事件增加一次哈希，并为每次导出增加一遍验证。在把 `dfir` profile 设为大型案件的默认之前，长 Session 需要先做测量。
