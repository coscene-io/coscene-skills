# coScene Automation Guide

## Overview

The coScene automation system connects events to actions through triggers.
The execution model is: an event occurs → a trigger matches → an action
executes → outputs are processed. All action authoring and trigger
configuration should use public product surfaces: cocli, public OpenAPI, or the
platform UI when the capability is not exposed by cocli/public OpenAPI.

```
Event (file upload, collection complete, record change, cron, task change)
    ↓
Trigger (matches event against configured conditions)
    ↓
Action (executes container code or sends HTTP webhook)
    ↓
Output (record.patch.json creates/updates records, chaining further triggers)
```

## Action Anatomy

An action (动作) is the fundamental building block. Each action has:

- **Name** — human-readable identifier
- **Description** — purpose and behavior
- **Labels** — categorization tags for filtering
- **Parameters** — key-value pairs, referenced as `{{parameter.key}}` in
  commands and bodies
- **Steps** — one or more execution steps (container or webhook)

### Parameter Interpolation

Double-brace syntax: `{{parameter.key}}`. Define key `input` with value
`hello world`, reference as `{{parameter.input}}` — platform substitutes at
invocation time. Parameters are project-scoped and reusable.

### Authoring Constraint

Actions are authored through public product surfaces only. Prefer `cocli` for
execution and inspection, use public OpenAPI only for supported public
operations, and use the platform web UI when neither exposes the authoring or
configuration capability. Do not use internal or undocumented APIs in public
skills. System-provided built-in actions (e.g., "Decompress File") are available
without authoring.

## Container Code Execution (镜像代码执行)

### Image Sources

| Source | Example | Notes |
|---|---|---|
| Organization registry | `cr.coscene.io/<org>/<image>:<tag>` or registry returned by `cocli` | Preferred for custom action images; each org has a coScene-provided registry with better pull locality and performance |
| Docker Hub | `docker.io/library/python:3.11-slim` | Public images; useful for quick tests, but org registry is preferred for repeatable action runs |
| Official coScene images | Image shown in public docs or the web UI | Use only when the image is exposed through public product surfaces; do not assume access to private registries |

Use `cocli registry create-credential -o json` or `cocli registry login` to
discover and authenticate the exact registry for the active profile. Do not
guess the registry host; IO and CN environments may differ.

### Action Image Hygiene

- Use fully qualified image references: `<registry>/<org>/<image>:<tag>`.
- Build and smoke-test for `linux/amd64`, then inspect the pushed manifest.
- Keep images small: use `.dockerignore`, multi-stage builds, slim runtime bases,
  and avoid copying raw datasets, build caches, notebooks, or unused toolchains.
- Split unrelated heavy workflows into separate images or steps instead of one
  catch-all image. Keep common base layers stable so repeated pulls are fast.
- Prefer immutable version tags or digests for production actions; avoid mutable
  `latest` when validating or sharing a workflow.

### Command Format

Commands use multi-line format with one parameter per line. This is a common
source of errors — each flag and argument goes on its own line.

**Correct:**
```
ls
-al
/cos/files
```

**Incorrect:**
```
ls -al /cos/files
```

The platform splits on newlines and passes each line as a separate argument to
the container entrypoint.

### Mount Points

Every container action automatically receives these mount points:

| Mount Path | Variable | Purpose | Permissions |
|---|---|---|---|
| `/cos/files` | `COS_FILE_VOLUME` | Record files (input) | Read-only or read-write (configurable) |
| `/cos/outputs` | `COS_OUTPUT_VOLUME` | Action outputs (auto-saved) | Read-write |
| `/cos/local_disk` | — | High-speed temporary storage | Read-write; cleared after execution |
| `/cos/codes` | `COS_CODE_VOLUME` | Code mount | Read-only |
| `/cos/bins` | `COS_BIN_VOLUME` | Binary files mount | Read-only |
| `/cos/bundles` | `COS_BUNDLE_VOLUME` | Batch test programs | Read-only |
| `/cos/artifacts` | `COS_ARTIFACT_VOLUME` | Batch test artifacts | Read-write |

**Output handling options:**

- Add output files to the existing record
- Replace record files with output files
- Both options auto-process `record.patch.json` in output directories

### Resource Options

| Tier | CPU | Memory | Notes |
|---|---|---|---|
| Small | 1 core | 2 GB | Default for lightweight processing |
| Medium | 2 cores | 4 GB | General-purpose |
| Large | 4 cores | 8 GB | Data-intensive processing |
| XL | 8 cores | 16 GB | Heavy computation; contact support for higher |

Custom resource configurations are available by contacting coScene support.

## HTTP Webhook (HTTP 请求)

Webhooks send HTTP requests to external endpoints when triggered. Common uses
include notification integrations (DingTalk, Slack, custom alerting).

### Configuration

| Field | Description | Example |
|---|---|---|
| Method | HTTP method | `POST`, `GET` |
| URL | Target endpoint | `https://hooks.slack.com/...` |
| Headers | Key-value pairs | `Content-Type: application/json` |
| Body | Request body (supports variable interpolation) | JSON payload with `{{task.title}}` |
| Timeout | Request timeout | 30 seconds (default) |

### Built-in Variables for Webhooks

These variables are automatically available in webhook URL, headers, and body:

| Variable | Description |
|---|---|
| `{{task.title}}` | Title of the associated task |
| `{{record.link}}` | Direct URL to the record in the web UI |
| `{{device.id}}` | Device identifier |
| `{{device.display_name}}` | Human-readable device name |
| `{{task.create_time}}` | Task creation timestamp |

## Runtime Environment Variables

The platform automatically injects these environment variables into every
container action execution:

### Volume Paths

| Variable | Default Value | Description |
|---|---|---|
| `COS_FILE_VOLUME` | `/cos/files` | Record files mount directory |
| `COS_OUTPUT_VOLUME` | `/cos/outputs` | Action output directory |
| `COS_CODE_VOLUME` | `/cos/codes` | Code mount directory |
| `COS_BIN_VOLUME` | `/cos/bins` | Binary files mount directory |
| `COS_BUNDLE_VOLUME` | `/cos/bundles` | Batch test program mount directory |
| `COS_ARTIFACT_VOLUME` | `/cos/artifacts` | Batch test artifact directory |

### Identity and Context

| Variable | Description | Notes |
|---|---|---|
| `COS_ORGID` | Organization ID (UUID) | Always set |
| `COS_USERID` | User ID (UUID) of the trigger initiator | Always set |
| `COS_WAREHOUSEID` | Warehouse ID | Always set |
| `COS_PROJECT` | Current project slug (human-readable) | Always set |
| `COS_PROJECTID` | Project ID (UUID) | Always set |
| `COS_RECORDID` | Record ID (UUID) | Set when action is record-scoped |

### Platform Access

| Variable | Description | Notes |
|---|---|---|
| `COS_ENDPOINT` | coScene API endpoint URL | `openapi.coscene.cn` or `openapi.coscene.io` |
| `COS_TOKEN` | Authentication token | See security note below |

### COS_TOKEN Security

`COS_TOKEN` is platform-injected, inherits full permissions of the trigger
initiator. Valid only for action lifetime. Never log to stdout/stderr (visible
in invocation history), never exfiltrate, never persist. Cross-project
operations require initiator access to target project — otherwise 403.

## Trigger Types

Five trigger types connect events to actions. Use the platform UI for trigger
configuration when the operation is not exposed by cocli or public OpenAPI.

### 1. File Upload (文件上传)

Fires when a file uploaded to a record matches configured conditions.

| Setting | Description | Example |
|---|---|---|
| Glob pattern | Filename pattern | `*.zip`, `*.mcap`, `data/*.bag` |
| Uploader filter | Who uploaded | Member, Device, or Any |
| Record label match | Required record labels | `["production", "error"]` |

Labels are evaluated at upload time. Changing labels on an existing record
does NOT retroactively fire this trigger.

### 2. Device Collection Status Change (设备采集状态变更)

Fires when a device completes data collection. Filter by status:
`manual_complete` or `rule_complete`.

### 3. Record Change (记录变更)

Fires when a record's label or custom field is modified (not file upload).
Label mutation and custom field mutation are separate match conditions.

### 4. Task Status Change (任务状态变更)

Fires when a task's status changes to a specified value (e.g., `"已处理"`,
`"已关闭"`).

### 5. Cron Schedule (定时触发)

Standard 5-field cron: `minute hour day_of_month month day_of_week`

| Expression | Schedule |
|---|---|
| `0 * * * *` | Hourly at :00 |
| `0 0 * * *` | Daily at midnight |
| `0 0 * * 0` | Weekly on Sunday |
| `0 3 1,15 * *` | 1st and 15th at 03:00 |
| `30 6 * * 5` | Every Friday at 06:30 |
| `*/15 * * * *` | Every 15 minutes |

Runs in UTC by default (configurable via environment settings).

## record.patch.json Convention

Actions create or update records by writing structured output to the
`$COS_OUTPUT_VOLUME/records/` directory. The platform automatically scans
this directory after action completion and processes each subdirectory.

### Directory Structure

```
$COS_OUTPUT_VOLUME/
└── records/
    ├── analysis-report/
    │   ├── summary.pdf
    │   ├── charts/
    │   │   ├── error-rate.png
    │   │   └── timeline.png
    │   └── .cos/
    │       └── record.patch.json
    └── extracted-frames/
        ├── frame-001.jpg
        ├── frame-002.jpg
        └── .cos/
            └── record.patch.json
```

Each subdirectory under `records/` becomes a record operation (create or
update). The `.cos/record.patch.json` file controls what happens.

### record.patch.json Schema

```json
{
  "projectSlug": "target-project",
  "id": "existing-record-uuid",
  "title": "Record Title",
  "description": "Human-readable description",
  "labels": ["label-1", "label-2"],
  "patch": [
    { "op": "replace", "path": "/title", "value": "New Title" },
    { "op": "add", "path": "/labels/-", "value": "new-label" },
    { "op": "remove", "path": "/labels/0" },
    { "op": "add", "path": "/files/path/to/file", "value": "../relative/path.jpg" }
  ]
}
```

### Operations

**Create:** Omit `id`, include `title` (required). All files in the subdirectory
are uploaded to the new record.

```json
{ "title": "Analysis Results", "labels": ["automated", "analysis"] }
```

**Update:** Include `id` (UUID), use `patch` array (RFC 6902 JSON Patch).

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "patch": [
    { "op": "replace", "path": "/title", "value": "Updated Title" },
    { "op": "add", "path": "/labels/-", "value": "processed" }
  ]
}
```

**File references:** relative paths from record subdirectory — e.g.,
`{ "op": "add", "path": "/files/report.pdf", "value": "../report.pdf" }`

**Cross-project:** specify `projectSlug` to target a different project. The
`COS_TOKEN` initiator must have write access to that project.

### Automatic Processing

After action completion, the platform scans `$COS_OUTPUT_VOLUME/records/` for
subdirectories, reads `.cos/record.patch.json` in each, creates/patches records,
uploads files, and logs results to invocation history.

## Action Chaining

Chain actions by having Action A write `record.patch.json` to create a record
with specific labels, then a file upload trigger matching those labels fires
Action B on the new record.

```
Action A → writes record.patch.json (label: "stage-1") → platform creates record
    → file upload trigger (glob: *.csv, label: "stage-1") → Action B fires
```

**Design notes:**
- Use labels to distinguish pipeline stages and route to the correct next action
- No built-in pipeline orchestrator — chaining is emergent from trigger-action
- Circular chains are possible — use labels or naming conventions to break cycles
- Each action gets its own `COS_TOKEN` (inheriting original initiator perms)

## Best Practices

- **Deduplication:** always configure event deduplication windows to prevent
  duplicate data collection from rapid-fire events
- **Glob patterns:** be precise — `*.zip` not `*.zip*`; test patterns before
  deploying triggers
- **Labels for routing:** use labels in `record.patch.json` to control which
  triggers fire on action output; do not rely on label changes to re-trigger
  file-upload workflows
- **Cross-project access:** verify `COS_TOKEN` initiator has target project
  permissions before deploying cross-project `record.patch.json` operations
- **Output structure:** always follow the `records/*/` convention in
  `$COS_OUTPUT_VOLUME` with `.cos/record.patch.json` — the platform ignores
  output files that do not follow this structure for record operations
- **Image registry:** for custom container actions, prefer the org registry
  returned by `cocli` over ad hoc public images; verify the full image reference
  and architecture before configuring the action
- **Resource sizing:** start with 2C/4G and scale up only if action execution
  hits OOM or timeout; over-provisioning wastes quota
- **Idempotency:** design actions to be re-runnable — use record `id` for
  updates to avoid creating duplicate records on retry
