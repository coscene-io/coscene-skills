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
| `./actions.md` | Running actions, monitoring runs, or understanding automation runtime |

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
| `-f` / `--force` | `delete`, `copy`, `move`, `action run` | Skip confirmation prompts |
| `-y` / `--yes` | `project create` | Skip creation confirmation |
| `--skip-params` | `action run` (without `-P`) | Skip param prompts, use defaults. Mutex with `-P`. |
| `--no-tty` | `upload` commands | Disable TTY progress bars |

### Destructive or Bulk Operations

For delete, move, overwrite, and bulk metadata updates:

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
| Create moment in record | `cocli record moment create <record> -n "event" -D 10 -T <epoch-seconds>` | No |

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
| List available actions | `cocli action list -o json` | Yes |
| Run action on record | `cocli action run <action> <record> -P key=val -f` | No |
| List action runs | `cocli action list-run -o json` | Yes |
| List runs for a record | `cocli action list-run -r <record> -o json` | Yes |

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
| Add login profile | `cocli login add -n <name> -t <token> -p <slug>` — history-suppress per setup.md § Profile Bootstrap (zsh: `setopt HIST_IGNORE_SPACE`; bash: `HISTCONTROL=ignorespace`) | No |
| Switch profile (interactive — TUI, will hang in automation) | `cocli login switch` | No |
| Switch profile (non-interactive) | `cocli login set -n <name>` | No |
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

**Run processing on a record:**

1. `cocli action list -o json` → find action name
2. `cocli action run <action> <record> -P key=val -f`
3. **CRITICAL:** action run is ASYNC. Exit 0 = submitted, not completed.
4. Poll: `cocli action list-run -r <record> -o json` until status shows completion.

**Cross-project transfer:**

- Copy: `cocli record copy <record> -P <dst-project> -f`
- Move: `cocli record move <record> -P <dst-project> -f`
- File-level: `cocli record file copy <src> <dst> -P <dst-project> --files "a.bag" -f`

**Modify existing record:**

- Metadata: `cocli record update <record> -t "new title" -l key=val`
- Labels: `--update-labels` (replace), `--delete-labels` (remove), `-l` (append)
- Moments: `cocli record moment create <record> -n "event" -D 10 -T <epoch-seconds>`

**Environment check:**
Go to Preflight section above.

## Output Parsing

### Commands with `-o json`

Parse JSON directly. Common examples: `action list`, `action list-run`,
`project list`, `project create`, `project file list`, `record list`,
`record create`, `record describe`, `record file list`, `record moment list`,
`user list`, `user get`, `role list`, `registry create-credential`.

Common JSON shapes:

```json
// List commands → array of objects
[{"name": "records/abc-123", "title": "Drive 42", "labels": {"env": "prod"}, ...}]

// Describe/get → single object
{"name": "records/abc-123", "title": "Drive 42", "labels": {"env": "prod"}, ...}

// Action list-run → array with status field
[{"name": "actionRuns/xyz", "state": "SUCCEEDED", ...}]
```

Key fields to extract:
- `name` — resource identifier (pass to other commands)
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
| "I don't need `-o json`" | ALWAYS use `-o json` when available. Human table output is not parseable. |
| "I can skip `-f` on action run" | NEVER. Without `-f`, cocli prompts interactively and hangs. |
| "I'll hardcode the endpoint URL" | NEVER. The active profile handles endpoint selection. |
| "I'll use `record view` to see the record" | `record view` only prints a URL (or opens browser with `-w`). Use `record describe -o json`. |
| "I'll skip `--skip-params` on action run" | Use `--skip-params` for defaults (no `-P`), or `-P key=val` for explicit params. They are mutually exclusive — never combine. |
| "I'll use `--page` for pagination" | Use `--page-token` or `--all`. `--page` is deprecated for records. |
| "Action run returned 0, so the action completed" | action run is ASYNC. Exit 0 means submitted, not completed. Poll `action list-run`. |
| "I'll upload without `--no-tty`" | In non-interactive contexts, always pass `--no-tty` to prevent TTY prompts. |
| "I'll use `login switch` to change profile" | NEVER in automation. `login switch` uses an interactive TUI menu — hangs without TTY. Use `cocli login set -n <name>` instead. |
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
