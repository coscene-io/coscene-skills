# Actions Reference

Actions are the automation building blocks of coScene. They execute data processing operations (cleaning, preprocessing, training, testing) through container-based code execution or HTTP webhook requests.

**Key constraint:** cocli is used to discover, run, and monitor actions. For
authoring or configuration, use public OpenAPI only if that capability is
documented and available; otherwise use the coScene web interface. Do not use
internal or undocumented APIs in public skills.

---

## Container Image Guidance

For custom container actions, prefer the image registry coScene provides for the
organization. It gives better pull locality and performance than ad hoc public
images, and keeps production action images under the org's control.

```bash
# Discover/authenticate the active org registry
cocli registry create-credential -o json
cocli registry login
```

Use the registry returned by cocli, or a fully qualified org image such as:

```text
cr.coscene.io/<org>/<image>:<tag>
```

Image checklist before configuring an action:

- Build for `linux/amd64` and run a small local smoke test.
- Keep the runtime image small with `.dockerignore`, multi-stage builds, and a
  slim runtime base.
- Do not copy raw datasets, build caches, notebooks, or unused toolchains into
  the runtime image.
- Split unrelated heavy workflows into separate images or steps instead of one
  catch-all image.
- Prefer immutable tags or digests for validated actions; avoid `latest` for
  repeatable workflows.

---

## Create an Action

`cocli action create` registers a new action in a project. Two authoring modes:

- **Inline flags** — minimal single-container action straight from the command
  line (`--name`, `--image`, `--command`, ...). Good for quick agent-spawned
  actions.
- **Spec file** (`-f`) — a full YAML/JSON `ActionSpec` for multi-job, HTTP,
  parameterized, or storage/mount/quota-configured actions. Use `--example` to
  get a skeleton, edit it, then `-f` it back in.

Flags override file fields when both are set.

### Flag Reference

| Flag | Long | Description |
|---|---|---|
| `-p` | `--project` | Project slug (falls back to the profile default) |
| `-f` | `--file` | Action spec YAML/JSON file (`-` for stdin) |
| | `--name` | Action name (inline mode) |
| | `--description` | Action description (inline mode) |
| | `--image` | Container image (inline mode) |
| | `--command` | Container command line — shell-split, quote-aware (inline mode; see below) |
| | `--env` | Container env `key=value` (repeatable; inline mode) |
| `-P` | `--param` | Action parameter default `key=value` (repeatable) |
| | `--quota` | Resource profile: `small` \| `medium` \| `large` \| `xlarge` |
| | `--dry-run` | Validate and print the lowered + server-defaulted spec; makes no API call |
| | `--example` | Print a skeleton action spec (no create) |
| `-o` | `--output` | Output format (`table` \| `json` \| `yaml`) |

There is **no** `--args`, `--job-name`, `--label`, `--cpu-quota`, or
`--memory-quota`. The CLI built-in single container job defaults to name
**`main`**. A YAML `-f` spec still accepts full `command`/`args` arrays and
`labels` directly — only the CLI convenience flags for those were removed.

### `--command` is a single shell-split flag

`--command` takes one full command line and splits it quote-aware into the
job's command tokens. Quotes are preserved; all tokens map to the container
`command` (k8s-style "just give me a command line") and `args` is left empty.

```bash
--command 'python train.py --epochs 10'   # → ["python","train.py","--epochs","10"]
--command 'sh -c "echo hi"'               # → ["sh","-c","echo hi"]
```

### Examples

```bash
# (a) Minimal inline create — job auto-named "main"
cocli action create -p my-proj --name train-yolo \
  --image cr.coscene.io/myorg/train:latest \
  --command "python train.py --epochs 50" \
  --env COS_KEY=secret \
  -P dataset=images \
  --quota large \
  -o json

# (b) Spec file — from disk, and from stdin (agent-generated spec)
cocli action create -p my-proj -f action.yaml
echo "$SPEC" | cocli action create -p my-proj -f -

# (c) Example skeleton + dry-run validate workflow (no API calls)
cocli action create --example > action.yaml      # skeleton, edit REPLACE-ME/image:tag
cocli action create -p my-proj -f action.yaml --dry-run   # validate the lowered spec
```

The `--example` skeleton ships a `REPLACE-ME/image:tag` sentinel; if an
unedited sentinel survives into a real create, cocli warns to stderr (non-fatal
— server-side validation is authoritative, but it almost always means a junk
action).

`--quota` profiles map to fixed CPU/memory tiers — see
[Resource Options](#resource-options) below. `-f` flags override file fields,
so you can keep a canonical spec file and tweak per-run.

### CRITICAL: RESOURCE_EXHAUSTED means do-not-retry

If `action create` fails with `RESOURCE_EXHAUSTED` / `NO_SUBSCRIPTION`, cocli
prints *"likely a missing permission grant, not a quota limit — do not retry"*
and exits non-zero. This is a **permanent config error** (missing subscription
or permission grant), **not** a transient quota limit — retrying will never
succeed. Surface the error to the user and stop; fix it in the coScene web UI
(grants/subscriptions), not by retrying the create.

---

## Discover Actions

List all actions available in the current project.

```bash
cocli action list -o json
```

**Output shape:**

```json
[
  {
    "name": "actions/abc-123",
    "displayName": "Decompress File",
    "description": "Decompress uploaded archives",
    "parameters": [
      {"key": "input_format", "defaultValue": "zip"}
    ]
  },
  {
    "name": "actions/def-456",
    "displayName": "YOLO Inference",
    "description": "Run object detection on camera frames",
    "parameters": [
      {"key": "model", "defaultValue": "yolov8n"},
      {"key": "threshold", "defaultValue": "0.5"}
    ]
  }
]
```

**Key fields:**
- `name` — action identifier, pass to `action run`
- `displayName` — human-readable label
- `parameters` — array of parameter definitions with keys and default values

**Flags:**

| Flag | Long | Description |
|---|---|---|
| `-p` | `--project` | Project slug |
| `-v` | `--verbose` | Verbose output |
| `-o` | `--output` | Output format (`json`) |

---

## Run an Action

Submit an action for execution against a record.

```bash
cocli action run <action-name> <record-name> -P key=val -f
```

### CRITICAL: action run is ASYNC

`action run` exits immediately on submission. **Exit 0 means the run was submitted, NOT that it completed.** There is no wait/poll built into the command. You must use `action list-run` to monitor status.

### Full Flag Reference

| Flag | Long | Description |
|---|---|---|
| `-p` | `--project` | Project slug |
| `-P` | `--param` | Parameter key=value pair (repeatable) |
| | `--skip-params` | Skip interactive prompts for missing parameters |
| `-f` | `--force` | Skip confirmation prompt |

### Non-Interactive Pattern (required for automation)

Two valid patterns — `-P` and `--skip-params` are **mutually exclusive**:

```bash
# Pattern 1: Explicit params (overrides defaults)
cocli action run actions/yolo-inference records/abc-123 \
  -P model=yolov8n \
  -P threshold=0.5 \
  -f

# Pattern 2: All defaults (skip param prompts)
cocli action run actions/decompress records/abc-123 -f --skip-params

# WRONG — will hang waiting for input
cocli action run actions/yolo-inference records/abc-123

# WRONG — -P and --skip-params are mutually exclusive
cocli action run actions/yolo-inference records/abc-123 -P model=yolov8n -f --skip-params
```

Without `-f`: the command prompts for confirmation.
Without `-P` or `--skip-params`: the command prompts for each parameter interactively.

### Multiple Parameters

Use multiple `-P` flags, one per parameter:

```bash
cocli action run actions/my-action records/abc-123 \
  -P model=yolo \
  -P threshold=0.5 \
  -P output_format=json \
  -f
```

### Skipping All Parameters

If the action has parameters but you want all defaults, pass `--skip-params` alone:

```bash
cocli action run actions/decompress records/abc-123 -f --skip-params
```

---

## Monitor Runs

List action execution runs, optionally filtered by record.

```bash
# All runs in the project
cocli action list-run -o json

# Runs for a specific record
cocli action list-run -r records/abc-123 -o json
```

**Flags:**

| Flag | Long | Description |
|---|---|---|
| `-p` | `--project` | Project slug |
| `-r` | `--record` | Filter by record name |
| `-v` | `--verbose` | Verbose output |
| `-o` | `--output` | Output format (`json`) |

**Output shape:**

```json
[
  {
    "name": "actionRuns/run-789",
    "action": "actions/yolo-inference",
    "record": "records/abc-123",
    "state": "SUCCEEDED",
    "createTime": "2026-04-28T15:00:00Z",
    "endTime": "2026-04-28T15:05:00Z"
  }
]
```

**Key fields:**
- `state` — `PENDING`, `RUNNING`, `SUCCEEDED`, `FAILED`
- `name` — run identifier
- `action` — which action was executed
- `record` — which record it operated on

### Polling Pattern

Poll `action list-run` in a loop until the run reaches a terminal state:

```bash
ACTION="actions/yolo-inference"
RECORD="records/abc-123"

# Submit
cocli action run "$ACTION" "$RECORD" -P model=yolov8n -f

# Poll
while true; do
  STATE=$(cocli action list-run -r "$RECORD" -o json | jq -r '.[0].state')
  echo "$(date +%H:%M:%S) — $STATE"
  case "$STATE" in
    SUCCEEDED)
      echo "Run completed successfully."
      break
      ;;
    FAILED)
      echo "Run failed. Check logs in coScene web UI."
      exit 1
      ;;
    *)
      sleep 15
      ;;
  esac
done
```

**Notes on polling:**
- `.[0]` assumes most-recent run is first. If multiple actions target the same record, filter by action name: `jq '.[] | select(.action == "actions/yolo-inference") | .state'`
- Poll interval of 10-15 seconds is reasonable. Actions typically take minutes to hours.
- There is no webhook or callback mechanism for run completion via CLI. Polling is the only option.

---

## Action Runtime Environment

When an action executes, the platform injects environment variables and mounts record data automatically.

### Injected Environment Variables

| Variable | Value | Description |
|---|---|---|
| `COS_TOKEN` | Bearer token | API auth — inherits trigger initiator's permissions |
| `COS_ENDPOINT` | API URL | coScene API endpoint |
| `COS_PROJECT` | Project slug | Current project |
| `COS_PROJECTID` | UUID | Project ID |
| `COS_RECORDID` | UUID | Record ID the action operates on |
| `COS_ORGID` | UUID | Organization ID |
| `COS_USERID` | UUID | User who triggered the action |
| `COS_WAREHOUSEID` | ID | Warehouse identifier |

### Mount Points

| Variable | Path | Description |
|---|---|---|
| `COS_FILE_VOLUME` | `/cos/files` | Record files (input — read or read-write) |
| `COS_OUTPUT_VOLUME` | `/cos/outputs` | Action output (auto-saved to record) |
| `COS_CODE_VOLUME` | `/cos/codes` | Code mount |
| `COS_BIN_VOLUME` | `/cos/bins` | Binary files |
| `COS_BUNDLE_VOLUME` | `/cos/bundles` | Batch test programs |
| `COS_ARTIFACT_VOLUME` | `/cos/artifacts` | Batch test artifacts |

### Resource Options

Actions can be configured with these CPU/memory profiles:

| Profile | CPU | Memory |
|---|---|---|
| Small | 1 core | 2 GB |
| Medium | 2 cores | 4 GB |
| Large | 4 cores | 8 GB |
| XLarge | 8 cores | 16 GB |

Contact support for custom resource profiles above 8C/16G.

### COS_TOKEN Permission Model

`COS_TOKEN` inherits the full permissions of the user or trigger that initiated the action. This has critical implications:

- If a human user triggers the action via UI, `COS_TOKEN` carries that user's permissions.
- If a trigger fires the action, the token inherits the trigger creator's permissions.
- For cross-project API calls within an action, the initiating user must have access to BOTH projects.
- If permissions are revoked for the initiating user after the run starts, in-flight API calls may fail.

---

## Trigger Types

Triggers are configured in the coScene web UI, not via CLI. Listed here for context when debugging action runs.

| Trigger Type | Fires When | Configuration |
|---|---|---|
| File Upload | File uploaded matching glob pattern | Glob pattern, uploader filter (member/device), record label match |
| Device Collection Status | Manual or rule collection completes | Status filter (manual_complete, rule_complete) |
| Record Change | Label or custom field modified on a record | Label/field change conditions |
| Task Status Change | Task status field changes | Status value filter |
| Cron Schedule | Time-based schedule | 5-field cron syntax (minute hour dom month dow) |

**Key behavior:** Label changes alone do not retroactively trigger file-upload triggers. To re-trigger processing after a label change, upload a new file to the record.

---

## record.patch.json Convention

Actions can create or update records by writing structured output to `COS_OUTPUT_VOLUME`.

### Directory Structure

```
/cos/outputs/
└── records/
    ├── new-record-name/
    │   ├── output-file.jpg
    │   └── .cos/
    │       └── record.patch.json
    └── another-record/
        ├── result.csv
        └── .cos/
            └── record.patch.json
```

### record.patch.json Schema

```json
{
  "projectSlug": "target-project",
  "id": "uuid-for-update",
  "title": "Record Title",
  "description": "Optional description",
  "labels": ["label-1", "label-2"],
  "patch": [
    {"op": "replace", "path": "/title", "value": "New Title"},
    {"op": "add", "path": "/labels/-", "value": "new-label"},
    {"op": "remove", "path": "/labels/0"},
    {"op": "add", "path": "/files/path/to/file", "value": "../relative/path.jpg"}
  ]
}
```

**Operations:**
- **Create a new record:** Omit `id`, include `title`. Files in the same directory are uploaded to the new record.
- **Update an existing record:** Include `id`, use `patch` array with RFC 6902 JSON Patch operations.
- **Cross-project output:** Set `projectSlug` to direct the record to a different project.
- **File references:** Use relative paths from the record directory to files in `/cos/outputs`.

The platform automatically scans `/cos/outputs/records/` after action completion and processes each `record.patch.json` found.
