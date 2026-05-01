# Workflows Reference

Multi-step workflow templates using cocli commands. Each template is copy-pasteable with concrete commands, expected output notes, and error handling.

All workflows assume an active cocli profile. Run `cocli login current` first.

---

## 1. Upload Sensor Data

Create a record, upload files, add labels, verify.

```bash
# Step 1: Create record, capture name
RECORD=$(cocli record create -t "Lidar Run 2026-04-28" \
  -d "Highway segment, vehicle A" \
  -l env=prod -l robot=lidar01 \
  -o json | jq -r '.name')
echo "Created: $RECORD"
# Expected: records/<uuid>

# Step 2: Upload files
cocli record upload "$RECORD" ./sensor_data -P 8 --no-tty
# Exit 0 = success. On failure, check stderr for network/auth errors.

# Step 3: Add additional labels after upload
cocli record update "$RECORD" -l uploaded=true -l batch=2026-04-28

# Step 4: Verify upload
cocli record file list "$RECORD" -R -o json | jq 'length'
# Expected: file count matching local directory
```

**What can go wrong:**
- Step 1: `exit 1` + "token" → profile token expired. Re-add with `cocli login add`.
- Step 2: Upload stalls → increase `-P` parallelism or decrease `-s` part size. Check network.
- Step 2: "not found" → record name wrong. Verify Step 1 captured the name correctly.

---

## 2. Find and Download

Search for records, inspect metadata, download specific files.

```bash
# Step 1: Search by keyword
cocli record list --keywords "highway" -o json | jq -c '.[] | {name, title}'
# Expected: array of matching records

# Step 2: Narrow by labels if needed
cocli record list --keywords "highway" --labels "env=prod" -o json | jq -c '.[] | {name, title}'

# Step 3: Describe the target record
RECORD="records/abc-123"  # from step 1/2
cocli record describe "$RECORD" -o json | jq '.'
# Inspect labels, file count, timestamps

# Step 4a: Download specific files
cocli record file download "$RECORD" ./output --files "lidar.bag,camera.bag"

# Step 4b: Or download everything
cocli record download "$RECORD" ./output
```

**What can go wrong:**
- Step 1: Empty result → wrong project context. Check `cocli login current`. Try `--all` or `--include-archive`.
- Step 3: "not found" → record name/ID is wrong. Re-list and verify.
- Step 4: Download fails → add `-r 5` for retries.

---

## 3. Cross-Project Transfer

Copy a record from one project to another and verify.

```bash
# Step 1: Identify the record in the source project
cocli record describe records/abc-123 -o json | jq '{name, title, labels}'

# Step 2: Copy to destination project
cocli record copy records/abc-123 -P target-project -f
# Exit 0 = copy initiated. No JSON output.

# Step 3: Switch to target project and verify
cocli record list -p target-project --keywords "$(cocli record describe records/abc-123 -o json | jq -r '.title')" -o json
# Expected: record appears in target project list

# Step 4: Verify file count matches
cocli record file list -p target-project records/abc-123 -R -o json | jq 'length'
```

**What can go wrong:**
- Step 2: "permission" error → current profile user lacks access to target project.
- Step 2: Without `-f`, command hangs waiting for confirmation.
- Step 3: Record name may differ in target project. Search by title or labels instead.

**Alternative — move (destructive):**

Stop and ask the user for explicit confirmation before running this command.

```bash
cocli record move records/abc-123 -P target-project -f
# Source record is REMOVED after move.
```

---

## 4. Batch Label Update

Add or replace labels on all records matching a filter.

```bash
# Step 1: List records that need updating
RECORDS=$(cocli record list --labels "env=staging" --all -o json | jq -r '.[].name')
echo "Found $(echo "$RECORDS" | wc -l | tr -d ' ') records"

# Step 2: Dry run — show what will change
for RECORD in $RECORDS; do
  TITLE=$(cocli record describe "$RECORD" -o json | jq -r '.title')
  echo "Will update: $RECORD ($TITLE)"
done

# Step 3: Apply label changes
for RECORD in $RECORDS; do
  cocli record update "$RECORD" --update-labels "env=prod" -l promoted=true
  echo "Updated: $RECORD"
done

# Step 4: Verify
cocli record list --labels "env=prod" --labels "promoted=true" --all -o json | jq 'length'
```

**What can go wrong:**
- Step 1: `--all` may be slow on projects with thousands of records. Use `--page-size` + `--page-token` loop instead.
- Step 3: `-l` appends a label; `--update-labels` replaces. Use `--delete-labels` to remove old keys.

---

## 5. Action Pipeline

Upload data, run an action, poll for completion, check result.

```bash
# Step 1: Create record and upload
RECORD=$(cocli record create -t "Processing input" -o json | jq -r '.name')
cocli record upload "$RECORD" ./input_data -P 8 --no-tty

# Step 2: Discover available actions
cocli action list -o json | jq -c '.[] | {name, displayName}'
# Pick the action name from output

# Step 3: Run the action (ASYNC — exits immediately on submission)
ACTION="actions/my-processor"
cocli action run "$ACTION" "$RECORD" -P model=yolo -P threshold=0.5 -f
# Exit 0 = SUBMITTED, NOT COMPLETED.

# Step 4: Poll for completion
while true; do
  STATUS=$(cocli action list-run -r "$RECORD" -o json | jq -r '.[0].state')
  echo "Status: $STATUS"
  case "$STATUS" in
    SUCCEEDED) echo "Action completed successfully."; break ;;
    FAILED)    echo "Action failed. Check logs in UI."; exit 1 ;;
    *)         sleep 10 ;;  # RUNNING, PENDING, etc.
  esac
done

# Step 5: Check outputs (action may have created new files or records)
cocli record file list "$RECORD" -R -o json | jq '.'

# Optional: download only representative outputs instead of the whole record
cocli record file download "$RECORD" ./outputs --files "output.mcap,report.json"
```

**What can go wrong:**
- Step 3: Without `-f` and either `-P` or `--skip-params`, command hangs waiting for interactive input. Note: `-P` and `--skip-params` are mutually exclusive.
- Step 3: Exit 0 does NOT mean the action succeeded — it means it was submitted. Always poll.
- Step 4: `.[0]` assumes the most recent run is first. Filter by action name if multiple actions target the same record.
- Step 4: If status stays `RUNNING` indefinitely, check action logs in the web UI.

---

## 6. Moment Annotation

Describe a record, then create moment annotations at specific timestamps.

```bash
# Step 1: Get record metadata
cocli record describe records/abc-123 -o json | jq '{name, title, createTime}'

# Step 2: List existing moments
cocli record moment list records/abc-123 -o json | jq '.'

# Step 3: Create a moment with timestamp and attributes
cocli record moment create records/abc-123 \
  -n "Collision event" \
  -D 10 \
  -T 1777386600 \
  -d "Front lidar detected obstacle at 2m range" \
  -j '{"severity": "critical", "sensor": "lidar_front", "distance_m": 2.1}'

# Step 4: Create a second moment
cocli record moment create records/abc-123 \
  -n "Lane departure" \
  -D 5 \
  -T 1777386735 \
  -d "Vehicle crossed lane boundary" \
  -j '{"severity": "warning", "camera": "front_wide"}'

# Step 5: Verify moments
cocli record moment list records/abc-123 -o json | jq 'length'
# Expected: previous count + 2
```

**What can go wrong:**
- Step 3: Invalid trigger time → `-T` takes epoch seconds (float), not RFC 3339. Convert: `date -d '2026-04-28T14:30:00Z' +%s` (Linux) or `gdate` (macOS).
- Step 3: `-j` JSON parse error → validate JSON string before passing.
- The moment `create` command has no JSON output; verify with `moment list`.

---

## 7. Environment Audit

Verify the full access chain: login profile → available projects → records.

```bash
# Step 1: Check current profile
cocli login current
# Shows active profile name, endpoint, project

# Step 2: List all profiles
cocli login list -v
# Shows all configured profiles

# Step 3: List accessible projects
cocli project list -o json | jq -c '.[] | {name, displayName}'

# Step 4: Check record access in a specific project
cocli record list -p my-project --page-size 5 -o json | jq 'length'
# Expected: > 0 if project has records and user has access

# Step 5: Check user permissions
cocli user get "$(cocli user list -o json | jq -r '.[0].name')" -o json | jq '.'

# Step 6: List available roles
cocli role list --level project -o json | jq -c '.[] | {name, title}'
```

**What can go wrong:**
- Step 1: No active profile → `cocli login add` to create one.
- Step 3: Empty project list → user not a member of any projects in this org.
- Step 4: "permission" error → user role lacks read access on that project.

---

## 8. Bulk Download

Download all records matching a filter to local directories.

```bash
# Step 1: List matching records
RECORDS=$(cocli record list --labels "env=prod" --keywords "highway" --all -o json | jq -r '.[].name')
COUNT=$(echo "$RECORDS" | wc -l | tr -d ' ')
echo "Found $COUNT records to download"

# Step 2: Download each to a separate directory
for RECORD in $RECORDS; do
  BASENAME=$(basename "$RECORD")
  DIR="./bulk_download/$BASENAME"
  mkdir -p "$DIR"
  echo "Downloading $RECORD → $DIR"
  cocli record download "$RECORD" "$DIR" -r 5 || echo "FAILED: $RECORD"
done

echo "Bulk download complete."

# Step 3: Verify
ls -la ./bulk_download/
```

**What can go wrong:**
- Step 1: `--all` on a large project can be slow. Use `--page-size` + `--page-token` loop for incremental fetches.
- Step 2: Disk space exhaustion — check `df -h` before bulk downloads.
- Step 2: Individual record download failures are caught by `|| echo FAILED` — re-run for failed records.

---

## 9. Data Cleanup

Find and delete stale or archived records.

```bash
# Step 1: List archived records
STALE=$(cocli record list --include-archive --labels "status=stale" --all -o json | jq -r '.[].name')
COUNT=$(echo "$STALE" | wc -l | tr -d ' ')
echo "Found $COUNT stale records"

# Step 2: Dry run — inspect before deleting
for RECORD in $STALE; do
  cocli record describe "$RECORD" -o json | jq '{name, title, labels, updateTime}'
done

# Step 3: Ask the user to confirm the exact list above, then delete (irreversible)
for RECORD in $STALE; do
  cocli record delete "$RECORD" -f
  echo "Deleted: $RECORD"
done

# Step 4: Verify
cocli record list --labels "status=stale" --all -o json | jq 'length'
# Expected: 0
```

**What can go wrong:**
- Step 3: Without `-f`, each delete prompts for confirmation — blocks in automation.
- Step 3: Deletion is permanent. There is no undo. The dry run in Step 2 is critical.
- Step 1: If `--labels` doesn't find stale records, check if the label key/value is exact.

---

## 10. Registry Setup

Log in to the organization's container registry and generate Docker credentials.
Use this before building or pushing custom container images for actions.

```bash
# Step 1: Log in to registry (uses active cocli profile)
cocli registry login
# Authenticates Docker client with org registry.

# Step 2: Generate a docker credential for external use
CRED=$(cocli registry create-credential -o json)
# → {"username": "...", "password": "...", "registry": "..."}

# Step 3: Use credential in Docker
REGISTRY=$(echo "$CRED" | jq -r '.registry')
USERNAME=$(echo "$CRED" | jq -r '.username')
PASSWORD=$(echo "$CRED" | jq -r '.password')

echo "$PASSWORD" | docker login "$REGISTRY" -u "$USERNAME" --password-stdin

# Step 4: Build and push an action image
IMAGE="$REGISTRY/<org>/<image>:<tag>"
docker build --platform linux/amd64 -t "$IMAGE" .
docker run --rm --platform linux/amd64 "$IMAGE" <smoke-test-command>
docker push "$IMAGE"

# Step 5: Verify
docker manifest inspect "$IMAGE" >/dev/null
```

**What can go wrong:**
- Step 1: "token" error → profile token expired. Re-authenticate with `cocli login add`.
- Step 2: Credential output is sensitive. Do not log or persist in plaintext.
- Step 3: Docker must be installed and running on the host.
- Step 4: If building on Apple Silicon or another non-amd64 host, keep
  `--platform linux/amd64` unless the action runtime explicitly supports another
  architecture.
