# Agent Note: DFIR investigation harness without China-hosted services or telemetry

Status: proposed

English | [中文](2026-09-29-dfir-investigation-harness.zh.md)

## Problem

The harness ships as a general-purpose coding agent, and four of its defaults conflict with incident response and digital forensics work.

1. **The tool surface can change the scene.** `dsh-base` mounts `tool-bash`, `tool-pwsh`, `tool-fs` write and edit, terminals, `ptc-runtime`, `plugin-manager`, `hmr`, `config-editor` and model-written plugin mounts. An investigator needs read-only access to evidence, and the agent must not alter the evidence, the harness or its own configuration.
2. **The Session log is not evidence-grade.** It is append-only and stored in checksummed frames, but no digest links one event to the next, no signature covers it, and no command verifies it. Attachments are content-addressed by sha256, but nothing records who acquired an item, from where, or when. Nothing models a case, a finding, a timeline entry or an indicator of compromise.
3. **Case data leaves the machine by default.** `session-telemetry-otel` exports to a DeepSeek-owned OTLP collector, `session-log-deepseek` attaches the canonical log to official requests, `product-analytics` reports desktop events, the desktop update policy check sends client metadata, and every profile creates an anonymous user id. Incident data is confidential and often under legal hold, so any of these paths is a disclosure.
4. **The default model and support services are China-hosted.** The default route is `deepseek-official` at `api.deepseek.com`, web search uses the same API, account calls go to `platform.deepseek.com`, the plugin manager falls back to `registry.npmmirror.com`, desktop releases are distributed through Tencent COS, and the contact form is on Feishu. The `llm-pi-ai` catalog also exposes Zhipu, Moonshot, MiniMax, Qwen and other China-hosted routes. Organizations that handle regulated or sensitive incident data need to exclude these dependencies and prove they are absent.

## Proposal

The DFIR harness is a new profile and bundle, plus a small set of investigation packages. General-purpose bundles stay unchanged in this note. The hard requirements (no telemetry, no China-hosted endpoints, no writes to evidence) are enforced by removing plugins and refusing to load, never by a configuration flag that could be flipped back.

### Profile and bundle

- A new bundle package `packages/bundle/dfir` layers after `dsh-base` and removes or replaces the rows listed below. A profile named `dfir` lists it in `dsh.profile.bundles`.
- The profile launches only through `dsh --profile dfir`, as [application launch](../../../../docs/architecture.md) requires. A headless variant serves scripted triage and a web variant serves interactive investigation.
- A composition check, `verify-dfir-profile-graph`, resolves the profile and fails when any row from the removal lists below, or any package on the denylist in China-hosted dependency removal, appears in the graph. The same check runs at profile load, so a user patch that re-adds a removed row is refused with the row id in the error.

### Investigation domain packages

A new package group `packages/dfir/` holds the domain. Each package follows the [adding a package](../../../../docs/cookbook/adding-a-package.md) and [adding a tool](../../../../docs/cookbook/adding-a-tool.md) cookbooks, including presentation of every tool result.

| Package | Role | Tools |
|---|---|---|
| `dfir-case` | Case record: id, title, scope, responders, opened and closed times. One Session tree belongs to one case. | `case_open`, `case_note` |
| `dfir-evidence` | Evidence register on top of `attachment-local`. Each item records sha256, byte length, source host or path, acquisition time, acquiring user and a free-text handling note. Items are immutable once registered. | `evidence_add`, `evidence_list`, `evidence_read` |
| `dfir-timeline` | Ordered timeline entries with an event time, a source evidence id and an offset or line reference, in UTC. | `timeline_add`, `timeline_query` |
| `dfir-finding` | Findings and hypotheses with a confidence level. A finding must cite at least one evidence id and location, or the tool marks it `hypothesis`. Indicators of compromise are typed (hash, IP, domain, path, registry key, account). | `finding_record`, `ioc_extract` |
| `dfir-report` | Renders the case, findings and timeline into a Markdown and JSON report and declares it through `tool-present`. Every statement carries evidence citations. | `report_generate` |

Each write goes through a Session event, so the model-visible and logged rule holds. New event types are required-on-read, and the `SessionEventMap` and both SDK projections are updated in the same change.

### Forensic-safe tool policy

- The evidence root is mounted read-only at the operating system level before the harness starts. The `dfir` bundle documents this as a precondition and the headless runner refuses to start when it can write to a configured evidence root.
- The bundle pins the `read-only` permission preset. `fs-sandbox` denies every mutation outside a single case output directory with `FS_SANDBOX_DENIED`. The model cannot request a wider mode, because the bundle removes the escalation request path from `sandbox-policy`.
- `tool-bash` and `tool-pwsh` are replaced by `tool-dfir-exec`, which runs an allowlist of read-only analysis programs (for example `strings`, `file`, `xxd`, `sha256sum`, `grep`, `jq`, `sqlite3 -readonly`, `tshark -r`, `vol`, `plaso`). The allowlist is enforced by the executing operation, not by the tool schema, and each invocation is recorded with its argument vector and exit status. The allowlist is a validated `Config` field.
- The bundle removes `plugin-manager`, `tool-plugin-manager`, `hmr`, `config-editor`, `settings` writes, `tool-cordis` creator mode, `skill-filesystem` writes, `schedule`, `webhook`, `hooks`, `goal-round-driver`, `tool-goal`, `subagent-fork` and `tool-ralph`. Subagents stay available for parallel read-only analysis and inherit the parent policy.
- `ptc-runtime` and `workflow-ptc` stay mounted only under the read-only sandbox and without network access.
- `credentials-local` cannot isolate secrets from the agent. The bundle never stores case credentials there, and remote acquisition uses operator-provided read-only mounts.

### Chain of custody

- Every Session event gains an optional envelope field `prev` holding the SHA-256 of the previous event's canonical bytes. The field is `ignorable`, so builds that do not know it can still read the log, and only structural format changes bump `SESSION_FORMAT_VERSION`. The `dfir` profile writes it on every event, and a verifier rejects a log where any `prev` is missing or wrong.
- A new command `dsh dfir verify <session>` walks the chain, re-hashes every referenced attachment and reports the first divergence. An optional Ed25519 signature over the final digest, made with an operator-held key that is never given to the agent process, seals the log at case close.
- `dsh dfir export <session>` produces a case bundle: the Session tree, attachments, evidence register, report, a manifest of digests and the signature. This path works without the Web UI. It reuses the `session-log-export` archive writer.
- Spill retention for the `dfir` profile is unlimited, so raw tool output referenced by a log event stays retrievable. Compaction, tool result pruning and image offload change model requests only, and a test asserts that the log is byte-identical with and without them.
- `workspace-changes` runs over the evidence root and the case output directory, and a non-empty change set in the evidence root fails the turn and is recorded.

### Investigation methods

- The bundle ships skills for triage order, timeline construction, Windows event and registry artifacts, Linux auth and audit logs, network capture review, mail header and phishing analysis, and cloud audit logs.
- Agent instructions require evidence-first reasoning: cite an evidence id and location for every factual claim, label everything else as a hypothesis, and never present an inference as observed fact.
- The model does not reach outside the case. Web fetch and search are absent by default because fetched content contaminates the case record and each request is egress. An operator may add a fetch provider with a host allowlist in the profile patch, and the graph check reports that opt-in in its output.

### China-hosted dependency removal

The `dfir` profile contains none of the following, and `verify-dfir-profile-graph` and a new repository gate, `verify-no-prohibited-endpoints`, reject them.

| Area | Current dependency | Action |
|---|---|---|
| Default model | `llm-deepseek`, `llm-deepseek-api-key`, `llm-deepseek-account`, route `deepseek-official`, `DEEPSEEK_API_KEY`, `DEEPSEEK_BASE_URL`, `api.deepseek.com` | Remove from the `dfir` bundle. The default route becomes an operator-configured route on `llm-pi-ai`, with no bundled default so a missing route fails at load. |
| Provider catalog | `llm-pi-ai` routes `deepseek`, `zai*`, `moonshotai*`, `kimi-coding`, `minimax*`, `qwen-token-plan*`, `xiaomi*`, `ant-ling` | Add a provider denylist to the `dfir` composition, enforced at route registration and at load. Self-hosted OpenAI-compatible endpoints stay allowed, including air-gapped ones. |
| Account and platform | `deepseek-account`, `deepseek-account-platform`, `platform.deepseek.com` | Remove. |
| Web search | `web-search-deepseek` | Remove. See Investigation methods. |
| Request metadata | `deepseek-llm-api-extensions`, `plugin-package-inventory-deepseek` | Remove. |
| Package registry | `NPMMIRROR_REGISTRY` fallback in `plugin-manager`, registry probe in `ui-plugin-manager`, mirror fallback in `scripts/wine-windows-gates.sh` | The `dfir` profile has no plugin manager. The repository gate rejects `npmmirror` hosts in every source, config and script path. |
| Desktop distribution | Tencent COS (`cos-nodejs-sdk-v5`, `desktop-cos*`), `download.deepseek.com`, Feishu test login | Out of scope for `dfir`. The desktop app is not part of this profile. |
| Support links | Feishu `contactFormUrl` | Set to empty for `dfir`. The client hides the contact entry when the value is empty. |

The denylist is one source-owned list of hosts, environment variable names and provider ids, shared by the graph check and the repository gate. Endpoints inside third-party dependencies, such as those in `@earendil-works/pi-ai`, are covered by the provider denylist because their hosts are not in this repository.

### Telemetry removal

The `dfir` profile makes no network calls except to the operator's configured model endpoint.

- Remove `otel`, `session-telemetry-otel`, `session-log-deepseek`, `desktop-product-telemetry`, `product-analytics`, `product-telemetry-otel` and `anonymous-user-id` from the `dfir` graph. The feedback commands remain as local Session events with no upload path.
- Remove the `DSH_TELEMETRY_*` and `DSH_PRODUCT_ANALYTICS_*` handling from the `dfir` profile context, and do not read `DSH_TELEMETRY_DISABLED` as the switch. Removal makes the setting irrelevant, so no environment value can enable a collector.
- Runtime invariants and crash reports remain local files under `$DSH_HOME`.
- `verify-dfir-profile-graph` rejects any plugin whose package declares an outbound OTLP, analytics or update-check endpoint, using an explicit `dsh.egress` field in the package manifest that every plugin with outbound calls declares. A plugin without the field is treated as making no outbound calls, and a network test in the `dfir` composition tests enforces that claim.
- A `sandbox` network policy denies all outbound connections except the configured model host. A composition test starts the headless runner with a recording resolver and a recorded model, runs a full investigation snapshot, and asserts zero connection attempts to any other host.

### Compatibility

- Public APIs are pre-stable, so removed rows and environment variables need no shims in `dfir`. General bundles keep their current behavior until a separate note retires them.
- The new event types and the `prev` field update both SDK expected outputs and the session format status document. `prev` is `ignorable` and needs no version bump.
- A new upgrade guide item records the `dfir` profile, the new commands and the new `dsh.egress` manifest field.
- A recorded keyless session snapshot covers an end-to-end investigation through the shipped `dfir` profile: register evidence, build a timeline, record findings, generate the report, verify and export.

## Alternatives considered

**A configuration-only profile.** Disabling rows in a patch removes nothing from the repository and a later patch can re-enable them. The graph check and load refusal make the constraints hold for every composition, so a patch-only profile is rejected.

**Fork the repository and delete the packages.** A fork removes the code but loses upstream updates and duplicates the session, sandbox and attachment work. The bundle approach keeps one code base. Physical deletion of the China-hosted and telemetry packages remains a follow-up once the general profiles no longer need them.

**A telemetry-off environment variable.** `DSH_TELEMETRY_DISABLED` already exists. A switch protects only while it is set, and it is invisible to a reviewer of a compliance claim. Removing the plugin and asserting an empty egress list is verifiable from the resolved graph and from a network test.

**Claude Code hooks for policy.** Hook bridges act after the tool call is built and are unattended automation, which the `dfir` profile removes. Policy enforced in the executing operation is the local convention.

**Add write access with approval.** Approval prompts train operators to click through. Analysis that must change data belongs on a copy in the case output directory.

## Acceptance criteria

- The resolved `dfir` graph contains no row or package from the removal lists and no denylisted host, environment variable or provider id, and `verify-dfir-profile-graph` fails on a patch that re-adds one.
- The recorded investigation snapshot makes no network connection to any host except the configured model endpoint.
- Loading the `dfir` profile with no configured model route fails with a message naming the missing route.
- A write, rename or delete under the evidence root fails with `FS_SANDBOX_DENIED` through the executor, and a shell command outside the allowlist is denied without running.
- `dsh dfir verify` accepts an untouched log and reports the first divergent event when one event, one attachment byte or the event order is changed.
- `dsh dfir export` on one case bundle verifies on a separate machine with the public key only.
- A finding without an evidence citation is stored as a hypothesis, and the report labels it as such.
- The Session log is byte-identical with and without compaction, tool result pruning and image offload enabled.
- Both SDK expected outputs, the session format status document and the upgrade guide are updated, and `pnpm run test`, `pnpm run typecheck`, `pnpm run lint`, `pnpm run doc-sync` and the snapshot suite pass.

## Risks

- [SAFETY.md](../../../../SAFETY.md) states that the harness is experimental and not audited. A hash chain shows tampering after the fact and does not make the harness a trusted forensic tool. This note makes no claim about legal admissibility.
- Case data still reaches whichever model endpoint the operator configures. The removal of China-hosted services does not make a third-party model provider safe. Air-gapped self-hosted models are the only route with no data leaving the site, at some cost in analysis quality.
- The `read-only` sandbox restricts file effects and does not restrict reads or network. The network policy and the operating system mount carry those guarantees, and a missing mount is a deployment fault the runner must detect at start.
- The `dsh.egress` field is a declaration. A plugin that makes undeclared calls passes the graph check, and only the network test catches it. Coverage of that test depends on the recorded snapshot exercising each mounted tool.
- The exec allowlist limits programs, not their behavior. Programs such as `sqlite3` or `tshark` can read outside the evidence root under the read-only sandbox, so the allowlist must be reviewed as part of the case environment.
- A denylist of provider ids goes stale as `@earendil-works/pi-ai` adds routes. The composition check reads the catalog at load, so an unlisted new China-hosted route must be caught by review of catalog updates.
- The hash chain adds a hash per event and a verification pass per export. Long Sessions need a measurement before the `dfir` profile is made the default for large cases.
