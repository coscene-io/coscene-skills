# coScene Troubleshooting

Diagnostic tree format: symptom, then check, then resolution.

## Authentication and Authorization

### Token expired or invalid

**Symptom:** 401 Unauthorized on API calls or cocli commands.
**Check:** `cocli login current` — verify endpoint and token status.
**Fix:** Regenerate token in web UI (User Settings), then `cocli login set -t <new-token>`.

### Wrong endpoint (CN vs IO)

**Symptom:** Connection timeouts or 404s. Data exists on one endpoint but not the other.
**Check:** `cocli login current` — China: `openapi.coscene.cn`, International: `openapi.coscene.io`.
**Fix:** `cocli login switch` to correct profile or create new one for the right endpoint (interactive TUI — for agents, use `cocli login set -n <profile-name>` instead).

### Profile mismatch

**Symptom:** Data from wrong org/project, or insufficient permissions.
**Check:** `cocli login list -v` — review all profiles.
**Fix:** `cocli login switch` to the correct profile (interactive TUI — for agents, use `cocli login set -n <profile-name>` instead).

### 403 Forbidden in action execution

**Symptom:** Action container API calls return 403; cross-project operations fail.
**Check:** `COS_TOKEN` inherits trigger initiator permissions. Initiator must have target project access.
**Fix:** Trigger the action as a user with required role on all target projects.

## Upload Failures

### File too large

**Symptom:** Upload fails midway or times out on large files.
**Check:** Default part size is 128 MiB.
**Fix:** Increase `--part-size` in cocli. For 50GB+ files, split before upload.

### Network timeout

**Symptom:** Upload stalls or connection timeout.
**Check:** Connectivity to `openapi.coscene.cn` (CN) or `openapi.coscene.io` (IO).
**Fix:** Check firewall/proxy/DNS. Reduce `--parallel` count for unstable connections.

### Permission denied on upload

**Symptom:** 403 on upload.
**Check:** User needs at least Member role on target project.
**Fix:** Request role from project admin. Verify profile with `cocli login current`.

### Upload stalls without error

**Symptom:** No progress, no error message.
**Check:** High `--parallel` count on limited bandwidth.
**Fix:** Reduce `--parallel` to 1-2 for constrained networks.

### Duplicate files skipped

**Symptom:** Files not appearing in record, or upload completes unexpectedly fast.
**Cause:** Expected behavior — SHA256 dedup skips files already in the record.
**Fix:** No action needed. Rename or modify content to force re-upload.

## Action Execution Failures

### Action not found

**Symptom:** "Not found" error on action invocation.
**Check:** `cocli action list -o json` — verify action exists in target project.
**Fix:** Actions are project-scoped. Use `-p <project-slug>` for correct project.

### Parameters missing

**Symptom:** Action fails or produces wrong output — parameters not substituted.
**Check:** All `{{parameter.key}}` refs need corresponding `-P key=value` flags.
**Fix:** Supply missing `-P` flags, or `--skip-params` for defaults.

### Container fails to start

**Symptom:** Immediate failure with no execution logs.
**Check:** Image URL may be wrong, image may not exist, or registry inaccessible.
**Fix:** Prefer a fully qualified org-registry image. Use
`cocli registry create-credential -o json` or `cocli registry login` to verify
the registry for the active profile, ensure the image is uploaded, and check the
image architecture matches the action runtime, normally `linux/amd64`.

### Multi-line command format error

**Symptom:** Command fails with unexpected arguments (e.g., `ls -al` as one string).
**Check:** Each flag/argument must be on its own line in the action UI.
**Fix:** `python3 -m pytest -v` becomes 4 lines: `python3`, `-m`, `pytest`, `-v`.

### Output files not saved

**Symptom:** Action succeeds but no output files in any record.
**Check:** Output must follow `$COS_OUTPUT_VOLUME/records/<name>/.cos/record.patch.json`.
**Fix:** Place files in subdirectories under `/cos/outputs/records/` with valid
`record.patch.json`. Files outside this structure are not processed.

### Cross-project token failure

**Symptom:** `record.patch.json` with `projectSlug` targeting another project fails 403.
**Check:** `COS_TOKEN` inherits initiator permissions — needs target project access.
**Fix:** Trigger as user with Member/Admin role on target project.

## Device and coScout Issues

### Device shows offline

**Symptom:** Device offline in UI, no uploads.
**Check:** Inspect `~/.local/state/cos/logs/cos.log` on the device.
**Fix:** Common causes: network loss (verify endpoint reachability), coScout
crash (restart service), expired credentials (re-register device).

### Rules not pushed to device

**Symptom:** Rules not taking effect, no rule-triggered collection.
**Check:** Device must be enabled in org settings AND allocated to the project.
Rules push every 60 seconds — wait at least one cycle.
**Fix:** Enable device in org, allocate to correct project, check device logs
for rule fetch errors.

### Event deduplication too aggressive

**Symptom:** Multiple events merged into one, missing expected triggers.
**Check:** Dedup window setting — long windows (e.g., 86400s) merge everything.
**Fix:** Reduce dedup window. Use distinct event codes for logically different events.

### Files not detected by coScout

**Symptom:** Files exist on device but coScout ignores them.
**Check:** Verify `listen_dirs` and `collect_dirs` are absolute paths matching
actual file locations.
**Fix:** Fix directory paths, ensure coScout has read permissions, verify topic
config for ROS/MCAP streams.

## Record Query Issues

### No results returned

**Symptom:** `cocli record list` returns empty.
**Check:** Records are project-scoped — verify `-p` flag or active profile.
**Fix:** `cocli login current` to check default project; specify `-p <slug>`.

### Missing archived records

**Symptom:** Previously visible records gone from query results.
**Check:** Archived records excluded from default queries.
**Fix:** Use `--include-archive`. If moved, search in target project.

### Pagination incomplete

**Symptom:** Only first page of results returned.
**Fix:** Use `--all` for full results, or iterate with `--page-token`.

## Moment Issues

### Moment not created by rule

**Symptom:** Rule configured but no moments appear.
**Check:** CEL expression must match actual message format from device topic.
**Fix:** Test CEL against sample messages. Common issues: case-sensitive field
names, nested fields need full path (`msg.data.error_code`), strings need quotes.

### Time range incorrect

**Symptom:** Moment covers wrong time range.
**Check:** Timestamp format must match device output format.
**Fix:** Verify format compatibility. Check "before"/"after" duration settings on
the rule match desired capture window.

### Moment not visible in playback

**Symptom:** Moment exists in metadata but not in visualization.
**Fix:** Reload visualization page. Verify layout includes moment/annotation panel.
Check that moment time range falls within record data timeline.
