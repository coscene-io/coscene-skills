---
name: cocli
description: >
  Use when interacting with the coScene robotics data platform via the cocli CLI —
  uploading records, querying data, managing projects, running actions, or checking
  auth/profile status. Paired with the coscene-docs skill for concept questions
  and troubleshooting. Use public OpenAPI only when cocli cannot cover a supported
  public operation; otherwise direct the user to the coScene web UI.
---

# cocli

> CLI-first interface to the coScene robotics data platform. Upload, query, process, move.

## Bundled References

Read on-demand — not on every invocation.

| File | Read when... |
|---|---|
| `./setup.md` | Preflight reports non-OK status, or user needs install/profile/prefs setup |
| `./records.md` | Working with record lifecycle, file operations, or moments |
| `./workflows.md` | Building multi-step workflows or following templates |
| `./actions.md` | Creating, running, or monitoring actions, or understanding automation runtime |

## Sister Skill

**coscene-docs** — platform concepts, automation model, and troubleshooting.

Load `coscene-docs` when the user asks about what things mean, how the platform
works conceptually, or needs troubleshooting guidance. This skill (`cocli`) handles
CLI execution; `coscene-docs` handles understanding.

- "What does Organization / Project / Record / Moment (一刻) mean?" → load `coscene-docs`
- "How does the automation system work?" → load `coscene-docs`
- "Why is my upload failing?" → check Error Recovery below first, then load `coscene-docs` for deeper diagnosis

## Preflight

Run once before any cocli work. Single invocation, read stdout for status:

```bash
bash -c '
  command -v cocli >/dev/null 2>&1 || { echo "TOOL_MISSING"; exit 0; }
  V=$(cocli --version 2>&1 | head -1)
  cocli login current >/dev/null 2>&1 || { echo "PROFILE_MISSING $V"; exit 0; }
  if command -v timeout >/dev/null 2>&1; then
    timeout 10s cocli project list -o json >/dev/null 2>&1 || { echo "ACCESS_DENIED $V"; exit 0; }
  else
    cocli project list -o json >/dev/null 2>&1 || { echo "ACCESS_DENIED $V"; exit 0; }
  fi
  echo "OK $V"
'
```

**Route by status code:**

| Status | Action |
|---|---|
| `TOOL_MISSING` | Read `./setup.md` § Install |
| `PROFILE_MISSING` | Read `./setup.md` § Profile Bootstrap |
| `ACCESS_DENIED` | Token expired or invalid. Check Error Recovery table below. If unresolvable, read `./setup.md` § Profile Bootstrap to re-add. |
| `OK` | Proceed. Version string included for reference. |
| *(other)* | Unexpected output. Read `./setup.md` § Install, verify binary, re-run preflight. |

If version in the OK output looks outdated or is a dev build (`v0.0.0+...`), suggest `cocli update` or read `./setup.md` § Update.

Security note: `~/.cocli.yaml` stores bearer tokens in plaintext. Ensure `chmod 0600 ~/.cocli.yaml`.

### Endpoint and Profile Selection

Profiles carry the target coScene instance. If the user gives a platform URL,
match the active profile endpoint to that URL's instance before running data
commands:

| Platform URL host | Required profile endpoint |
|---|---|
| `coscene.cn` | `https://openapi.coscene.cn` |
| `volc.coscene.cn` | `https://openapi.volc.coscene.cn` |
| `coscene.io` | `https://openapi.coscene.io` |

Do not treat `cocli --version` as the active endpoint. It shows the binary's
default release endpoint, while `cocli login current` shows the profile endpoint
actually used by commands. If no existing profile matches the user's URL, read
`./setup.md` § Profile Bootstrap and create one with `cocli login add -e`.

### Per-invocation profile override (`--profile`)

To target a profile for a single command without changing the persisted
current-profile, use the global `--profile` flag (in-memory only, safe for
concurrent runs against different profiles):

```bash
cocli record list --profile staging
```

Precedence (`--profile` > complete `COS_*` env > config current-profile) and CI
env usage: see `./setup.md` § Profile Bootstrap.

## Agent Preferences

On first interaction, check for user preferences:

```bash
cat ~/.coscene/AGENTS.md 2>/dev/null
```

If the file exists, adapt:
- **Language**: respond in the user's preferred language (zh/en). If `auto-detect`, infer from user's message language.
- **Communication style**: `concise` = terse, code-heavy. `explainer` = add context and rationale. `beginner` = step-by-step with platform concept explanations.
- **Memory / Gotchas**: check for relevant notes before answering — past corrections, environment quirks, known workarounds.

If the file is missing, use defaults (English, concise) and continue without prompting. Do not block on missing preferences.

When the user corrects your behavior, states a new preference, or you discover an environment quirk — append a bullet to `## Memory` or `## Gotchas` in `~/.coscene/AGENTS.md` and update `## Last updated`. Read the file first, append only, do not rewrite existing entries.

### Non-Interactive Flags

cocli has several commands that prompt for interactive input. In agent/automation contexts, always pass these flags:

| Flag | Commands | Purpose |
|---|---|---|
| `-f` / `--force` | `delete`, `copy`, `move`, `action run`, `action delete`, `action cancel-run` | Skip confirmation prompts (`action delete` also disables triggers; `action cancel-run` is irreversible) |
| `-y` / `--yes` | `project create` | Skip creation confirmation |
| `--skip-params` | `action run` (without `-P`) | Skip param prompts, use defaults. Mutex with `-P`. |
| `--no-tty` | `upload` commands | Disable TTY progress bars |

### Destructive or Bulk Operations

For delete, move, overwrite, action-run cancellation, and bulk metadata updates:

1. Read back and show the exact target list first.
2. Explain what will change and whether it is reversible.
3. Ask for explicit user confirmation before running the mutating command.
4. Use `-f` only after confirmation, then verify with a readback.

## Command Lookup Table

Task-to-command routing. Not flag-complete — run `cocli <cmd> --help` for full options.

### Upload Data

| Task | Command | JSON? |
|---|---|---|
| Create empty record | `cocli record create -t "title"` | Yes |
| Create record with labels | `cocli record create -t "title" -l env=prod -l robot=lidar01` | Yes |
| Upload files to record | `cocli record upload <record> ./data` | No |
| Upload files to project | `cocli project file upload <project-slug> ./data` | No |
| Upload with tuned concurrency (`-P` default is 4) | `cocli record upload <record> ./data -P 8 -s 256 --no-tty` | No |

### Query Records

| Task | Command | JSON? |
|---|---|---|
| List records | `cocli record list -o json` | Yes |
| List all records (no pagination) | `cocli record list --all -o json` | Yes |
| Search records by keyword | `cocli record list --keywords "lidar" -o json` | Yes |
| Search records by JSON Logic (`-s` mutex with each of `--labels`, `--keywords`, `--include-archive` — three separate constraints) | `cocli record list -s '<json-logic>' -o json` | Yes |
| Filter by labels | `cocli record list --labels "env=prod" -o json` | Yes |
| Include archived records | `cocli record list --include-archive -o json` | Yes |
| Describe single record | `cocli record describe <record> -o json` | Yes |
| Get record URL | `cocli record view <record>` | No |
| List moments in record | `cocli record moment list <record> -o json` | Yes |
| Download moments.json | `cocli record moment download <record> <dst-dir>` | No |

### Modify Records

| Task | Command | JSON? |
|---|---|---|
| Update record title | `cocli record update <record> -t "new title"` | No |
| Add labels to record | `cocli record update <record> -l newlabel=val` | No |
| Replace labels on record | `cocli record update <record> --update-labels "k=v"` | No |
| Delete labels from record | `cocli record update <record> --delete-labels "oldkey"` | No |
| Delete a record | `cocli record delete <record> -f` | No |
| Create moment in record (creates a task by default; add `-s` to skip) | `cocli record moment create <record> -n "event" -D 10 -T <epoch-seconds>` | No |

### Manage Files

| Task | Command | JSON? |
|---|---|---|
| List record files | `cocli record file list <record> -o json` | Yes |
| List record files recursively | `cocli record file list <record> -R` | Yes |
| Download record files | `cocli record file download <record> <dst-dir> --files "a.bag,b.bag"` | No |
| Download all record data | `cocli record download <record> <dst-dir>` | No |
| Download moments | `cocli record download <record> <dst-dir> -m` | No |
| Delete record files | `cocli record file delete <record> --files "x.bag" -f` | No |
| Copy files between records | `cocli record file copy <src> <dst> --files "a.bag" -f` | No |
| Move files between records | `cocli record file move <src> <dst> --files "a.bag" -f` | No |
| List project files | `cocli project file list <project-slug> -o json` | Yes |
| Download project files | `cocli project file download <project-slug> ./output --files "config.yaml"` | No |
| Delete project files | `cocli project file delete <project-slug> --files "old.log" -f` | No |

### Run Actions

| Task | Command | JSON? |
|---|---|---|
| Create action (inline single container; job auto-named `main`) | `cocli action create -p <slug> --name <n> --image <img> --command "python run.py" --quota small -o json` (inline `--quota small\|medium\|large\|xlarge` preset) | Yes |
| Create action from spec file / stdin (file is the full spec; inline spec flags like `--name`/`--quota` cannot be combined with `-f`) | `cocli action create -p <slug> -f action.yaml` (`-f -` for stdin) | Yes |
| List available actions | `cocli action list -o json` | Yes |
| Get one action (`-o yaml/json` is `update -f`'s input format) | `cocli action get <action> -p <slug> -o yaml` | Yes |
| Update an action's spec from a file (full spec replace; get → edit → update loop) | `cocli action update <action> -p <slug> -f spec.yaml` (`-f -` for stdin; `--dry-run` to preview) | Yes |
| Delete an action (soft delete; also disables its triggers) | `cocli action delete <action> -p <slug> -f` | No |
| Run action on record | `cocli action run <action> <record> -P key=val -f` | No |
| **Run one action over many records (default for any multi-record ask)** | `cocli action run <action> -p <slug> -s '<json-logic>' -P key=val -f` (cocli ≥ v1.7.7; dry-run the query with `record list -s` first; mutex with `<record>`) | No |
| List action runs | `cocli action list-run -o json` | Yes |
| List runs for a record | `cocli action list-run -r <record> -o json` | Yes |
| Cancel an action run (async; confirm the exact target before `-f`) | `cocli action cancel-run <action-run> -p <slug> -f` | No |
| Print a run's logs (running → live pod logs, finished → archived log, same command) | `cocli action logs <action-run> -p <slug>` | No |
| Follow live logs, wait for the run to start | `cocli action logs <action-run> -p <slug> -f` | No |
| Logs for a specific job index (1-based, `-j 1` = first) / DAG node | `cocli action logs <action-run> -p <slug> -j 1 --node <node>` | No |

`<action-run>` is a full resource name (`projects/<project>/actionRuns/<uuid>`) or a bare UUID.
For `action logs`, `-f` means **`--follow`** (stream and reconnect on transient errors), **not**
`--force`. For `action cancel-run`, `-f` means **`--force`** and skips the irreversible-action
confirmation. Without `-f`, logs reports a not-yet-started run and exits; logs is a streaming text
command (no `-o json`) — exit `0` = success, `1` = error (check stderr).

### Registry and Action Images

| Task | Command | JSON? |
|---|---|---|
| Log in to the active organization's registry | `cocli registry login` | No |
| Generate Docker registry credentials | `cocli registry create-credential -o json` | Yes |

For container actions, prefer the organization registry that coScene provides for
each org. Use `cocli registry create-credential -o json` or `cocli registry login`
to discover and authenticate against the exact registry for the active profile;
do not guess registry hosts from memory. Use fully qualified image references
such as `cr.coscene.io/<org>/<image>:<tag>` or the registry host returned by
cocli.

### Cross-Project

| Task | Command | JSON? |
|---|---|---|
| Copy record to another project | `cocli record copy <record> -P <dst-project> -f` | No |
| Move record to another project | `cocli record move <record> -P <dst-project> -f` | No |
| Copy files cross-project | `cocli record file copy <src> <dst> -P <dst-project> -f` | No |

### Admin

| Task | Command | JSON? |
|---|---|---|
| Add login profile | `cocli login add -n <name> -t <token> -p <slug> -e <endpoint>` — choose endpoint from the platform URL; history-suppress per setup.md § Profile Bootstrap (zsh: `setopt HIST_IGNORE_SPACE`; bash: `HISTCONTROL=ignorespace`) | No |
| Switch profile (interactive — TUI, will hang in automation) | `cocli login switch` | No |
| Switch profile persistently (non-interactive) | `cocli login set -n <name>` | No |
| Use a profile for ONE command (no config change) | `cocli <cmd> --profile <name>` — global flag; overrides current-profile in memory only, safe for concurrent runs against different profiles | No |
| Delete profile | `cocli login delete <name>` | No |
| Create project | `cocli project create -p <slug> -n "name" -b private -y` | Yes |
| List users | `cocli user list -o json` | Yes |
| Get user details | `cocli user get <user> -o json` | Yes |
| List roles | `cocli role list -o json` | Yes |
| Registry login | `cocli registry login` | No |
| Generate docker credential | `cocli registry create-credential -o json` | Yes |
| Update cocli | `cocli update` | No |

### `-P` Flag — Triple Meaning

The `-P` flag changes meaning by context. Getting this wrong silently does the wrong thing.

| Context | `-P` means | Example |
|---|---|---|
| `record upload`, `project file upload`, `record create` | `--parallel` (thread count) | `-P 8` |
| `record copy`, `record move`, `record file copy/move` | `--dst-project` (target project) | `-P other-project` |
| `action run` | `--param` (key=value pair) | `-P model=yolo -P threshold=0.5` |

## Decision Router

Match user intent to the correct command sequence.

**Upload new data:**

1. `cocli record create -t "title" -l env=prod -o json` → capture record name from JSON
2. `cocli record upload <record> ./data -P 8 --no-tty`
3. Verify: `cocli record file list <record> -o json`

**Find existing data:**

1. `cocli record list --keywords "lidar" -o json` (or `--labels`, `-s` for JSON Logic)
2. Pick record → `cocli record describe <record> -o json`
3. Download: `cocli record download <record> <dst-dir>` (all) or `cocli record file download <record> <dst-dir> --files "specific.bag"`

**Run processing on many records (the common case — one run, not a loop):**

1. `cocli -v` → confirm ≥ v1.7.7 (`-s` on `action run`); older, tell the user to `cocli update`
2. Dry-run the selection: `cocli record list -p <slug> -s '<json-logic>' -o json`
3. Same query, one run: `cocli action run <action> -p <slug> -s '<json-logic>' -P key=val -f`
4. Poll `cocli action list-run -p <slug> -o json` → `progress.succeeded` counts records

Never loop `action run` per record for a set — see `./actions.md` § Run One Action
Over Many Records.

**Run processing on a record:**

1. `cocli action list -o json` → find action name
2. `cocli action run <action> <record> -P key=val -f`
3. **CRITICAL:** action run is ASYNC. Exit 0 = submitted, not completed.
4. Poll: `cocli action list-run -r <record> -o json` until status shows completion.
5. Inspect logs: `cocli action logs <action-run> -p <slug>` — add `-f` to follow a running
   run live (waits if it hasn't started). Once the run finishes, the archived log is printed
   automatically, so the same command works mid-run or after completion.

**Cancel a running action:**

1. Identify the exact run with `cocli action list-run -o json`.
2. Show the target to the user and get explicit confirmation.
3. Request cancellation with `cocli action cancel-run <action-run> -p <slug> -f`.
4. Poll `action list-run` until the state is `ABORTED`; cancellation is asynchronous.

If the run is already `SUCCEEDED`, `FAILED`, or `ABORTED`, cocli reports that it has finished
and sends no cancellation request.

**Cross-project transfer:**

- Copy: `cocli record copy <record> -P <dst-project> -f`
- Move: `cocli record move <record> -P <dst-project> -f`
- File-level: `cocli record file copy <src> <dst> -P <dst-project> --files "a.bag" -f`

**Modify existing record:**

- Metadata: `cocli record update <record> -t "new title" -l key=val`
- Labels: `--update-labels` (replace), `--delete-labels` (remove), `-l` (append)
- Moments: `cocli record moment create <record> -n "event" -D 10 -T <epoch-seconds>` creates a task by default; add `-s` / `--skip-create-task` for moment-only annotations.

**Environment check:**
Go to Preflight section above.

## Output Parsing

### Commands with `-o json`

Parse JSON directly. Common examples: `action list`, `action list-run`,
`action get`, `action update`, `project list`, `project create`, `project file list`, `record list`,
`record create`, `record describe`, `record file list`, `record moment list`,
`user list`, `user get`, `role list`, `registry create-credential`.

Common JSON shapes:

```json
// List commands → array of objects
[{"name": "projects/abc-123/records/def-456", "title": "Drive 42", "labels": {"env": "prod"}, ...}]

// Describe/get → single object
{"name": "projects/abc-123/records/def-456", "title": "Drive 42", "labels": {"env": "prod"}, ...}

// Action list-run → array with status field
[{"name": "actionRuns/xyz", "state": "SUCCEEDED", ...}]
```

Key fields to extract:
- `name` — full resource name (`projects/{project-uuid}/records/{record-uuid}`). Use this value as-is when passing to other cocli commands. When the full resource name is used, `-p` is optional (the project is already encoded in the path).
- `title` — human-readable label
- `labels` — key-value metadata map
- `state` — action run status (`RUNNING`, `SUCCEEDED`, `FAILED`)

### Commands without `-o json`

Check exit code + capture stderr. stdout is human text — do not parse structurally.

```bash
# Pattern for non-JSON commands
if cocli record upload <record> ./data --no-tty 2>err.log; then
  echo "success"
else
  cat err.log  # diagnostic info in stderr
fi
```

Exit codes: `0` = success, `1` = error (check stderr for details).

## Anti-Rationalization Table

These are hard rules. Do not reason around them.

| Thought | Response |
|---|---|
| "I'll use an internal or undocumented API" | NEVER in public skills. Use `cocli` first, then public OpenAPI only for supported public operations. If neither exposes the capability, direct the user to the coScene web UI. |
| "The default profile is probably right" | ALWAYS run `cocli login current` first. Profiles carry org + project context. |
| "The URL is `volc.coscene.cn`, but the default endpoint is probably fine" | NEVER. Match the profile endpoint to the platform URL host, for example `volc.coscene.cn` → `https://openapi.volc.coscene.cn`. |
| "I don't need `-o json`" | ALWAYS use `-o json` when available. Human table output is not parseable. |
| "I can skip `-f` on action run" | NEVER. Without `-f`, cocli prompts interactively and hangs. |
| "`-f` always means `--force`" | NOT for `action logs`, where `-f` is `--follow` (stream live). It is `--force` on `action run`, `delete`, `copy`, `move`. |
| "I'll hardcode the endpoint URL for data commands" | NEVER. The active profile handles endpoint selection. Only set an endpoint when creating/updating a profile, then verify with `cocli login current`. |
| "I'll use `record view` to see the record" | `record view` only prints a URL (or opens browser with `-w`). Use `record describe -o json`. |
| "I'll skip `--skip-params` on action run" | Use `--skip-params` for defaults (no `-P`), or `-P key=val` for explicit params. They are mutually exclusive — never combine. |
| "I'll use `--page` for pagination" | Use `--page-token` or `--all`. `--page` is deprecated for records. |
| "Action run returned 0, so the action completed" | action run is ASYNC. Exit 0 means submitted, not completed. Poll `action list-run`. |
| "`cancel-run` returned 0, so the run is already aborted" | Cancellation is ASYNC. Poll `action list-run` until `ABORTED`; an already-finished run also exits 0 without sending a cancellation request. |
| "I'll upload without `--no-tty`" | In non-interactive contexts, always pass `--no-tty` to prevent TTY prompts. |
| "I'll use `login switch` to change profile" | NEVER in automation. `login switch` uses an interactive TUI menu — hangs without TTY. Use `cocli login set -n <name>` to switch persistently, or `--profile <name>` on a single command (no config mutation, safe for concurrent runs). |
| "I'll guess the registry host" | NEVER. Use `cocli registry create-credential -o json` or `cocli registry login` to discover/authenticate the active org registry. |

## Cross-Skill Referral

| Question | Action |
|---|---|
| "What does Organization / Project / Record / Moment (一刻) mean?" | Load `coscene-docs` skill |
| "How does the automation / action system work conceptually?" | Load `coscene-docs` skill |

## Error Recovery

| Symptom | Cause | Fix |
|---|---|---|
| Exit 1 + "token" in stderr | Auth expired or invalid | `cocli login current`, re-add token if expired |
| Exit 1 + "not found" | Wrong resource name/ID | Verify with `record list` or `action list` |
| Exit 1 + "permission" | User lacks access | Check role with `user get`, request access |
| Upload stalls or is slow | Default parallelism | Increase `-P` (parallel) and tune `-s` (part-size) |
| Action run fails | Bad params or wrong action name | Run `action list -o json` first; check `-P key=val` pairs |
| Action run returns 0 but nothing happened | action run is ASYNC | Poll `action list-run -r <record> -o json` for actual status |
| "interactive input" error | Missing `-f`, `-P`, or `--skip-params` | Add `-f` and either `-P key=val` or `--skip-params` to action run; `--no-tty` to uploads |
| Record search returns empty | Wrong project context | Verify project with `cocli login current`; try `--all` flag |
| Exit 1 + "labels not found" | `--labels` value absent in project (typo/casing, or wrong project) | Fix the label spelling; list valid labels. The query does not fall back to unfiltered results |
