# Records Reference

Complete flag reference and patterns for record lifecycle, file operations, moments, pagination, filtering, and batch processing.

## Record Lifecycle

### record create — JSON: Yes

Create a new record in the active project.

**Flags:** `-p` (project), `-t` (title, default: "cocli created record"), `-d` (description), `-l` (labels `key=val`, repeatable), `--custom` (custom field JSON), `-i` (thumbnail), `-P` (parallel upload threads), `-s` (part-size), `--no-tty`, `--tty`, `-o json`

Pass `-t` explicitly for meaningful titles — agents should always provide one.

```bash
# Minimal — capture name from JSON
RECORD=$(cocli record create -t "Drive 42" -o json | jq -r '.name')

# With labels and description
cocli record create -t "Lidar Run 7" -d "Highway segment" \
  -l env=prod -l robot=lidar01 -o json
```

**Output shape:** `{"name": "records/abc-123", "title": "Drive 42", "labels": {...}, ...}`

### record upload — JSON: No

Upload files from a local directory into a record.

**Flags:** `-p` (project), `-H` (include hidden), `--dir` (remote target subdirectory, optional), `-P` (parallel threads, default 4), `-s` (part-size, default 128MiB), `--no-tty`, `--tty`

```bash
cocli record upload records/abc-123 ./data -P 8 -s 256 --no-tty
```

### record describe — JSON: Yes

Get full metadata for a single record.

**Flags:** `-p` (project), `-o json`

```bash
cocli record describe records/abc-123 -o json
```

**Output shape:** `{"name": "records/abc-123", "title": "...", "description": "...", "labels": {...}, "createTime": "...", "updateTime": "..."}`

### record update — JSON: No

Update metadata on an existing record.

**Flags:** `-p` (project), `-t` (title), `-d` (description), `-l` (append label `key=val`, repeatable), `--update-labels` (replace label `key=val`), `--delete-labels` (remove label by key), `--custom` (custom field JSON), `-i` (thumbnail), `-P` (parallel), `-s` (part-size), `--no-tty`, `--tty`

```bash
cocli record update records/abc-123 -t "Drive 42 — corrected"
cocli record update records/abc-123 -l validated=true          # append
cocli record update records/abc-123 --update-labels "env=prod" # replace
cocli record update records/abc-123 --delete-labels "env"      # delete
```

**Mutually exclusive:** pick exactly one of `-l/--labels`, `--update-labels`, or `--delete-labels` per invocation.

### record list — JSON: Yes

List records with filtering and pagination.

**Flags:** `-p` (project), `-v` (verbose), `-s` (JSON Logic search — mutex with `--labels`, `--keywords`, `--include-archive`), `--all` (fetch all — mutex with `--page-size`, `--page-token`), `--page-size` (10-100), `--page-token` (preferred), `--include-archive`, `--labels` (`key=val`), `--keywords` (text search), `-o json`

```bash
cocli record list -o json
cocli record list --labels "env=prod" --keywords "lidar" --all -o json
```

**Output shape:** `[{"name": "records/abc-123", "title": "...", "labels": {...}, ...}, ...]`

### record download — JSON: No

Download all files from a record.

**Flags:** `-p` (project), `-m` (include moments), `-r` (max retries, default 3), `--flat` (flatten dirs)

```bash
cocli record download records/abc-123 ./output
cocli record download records/abc-123 ./output -m -r 5 --flat
```

### record copy — JSON: No

Copy a record to another project. Source is preserved.

**Flags:** `-p` (source project), `-P` (destination project), `-f` (force, skip confirmation)

```bash
cocli record copy records/abc-123 -P target-project -f
```

### record move — JSON: No

Move a record to another project. Source is removed.
Ask the user for explicit confirmation before running this command.

**Flags:** `-p` (source project), `-P` (destination project), `-f` (force)

```bash
cocli record move records/abc-123 -P target-project -f
```

### record delete — JSON: No

Permanently delete a record.
Ask the user for explicit confirmation before running this command.

**Flags:** `-p` (project), `-f` (force, skip confirmation)

```bash
cocli record delete records/abc-123 -f
```

### record view — JSON: No

Print the web URL for a record.

**Flags:** `-p` (project), `-w` (open in browser — macOS only, do NOT use in automation)

```bash
cocli record view records/abc-123
# → The record url is: https://...
# For structured data, use: cocli record describe <record> -o json
```

---

## File Sub-commands

Operate on individual files within a record.

### record file list — JSON: Yes

**Flags:** `-p` (project), `-v` (verbose), `--all`, `--page-size`, `--page`, `--dir` (filter directory), `-R` (recursive), `-o json`

```bash
cocli record file list records/abc-123 -o json
cocli record file list records/abc-123 -R --dir "sensor/" -o json
```

### record file download — JSON: No

**Flags:** `-p` (project), `-r` (max retries, default 3), `-d` (filter by directory path within record), `--files` (comma-separated), `--flat`

```bash
cocli record file download records/abc-123 ./output --files "lidar.bag,camera.bag"
```

### record file delete — JSON: No

Ask the user for explicit confirmation before running this command.

**Flags:** `-p` (project), `-f` (force), `--files` (comma-separated)

```bash
cocli record file delete records/abc-123 --files "old_data.bag" -f
```

### record file copy — JSON: No

**Flags:** `-p` (project), `-P` (destination project for cross-project), `--files` (comma-separated), `-f` (force)

```bash
cocli record file copy records/src-123 records/dst-456 --files "data.bag" -f
cocli record file copy records/src-123 records/dst-456 -P other-project --files "data.bag" -f
```

### record file move — JSON: No

Ask the user for explicit confirmation before running this command.

**Flags:** `-p` (project), `-P` (destination project), `--files` (comma-separated), `-f` (force)

```bash
cocli record file move records/src-123 records/dst-456 --files "data.bag" -f
```

---

## Moment Sub-commands

Moments mark key time segments within a record — critical events, failures, or notable occurrences.

### record moment create — JSON: No

**Flags:** `-p` (project), `-n` (name, required), `-d` (description), `-D` (duration in seconds, required), `-T` (trigger-time in epoch seconds, required), `-j` (custom fields JSON), `-a` (assigner), `-e` (assignee), `-R` (rule), `-s` (skip-create-task), `-S` (sync-task)

```bash
cocli record moment create records/abc-123 \
  -n "Collision detected" \
  -D 10 \
  -T 1777386600 \
  -d "Front lidar detected obstacle at 2m" \
  -j '{"severity": "high", "sensor": "lidar_front"}'
```

**Tip:** Convert timestamps: `date -d '2026-04-28T14:30:00Z' +%s` (Linux) or `gdate -d '...' +%s` (macOS with coreutils).

### record moment list — JSON: Yes

**Flags:** `-p` (project), `-v` (verbose), `-o json`

```bash
cocli record moment list records/abc-123 -o json
# → [{"name": "moments/m-001", "displayName": "Collision detected", ...}]
```

### record moment download — JSON: No

**Flags:** `-p` (project), `--flat`

```bash
cocli record moment download records/abc-123 ./output
```

---

## Pagination Patterns

### Token-based (preferred for records)

Use `--page-size` + `--page-token`. First call omits `--page-token`; pass the token from previous response for next page.

```bash
# Iterate all pages
PAGE_TOKEN=""
while true; do
  if [ -z "$PAGE_TOKEN" ]; then
    RESULT=$(cocli record list --page-size 50 -o json)
  else
    RESULT=$(cocli record list --page-size 50 --page-token "$PAGE_TOKEN" -o json)
  fi
  echo "$RESULT" | jq -c '.[]'
  PAGE_TOKEN=$(echo "$RESULT" | jq -r '.nextPageToken // empty')
  [ -z "$PAGE_TOKEN" ] && break
done
```

### Fetch-all

```bash
cocli record list --all -o json
```

Convenient but slow on large datasets.

### Integer-based (deprecated for records)

`--page` still works for `project list`, `project file list`, and `record file list` (integer `--page` only, no `--page-token`). Prefer `--page-token` for records.

---

## Label-Based Filtering

```bash
# Single label
cocli record list --labels "env=prod" -o json

# Multiple labels (AND logic)
cocli record list --labels "env=prod" --labels "robot=lidar01" -o json

# Keyword search (titles + descriptions)
cocli record list --keywords "highway" -o json

# JSON Logic — advanced filtering
cocli record list -s '{"==": [{"var": "labels.env"}, "prod"]}' -o json

# Combine filters
cocli record list --labels "env=prod" --keywords "lidar" -o json

# Include archived
cocli record list --include-archive -o json
```

---

## Batch Patterns

### Loop over record list

```bash
RECORDS=$(cocli record list --all -o json | jq -r '.[].name')
for RECORD in $RECORDS; do
  cocli record describe "$RECORD" -o json
done
```

### Batch label update

For bulk metadata changes, show the target list and ask the user for explicit
confirmation before running the loop.

```bash
RECORDS=$(cocli record list --labels "env=staging" --all -o json | jq -r '.[].name')
for RECORD in $RECORDS; do
  cocli record update "$RECORD" --update-labels "env=prod" -l promoted=true
done
```

### Parallel download

```bash
RECORDS=$(cocli record list --labels "env=prod" --all -o json | jq -r '.[].name')
for RECORD in $RECORDS; do
  (
    DIR="./downloads/$(basename "$RECORD")"
    mkdir -p "$DIR"
    cocli record download "$RECORD" "$DIR" -r 5
  ) &
done
wait
```

### Extract fields with jq

```bash
# Titles and labels
cocli record list --all -o json | jq -c '.[] | {name, title, labels}'

# Count records per label value
cocli record list --all -o json | jq -r '.[].labels.env // "unlabeled"' | sort | uniq -c

# Filter in jq
cocli record list --all -o json | jq -r '.[] | select(.labels.env == "prod") | .name'
```

### Batch delete with dry run

```bash
STALE=$(cocli record list --labels "status=stale" --all -o json | jq -r '.[].name')
echo "Will delete $(echo "$STALE" | wc -l | tr -d ' ') records:"
echo "$STALE"

# Ask the user to confirm the exact list above, then uncomment to execute:
# for RECORD in $STALE; do cocli record delete "$RECORD" -f; done
```
