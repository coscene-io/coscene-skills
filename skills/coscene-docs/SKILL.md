---
name: coscene-docs
description: >
  Use when answering questions about the coScene platform — what Organization/Project/
  Record/Moment mean, how automation and actions work, how the data model connects,
  or when debugging coScene issues. Grounds answers in the official documentation
  corpus — paired with the cos skill for CLI execution.
---

# coScene Docs

> Platform concepts, automation model, and troubleshooting for coScene — the scenario-oriented robotics data platform.

## Bundled References

Read these on-demand — not on every invocation.

| File | Read when... |
|---|---|
| `./platform-architecture.md` | Deep dive on data model, device system, storage, API surface |
| `./automation-guide.md` | Setting up automation — triggers, action runtime, env vars, record.patch.json |
| `./troubleshooting.md` | Diagnosing failures: auth, upload, action execution, device connectivity |

## Sister Skill

**cos** — CLI execution for the coScene platform via cocli.

Load `cos` when the user wants to execute operations — uploading data, querying
records, running actions, managing profiles. This skill (`coscene-docs`) handles
understanding; `cos` handles doing.

- "How do I upload files to a record?" → load `cos`
- "Run this action on my record" → load `cos`
- "Set up my cocli profile" → load `cos` (reads `setup.md` for bootstrap)

## Agent Preferences

On first interaction, check for user preferences:

```bash
cat ~/.coscene/agent-prefs.md 2>/dev/null
```

If the file exists, adapt:
- **Language**: respond in the user's preferred language (zh/en). If `auto-detect`, infer from user's message language.
- **Communication style**: `concise` = terse, direct answers. `explainer` = add context, cite doc sections. `beginner` = walk through concepts step-by-step with examples.

If the file is missing, use defaults (English, concise) and continue without prompting. Do not block on missing preferences.

## Concept Glossary

### Organization (组织)

Top-level container representing a company or team. Organizations manage multiple
projects and members, with organization administrators controlling user permissions,
data access, billing, device fleet management, and the container image registry. All
platform resources — projects, devices, images — are scoped under exactly one
organization. Organization-level roles: Admin (full control), Member (assigned
project access), Viewer (read-only per visibility).

### Project (项目)

The minimal permission management unit within an organization. A project groups
related records, devices, moments, automation rules, and actions into one workspace.
Visibility can be private (members only), internal (org-wide), or public. Projects
are the RBAC boundary — membership and role determine what a user can read, write,
or administer. Actions and triggers are project-scoped; a trigger can only reference
actions within the same project.

### Record (记录)

The core data container in coScene. A record holds ALL files describing a specific
scenario — ROS bags, MCAP files, logs, maps, configuration files, images, videos.
Records belong to a project, inherit its visibility, and carry labels and custom
fields for organization and filtering. A record is NOT a single file; it is a
collection of multimodal files for one event or data collection session. Records
can be archived (read-only), copied, or moved across projects.

### Moment (一刻)

A timestamped annotation marking a key time segment within a record. Moments
represent critical events — failures, anomalies, notable occurrences — during data
collection. Each moment has a name, description, start/end timestamps, and optional
structured attributes. Created manually during visualization playback or
automatically by rule engine triggers on the device edge. Moments are NOT comments;
they are anchored to sensor data timelines with precise time ranges, used for data
diagnosis and quality management.

### Action (动作)

A building block of the automation system representing a specific processing task.
Actions execute through two mechanisms: container-based code execution (Docker images
with record files mounted) or HTTP webhook requests. Each action has one or more
steps, supports `{{parameter.key}}` interpolation for configuration, and runs with
configurable resources (1-8 CPU cores, 2-16GB memory). Actions are authored
exclusively in the platform UI — there is no CLI or API for creating or editing
action definitions. System-provided built-in actions (e.g., "Decompress File") are
also available.

### Device / coScout (设备 / 端助理)

A physical or logical entity registered on the platform that collects and uploads
data. Devices are managed at the organization level and allocated to projects as
needed. The coScout (端助理) edge agent is pre-built client software running on
devices to handle: data capture from configured directories, SHA256-based file
deduplication, rule-based trigger evaluation (rules pushed every 60 seconds),
compression, filtering, and upload orchestration. Device configuration includes: SN
registration, listener directories (`listen_dirs`) for rule monitoring, collection
directories (`collect_dirs`) for data capture, and topic configurations for ROS/MCAP
streams.

### Label (标签)

User-defined categorization applied to records for organization, filtering, and
automation. Labels can filter record lists in the UI, serve as matching conditions
in file-upload triggers (glob + label match), and logically group records across a
project. Records can carry multiple labels. Critical constraint: changing labels on
an existing record does NOT retroactively fire file-upload triggers — only new file
uploads and field mutations activate triggers.

### Rule Engine (规则引擎)

Cloud-based event-driven automation for data collection and processing at the
device/edge layer. The rule engine monitors device logs and ROS/MCAP data streams
for triggering conditions (topic messages, log keywords, file updates, event codes).
Rules are defined at the project level, pushed to edge devices every 60 seconds for
decentralized evaluation. Supports CEL expressions for complex event matching,
parametric event code tables (CSV/JSON upload), time-based aggregation, deduplication
windows (1-86400 seconds), and variable interpolation using `{scope.*}` and
`{msg.*}` syntax. Distinct from project-level triggers — the rule engine operates
on the device edge.

### Trigger (触发器)

A conditional rule specifying when a project action should execute. Five supported
event types: file upload (glob pattern + uploader role + record label match), device
collection status change (manual/rule completion), record change (label or custom
field mutation), generic task status change, and cron schedule (standard 5-field
syntax, UTC). Triggers reference project-scoped actions, can be enabled/disabled
via the UI, and support system-provided built-in actions. Trigger configuration is
UI-only — no API for creating or modifying triggers.

### Invocation (调用)

A single execution instance of an action. Triggered manually via UI, by triggers,
or via API. During execution, record files mount to `/cos/files` (read-only or
read-write), outputs to `/cos/outputs`, with optional `/cos/local_disk` for
high-speed temporary storage. Each invocation receives `COS_TOKEN` inheriting the
full permissions of the trigger initiator — cross-project operations require the
initiator to have target project access. Invocations are logged with execution time,
status, stdout/stderr, and results.

## Domain Routing Table

| Question type | Where to look |
|---|---|
| "What is X?" / platform concepts | This skill's glossary above |
| "How does the data model work?" / hierarchy, permissions | `./platform-architecture.md` |
| "How are org/project/record related?" / visibility, RBAC | `./platform-architecture.md` |
| "How do I set up automation / actions / triggers?" | `./automation-guide.md` |
| "What env vars does my action get?" / runtime, mounts | `./automation-guide.md` |
| "How does record.patch.json work?" / chaining actions | `./automation-guide.md` |
| "Why is X broken?" / upload failed / action didn't run | `./troubleshooting.md` |
| "Device offline / rules not pushed / collection stuck" | `./troubleshooting.md` |
| "How do I use cocli?" / CLI commands / API calls | Load the `cos` skill — this skill covers concepts, not CLI |

## Automation Capability Map

### What the platform CAN do

- **Trigger on:** file upload (glob + label match), device collection status, record
  change (label/field mutation), task status change, cron schedule
- **Run containers:** custom org registry images, public Docker Hub images, or
  official coScene images — with record files mounted at `/cos/files`
- **HTTP webhooks:** send requests with built-in variable interpolation
  (`{{task.title}}`, `{{record.link}}`, `{{device.id}}`, etc.)
- **Chain actions:** write `record.patch.json` in `/cos/outputs/records/*/` to
  create, update, or patch records using RFC 6902 JSON Patch format
- **Runtime injection:** 13+ env vars including `COS_TOKEN`, `COS_RECORDID`,
  `COS_PROJECT`, `COS_ENDPOINT`, `COS_FILE_VOLUME`, `COS_OUTPUT_VOLUME`
- **Resource tiers:** 1C/2G, 2C/4G, 4C/8G, 8C/16G per action (contact support
  for custom)
- **Temporary storage:** `/cos/local_disk` mount for high-speed scratch within
  action execution

### What the platform CANNOT do

- **Action authoring via CLI/API** — actions are defined and edited in the UI only
- **Programmatic access to action source** — step definitions, images, and commands
  are not exposed via API
- **Trigger configuration via API** — trigger conditions are set and modified in
  the UI only
- **Retroactive trigger on label change** — label changes on existing records do not
  fire file-upload triggers; must add file or mutate custom field
- **Local execution** — all action execution is containerized cloud; nothing runs on
  the user's machine
- **Dedup windows beyond 1 day** — maximum event deduplication window is 86400
  seconds

### Action Output Convention (record.patch.json)

Actions create or update records by writing to a structured output directory:

```
$COS_OUTPUT_VOLUME/records/<record-name>/.cos/record.patch.json
```

- **Create:** omit `id`, include `title` — platform creates a new record
- **Update:** include `id`, use `patch` array (RFC 6902 JSON Patch)
- **Cross-project:** specify `projectSlug` if different from trigger context
- **File references:** relative paths from record directory to output files

Platform automatically scans output directories after action completion.

## Bilingual Awareness

Documentation lives at `docs.coscene.cn` with zh as primary language and en as
translation. The Chinese source is authoritative; English translations occasionally
lag behind or simplify.

**Routing rules:**
- When the user asks in Chinese, cite Chinese doc sections and use Chinese terms
- When the user asks in English, cite English translations
- When precision matters, prefer the zh source and note any discrepancy

**Key terms for cross-referencing:**

| Chinese | English | Notes |
|---|---|---|
| 组织 | Organization | Top-level container, billing + device scope |
| 项目 | Project | RBAC boundary, action/trigger scope |
| 记录 | Record | Data container (NOT a single file) |
| 一刻 | Moment | Timestamped annotation on sensor timeline |
| 动作 | Action | Automation building block, UI-authored only |
| 设备 / 端助理 | Device / coScout | Edge agent for capture + upload |
| 标签 | Label | Record categorization, trigger filter |
| 触发器 | Trigger | Event-based action activation |
| 规则引擎 | Rule Engine | Edge-layer automation (distinct from triggers) |
| 调用 | Invocation | Action execution instance |
| 镜像 | Image | Container image for action execution |
| 采集 | Collection | Data capture from device directories |
| 手动采集 / 规则采集 | Manual / Rule Collection | Two collection modes |
| 可视化 | Visualization | Data playback and replay UI |
| 工作流 | Workflow | Automation pipeline |
| 事件码 / 事件码表 | Event Code / Event Code Table | Error identifiers for rule matching |
| 监听目录 / 采集目录 | Listen Dir / Collect Dir | Device config directories |
| 自定义字段 | Custom Field | User-defined record metadata |

## Anti-Rationalization

| Temptation | Reality |
|---|---|
| "I can answer this from training data" | NEVER for platform-specific facts. Always ground in bundled references or glossary. coScene is a living product; training data is stale or wrong. |
| "The English and Chinese docs say the same thing" | Usually but not always. zh is the primary source; en translations can lag. When precision matters, check both. |
| "I know how coScene automation works" | CHECK the capability map above. Action authoring and trigger configuration are UI-only — no CLI, no API for definitions. |
| "Record and file are the same thing" | NO. A record is a container. Files live inside records. One record holds many files from one scenario. |
| "Moments are comments" | NO. Timestamped annotations anchored to sensor data timelines, with start/end times and structured attributes. |
| "Actions run on the user's machine" | NO. Containerized cloud environment with mounted record files at `/cos/files` and injected env vars. |
| "Organization/Project is just folders" | NO. RBAC permission boundaries. Org controls billing, devices, registry. Project controls record access, automation scope, member roles. |
| "Labels trigger actions when changed" | NOT directly. Label changes on existing records do not fire file-upload triggers. Only field mutations and new file uploads do. |
| "Rule engine and triggers are the same" | NO. Rule engine = device/edge (CEL, event codes, dedup). Triggers = project/cloud (file upload, cron, status). |
| "I can create actions with cocli" | NO. cocli handles records, files, devices, projects. Action/trigger authoring is UI-only. |

## Common Misconceptions

- **coScene is not just a file store** — it is a scenario-oriented data platform for
  robotics with built-in visualization, automation, device management, and RBAC.

- **Moments are not comments** — they are timestamped annotations on sensor data
  timelines with structured start/end ranges and attributes.

- **Actions are not local** — they execute in containerized cloud environments with
  mounted record files, never on user machines.

- **Org/Project hierarchy is RBAC, not folders** — membership and roles determine
  access; organizations scope billing and devices, projects scope records and
  automation.

- **Rule engine and triggers are different layers** — rule engine is edge/device-side
  (CEL expressions, event codes, dedup windows); triggers are cloud/project-side
  (file upload, cron, status change).

- **cocli cannot author automation** — CLI handles data operations (records, files,
  devices); action and trigger definitions require the platform UI.
