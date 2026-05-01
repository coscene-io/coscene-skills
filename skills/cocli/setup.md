# Setup

Setup and environment bootstrap for cocli. The agent reads this file when preflight reports a non-OK status.

## Install

Detect the user's OS and architecture, then install cocli.

**Supported platforms:** macOS (amd64, arm64), Linux (amd64, arm64). No Windows native binary — use WSL.

| Method | When to use | Command |
|---|---|---|
| curl (recommended) | macOS or Linux, no Go toolchain | See below |
| go install | Go toolchain available | `go install github.com/coscene-io/cocli@latest` |

### curl install

Single command — downloads, decompresses, installs to `/usr/local/bin`:

```bash
curl -fsSL "https://download.coscene.cn/cocli/latest/$(uname -s | tr A-Z a-z)-$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/').gz" | gzip -d > cocli && chmod +x cocli && sudo mv cocli /usr/local/bin/
```

If `sudo` is unavailable or the user prefers a local install:

```bash
mkdir -p ~/.local/bin
curl -fsSL "https://download.coscene.cn/cocli/latest/$(uname -s | tr A-Z a-z)-$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/').gz" | gzip -d > ~/.local/bin/cocli && chmod +x ~/.local/bin/cocli
```

Ensure `~/.local/bin` is in `$PATH`.

**AskUserQuestion:** "Which install method do you prefer? (recommend curl unless you have Go set up)"

Post-install verify:

```bash
cocli --version
```

Expected output: `cocli version vX.Y.Z` (release) or `cocli version v0.0.0+<commit>.dirty` (dev build).

## Update

For outdated versions or manual upgrade:

```bash
cocli update
```

`cocli update` replaces the binary in-place via go-selfupdate. If cocli was installed to `/usr/local/bin` with sudo, the update also requires sudo:

```bash
sudo cocli update
```

To install a specific version instead:

```bash
curl -fsSL "https://download.coscene.cn/cocli/v1.2.3/$(uname -s | tr A-Z a-z)-$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/').gz" | gzip -d > cocli && chmod +x cocli && sudo mv cocli /usr/local/bin/
```

## Profile Bootstrap

First-time auth setup. Requires a coScene bearer token and project slug.

### Gather credentials

**AskUserQuestion:** "What is your coScene bearer token? (Find it in the platform UI: Settings > API Keys)"

**AskUserQuestion:** "What is your default project slug? (Format: org-slug/project-slug)"

**AskUserQuestion:** "What is your coScene endpoint? (Default: https://openapi.coscene.cn — most users keep the default)"

### Add profile

Execute with token hygiene — leading space prevents the token from entering shell history. Shell-detect ensures this works on both bash and zsh:

```bash
if [ -n "$ZSH_VERSION" ]; then setopt HIST_IGNORE_SPACE
elif [ -n "$BASH_VERSION" ]; then HISTCONTROL=ignorespace
else echo "Unknown shell — manually clear history after this command"; fi
 cocli login add -n default -t '<bearer-token>' -p '<project-slug>' -e '<endpoint>'
```

The leading space before `cocli` is intentional. With history-ignore-space enabled (bash: `HISTCONTROL=ignorespace`; zsh: `setopt HIST_IGNORE_SPACE`), commands starting with a space are excluded from shell history, keeping the bearer token out of `~/.bash_history` / `~/.zsh_history`.

**Process-table exposure:** The token is briefly visible in `/proc/<pid>/cmdline` while the command runs. For higher security, use the `COS_TOKEN` environment variable instead:

```bash
read -rs COS_TOKEN && export COS_TOKEN
cocli login add -n default -t "$COS_TOKEN" -p '<project-slug>' -e '<endpoint>'
unset COS_TOKEN
```

`read -rs` suppresses echo and avoids both history and process-table exposure. cocli reads `COS_TOKEN` from the environment (see `COS_TOKEN`, `COS_ENDPOINT`, `COS_PROJECT` in cocli source).

### Verify

```bash
cocli login current
cocli project list -o json
```

Both should succeed without errors. If `project list` fails with a permission error, the token may lack project access — check with the organization admin.

### Secure config file

```bash
chmod 0600 ~/.cocli.yaml
```

`~/.cocli.yaml` stores bearer tokens in plaintext. Restrict read access to the owner.

## Preferences

Agent preferences and persistent memory. Stored at `~/.coscene/AGENTS.md` — agents both read and write this file.

**AskUserQuestion:** "Preferred language? (zh / en / auto-detect) — recommend auto-detect"

**AskUserQuestion:** "Communication style? (concise / explainer / beginner) — recommend concise for developers, explainer for new users"

Write initial preferences:

```bash
mkdir -p ~/.coscene
cat > ~/.coscene/AGENTS.md << 'AGENTSMD'
# coScene AGENTS.md

## Language
<user-choice>

## Communication style
<user-choice>

## Memory
<!-- Agent-maintained notes: environment details, user patterns, workflow preferences -->

## Gotchas
<!-- Lessons learned: environment quirks, user corrections, platform edge cases -->

## Last updated
<YYYY-MM-DD>
AGENTSMD
```

Replace `<user-choice>` placeholders with the user's answers and `<YYYY-MM-DD>` with the current date.

If `mkdir` or write fails (permissions, disk full), warn the user and continue — preferences are optional. The CLI works without them.

### AGENTS.md format

Both the `cocli` and `coscene-docs` skills read this file at first interaction via `cat ~/.coscene/AGENTS.md 2>/dev/null`. The sections are parsed by heading:

| Section | Values | Default (if missing) |
|---|---|---|
| `## Language` | `zh`, `en`, `auto-detect` | `en` |
| `## Communication style` | `concise`, `explainer`, `beginner` | `concise` |
| `## Memory` | Free-form notes (agent-maintained) | empty |
| `## Gotchas` | Free-form notes (agent-maintained) | empty |
| `## Last updated` | ISO 8601 date | — |

### When to update AGENTS.md

Agents should append to `## Memory` or `## Gotchas` when:

- User corrects agent behavior or states a preference not captured in the file
- Agent discovers an environment detail that affects future interactions (OS, shell, cocli version quirks)
- A workaround is found for a platform edge case the user will likely hit again

**Update procedure:** read the file, append a bullet under the relevant section, update `## Last updated`. Do not rewrite existing entries — append only. Keep each entry to one line.

## Help

### Platform concepts and troubleshooting

Load the `coscene-docs` skill, then read `troubleshooting.md` for diagnosis guides covering auth failures, upload issues, action execution problems, and device connectivity.

### Access troubleshooting

If preflight returned `ACCESS_DENIED`:

1. Check that the token has not expired — regenerate in Platform UI > Settings > API Keys
2. Verify the project slug matches an accessible project — `cocli project list -o json`
3. Check organization membership with the admin

### Human support

For issues beyond what the agent or documentation can resolve: contact@coscene.io
